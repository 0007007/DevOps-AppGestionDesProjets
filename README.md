# spring-boot-devops — Gestion des Projets

Application **Spring Boot** (API REST de gestion d'entreprises, d'équipes et de projets) avec un pipeline **Jenkins CI/CD** complet : build et tests Maven, création d'une image Docker, publication dans un registre Docker privé, puis déploiement automatique avec MySQL.

---

## Structure du dépôt

```
spring-boot-devops/
├── src/
│   ├── main/
│   │   ├── java/tn/esprit/backend/   # entity, repository, service, controller
│   │   └── resources/application.properties
│   └── test/
│       ├── java/tn/esprit/backend/BackendApplicationTests.java
│       └── resources/application.properties   # base H2 en mémoire pour les tests
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── README.md
```

---

## Stack technique

| Élément | Technologie |
|---|---|
| Backend | Spring Boot 4.1, Spring Data JPA, Spring Web MVC |
| Java | 17 (exécuté sur l'image `alpine/java:22-jdk`) |
| Base de données | MySQL (image `mysql:latest`, H2 en mémoire pour les tests) |
| Build | Maven |
| CI/CD | Jenkins (pipeline déclaratif) |
| Conteneurisation | Docker |
| Registre | Docker Hub (dépôt privé `hamzanajj/backend-app`) |
| Environnement | VM Ubuntu 22.04 (Vagrant + VirtualBox), IP `192.168.33.10` |

---

## Pipeline CI/CD

```
Git
 ↓
Jenkins
 ↓
Build & Test Maven
 ↓
Docker Build
 ↓
Docker Push
 ↓
Déploiement MySQL
 ↓
Déploiement backend-app
 ↓
Vérification avec logs
```

| Étape | Description | Commandes principales |
|---|---|---|
| **Checkout Git** | Récupère le code depuis GitHub | `checkout scm` |
| **Build & Test Maven** | Exécute les tests unitaires (H2) puis génère le `.jar` | `mvn clean test` · `mvn package -DskipTests` |
| **Docker Build** | Construit l'image à partir du `Dockerfile` | `docker build -t backend-app:latest .` |
| **Docker Push** | S'authentifie avec les Credentials Jenkins et publie l'image dans le registre privé | `docker login` · `docker tag` · `docker push` |
| **Deploy MySQL** | Crée le réseau Docker et lance (ou redémarre) le conteneur `mysql` | `docker network create` · `docker run --name mysql` |
| **Deploy backend-app** | Supprime l'ancien conteneur, récupère l'image publiée et lance le nouveau conteneur | `docker stop` · `docker rm` · `docker pull` · `docker run --name backend-app` |
| **Vérification** | Vérifie que les conteneurs tournent et que le backend a démarré | `docker ps` · `docker logs backend-app` |

---

## Dockerfile

```dockerfile
FROM alpine/java:22-jdk
WORKDIR /app
COPY target/backend-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Le `.jar` est produit par Maven à l'étape précédente, puis copié dans l'image.

---

## Configuration Jenkins

### Prérequis sur la VM
- Java 21, Maven, Git, Docker
- L'utilisateur `jenkins` doit appartenir au groupe `docker` :
  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart jenkins
  ```

### Credentials
| ID | Type | Usage |
|---|---|---|
| `dockerhub-cred` | Username with password (token Docker Hub) | Authentification au registre Docker privé |

### Job
- Type : **Pipeline**
- Definition : **Pipeline script from SCM** (Git, branche `*/main`)
- Script Path : `Jenkinsfile`
- Déclencheur : Scrutation de l'outil de gestion de version (`H/5 * * * *`)

---

## Déploiement

Les deux conteneurs sont reliés par le réseau Docker `devops-net`, ce qui permet au backend de joindre MySQL par son nom (`mysql`).

```bash
# Réseau
docker network create devops-net

# MySQL
docker run -d --name mysql --network devops-net -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=test_db \
  --restart always mysql:latest

# Backend
docker run -d --name backend-app --network devops-net -p 8089:8080 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true" \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=root \
  hamzanajj/backend-app:latest
```

> Le port **8080** de la VM étant utilisé par Jenkins, le backend est exposé sur le port **8089**.

---

## Vérification

```bash
docker ps                   # les conteneurs mysql et backend-app sont "Up"
docker logs backend-app     # doit contenir "Started BackendApplication"
```

API accessible sur : **http://192.168.33.10:8089/entreprise/all**

---

## Endpoints REST

| Ressource | Endpoints |
|---|---|
| Entreprise | `GET /entreprise/all` · `GET /entreprise/get/{id}` · `POST /entreprise/add` · `PUT /entreprise/update` · `DELETE /entreprise/delete/{id}` |
| Équipe | `GET /equipe/all` · `GET /equipe/get/{id}` · `POST /equipe/add` · `PUT /equipe/update` · `DELETE /equipe/delete/{id}` · `PUT /equipe/assign-entreprise/{equipeId}/{entrepriseId}` · `PUT /equipe/assign-projet/{equipeId}/{projetId}` |
| Projet | `GET /projet/all` · `GET /projet/get/{id}` · `POST /projet/add` · `PUT /projet/update` · `DELETE /projet/delete/{id}` |
| Projet détaillé | `GET /projet-detaille/all` · `GET /projet-detaille/get/{id}` · `POST /projet-detaille/add` · `PUT /projet-detaille/update` · `DELETE /projet-detaille/delete/{id}` · `PUT /projet-detaille/assign-projet/{pdId}/{projetId}` |

---

## Auteur

**Hamza Najjar** — ESPRIT, UP ASI — Module DevOps
