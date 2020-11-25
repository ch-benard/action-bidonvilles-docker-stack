# Créer les outils de déploiement

> Ces outils sont regroupés dans un dépôt spécifiques intitulé "action-bidonvilles-deploy".

## docker-compose.yml 

* Le fichier *docker-compose.yml* permet l'exécution de la commande *docker-compose*, qui va lancer la construction des conteneurs correspondant à chaque micro-services.

```yaml
version: '3.4'
services:
    rb-api:
        build:
            context: ../action-bidonvilles-api
            target: prod
        container_name: rb_api
        env_file: ./config/.env.staging
        depends_on:
            - rb-database
        ports:
            - "3000:3000"
        networks:
            - rb-network

    rb-database:
        image: postgres:13
        container_name: rb_database
        volumes:
            - rb-pg-data:/var/lib/postgresql/data
        env_file: ./config/.env.staging
        healthcheck:
            test: pg_isready -U postgres -h rb-database
        networks:
            - rb-network

    rb-front:
        build:
            context: ../action-bidonvilles
            target: production-stage
        container_name: rb_front
        env_file: ./config/.env.staging
        restart: unless-stopped
        ports:
            - "80:80"
            - "443:443"
        volumes:
            - ./nginx-conf-tweak/nginx-alpine.conf:/etc/nginx/nginx.conf
            - ./nginx-conf:/etc/nginx/conf.d
            - certbot-etc:/etc/letsencrypt
            - certbot-var:/var/lib/letsencrypt
            - ssl:/etc/ssl/
            - web-root:/var/www/html
            - dhparam:/etc/ssl/certs
        depends_on:
            - rb-api
        networks:
            - rb-network

    rb-certbot:
        image: certbot/certbot
        container_name: rb_certbot
        volumes:
            - certbot-etc:/etc/letsencrypt
            - certbot-var:/var/lib/letsencrypt
            - ssl:/etc/ssl/
            - web-root:/var/www/html
        depends_on:
            - rb-front
        # command: certonly --webroot --webroot-path=/var/www/html --email email@domaine.com --agree-tos --no-eff-email --staging -d url.action.bidonville.fr  -d url.api.action.bidonville.fr
        # command: certonly --webroot --webroot-path=/var/www/html --email email@domaine.com --agree-tos --no-eff-email --force-renewal -d url.action.bidonville.fr -d url.api.action.bidonville.fr

volumes:
    rb-pg-data:
    certbot-etc:
    certbot-var:
    ssl:
    web-root:
        driver: local
        driver_opts:
            type: none
            device: /home/user_a_creer/rb-project/action-bidonvilles/dist
            o: bind
    dhparam:
        driver: local
        driver_opts:
            type: none
            device: /home/user_a_creer/rb-project/action-bidonvilles-deploy/dhparam/
            o: bind

networks:
    rb-network:
        driver: bridge
```

## Modification du fichier de configuration de Nginx

* Fichier *nginx-conf-tweak/nginx.conf* monté sur */etc/nginx/nginx.conf*

* Il s'agit d'une copie du fichier de l'image Docker Nginx à laquelle a été ajoutée une ligne permettant l'utilisation d'un nom de domaine long: *(server_names_hash_bucket_size 128;)*

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
    # avoid a possible hash bucket memory problem
    server_names_hash_bucket_size 128;
    include /etc/nginx/conf.d/*.conf;
}
```

## Création du fichier de définition du server block

* Fichier *nginx-conf/action-bidonvilles.conf* monté sur */etc/nginx/conf.d/action-bidonvilles.conf*

```
# 443 - https on frontend
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name preprod.resorption-bidonvilles.beta.gouv.fr;

    ssl_certificate /etc/letsencrypt/live/preprod.resorption-bidonvilles.beta.gouv.fr/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/preprod.resorption-bidonvilles.beta.gouv.fr/privkey.pem; # managed by Certbot

    root /var/www/html;
    index index.html app.html;
    location / {
        try_files $uri $uri/ =404;
    }
}

# 443 - https on backend
server {
     listen       443 ssl http2;
     listen       [::]:443 ssl http2;     
     server_name  api.preprod.resorption-bidonvilles.beta.gouv.fr;

     ssl_certificate /etc/letsencrypt/live/preprod.resorption-bidonvilles.beta.gouv.fr/fullchain.pem; # managed by Certbot
     ssl_certificate_key /etc/letsencrypt/live/preprod.resorption-bidonvilles.beta.gouv.fr/privkey.pem; # managed by Certbot

     location / {
          proxy_pass                 http://api.preprod.resorption-bidonvilles.beta.gouv.fr:3000;
          proxy_http_version         1.1;
          proxy_set_header Host      $host;
          proxy_set_header X-Real-IP $remote_addr;
     }
}

# Force https
server {
    listen 80;
    listen [::]:80;
    server_name preprod.resorption-bidonvilles.beta.gouv.fr;

    return 301 https://$host$request_uri;
}
```

## Création du fichier d'initialisation des variables d'environnement

* Fichier *config/.env.staging*

```
NODE_ENV=Staging
DB_USERNAME=userPostgres
DB_PASSWORD=passwordPostgres
DB_NAME=bddName
DB_HOST=
DB_PORT=5432
DB_DIALECT=postgres
FRONT_HOST=url.action.bidonville.fr
FRONT_PORT=80
API_HOSTNAME=url.api.action.bidonville.fr
API_CONTAINER_PORT=3000
API_EXTERNAL_PORT=3000
SECRET_FOR_TOKENS=any secret string
POSTGRES_USER=userPostgres
POSTGRES_PASSWORD=passwordPostgres
POSTGRES_DB=bddName
```
