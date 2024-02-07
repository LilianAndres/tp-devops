## Mise en place de la CI

Testcontainers est une bibliothèque Java qui permet de créer, gérer et détruire des conteneurs Docker lors de l'exécution des tests d'intégration. Dans notre contexte, Testcontainers est utilisé pour simplifier les tests en fournissant un environnement Dockerisé pour exécuter des tests qui nécessitent des ressources externes telles que des bases de données.

Dans cette première architecture d'intégration continue, le workflow est déclenché à chaque push sur les branches master et develop du répertoire distant. La première étape de ce workflow consiste à installer et à configurer le JDK 17 de Java de le conteneur du job afin d'être en mesure de pouvoir d'exécuter la seconde étape de ce job : la compilation et le lancement des tests d'intégrations du projet Spring Boot. Il ne faut pas oublier de spécifier le chemin du fichier pom.xml au lancement de la commande `mvn verify` pour que Maven le trouve (il ne se trouve pas à la racine).

```yaml
name: CI devops 2023
on:
  # to begin you want to launch this job in main and develop
  push:
    branches:
      - master
      - develop

jobs:
  build-and-test-backend:
    runs-on: ubuntu-22.04
    steps:
      # checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0
      # do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: "17"
          distribution: "temurin"
          cache: "maven" # cache Maven dependencies
      - name: Build and test with Maven
        run: mvn --batch-mode --update-snapshots verify --file docker/backend/pom.xml
```

## Mise en place de la CD

Afin de mettre en place un second job dans ce même pipeline dont l'objectif est de build et pousser les images Docker sur Docker Hub, il faut veiller à plusieurs choses. Premièrement, la création de secrets Github s'est avérée utile afin de stocker de manière sécurisée des éléments tels que les token d'authentification à Docker Hub. Ensuite, il ne faut surtout pas oublier d'ajouter la commande `needs: build-and-test-backend` avant le lancement des premières steps du job. Cette commande nous permet de bloquer l'exécution de ce second job si le premier n'est pas terminé ou ne s'est pas terminé correctement. En enlevant cette ligne, le second job aurait été executé paralèllement au premier, ce qui n'aurait pas eu de sens dans notre cas. En effet, ce n'est probablement pas la meilleure idée de build et push les images Docker si les tests unitaires de notre application échouent. Cela signifierait que des régréssions seraient apparues dans notre code source à la suite de nos nouveaux développements et que le déploiement continu ne devrait pas avoir lieu, au risque de mettre à disposition des utilisateurs une application 'cassée'.

Cependant, plusieurs étapes notables sont à relever dans ce job :

- Comme d'habitude, il faut toujours veiller à checkout sur notre code afin de se placer sur la bonne branche
- Une connexion à Docker Hub est ensuite nécessaire afin d'anticiper le déploiement de nos images sur Internet. On utilise pour ce faire la commande `uses: docker/login-action@v3` en précisant nos credentials, précédemment stockés dans les secrets Github
- Les trois étapes suivantes permettent ensuite de compiler chacune de nos trois images Docker, de les tagger correctement et de les pousser sur Docker Hub. Il ne faut pas oublier de fournir le chemin d'accès relatif à notre Dockerfile grâce à l'attribut `context`.

La commande `push: ${{ github.ref == 'refs/heads/master' }}` nous permet d'indiquer à l'action docker build-push-action@v3 de ne publier que lorsqu'un push est réalisé sur la branche master.

```yaml
build-and-push-docker-image:
  needs: build-and-test-backend # run only when code is compiling and tests are passing
  runs-on: ubuntu-22.04
  steps:
    - name: Checkout code
      uses: actions/checkout@v2.5.0
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - name: Build image and push backend
      uses: docker/build-push-action@v3
      with:
        context: ./docker/backend # relative path to the place where source code with Dockerfile is located
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
        push: ${{ github.ref == 'refs/heads/master' }}
    - name: Build image and push database
      uses: docker/build-push-action@v3
      with:
        context: ./docker/database
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-database:latest
        push: ${{ github.ref == 'refs/heads/master' }}
    - name: Build image and push httpd
      uses: docker/build-push-action@v3
      with:
        context: ./docker/web
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-web-server:latest
        push: ${{ github.ref == 'refs/heads/master' }}
```

Le fait de compiler et pousser chacune de nos images Docker sur Docker Hub lors de la mise à jour de notre code de manière totalement automatisée est extrêment pratique. En effet, dans une optique d'intégration et de déploiement continue, l'utilisateur peut bénéficier des dernières mises à jour des images de notre application régulièrement. En tant que développeurs, c'est un gain de temps énorme d'être en mesure de déployer nos applications de manière automatique.

