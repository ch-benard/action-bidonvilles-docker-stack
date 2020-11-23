# Créer les outils de déploiement

> Ces outils sont regroupés dans un dépôt spécifiques intitulé "action-bidonvilles-deploy".

## docker-compose.yml 

* Le fichier *docker-compose.yml* permet l'exécution de la commande *docker-compose*, qui va lancer la construction des conteneurs correspondant à chaque micro-services.

```yaml
version: '3.4'

services:
    rb-api:
        build:
            context: ../rb-api
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
        environment:
            - POSTGRES_USER=userPostgres
            - POSTGRES_PASSWORD=passwordPostgres
            - POSTGRES_DB=bddName
        healthcheck:
            test: pg_isready -U postgres -h rb-database
        networks:
            - rb-network

    rb-front:
        build:
            context: ../rb-front
            target: production-stage
        container_name: rb_front
        env_file: ./config/.env.staging
        restart: unless-stopped
        ports:
            - "80:80"
            - "443:443"
        volumes:
            - ./nginx-conf:/etc/nginx/conf.d
            - certbot-etc:/etc/letsencrypt
            - certbot-var:/var/lib/letsencrypt
            - ssl:/etc/ssl/
            - web-root:/var/www/html
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
        #depends_on:
        #    - rb-front
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
            device: /home/user_a_creer/rb-project/rb-front/dist
            o: bind
    dhparam:
        driver: local
        driver_opts:
            type: none
            device: /home/user_a_creer/rb-project/rb-deploy/dhparam/
            o: bind

networks:
    rb-network:
        driver: bridge
```


## nginx-conf/default.conf

* Configurer le serveur Web/reverse-proxy

```
# 443 - https on frontend
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name url.action.bidonville.fr;

    ssl_certificate /etc/letsencrypt/live/url.action.bidonville.fr/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/url.action.bidonville.fr/privkey.pem; # managed by Certbot

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
     server_name  url.api.action.bidonville.fr;

     ssl_certificate /etc/letsencrypt/live/url.action.bidonville.fr/fullchain.pem; # managed by Certbot
     ssl_certificate_key /etc/letsencrypt/live/url.action.bidonville.fr/privkey.pem; # managed by Certbot

     location / {
          proxy_pass                 http://url.api.action.bidonville.fr:3000;
          proxy_http_version         1.1;
          proxy_set_header Host      $host;
          proxy_set_header X-Real-IP $remote_addr;
     }
}

# Force https
server {
    listen 80;
    listen [::]:80;
    server_name url.action.bidonville.fr;

    return 301 https://$host$request_uri;
}
```

## config/.env.staging

* Initialiser les variables d'environnement

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
