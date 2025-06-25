# Introduction
This repository describes how to run wordpress in docker usind different methods.
 - [Run Wordpress with docker command](#run-wordpress-with-docker-command)
 - [Wordpress Dev Environment](#wordpress-dev-environment)
 - [Wordpress Nginx Letsencrypt Docker Compose Version](#wordpress-nginx-letsencrypt-docker-compose-version)
 - [Bitnami Wordpress Traefik Letsencrypt Version](#bitnami-wordpress-traefik-letsencrypt-version)


---


# Run Wordpress with docker command
Based on the official Wordpress docker image:
 - [Official wordpress docker image](https://hub.docker.com/_/wordpress)

This option is only viable if you already have a MySQL/MariaDB container and want to use the same one, otherwise the option to [run wordpress as a docker container](#wordpress-dev-environment) is better.

The docker image can be run using this command:
```bash
docker run -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=<password> --name wordpress --link wordpressdb:mysql -p 80:80 -v "$PWD/html":/var/www/html -d wordpress
```
Other parameters include:
 - -e WORDPRESS_DB_HOST=...
 - -e WORDPRESS_DB_USER=...
 - -e WORDPRESS_DB_PASSWORD=...
 - -e WORDPRESS_DB_NAME=...
 - -e WORDPRESS_TABLE_PREFIX=...
 - -e WORDPRESS_AUTH_KEY=..., -e WORDPRESS_SECURE_AUTH_KEY=..., -e WORDPRESS_LOGGED_IN_KEY=..., -e WORDPRESS_NONCE_KEY=..., -e WORDPRESS_AUTH_SALT=..., -e WORDPRESS_SECURE_AUTH_SALT=..., -e WORDPRESS_LOGGED_IN_SALT=..., -e WORDPRESS_NONCE_SALT=... (default to unique random SHA1s, but only if other environment variable configuration is provided)
 - -e WORDPRESS_DEBUG=1 (defaults to disabled, non-empty value will enable WP_DEBUG in wp-config.php)
 - -e WORDPRESS_CONFIG_EXTRA=... (defaults to nothing, the value will be evaluated by the eval() function in wp-config.php. This variable is especially useful for applying extra configuration values this image does not provide by default such as WP_ALLOW_MULTISITE; see docker-library/wordpress#142⁠ for more details)

The `WORDPRESS_DB_NAME` has to be a MySQL/Mariadb database that needs to already exists since the container will not create a new database.

This is a full command including volumes, db settings, passwords, etc:
```bash
docker run -d --name wordpress --restart always -p 8080:80 \
    -v $PWD/html:/var/www/html \
    -v $PWD/dump.sql:/docker-entrypoint-initdb.d/dump.sql \
    -e WORDPRESS_DB_NAME=wordpress \
    -e WORDPRESS_TABLE_PREFIX=wp_ \
    -e WORDPRESS_DB_HOST=<wordpressdb_docker_ip> \
    -e WORDPRESS_DB_USER=root \
    -e WORDPRESS_DB_PASSWORD=password \
    -e WORDPRESS_DEBUG=1 \
    wordpress:latest
```
 /!\ It is important that the `wordpress` database has already been created on the database.
 ```sql
 CREATE DATABASE wordpress COLLATE utf8mb4_general_ci;
 ```

The line for `dump.sql` can be ommitted if no initial db is required to be imported.

To link the wordpress container to the db you need to use the ip defined in docker for the db. To identify the ip you can run this command:
```bash
docker inspect mariadb | grep IPAddress
```
Alternatively to link the wordpress and db container together we can use the default docker network `bridge` or create a new network and use the `--link` parameter when running `docker run`.

To view a list of networks run
```bash
docker network ls
```

To create a new network run:
```bash
docker network create <network_name>
```

To test that communication is working you can run this command:
```bash
docker exec -it container2 ping container1
```

## Run MariaDB as a container
To run MariaDB as a container run the following docker command:
```bash
docker run -e MARIADB_ROOT_PASSWORD=<password> -e MARIADB_DATABASE=wordpress --name wordpressdb -v "$PWD/database":/var/lib/mysql -d mariadb:latest
```


---


# [Wordpress Dev Environment](./wordpress-dev/)
Based on this article:
 - [Medium blog](https://medium.com/@richardevcom/wordpress-development-environment-with-docker-ba52427bdd65)

## Explanation
[compose.yml](./wordpress-dev/compose.yml) is a standard configuration file to create a WordPress development environment. However, there are  a couple of adjustments to tailor it to this specific case:
 - volumes: - `./wp-content:/var/www/html/wp-content`: this line syncs the local [wp-content/](./wordpress-dev/wp-content/) folder with the container's `wp-content` directory, allowing to make changes locally and see them reflected in the container.
 - dump.sql: `./dump.sql:/docker-entrypoint-initdb.d/dump.sql` will automatically import your database dump file after MySQL server is installed.

Before running the environment, we need to adjustment the permissions of the `wp-content` directory. Docker has specific rules about file permissions, and the WordPress image is set up in a particular way. To ensure we can easily edit the WordPress files from outside the container, we need to change the ownership of the `wp-content` directory:
```bash
sudo chown -R {your-username}:{your-username} wp-content
```

To run the container run:
```bash
docker-compose up -d
```

The installation is accessible from [localhost:8080](http://localhost:8080)
---


# [Wordpress Nginx Letsencrypt Docker Compose Version](./custom-wordpress-nginx)
Based on this article:
 - [DigitalOcean blog](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose)

_/!\ Double check the blog for important steps re the SSL certificate_

To run the container run:
```bash
docker-compose up -d
```
There should be 4 services

To check the logs of a specific service run
```bash
docker-compose logs _service_name_
```
You can check that the certificate has been downloaded by running:
```bash
docker-compose exec webserver ls -la /etc/letsencrypt/live
```

Once the certificate has been created, you can use the second nginx configuration and restart the services.


---


# [Bitnami Wordpress Traefik Letsencrypt Version](./bitnami-wordpress-traefik-letsencrypt/)
Based on this article:
 - [Docker.com blog](https://www.docker.com/blog/how-to-dockerize-wordpress/)

In this version we are using the [Bitnami version of Wordpress](https://hub.docker.com/r/bitnami/wordpress).
This version is production ready and already optimized with all security patches and latest updates.
The version of wordpress we are using is specified in the .env file under the `WORDPRESS_IMAGE_TAG` variable.

[Traefik](https://doc.traefik.io/traefik/) is a modern reverse proxy and load balancer designed for microservices. It integrates seamlessly with Docker and can automatically obtain and renew TLS certificates from [Let’s Encrypt](https://letsencrypt.org/)

However to use this version a domain name is required.

Because we’ve mapped the wordpress-data volume, any changes you make within the WordPress container (like installing plugins or themes) will persist across container restarts.

## How to deploy
Before deploying the docker compose we need to create docker networks:
```bash
docker network create traefik-network
docker network create wordpress-network
```

After the network has been created, we can run the docker compose:
```bash
docker compose -f wordpress-traefik-letsencrypt-compose.yml -p website up -d
```

Running `docker ps` should show three services:
    - MariaDB
    - Wordpress
    - Traefik

## Accessing the sites
To access the website you can go to these addresses:
    - https://wordpress.yourdomain.com
    - https://traefik.yourdomain.com

Passwords for these sites have been set in the .env file.

## Backup site
To backup the site run:
```bash
docker cp ./wordpress-backup/. wordpress_wordpress_1:/bitnami/wordpress/
```
To backup the database run:
```bash
mysqldump -u your_db_user -p your_db_name > wordpress_db_backup.sql
```

## Import site
First copy the db in the current mariadb:
```bash
docker cp wordpress_db_backup.sql wordpress_mariadb_1:/wordpress_db_backup.sql
```
Access the mariadb container:
```bash
docker exec -it wordpress_mariadb_1 bash
```
And import the database
```bash
mysql -u root -p${WORDPRESS_DB_ADMIN_PASSWORD} ${WORDPRESS_DB_NAME} < wordpress_db_backup.sql
```


---