## Qualité de code

L'utilisation d'un outil de qualimétrie statique de code tel que Sonar est très pratique pour plusieurs raisons. Cela nous permet de détecter rapidement les bugs, les failles de sécurité, la duplication de code, le pourcentage de couverture du code...etc...

Les étapes de ce premier job (build-and-test-backend) ont été revues afin d'automatiser les tests et le scanner Sonar sur notre projet :

- On checkout d'abord sur notre code
- On met en place le JDK de Java 17 sur le conteneur du job
- On met en cache les packages SonarCloud ainsi que les packages Maven pour des questions de performance et dans une optique de réutilisabilité future
- On exécute ensuite la commande `run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=lilianandres_lilianandres --file docker/backend/pom.xml` afin de lancer l'exécution des tests du projet et de lancer le scan Sonar.

Ces scans sont ensuite accessibles directement depuis notre projet Sonar, sur lesquels on retrouve le rapport d'analyse du code. Le niveau de qualimétrie de notre code peut facilement être paramétré et les seuils de chacun des critères d'analyses peuvent être ajustés.

Le second job (pour les images Docker) est resté inchangé.

```yaml
name: CD devops 2024
on:
  push:
    branches:
      - master
      - develop

jobs:
  build-and-test-backend:
    name: Build and analyze
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0 # Shallow clones should be disabled for a better relevancy of analysis
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: "zulu"
      - name: Cache SonarCloud packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2
      - name: Build and analyze
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=lilianandres_lilianandres --file docker/backend/pom.xml

  build-and-push-docker-image:
    name: Build and push images
    needs: build-and-test-backend
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./docker/backend
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./docker/database
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-database:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./docker/web
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-web-server:latest
          push: ${{ github.ref == 'refs/heads/master' }}
```

## Découpage des pipelines

Le découpage des pipelines dans des fichiers distincts peut être assez pratique afin de séparer efficacement les responsabilités de chacun de nos pipelines et de faciliter la lecture et la maintenabilité de nos workflows.

Cependant, lorsque les pipelines sont inter-dépendants, cela peut devenir plus complexe à gérer. Pour ce faire, nous pouvons utiliser la commande `on: workflow_run`, comme dans le fichier `build-and-push-images.yml`. Etant donné que ce second pipeline dépend évidemment toujours de la bonne exécution du premier pipeline qui teste notre code, mais que chacun de ces jobs s'exécute paralèllement, le problème devient difficile à résoudre. Dans cette situation, il ne serait simplement pas envisageable d'utiliser l'attribut `need` puisque le second pipeline n'aurait pas connaissance des jobs présents dans le premier pipeline.

L'utilisation du bloc suivant dans le second pipeline nous permet de déclencher son contenu seulement lorsque l'exécution du premier pipeline nommé 'CI devops 2024', sur la branche master, est terminé.

```yaml
on:
  workflow_run:
    workflows: [CI devops 2024]
    branches: [master]
    types:
      - completed
```

Attention toutefois, cette méthode de déclenchement ne suffit pas. Dans ce cas, on attend que l'exécution du premier pipeline soit terminée avant de déclencher l'exécution du second pipeline mais on ne regarde pas si le premier pipeline s'est terminée en erreur et de manière anticipée. Afin de nous prémunir de ce type de problématique, il faut ajouter une condition `if: ${{ github.event.workflow_run.conclusion == 'success' }}` avant le bloc `steps` pour n'exécuter ces steps uniquement si le précédent pipeline, celui dont on attendait la fin de l'exécution, s'est terminé avec succès. On conserve donc le comportement attendu.

### build-and-test-backend.yml

```yaml
name: CI devops 2024
on:
  push:
    branches:
      - "*"

jobs:
  build-and-test-backend:
    name: Build and analyze
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0 # Shallow clones should be disabled for a better relevancy of analysis
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: "zulu"
      - name: Cache SonarCloud packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2
      - name: Build and analyze
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=lilianandres_lilianandres --file docker/backend/pom.xml
```

### build-and-push-images.yml

```yaml
name: CD devops 2024
on:
  workflow_run:
    workflows: [CI devops 2024]
    branches: [master]
    types:
      - completed

jobs:
  build-and-push-docker-image:
    name: Build and push images
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./docker/backend
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./docker/database
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-database:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./docker/web
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-web-server:latest
          push: ${{ github.ref == 'refs/heads/master' }}
```
