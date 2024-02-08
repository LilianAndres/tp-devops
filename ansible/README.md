# Ansible

## Inventory

Un inventaire Ansible est un fichier qui répertorie les hôtes sur lesquels Ansible doit agir. C'est essentiellement une liste des serveurs, des machines virtuelles ou d'autres appareils réseau sur lesquels on souhaite exécuter des playbooks (séries d'instructions) ou des commandes Ansible.

```yml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ~/.ssh/id_rsa_ansible
  children:
    prod:
      hosts: lilian.andres.takima.cloud
```

- `all`: groupe principal de l'inventaire
- `vars`: variables spécifiques à tous les hôtes répertoriés dans le groupe `all`
- `ansible_user`: nom d'utilisateur à utiliser lors de la connexion aux hôtes contenus dans le groupe `all`
- `ansible_ssh_private_key_file`: chemin vers la clé privé SSH à utiliser
- `children`: sous-groupe d'hôtes qui hériteront des variables du groupe parent `all`
- `prod`: nom du premier sous-groupe d'hôtes
- `hosts`: hôtes inclus dans ce groupe

## Facts

Les "facts" Ansible sont des informations sur l'état d'un système cible que Ansible collecte lors de l'exécution de tâches ou de playbooks. Ces informations comprennent des détails tels que le système d'exploitation, la version du noyau, l'adresse IP, les interfaces réseau, les partitions de disque, etc.

```shell
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

- `all`: exécuter la commande sur tous les hôtes répertoriés dans l'inventaire
- `-i inventories/setup.yml`: spécifier le chemin de l'inventaire
- `-m setup`: utiliser le module setup d'Ansible pour collecter des facts sur les hôtes cibles
- `-a "filter=ansible_distribution*"`: fournir les arguments à passer au module setup. L'argument "filter" permet de filtrer les facts collectés uniquement pour les variables liées à la distribution du système d'exploitation.

```shell
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

- `all`: exécuter la commande sur tous les hôtes répertoriés dans l'inventaire
- `-i inventories/setup.yml`: spécifier le chemin de l'inventaire
- `-m yum`: utiliser le module yum d'Ansible pour gérer les packages Python
- `-a "name=httpd state=absent`: fournir les arguments à passer au module setup. Dans ce cas, on indique au module yum que l'on souhaite que le package httpd soit dans l'état absent, donc que l'on souhaite le désinstaller
- `--become`: devenir superutilisateur (root) lors de l'exécution de la commande

## Playbooks

### Playbook simple

Dans ce premier exemple, l'objectif est simple : il faut ping tous les hôtes répertoriés dans notre inventaire.

```yml
- hosts: all # les tâches du playbook seront exécutées pour tous les hôtes de l'inventaire (groupe all)
  gather_facts: false # désactive la collecte automatique des facts sur les hôtes ciblés
  become: true # exécuter les tâches du playbook en tant que superutilisateur (root)

  tasks:
    - name: Test connection # nom de la tâche (indiqué dans la trace d'exécution du playbook)
      ping: # utilisation du module ping d'Ansible
```

## Playbook avancé

Dans ce second exemple, l'objectif est légèrement plus complexe : il faut installer Docker sur notre serveur.

```yml
- hosts: all
  gather_facts: false
  become: true

  tasks:
    # Installer la dernière version du package device-mapper-persistent-data
    - name: Install device-mapper-persistent-data
    yum:
        name: device-mapper-persistent-data
        state: latest

    # Installer la dernière version du package lvm2
    - name: Install lvm2
    yum:
        name: lvm2
        state: latest

    # Ajouter le répository Docker au référentiel des paquets yum (cf. Docker CentOS)
    - name: add repo docker
    command:
        cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

    # Installer Docker Community Edition
    - name: Install Docker
    yum:
        name: docker-ce
        state: present # si ce n'est pas déjà présent dans le système cible

    # Installer Python 3
    - name: Install python3
    yum:
        name: python3
        state: present # si ce n'est pas déjà présent dans le système cible

    # Installer le package Python 'docker'
    - name: Install docker with Python 3
    pip:
        name: docker
        executable: pip3
    vars:
        ansible_python_interpreter: /usr/bin/python3 # indiquer à Ansible d'utiliser python3 pour l'interprétation

    # Vérifier que Docker tourne sur le système avec le module service
    - name: Make sure Docker is running
    service: name=docker state=started # s'il n'est pas en cours d'exécution, Ansible le démarrera
    tags: docker
```

## Création des rôles

La commande suivante nous permet de créer le rôle Docker :

```shell
ansible-galaxy init roles/docker
```

Cette commande génère une arborescence de fichier dans notre projet. Le dossier roles/docker contient les sous-répertoires suivants :

- `defaults` contient les fichiers YAML qui définissent les valeurs par défaut des variables utilisées dans le rôle
- `files` est utilisé pour stocker les fichiers (scripts, configurations, certificats...) qui doivent être copiés sur les hôtes cibles lors de l'exécution du rôle
- `handlers` sont des tâches qui sont déclenchées par d'autres tâches lorsque celles-ci modifient l'état du système. Par exemple, si une tâche installe un nouveau logiciel, un handler peut être déclenché pour redémarrer un service associé à ce logiciel
- `meta` contient un fichier nommé main.yml qui peut être utilisé pour spécifier les dépendances du rôle. Les dépendances peuvent être d'autres rôles Ansible ou des collections Ansible
- `tasks` contient le coeur du rôle, c'est-à-dire les tâches à exécuter sur les hôtes cibles lors de l'utilisation du rôle
- `templates` est utilisé pour stocker les modèles de fichiers qui seront rendus sur les hôtes cibles. Les modèles peuvent inclure des variables et des boucles, ce qui permet de générer des fichiers de configuration personnalisés en fonction des spécificités de chaque hôte
- `tests` est utilisé pour stocker les tests unitaires et les tests d'intégration du rôle
- `vars` contient des fichiers YAML qui définissent des variables spécifiques au rôle

Dans notre cas, seul le dossier `tasks` est utile. Nous pouvons supprimer les autres (ou les conserver à des fins d'évolution potentielle).

```yml
# ansible/playbook.yml

- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker # exécuter les tasks du rôle Docker
```

```yml
# ansible/roles/docker/tasks/main.yml

# Installer la dernière version du package device-mapper-persistent-data
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

# Installer la dernière version du package lvm2
- name: Install lvm2
  yum:
    name: lvm2
    state: latest

# Ajouter le répository Docker au référentiel des paquets yum (cf. Docker CentOS)
- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

# Installer Docker Community Edition
- name: Install Docker
  yum:
    name: docker-ce
    state: present # si ce n'est pas déjà présent dans le système cible

# Installer Python 3
- name: Install python3
  yum:
    name: python3
    state: present # si ce n'est pas déjà présent dans le système cible

# Installer le package Python 'docker'
- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3 # indiquer à Ansible d'utiliser python3 pour l'interprétation

# Vérifier que Docker tourne sur le système avec le module service
- name: Make sure Docker is running
  service: name=docker state=started # s'il n'est pas en cours d'exécution, Ansible le démarrera
  tags: docker
```

## Déploiement

Afin de gérer le déploiement de nos différentes applications de manière automatisée, plusieurs rôles ont été créés, en plus du rôle Docker précédemment évoqué.

```yaml
# ansible/roles/network/tasks.yml

- name: Create the Docker network
  docker_network:
    name: "{{ NETWORK_NAME }}"
```

```yaml
# ansible/roles/database/tasks.yml

- name: Create the database container
  docker_container:
    name: "{{ DATABASE_HOST }}"
    image: lilianandres/tp-devops-simple-database:latest
    networks:
      - name: "{{ NETWORK_NAME }}"
    env:
      POSTGRES_DB: "{{ DATABASE_NAME }}"
      POSTGRES_USER: "{{ DATABASE_USER }}"
      POSTGRES_PASSWORD: "{{ DATABASE_PASSWORD }}"
    volumes:
      - datavolume:/var/lib/postgresql/data
```

```yaml
# ansible/roles/app/tasks.yml

- name: Create the app container
  docker_container:
    name: "{{ BACKEND_HOST }}"
    image: lilianandres/tp-devops-simple-api:latest
    networks:
      - name: "{{ NETWORK_NAME }}"
    env:
      DATABASE_HOST: "{{ DATABASE_HOST }}"
      DATABASE_NAME: "{{ DATABASE_NAME }}"
      DATABASE_USER: "{{ DATABASE_USER }}"
      DATABASE_PASSWORD: "{{ DATABASE_PASSWORD }}"
```

```yaml
# ansible/roles/proxy/tasks.yml

- name: Create the proxy container
  docker_container:
    name: proxy
    image: lilianandres/tp-devops-web-server:latest
    networks:
      - name: "{{ NETWORK_NAME }}"
    env:
      BACKEND_HOST: "{{ BACKEND_HOST }}"
    ports:
      - "80:80"
```

Il faut ensuite appeler ces rôles dans le bon ordre dans notre playbook afin de les déployer les uns à la suite des autres sur notre serveur cible.

```yaml
# ansible/playbook.yml

- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
    - network
    - database
    - app
    - proxy
```

Enfin, étant donné que ces rôles utilisent des variables de configuration commune, j'ai pris la décision de ne pas dupliquer ces variables dans chacun des fichiers vars/main.yml dans tous les rôles. J'ai placé ces variables dans un fichier global et partagé afin de faciliter la maintabilité et ne pas dupliquer du code dans pleins de fichiers différents.

```yaml
# ansible/group_vars/all.yml

NETWORK_NAME: app-network
BACKEND_HOST: backend
DATABASE_HOST: database
DATABASE_NAME: db
DATABASE_USER: usr
DATABASE_PASSWORD: pwd
```

La commande pour permettre le déploiement de mon projet s'est déroulé sans difficultés.

```shell
ansible-playbook -i inventories/setup.yml playbook.yml
```

## Application front

Après avoir récupéré le projet frontend, il faut ajouter un rôle front. Ce rôle utilisera évidemment l'image Docker présente dans le projet frontend et préalablement compilée et poussée sur Docker Hub.

```yaml
# ansible/roles/front/main.yml

- name: Create the frontend container
  docker_container:
    name: front
    image: lilianandres/tp-devops-front:latest
    networks:
      - name: "{{ NETWORK_NAME }}"
    ports:
      - "80:80" # on mappe le port 80 de l'hôte avec le port 80 du conteneur
```

Il ne faut pas oublier d'ajouter ce rôle dans la liste des rôles à jouer dans le playbook.

## Déploiement continu

Afin de gérer le déploiement continu de notre application, j'ai ajouté un nouveau pipeline GitHub Actions. Ce pipeline est executé automatiquement lorsque le pipeline précédent (celui qui compile et pousse les images Docker) se termine et se termine en succès. J'ai également ajouté ma clé SSH Ansible ainsi que le nom de l'user Ansible dans les secrets GitHub.

```yaml
name: Ansible
on:
  workflow_run:
    workflows: [CD devops 2024]
    branches: [master]
    types:
      - completed

jobs:
  ansible-deployment:
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      - name: Run Ansible playbook
        uses: dawidd6/action-ansible-playbook@v2
        with:
          # Required, playbook filepath
          playbook: playbook.yml
          # Optional, directory where playbooks live
          directory: ./ansible/
          # Optional, SSH private key
          key: ${{secrets.ANSIBLE_SSH_PRIVATE_KEY}}
          # Optional, literal inventory file contents
          inventory: |
            [all]
            lilian.andres.takima.cloud
          options: |
            -u${{secrets.ANSIBLE_USER}}
```

## Ansible Vault

Ansible Vault nous permet de chiffrer certaines variables utilisées par Ansible afin de les sécuriser. Afin de mettre en place un Vault, j'ai d'abord lancé la commande suivante en spécifiant un mot de passe pour mon Vault :

```shell
ansible-vault create ansible-vault/vars/vault.yml
```

Esnuite, j'ai édité ce fichier avec la commande suivante...

```shell
ansible-vault edit ansible-vault/vars/vault.yml
```

...afin d'y placer mes variables sensibles au format suivant :

```yaml
# ansible-vault/vars/vault.yml

VAULT_VARIABLE: "myvar"
```

Ces variables sont ensuite utilisables de manière globale dans les différents rôles de mon projet (exemple ci-dessous).

```yml
---
# vars file for roles/app

BACKEND_HOST: "{{ VAULT_BACKEND_HOST }}"
NETWORK_NAME: "{{ VAULT_NETWORK_NAME }}"

DATABASE_HOST: "{{ VAULT_DATABASE_HOST }}"
DATABASE_NAME: "{{ VAULT_DATABASE_NAME }}"
DATABASE_USER: "{{ VAULT_DATABASE_USER }}"
DATABASE_PASSWORD: "{{ VAULT_DATABASE_PASSWORD }}"
```

```yml
# ansible/roles/app/tasks/main.yml

- name: Create the app container
  docker_container:
    name: "{{ BACKEND_HOST }}"
    image: lilianandres/tp-devops-simple-api:latest
    networks:
      - name: "{{ NETWORK_NAME }}"
    env:
      DATABASE_HOST: "{{ DATABASE_HOST }}"
      DATABASE_NAME: "{{ DATABASE_NAME }}"
      DATABASE_USER: "{{ DATABASE_USER }}"
      DATABASE_PASSWORD: "{{ DATABASE_PASSWORD }}"
```
