# Environnement de DEV Resporption-Bidonvilles

## Pour aller vite

### Prérequis

* Les outils de développement doivent être installés sur le poste du dveloppeur (Nodejs, Yarn, Docker, Docker-compose, etc).

* Pour déployer rapidement un environnement de développement sur un poste Linux:

```bash
mkdir resorption-bidonvilles
cd resorption-bidonvilles
git clone https://github.com/MTES-MCT/action-bidonvilles-api.git
git clone https://github.com/MTES-MCT/action-bidonvilles.git
git clone https://github.com/MTES-MCT/action-bidonvilles-infra.git

cd action-bidonvilles-infra
docker-compose -f docker-compose-dev.yml up -d

cd ../action-bidonvilles
yarn install
```

## Composition de l'environnment pour le développement

* Script ***scripts-dev/startup.sh*** monté à la racine du conteneur de l'API

```bash
#!/bin/sh
echo "Chargement du répertoire de l'application"
echo "-----------------------------------------"
cd /home/node/app
echo "-----------------------------------------"
echo "Installation des modules Node"
yarn install --dev
echo "-----------------------------------------"
echo "Lancement de nodemon"
node_modules/.bin/nodemon server/index.js --inspect=0.0.0.0:9229
```

### docker-compose-dev.yml

```yml
version: '3.4'
services:
    rb-api:
        image: node:14.15.0-alpine
        container_name: rb_api
        working_dir: /home/node/app
        depends_on:
            - rb-database
        ports:
            - "1236:8080"
        volumes:
            - ../action-bidonvilles-api:/home/node/app
            - ./config-dev/config.js.db:/home/node/app/db/config/config.js
            - ./config-dev/config.js.server:/home/node/app/server/config.js
            - ./scripts-dev/startup.sh:/home/node/app/startup.sh
        networks:
            - rb-network
        command: /home/node/app/startup.sh

    rb-database:
        image: postgres:13
        container_name: rb_database
        volumes:
            - rb-pg-data:/var/lib/postgresql/data
        env_file: ./config/.env.sample
        healthcheck:
            test: pg_isready -U postgres -h rb-database
        networks:
            - rb-network

volumes:
    rb-pg-data:

networks:
    rb-network:
        driver: bridge
```

### Fichiers config-dev/...

* ***config.js.db***

```javascript
module.exports = {
    username: 'fabnum',
    password: 'fabnum',
    database: 'bidonvilles',
    host: 'rb-database',
    port: 5432,
    dialect: 'postgres',
};
```

* ***config.js.server***

```javascript
const path = require('path');

const config = {
    assetsSrc: path.resolve(__dirname, '../assets'),
    frontUrl: 'http://localhost:1234',
    port: 8080,
    externalPort: 1236,
    auth: {
        secret: 'secret used for generating authentication tokens',
        expiresIn: '168h', // 7 days
    },
    activationTokenExpiresIn: '48h',
    passwordResetDuration: '24h',
    mail: {
        publicKey: 'MAILJET_API_PUBLIC_KEY',
        privateKey: 'MAILJET_API_PRIVATE_KEY',
    },
};
config.backUrl = `http://localhost:${config.externalPort}`;
```
