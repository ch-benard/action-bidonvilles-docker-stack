# Conteneuriser le projet

> Le principe

* Il s'agit d'automatiser la contruction des images Docker de l'API et du Frontend qui seront envoyées sur le registre Dockerhub de la startup *resorption-bidonvilles*.

* La construction de ces images est automatisée grâce à l'utilisation des outils de déploiement continu, les *Github Actions*, disponibles sur [Github](https://github.com/features/actions).

* A chaque nouveau *commit* ou *pull-request* validé, par exemple, l'image Docker du service concerné peut être automatiquement contruite, taguée et déployée sur le registre Dockerhub.

* Il suffit ensuite de déclencher sur l'environnement concerné (pré-production - staging - ou production) la construction du conteneur Docker en se basant sur les dernières versions des images poussées vers le registre.

