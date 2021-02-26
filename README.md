# RaspberryPi-Dockerized-Nextcloud-PostgreSQL-Redis-Nginx-SSL
A dockerized preconfigured Nextcloud FPM on Raspberry Pi, using PostgreSQL, Redis, Nginx and also SSL certificated with Certbot. 

It uses PostgreSQL instead of MariaDB or MySQL, personally I prefer PostgreSQL over them. 
You can significantly improve your Nextcloud server performance with memory caching provided by Redis, where frequently-requested objects are stored in memory for faster retrieval. It also uses Nginx wich actues like a proxy server 

With this compose file you will deploy a fast and easy installation. All images are `arm` friendly, so fits perfect with a `RaspberryPi` or another `arm device`.

## Prerequisites
* Docker (obviously)
* Docker Compose **version 1.24 minimum**

## Configuration
Before build it, you want to configure your installation with your own data and domain.
* build.sh

Modify lanes 3 and 4 with your `domain` and `email` (valid address is strongly recommended)
```
domain=(mydomain.com)
email="email@mydomain.com"
```
**Optional:** set `staging` to 1 if you're testing your setup to avoid hitting request limits while validating your SSL certificate, the generated certificate will not be valid. You might want to do this at the first execution, to check that the compose built right. When it's built you will have to execute this script once again but this time with `staging` to 0.

Let’s Encrypt provides rate limits to ensure fair usage by as many people as possible. 

More info: https://letsencrypt.org/docs/rate-limits/

By default `staging` is set to 0 (lane 7).

```
staging=0
```
* docker-compose.yml

Change these PostgreSQL environment values to your desired `POSTGRES_USER` and `POSTGRES_PASSWORD`. 

You will have to change it on `db` service (lanes 12 and 13) and `app` service (lanes 34 and 35).
```
- POSTGRES_USER=mypostgresuser
- POSTGRES_PASSWORD=mypostgrespassword
```
Also for Nextcloud environment values `NEXTCLOUD_ADMIN_USER`, `NEXTCLOUD_ADMIN_PASSWORD`, and `NEXTCLOUD_TRUSTED_DOMAINS` (lanes 38, 39 and 40).
```
- NEXTCLOUD_ADMIN_USER=mynextcloudadmin
- NEXTCLOUD_ADMIN_PASSWORD=mynextcloudadminpassword
- NEXTCLOUD_TRUSTED_DOMAINS=mytrustedomain.com mytrusteddomain2.com 192.168.XXX.XXX #separated by spaces
```
* data/nginx/nginx.conf

Change `server_name` on port 80 (lane 44) and port 443 (lane 59) with your own domain.
```
server_name mydomain.com;
```
Change `ssl_certificate` and `ssl_certificate_key` location modifying `MYDOMAIN.COM`.
```
ssl_certificate /etc/letsencrypt/live/MYDOMAIN.COM/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/MYDOMAIN.com/privkey.pem;
```

## Building it
Execute `./build.sh` and that's it. 

Remember, by default `staging` is set to 0, so it will generate a valid SSL certificate on the first (and it should be one time only) run. **If it fails, please don't execute it again before setting `staging` to 1, or you would got a Let’s Encrypt rate limit.** To solve this, check what happened with your domain, if your server has ports 80 and 443 open, if you modified data/nginx/nginx.conf, ... 

If you execute `./build.sh` nothing would happen except an attempt to generate a valid SSL certificate, it only does a **docker-compose up -d** at the end, so don't be afraid of execeuting it again. You will not have data loss.

## Automatic Certificate Renewal
The `certbot` image does this automatically. 

On the `certbot` section of `docker-compose.yml` we can see this:
```
entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```
This will check if your certificate is up for renewal every 12 hours as recommended by Let’s Encrypt.

The `nginx` section also reloads the newly obtained certificates:
```
command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```
This makes nginx reload its configuration (and certificates) every six hours in the background and launches nginx in the foreground.

## Updating Nextcloud
Updating the Nextcloud container is done by pulling the new image, throwing away the old container and starting the new one.
```
docker-compose pull && docker-compose up -d
```

**It is only possible to upgrade one major version at a time. For example, if you want to upgrade from version 18 to 20, you will have to upgrade from version 18 to 19, then from 20**

Nextcloud has an own volume so you will not have data loss.

## Fix missing indices
When you install Nextcloud or update it it's a common problem that it has missing indices. To solve this, execute this:
```
docker exec --user www-data nextcloud-app php occ db:add-missing-indices
```
Sends command `php occ db:add-missing-indices` to the Nextcloud docker container as the user `www-data`, assuming your `docker-compose.yml` has nextcloud-fpm under the header `app` and named by `nextcloud-app`. You can also do this with its container id.
