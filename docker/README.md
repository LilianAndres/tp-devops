# TP Docker

## Base de données

Afin de créer notre image, on lance les commandes suivantes :

```docker
docker build -t lilianandres/database .
docker run -p 5432:5432 --name database lilianandres/database
```

A cet instant, la base de données est lancée et tourne dans notre conteneur. Cependant, pour utiliser Adminer avec PostgreSQL, il faut créer un second conteneur et un réseau virtuel afin de permettre à ces deux conteneurs de communiquer. Il ne faut pas oublier, au lancement de nos deux conteneurs, de les ajouter dans le réseau précédemment créé.

```docker
docker network create app-network

docker run -d --network app-network -p 8082:8080 adminer
docker run -d --network app-network -p 5432:5432 --name database lilianandres/database
```

- `d`: détacher le conteneur du terminal (lancement en arrière-plan)
- `--network`: placer le conteneur dans un réseau nommé
- `-p`: mapper les ports exposés par le conteneur sur le host
- `--name`: nommer le conteneur

Afin de ne pas écrire le mot de passe de la base de données dans un fichier de configuration statique, nous pouvons le placer dans une variable d'environnement au lancement du conteneur.

```docker
docker run -d \
--network app-network \
-e POSTGRES_PASSWORD="pwd" \
-p 5432:5432 \
--name database lilianandres/database
```

- `-e`: fournir une variable d'environnement au conteneur

Afin que la base de données soit initialisée avec une structure et de données, il faut ajouter des scripts d'initialisation dans le projet (cf dossier sql). Il faut également ajouter la ligne suivante dans le Dockerfile pour copier ces fichiers dans notre conteneur afin que ces derniers soient joués au montage du conteneur.

```docker
COPY ./sql /docker-entrypoint-initdb.d
```

En revanche, lorsque le conteneur est détruit, les données ne persistent pas. Il faut pour cela créer un volume qui va nous permettre de sauvegarder les changements localement sur le disque de la machine hôte. De ce fait, lorsque le conteneur sera détruit puis relancer, les données présentes sur le volume seront "restaurées". Par exemple, si je change une donnée dans une des tables de la base, et que je détruis puis relance mon conteneur, je verrais la modification que j'avais faite avant la destruction du conteneur dans la base de données.

```docker
docker run -d \
--network app-network \
-e POSTGRES_USER="usr" \
-e POSTGRES_PASSWORD="pwd" \
-p 5432:5432 \
-v /Users/lilian/datadir:/var/lib/postgresql/data \
--name database lilianandres/database
```

- `-v`: attacher un volume physique de l'hôte au conteneur

Il existe deux manières de créer des volumes : avec la commande `docker create volume <name>` ou en spécifiant un volume lors du démarrage d'un conteneur (comme dans l'exemple ci-dessus). En l'absence d'un chemin, Docker gérera lui-même l'endroit où il stocke le volume sur la machine hôte afin d'assurer la persistence des données.

## Backend

Afin de construire une image contenant un JRE Java et pouvoir exécuter notre code dedans, il est possible d'utiliser le Dockerfile suivant :

```docker
FROM eclipse-temurin:17

# Copy the compiled Java code
COPY Main.class .

# Execute the program at the container startup
CMD ["java", "Main"]
```

Il suffit donc de build notre image personnalisée et de run le conteneur avec cette image et le message "Hello World" apparaît dans notre console.

```docker
docker build -t liliandres/backend .
docker run --name api liliandres/backend
```

Afin d'utiliser le multistage pour notre projet Spring Boot, on utilise le Dockerfile suivant :

```docker
# Build
# Source image with a name alias
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

# Create env variable
ENV MYAPP_HOME /opt/myapp

# Set the workdir path
WORKDIR $MYAPP_HOME

# Copy files from source to destination on the container
COPY pom.xml .
COPY src ./src

# Run mvn package command to build the app
RUN mvn package -DskipTests

# Run
# Source image
FROM amazoncorretto:17

# Create env variable
ENV MYAPP_HOME /opt/myapp

# Set the workdir path
WORKDIR $MYAPP_HOME

# Copy the compiled code from the previous stage into the new one as myapp.jar
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Run the JAR and start the backend
ENTRYPOINT java -jar myapp.jar
```

