Regroupe tous les TD & TP de DevOps

# TP1 : Compte rendu 

## Docker Postgres avec Adminer: 
    docker pull adminer -> interface web pour accéder aux données (opt.)
    docker network create app-network -> créer un réseau pour autoiriser la communication entre les process
    docker run -d --name DevOpsTP1 --network app-network KilianVLT/postgres -> container postgres
    docker run --network app-network -p 8080:8080 --name adminer -d adminer -> run sur le réseau

## 1-1 Why should we run the container with a flag -e to give the environment variables?
    Pour empêcher de mettre en clair dans un fichier des informations comme des mots de passe (exemple password de postgres)

## 1-2 Why do we need a volume to be attached to our postgres container?
    Pour conserver les données existantes en cas de changement dans le container Postgres

## 1-3 Document your database container essentials: commands and Dockerfile.
    docker pull adminer -> interface web pour accéder aux données (opt.)
    docker network create app-network -> créer un réseau pour autoiriser la communication entre les process
    docker run -d --name DevOpsTP1 --network app-network KilianVLT/postgres -> container postgres
    docker run --network app-network -p 8080:8080 -v D:/Documents/DevOps-S8/TP1:/var/lib/postgresql/data --name adminer -d adminer -> run sur le réseau

## 1-4 Why do we need a multistage build? And explain each step of this dockerfile.
    Parce qu'on a deux processus à build pour notre projet, un sur Maven et l'autre avec AmazonCorretto
    
    Dans le 1er on construit notre image Maven, On défini une variable d'env pour la source du projet. Avec Workdir, le Dockerfile détecte la zone de départ du projet
    On copie pom.xml à la racine de l'image
    On copie tout le répertoire src dans le dossier /src de l'image 
    On run les packages Maven

    Dans le 2ème on utilise une image AmazonCorretto (it’s designed to provide everything you need to create and run Java applications without limits)
    On redéclare la même variable d'env que dans le premier process
    On se fixe dessus avec Workdir

    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
    On copie depuis le premier process tous les contenus en .jar sur le chemin $MYAPP_HOME/target/*.jar vers $MYAPP_HOME/myapp.jar

    ENTRYPOINT ["java", "-jar", "myapp.jar"] -> démarre l'appli 

    Note: Dans le multistaging, seul ce qu'il y a après le dernier FROM est conservé pour la création de l'image final
    Dans le DockerFile de l'API, les lignes 9 à 13 gardent seulement les fichiers .jar (jdk et jre) qui sont des fichiers compilés et beaucoup plus léger
    et qui ne garde que l'essentiel pour le projet.  

## 1-5 Why do we need a reverse proxy?

    Cache les adresses IP côté client, permet de faire du load-balancing, 
    Protège un serveur Web des attaques provenant de l'extérieur
    Permet de créer un point unique pour les accès aux ressources Web 

## 1-6 Why is docker-compose so important?

    simplifie enormément un déploiement lorsqu'il y a plusieurs containers
    réduit les erreurs humaines
    regroupe toutes les dépendances

## 1-7  Document docker-compose most important commands.
    docker compose up -d pour 

# TP2 : Compte rendu

## 2-1 What are testcontainers?
    C'est une bibliothèque OPS qui permet d'executer des tests facilement avec des bdd simplifiées et des environnements dédiés.
    java libraries that allow you to run a bunch of docker containers while testing

## 2-3
    Rapidité de déploiment  
    Travail collaboratif

## Pour la partie Bonus: 
    On a deux fichiers, le premier "main.yml" gère le premier job au moment du push depuis la branche main ou staging
    Le deuxième se déclenche a la fin du premier workflow et ne s'exécute pas si le push vient initialement de la branche staging (branch_ignore: ...)
    dans le job du fichier second.yml j'ai également ajouté la condition "if: ${{ github.ref == 'refs/heads/main' }}" pour être sûr qu'il s'agit de la branche main
    Résultat: on peut observer que le premier workflow se déroule dans un premier temps et après le deuxième

# TP3: Compte rendu:

Extrait des fichiers ansible concernés

## PlayBook

- hosts: all
  gather_facts: true
  become: true
  roles:
    - docker
    - network
    - database
    - app
    - front
    - proxy

Le playbook va exécuter toutes les actions contenus dans les fichiers main.yml des roles cités et dans l'ordre indiqué
hosts: all signifie que les actions seront effectués sur tous les hôtes inscrits dans inventories/setup.yml
become: execute en tant que root

## Role Docker 

---
### tasks file for roles/docker
### Install prerequisites for Docker
- name: Install required packages
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - python3-venv
    state: latest
    update_cache: yes


### Add Docker’s official GPG key
- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/debian/gpg
    state: present


### Set up the Docker stable repository
- name: Add Docker APT repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
    state: present
    update_cache: yes


### Install Docker
- name: Install Docker
  apt:
    name: docker-ce
    state: present


### Install Python3 and pip3
- name: Install Python3 and pip3
  apt:
    name:
      - python3
      - python3-pip
    state: present


### Create a virtual environment for Python packages
- name: Create a virtual environment for Docker SDK
  command: python3 -m venv /opt/docker_venv
  args:
    creates: /opt/docker_venv  # Only runs if this directory doesn’t exist


### Install Docker SDK for Python in the virtual environment
- name: Install Docker SDK for Python in virtual environment
  command: /opt/docker_venv/bin/pip install docker


### Ensure Docker is running
- name: Make sure Docker is running
  service:
    name: docker
    state: started
  tags: docker


Voici toutes les étapes du rôle Docker qui est cité dans le playbook.
Chaque tâche commence par un -name et sont exécutées à la suite. Ce rôle ne s'occupe que de la mise en place des dockers sur les hôtes concernés

### Network

### Création des deux réseaux
- name: Create the front network
  community.docker.docker_network:
    name: front-network

- name: Create the back network
  community.docker.docker_network:
    name: back-network

Le role network est chargé de créer deux réseaux pour les containers docker à venir (à la manière du docker-compose dans le TP2)


# Database 
---
### tasks file for roles/database
- name: Create Volume
  community.docker.docker_volume:
    name: db-volume

- name: Run Database
  docker_container:
    name: db
    image: billlameduse/tp-devops-postgres:latest
    pull: yes
    networks:
      - name: back-network
    volumes:
      - db-volume:/var/lib/postgresql/data
    env_file: /opt/environments/dbenv/.env

Le role database créé un volume pour le container de base de données et run l'image de la database stockée sur dockerhub. on lui attribue le réseau back-network et le fichier d'environnement est déjà contenu sur la machine hôte au chemin indiqué

# Le reste des rôles 

Le reste des rôles c'est le même principe que le rôle database pour l'api, le front et le httpd proxy. Hormis qu'on attribue pas de volume car on a pas de données à stocker