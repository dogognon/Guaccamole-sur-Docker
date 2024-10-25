# Guacamole avec Docker Compose

Voici une petite documentation sur la façon de faire fonctionner une instance pleinement opérationnelle d'**Apache Guacamole (en incubation)** avec Docker (Docker Compose). Le but de ce projet est de faciliter les tests de Guacamole.

## À propos de Guacamole

Apache Guacamole (en incubation) est une passerelle de bureau à distance sans client. Il prend en charge les protocoles standard comme VNC, RDP et SSH. Il est qualifié de "sans client" car aucun plugin ou logiciel client n'est nécessaire. Grâce à HTML5, une fois Guacamole installé sur un serveur, tout ce dont vous avez besoin pour accéder à vos bureaux est un navigateur web.

Il prend en charge RDP, SSH, Telnet et VNC, et c'est la passerelle HTML5 la plus rapide que je connaisse. Consultez la [page d'accueil du projet](https://guacamole.incubator.apache.org/) pour plus d'informations.

## Prérequis

Vous avez besoin d'une installation fonctionnelle de **Docker** et de **Docker Compose** sur votre machine.

## Démarrage rapide

Clonez le dépôt Git et démarrez Guacamole :

~~~bash
git clone "https://github.com/boschkundendienst/guacamole-docker-compose.git"
cd guacamole-docker-compose
./prepare.sh
docker compose up -d
~~~

Votre serveur Guacamole devrait maintenant être accessible à l'adresse `https://ip de votre serveur:8443/`. Le nom d'utilisateur par défaut est `guacadmin` avec le mot de passe `guacadmin`.

## Détails

Pour comprendre certains détails, examinons de plus près les parties du fichier `docker-compose.yml` :

### Réseau

La partie suivante du fichier `docker-compose.yml` crée un réseau nommé `guacnetwork_compose` en mode "bridged".

~~~yaml
...
# réseaux
# créer un réseau 'guacnetwork_compose' en mode 'bridged'
networks:
  guacnetwork_compose:
    driver: bridge
...
~~~

### Services

#### guacd

La partie suivante du fichier `docker-compose.yml` crée le service guacd. guacd est le cœur de Guacamole, qui charge dynamiquement le support des protocoles de bureau à distance (appelés "plugins client") et les connecte aux bureaux distants en fonction des instructions reçues de l'application web. Le conteneur sera nommé `guacd_compose` et sera basé sur l'image Docker `guacamole/guacd`, connecté au réseau `guacnetwork_compose`. De plus, nous mappons les deux dossiers locaux `./drive` et `./record` dans le conteneur pour pouvoir les utiliser ultérieurement pour mapper les lecteurs utilisateur et enregistrer les sessions.

~~~yaml
...
services:
  # guacd
  guacd:
    container_name: guacd_compose
    image: guacamole/guacd
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/record:rw
...
~~~

#### PostgreSQL

La partie suivante du fichier `docker-compose.yml` crée une instance de PostgreSQL en utilisant l'image officielle Docker. Cette image est hautement configurable à l'aide de variables d'environnement. Par exemple, elle initialise une base de données si un script d'initialisation est trouvé dans le dossier `/docker-entrypoint-initdb.d` de l'image. Puisque nous mappons le dossier local `./init` à l'intérieur du conteneur comme `docker-entrypoint-initdb.d`, nous pouvons initialiser la base de données pour Guacamole en utilisant notre propre script (`./init/initdb.sql`). Vous pouvez lire plus de détails sur l'image officielle de PostgreSQL [ici](http://).

~~~yaml
...
  postgres:
    container_name: postgres_guacamole_compose
    environment:
      PGDATA: /var/lib/postgresql/data/guacamole
      POSTGRES_DB: guacamole_db
      POSTGRES_PASSWORD: ChoisissezVotreMotDePasseIci1234
      POSTGRES_USER: guacamole_user
    image: postgres
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:ro
    - ./data:/var/lib/postgresql/data:rw
...
~~~

#### Guacamole

La partie suivante du fichier `docker-compose.yml` crée une instance de Guacamole en utilisant l'image Docker `guacamole` de Docker Hub. Elle est également hautement configurable à l'aide de variables d'environnement. Dans cette configuration, elle est configurée pour se connecter à l'instance PostgreSQL précédemment créée en utilisant un nom d'utilisateur et un mot de passe, ainsi que la base de données `guacamole_db`. Le port 8080 n'est exposé que localement ! Nous attacherons une instance de Nginx pour l'accès public dans l'étape suivante.

~~~yaml
...
  guacamole:
    container_name: guacamole_compose
    depends_on:
    - guacd
    - postgres
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_HOSTNAME: postgres
      POSTGRES_PASSWORD: ChoisissezVotreMotDePasseIci1234
      POSTGRES_USER: guacamole_user
    image: guacamole/guacamole
    links:
    - guacd
    networks:
      guacnetwork_compose:
    ports:
    - 8080/tcp
    restart: always
...
~~~

#### Nginx

La partie suivante du fichier `docker-compose.yml` crée une instance de Nginx qui mappe le port public 8443 au port interne 443. Le port interne 443 est ensuite mappé à Guacamole en utilisant le fichier `./nginx/templates/guacamole.conf.template`. Le conteneur utilise le certificat auto-signé généré précédemment (`prepare.sh`) dans `./nginx/ssl/` avec `./nginx/ssl/self-ssl.key` et `./nginx/ssl/self.cert`.

~~~yaml
...
  # nginx
  nginx:
   container_name: nginx_guacamole_compose
   restart: always
   image: nginx
   volumes:
   - ./nginx/templates:/etc/nginx/templates:ro
   - ./nginx/ssl/self.cert:/etc/nginx/ssl/self.cert:ro
   - ./nginx/ssl/self-ssl.key:/etc/nginx/ssl/self-ssl.key:ro
   ports:
   - 8443:443
   links:
   - guacamole
   networks:
     guacnetwork_compose:
...
~~~

## prepare.sh

`prepare.sh` est un petit script qui crée `./init/initdb.sql` en téléchargeant l'image Docker `guacamole/guacamole` et en la démarrant ainsi :

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > ./init/initdb.sql
~~~

Il crée le fichier d'initialisation de la base de données nécessaire pour PostgreSQL.

`prepare.sh` crée également le certificat auto-signé `./nginx/ssl/self.cert` et la clé privée `./nginx/ssl/self-ssl.key` qui sont utilisés par Nginx pour HTTPS.

## reset.sh

Pour tout réinitialiser à l'état initial, exécutez simplement `./reset.sh`.

## WOL

Wake on LAN (WOL) ne fonctionne pas et cela ne sera pas corrigé car cela dépasse le cadre de ce dépôt. Cependant, [zukkie777](https://github.com/zukkie777), qui a également soumis [ce problème](https://github.com/boschkundendienst/guacamole-docker-compose/issues/12), l'a corrigé. Vous pouvez en lire plus sur la [liste de diffusion Guacamole](http://apache-guacamole-general-user-mailing-list.2363388.n4.nabble.com/How-to-docker-composer-for-WOL-td9164.html).

**Avertissement**

Le téléchargement et l'exécution de scripts provenant d'Internet peuvent nuire à votre ordinateur. Assurez-vous de vérifier la source des scripts avant de les exécuter !