L'intérêt d'utiliser un Dockerfile multistage est multiple. Cela nous permet de faciliter la lecture et la maintenabilité du fichier. Cela nous permet également de centraliser les étapes de build et de run (de l'API) dans un seul fichier, laissant uniquement dans l'image final l'état du dernier stage. En d'autres termes, le stage du build n'est utilisé que pour générer le fichier compilé. Aucun des outils ou des configurations nécessaires au build ne sont présentes dans l'image finale, ce qui s'avère être extrêmement pratique. L'image finale est donc propre, légère et ne contient que le strict nécessaire.

Dans ce Dockerfile, nous utilisons la commande WORKDIR. Cette commande permet de spécifier le répertoire de travail dans l'image. Ce répertoire sera utilisé comme répertoire de base pour toutes les commandes de type RUN, CMD, ENTRYPOINT, COPY et ADD. Spécifier un répertoire de travail est généralement une bonne pratique. Cela permet notamment d'éviter de se promener avec des CD.

Afin de configurer le second projet Spring Boot (le projet plus complet), on peut évidemment utiliser le même Dockerfile que le précédent. Il faut simplement paramétrer le projet Spring pour qu'il puisse joindre la base de données.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://database:5432/${DATABASE_NAME}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
    driver-class-name: org.postgresql.Driver
management:
  server:
    add-application-context-header: false
  endpoints:
    web:
      exposure:
        include: health,info,env,metrics,beans,configprops
```

```docker
docker run -d \
--network app-network \
-e DATABASE_NAME="db" \
-e DATABASE_USER="usr" \
-e DATABASE_PASSWORD="pwd" \
-p 8080:8080 \
--name backend liliandres/backend
```

Il ne faut pas oublier de fournir les variables d'environnements essentielles lors du lancement du conteneur. Il ne faut pas non plus oublier de placer le projet dans le même réseau Docker que la base de données.

## Web Server

Afin de monter un conteneur et d'avoir un serveur HTTP, on peut, par exemple, utiliser Apache. Le Dockefile ressemble au suivant :

```docker
FROM httpd:2.4

COPY ./public/ /usr/local/apache2/htdocs/
```

Dans cette image, on copie notre dossier public contenant nos fichiers HTML à l'emplacement indiqué sur la documentation de l'installation d'Apache (/usr/local/apache2/htdocs/). Pour vérifier que notre serveur web fonctionne correctement, on lance la commande suivante :

```docker
docker build -t lilianandres/web .
docker run -d -p 80:80  --network app-network --name web lilianandres/web
```

Il suffit ensuite de nous rendre à l'adresse http://localhost:80/ et observer notre page `index.html`.

Ensuite, il est possible de récupérer un fichier présent dans un conteneur de plusieurs façons. Une des façons la plus simple est d'utiliser la commande docker cp tel que :

```docker
docker cp web:/usr/local/apache2/conf/httpd.conf /Users/lilian/Documents/dev/CPE/S8/DevOps/tp-devops/web/
```

Afin de configurer Apache comme étant un Reverse Proxy, il faut modifier la configuration Apache et ne pas oublier de rebuild l'image Apache en y ajoutant la ligne suivante :

```docker
# Copy the custom Apache configuration into the container
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

## Docker Compose

Le contenu de mon fichier Docker Compose est le suivant :

```docker
version: "3.7"

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - DATABASE_NAME=${DB_NAME}
      - DATABASE_USER=${DB_USER}
      - DATABASE_PASSWORD=${DB_PWD}
    networks:
      - app-network
    depends_on:
      - database

  database:
    build:
      context: ./database
      dockerfile: Dockerfile
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PWD}
    volumes:
      - /Users/lilian/datadir:/var/lib/postgresql/data
    networks:
      - app-network

  httpd:
    build:
      context: ./web
      dockerfile: Dockerfile
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - database
      - backend

networks:
  app-network:
```

Pour lancer tous nos conteneurs avec une seule commande :

```docker
docker compose build && docker compose up -d
```

Dans chacun de nos 3 services, on indique à Docker où trouver notre image custom grâce au build/context qui indique où se trouve le fichier de configuration de notre image et au build/dockerfile qui indique le nom du fichier qui porte la configuration de notre image. De plus chacun de ces services est placé dans le réseau app-network que l'on a déclaré en fin de fichier. Les variables d'environnements sont ici gérés de manière dynamique pour plus de sécurité. Il existe plusieurs moyens différents de les fournir à notre stack (fichier de variables d'environnements, variables d'environnements globales Shell, ligne de commande...). Ces différentes manières sont classées par ordre de priorité (il faut se référer à la documentation).

Dans cet exemple, l'utilisation de Docker Compose s'est avérée très pratique pour plusieurs raisons évidentes. L'administration de 3 conteneurs différents peut rapidement devenir fastidieuse et redondante. Docker Compose nous permet de centraliser chacun de nos conteneurs au même endroit, ce qui rend la gestion de notre environnement plus simple, plus lisible et plus maintenable. Ainsi, il devient bien plus facile de démarrer/arrêter les différents conteneurs de notre stack, visualiser leurs états...

## Publish

Etant donné que Docker Hub peut également être utilisé pour host nos images personnalisées, il est possible de simplifier notre fichier `docker-compose.yml` tel que :

```docker
version: "3.7"

services:
  backend:
    image: lilianandres/backend:1.0
    environment:
      - DATABASE_NAME=${DB_NAME}
      - DATABASE_USER=${DB_USER}
      - DATABASE_PASSWORD=${DB_PWD}
    networks:
      - app-network
    depends_on:
      - database

  database:
    image: lilianandres/postgresql:1.0
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PWD}
    volumes:
      - /Users/lilian/datadir:/var/lib/postgresql/data
    networks:
      - app-network

  httpd:
    image: lilianandres/httpd:1.0
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - database
      - backend

networks:
  app-network:
```

Attention tout de même, il ne faudra pas oublier de les compiler ET de les pousser lorsque ces dernières seront mises à jour.
