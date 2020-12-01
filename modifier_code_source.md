# Modifications du code

> Il est nécessaire d'adapter certaines parties de l'application pour permettre la conteneurisation et le déploiement automatisé du projet

## Préparer la partie API

### Variabiliser certaines valeurs

* Il s'agit d'essayer de se conformer au point III des [12 facteurs](https://12factor.net/fr/) qui consiste à *stocker la configuration dans l'environnement*.

* Dans le fichier **db/config/config.js.sample**, remplacer les valeurs des éléments de configuration de l'environnement qui permettent l'accès à la base de données:

```javascript
module.exports = {
    username: '${DB_USERNAME}',
    password: '${DB_PASSWORD}',
    database: '${DB_NAME}',
    host: '${DB_HOST}',
    port: ${DB_PORT},
    dialect: '${DB_DIALECT}',
};
```

* Dans le fichier **server/config.js.sample**, remplacer les valeurs des éléments de configuration de l'environnement qui permettent de définir certains éléments de paramétrages:

```javascript
const path = require('path');

const config = {
    assetsSrc: path.resolve(__dirname, '../assets'),
    frontUrl: `${FRONT_HOST}:${FRONT_PORT}`,
    port: ${API_CONTAINER_PORT},
    apiHostname: '${API_HOSTNAME}',
    apiExternalPort: ${API_EXTERNAL_PORT},
    auth: {
        secret: '${SECRET_FOR_TOKENS}',
        expiresIn: '168h', // 7 days
    },
    activationTokenExpiresIn: '48h',
    passwordResetDuration: '24h',
    mail: {
        publicKey: 'MAILJET_API_PUBLIC_KEY',
        privateKey: 'MAILJET_API_PRIVATE_KEY',
    },
};
config.backUrl = `${config.apiHostname}:${config.apiExternalPort}`;
module.exports = config;
```

* Les valeurs des variables seront définies dans un fichier d'environnement (*.env*) propre à chaque contexte (*.env.production*, *.env.developement*, etc.), et référencé dans le fichier *docker-compose* qui servira à lancer la pile de conteneurs.

### Préparer l'image Docker

* Le challenge est de permettre de concevoir une image Docker optimisée, c'est à dire la plus compacte possible et exempt de tout artefact non directement utile à l'exécution du conteneur, tel que les dépendances de construction par exemple. 

* Le mutli-staging permet de générer des images intermédiaires, que l'on pourra ensuite invoquer lors de la constructon (build), pour spécifier la cible visée: développement, staging (pré-production), production etc.

* Contenu du Dockerfile de l'API:

```
## Stage 1: (production base)
FROM node:14.15.0-alpine AS base
# Information about this Docker image
LABEL org.opencontainers.image.authors=ch.benard[at]gmail.com
LABEL org.opencontainers.image.title="Resorption Bidonvilles API Docker Images"
LABEL org.opencontainers.image.licenses=MIT
EXPOSE $API_CONTAINER_PORT
# Defining the environment (development, staging, production, test)
ENV NODE_ENV=production
# Create the app directory
WORKDIR /home/node/rb/api
COPY . .

## Stage 2: (builddev)
FROM base as builddev
ENV NODE_ENV=development
ENV PATH=/home/node/rb/api/node_modules/.bin:$PATH
# Copy the files holding various metadata relevant to the project
COPY package.json yarn.lock ./
RUN yarn config list && yarn install --development

# Stage 3: (development)
FROM builddev as development
RUN apk add gettext libintl && apk add --no-cache 'su-exec>=0.2'
COPY rb-api-entrypoint.sh /usr/local/bin/
RUN chmod u+x /usr/local/bin/rb-api-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/rb-api-entrypoint.sh"]
CMD ["nodemon", "server/index.js", "--inspect=0.0.0.0:9229"]

## Stage 4: (test)
FROM builddev as test
RUN echo "Skipping test for the moment..."
# Pour qu'on ait à la fois les dépendances de developpement et de production:
# COPY --from=development /home/node/rb/api/node_modules /home/node/rb/api/node_modules
# RUN eslint .
# RUN yarn test
# RUN yarn test:unit
# RUN yarn audit

## Stage 5: (buildproduction)
FROM base as prepaprod
# Install project external dependencies and clean cache 
# yarn install --frozen-lockfile is the closest yarn alternative to npm -ci
RUN yarn config list && yarn install --production --frozen-lockfile --silent && yarn cache clean
# Suppress the test directory not reqired in production
RUN rm -rf ./test

## Stage 6: (production)
FROM prepaprod as production
RUN apk add --no-cache tini
RUN apk add gettext libintl && apk add --no-cache 'su-exec>=0.2'
COPY rb-api-entrypoint.sh /usr/local/bin/
RUN chmod u+x /usr/local/bin/rb-api-entrypoint.sh
ENTRYPOINT ["tini", "--", "/usr/local/bin/rb-api-entrypoint.sh"]
CMD ["node", "server/index.js"]
```

### Préparer le fichier .dockerignore

* Ce fichier, à l'instar du fichier .gitignore pour Git, permet de préciser la liste des fichiers et répertoires que l'image Docker ne doit pas embarquer.

```
node_modules
config.js
.vscode
coverage
.nyc_output
TODO
.DS_Store
postgres-data
.idea
```

### Modifier le fichier server/app.html (facultatif)

* Permet d'afficher, en plus du port, le nom du host qui exécute Node.

```javascript
const loaders = require('#server/loaders');
const { port } = require('#server/config');
const { backUrl } = require('#server/config');
module.exports = {
    start() {
        const app = loaders.express();
        loaders.routes(app);

        app.listen(port, () => {
            console.log(`Node is now running the API on ${backUrl}! :)`);
        });
    },
};
```

## Préparer la partie Front

### Ajouter la commande build-staging aux scripts dans le fichier package.json

* dans la partie *scripts* du fichier *package.json, après la commande *build*, ajouter la commande *build-staging*:

```
  "scripts": {
    "build": "vue-cli-service build",
    "build-staging": "vue-cli-service build --mode staging",
    ...
```

### Préparer l'image Docker

* Pour le moment, les étapes (stage) *develop-stage* et *build-stage* sont lancées manuellement pour créer une *release*. Le contenu du répertoire *dist* contenant la release est récupéré lors du clônage du dépôt Git. Ces étapes pourront être activées lors de la création des *github-actions* dans le cadre du déploiement continu.

```
## develop stage
# FROM node:14-alpine as develop-stage
# WORKDIR /app
# COPY package*.json ./
# RUN yarn install
# COPY . .

## build stage
# FROM develop-stage as build-stage
# RUN yarn build-staging
# # Remplacer ci-dessous x.x.x par le numéro de la release
# RUN yarn release ${FRONT_RELEASE}

## production stage
FROM nginx:mainline-alpine as production-stage
WORKDIR /var/www/html
#COPY --from=build-stage /app/dist /var/www/html
COPY dist/* ./
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
```

### Préparer le fichier .env.staging

* Ce fichier contient le nom du serveur servant l'api.

* .env.staging

```env
VUE_APP_API_URL=url.api.action.bidonville.fr
```
