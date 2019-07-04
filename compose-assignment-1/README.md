# Assignment: Writing a Compose File
drupal - a content management server which is basically just a website builder, a fully functional webapp needs a database running behind it

> Goal: Create a compose config for a local Drupal CMS website

- This empty directory is where you should create a docker-compose.yml 
- Use the `drupal` image along with the `postgres` image
- Set the version to 2
- Use `ports` to expose Drupal on 8080 so you can localhost:80
- Be sure to setup POSTGRES_PASSWORD on postgres image
- Walk though Drupal config in browser at http://localhost:8080
- Tip: Drupal assumes DB is "localhost", change it to the database server dns name ( container service name) . drupul will talk to database server over the docker network.
- Use Docker Hub documentation to figure out the right environment and volume settings
- Extra Credit: Use volumes to store Drupal unique data/app data

Dockerhub documentation:
https://hub.docker.com/_/drupal/

PostgreSQL

$ docker run --name some-drupal --network some-network -d drupal

    Database type: PostgreSQL
    Database name/username/password: <details for> (POSTGRES_USER, POSTGRES_PASSWORD; see environment variables in the description for postgres)
    ADVANCED OPTIONS; Database host: some-postgres (for using the DNS entry added by --network to access the PostgreSQL container)


volumes:
$ docker run --name some-drupal --network some-network -d \
    -v /path/on/host/modules:/var/www/html/modules \
    -v /path/on/host/profiles:/var/www/html/profiles \
    -v /path/on/host/sites:/var/www/html/sites \
    -v /path/on/host/themes:/var/www/html/themes \
    drupal
