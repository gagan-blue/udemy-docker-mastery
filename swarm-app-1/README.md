# Assignment: Create A Multi-Service Multi-Node Web App
Docker's distributed voting ap
Different teams are writing different parts of a solution and they are able to just pick the language they think is best for that scenario. This kind of application really shines in the world of containers.
Because our applications are all segmented, they can still technically run on the same machine but they're all isolated and protected from each other so that you don't end up python/ jvm version conflicts and dependancy conflicts.

-voting-app (python) web front end for end user
-key-value store (Redis)
-worker (.NET) - talks to both redis and postgres
-db (postgres) on backed network so that frontend user doesnt have direct access
-result-app (Node.js) web back end for admins

1 volume, 2 networks, 5 services needed

## Goal: create networks, volumes, and services for a web-based "cats vs. dogs" voting app.
Here is a basic diagram of how the 5 services will work:

![diagram](./architecture.png)
- All images are on Docker Hub, so you should use editor to craft your commands locally, then paste them into swarm shell (at least that's how I'd do it)
- a `backend` and `frontend` overlay network are needed. Nothing different about them other then that backend will help protect database from the voting web app. (similar to how a VLAN setup might be in traditional architecture)
- The database server should use a named volume for preserving data. Use the new `--mount` format to do this: `--mount type=volume,source=db-data,target=/var/lib/postgresql/data`

### Infrastructure

Create a 3 node swarm cluster (swarm init)
docker-machine create node1  192.168.99.107
docker-machine create node2  192.168.99.108
docker-machine create node3  192.168.99.109

work from any swarm manager:

docker-machine ssh node1
docker swarm init --advertise-addr 192.168.99.107
docker swarm join-token worker
docker swarm join worker on node2 and node3

docker node ls
node1 - leader
node2
node3

docker network create -d overlay backend (subnet:10.0.0.0/24)
docker network create -d overlay frontend (subnet:10.0.1.0/24)



### Services (names below should be service names)

- vote (python) web front end pushes to redis
    - dockersamples/examplevotingapp_vote:before
    - web front end for users to vote dog/cat
    - ideally published on TCP 80. Container listens on 80
    - on frontend network
    - 2+ replicas of this container

Test and inspect standalone container:

docker image pull dockersamples/examplevotingapp_vote:before
docker image inspect ..
working dir /app
exposed ports 80

docker  inspect --format='{{ .Config.Cmd}}' dockersamples/examplevotingapp_vote:before
[gunicorn app:app -b 0.0.0.0:80 --log-file - --access-logfile - --workers 4 --keep-alive 0]

docker container run -d -p 80:80 dockersamples/examplevotingapp_vote:before

from node1 host: curl http://localhost:80 (since host port80 forwarded to cotnainer port80)

from localhost: curl http://192.168.99.107:80 or brower
(network 192.168.99.0 directly reachable from localhost through interface vboxnet2)

docker  exec -it awesome_diffie sh

/app/
    
    app.py
    app.pyc
    requirements.txt
    static
    templates

cat /app/Dockerfile

FROM python:2.7-alpine
WORKDIR /app
ADD requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt
ADD . /app
EXPOSE 80
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]


docker service create \
--name vote \
-p 80:80 \
--network frontend \
--replicas 2 \
dockersamples/examplevotingapp_vote:before

docker service ps vote
node1
node2


- redis
    - redis:3.2
    - key/value storage for incoming votes
    - no public ports (that means inside the container we can access or on overlay front end network)
    - on frontend network
    - 1 replica NOTE VIDEO SAYS TWO BUT ONLY ONE NEEDED

docker service create \
--name redis \
--network frontend \
--replicas 1 \
redis:3.2

docker service ps redis
node3

node3>docker exec -it redis.1.j3psttw5uam4xd8rrnegowben bash
redis-cli -h localhost -p 
redis container>redis-cli -h localhost -p 6379
localhost:6379> ping
PONG

Lets ping from vote container on node1 which is on the same frontend network

node1> docker container exec -it vote.2.abcfds sh
vote container>ping redis
    PING redis (10.0.1.7): 56 data bytes
    64 bytes from 10.0.1.7:

vote container> apk add --no-cache redis
vote container>redis-cli -h localhost -p 6379
vote container>redis-cli -h redis -p 6379
redis:6379> ping
PONG


- worker
    - dockersamples/examplevotingapp_worker
    - backend processor of redis and storing results in postgres
    - no public ports
    - on frontend and backend networks
    - 1 replica

docker image pull dockersamples/examplevotingapp_worker
docker  inspect --format='{{ .Config.Cmd}}' dockersamples/examplevotingapp_worker

/bin/sh -c dotnet src/Worker/Worker.dll

#create worker service after creating postgres service because the workerp app wont start if doesnt find postgress db.

Dockerfile:

FROM microsoft/dotnet:2.0.0-sdk
WORKDIR /code
ADD src/Worker /code/src/Worker
RUN dotnet restore -v minimal src/Worker \
    && dotnet publish -c Release -o "./" "src/Worker/" 
CMD dotnet src/Worker/Worker.dll

docker service create \
--name worker \
--network backend \
--network frontend \
--replicas 1 \
dockersamples/examplevotingapp_worker

- db
    - postgres:9.4
    - one named volume needed, pointing to /var/lib/postgresql/data
    - no public ports
    - on backend network, so worker can reach but front end cannot
    - 1 replica

docker service create \
--name db \
--network backend \
--mount type=volume,source=db-data,target=/var/lib/postgresql/ \
--replicas 1 \
postgres:9.4

docker service logs db
"No password has been set for the database."




- result
    - dockersamples/examplevotingapp_result:before
    - web app that shows results
    - runs on high port since just for admins (lets imagine)
    - so run on a high port of your choosing (I choose 5001), container port 80
    - on backend network
    - 1 replica

docker image pull dockersamples/examplevotingapp_result:before
docker image inspect dockersamples/examplevotingapp_result:before

working dir /app
port 80
cmd node server.js

Dockerfile:

FROM node:8.9-alpine
RUN mkdir -p /app
WORKDIR /app
RUN npm install -g nodemon
RUN npm config set registry https://registry.npmjs.org
COPY package.json /app/package.json
RUN npm install \
 && npm ls \
 && npm cache clean --force \
 && mv /app/node_modules /node_modules
COPY . /app
ENV PORT 80
EXPOSE 80
CMD ["node", "server.js"]

docker service create \
--name result \
--network backend \
-p 5001:80 \
dockersamples/examplevotingapp_result:before

result service (nodejs app) reachable from all swarm nodes 
http://192.168.99.107:5001/
http://192.168.99.108:5001/
http://192.168.99.109:5001/



docker service ps result
docker service ps redis
docker service ps db
docker service ps vote
docker service ps worker



deploy a complete application stack to the swarm
docker-compose.yml
docker stack deploy

https://github.com/dockersamples/example-voting-app/blob/master/docker-stack.yml

version: "3"
services:

  redis:
    image: redis:alpine
    networks:
      - frontend
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]
  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - 5000:80
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes:
db-data: