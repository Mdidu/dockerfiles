# Java 24 · Spring Boot · PostgreSQL

Stack backend conteneurisée, compatible **Windows (WSL2)** et **macOS Apple Silicon (ARM64)**.

---

## Stack technique

| Couche          | Technologie                         |
|-----------------|-------------------------------------|
| Backend         | Java 24 · Spring Boot 3.4.x · Maven |
| Base de données | PostgreSQL 17                       |
| Dev tooling     | pgAdmin 4                           |

---

## Structure des fichiers

```
.
├── backend/
│   ├── Dockerfile.dev   # mvn spring-boot:run + debug JDWP
│   └── Dockerfile.prod  # multi-stage : Maven build → JRE
├── compose.dev.yml      # environnement de développement
├── compose.prod.yml     # environnement de production
└── .env.example         # template des variables d'environnement
```

---

## Prérequis

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (avec WSL2 activé sur Windows)

---

## Mise en route

### 1. Configurer les variables d'environnement

```bash
cp .env.example .env
```

Édite le fichier `.env` :

```env
POSTGRES_DB=myapp
POSTGRES_USER=myuser
POSTGRES_PASSWORD=change_me

PGADMIN_EMAIL=admin@admin.com
PGADMIN_PASSWORD=change_me
```

> **Ne commite jamais le fichier `.env`** dans ton dépôt Git.

### 2. Lancer l'environnement

**Développement**
```bash
docker compose -f compose.dev.yml up --build
```

**Production**
```bash
docker compose -f compose.prod.yml up --build
```

### 3. Accéder aux services

| Service         | URL                   | Environnement |
|-----------------|-----------------------|---------------|
| Spring Boot API | http://localhost:8080 | Dev + Prod    |
| pgAdmin         | http://localhost:5050 | Dev           |
| PostgreSQL      | localhost:5432        | Dev           |

---

## Fonctionnalités par environnement

### Développement (`compose.dev.yml`)

- **Hot-reload Spring** : Spring DevTools détecte les changements dans `backend/src/`.
- **Debug distant (JDWP)** : le port `5005` est exposé pour connecter IntelliJ IDEA ou VS Code en mode debug.
- **Cache Maven** : les dépendances sont stockées dans un volume nommé `maven_cache` pour ne pas être re-téléchargées à chaque rebuild.
- **pgAdmin** : interface web disponible sur `http://localhost:5050`.

#### Connexion à PostgreSQL depuis pgAdmin

Dans l'interface pgAdmin, crée un nouveau serveur avec ces paramètres :

| Champ    | Valeur                          |
|----------|---------------------------------|
| Host     | `db` *(nom du service Compose)* |
| Port     | `5432`                          |
| Database | valeur de `POSTGRES_DB`         |
| Username | valeur de `POSTGRES_USER`       |
| Password | valeur de `POSTGRES_PASSWORD`   |

### Production (`compose.prod.yml`)

- **Image multi-stage** : l'image finale ne contient que le JRE + le JAR (pas de JDK, pas de Maven).
- **PostgreSQL non exposé** : la base de données n'est pas accessible depuis l'extérieur du réseau Docker.
- **Restart policy** : les services redémarrent automatiquement en cas de crash.
- **JVM optimisée** : `-XX:+UseContainerSupport` et `-XX:MaxRAMPercentage=75.0` pour respecter les limites mémoire du conteneur.

---

## Adaptations requises à ton projet

### Spring DevTools dans `pom.xml`

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
  <optional>true</optional>
</dependency>
```

---

## Compatibilité multi-architecture

| Image                            | amd64 | arm64 |
|----------------------------------|:-----:|:-----:|
| `eclipse-temurin:24-jdk-jammy`   | ✓     | ✓     |
| `eclipse-temurin:24-jre-jammy`   | ✓     | ✓     |
| `postgres:17-alpine`             | ✓     | ✓     |
| `dpage/pgadmin4`                 | ✓     | ✓     |

---

## Commandes utiles

```bash
# Arrêter les conteneurs
docker compose -f compose.dev.yml down

# Arrêter et supprimer les volumes (repart d'une DB vide)
docker compose -f compose.dev.yml down -v

# Rebuild du backend seul
docker compose -f compose.dev.yml up --build backend

# Consulter les logs
docker compose -f compose.dev.yml logs -f backend

# Ouvrir un shell dans le conteneur
docker compose -f compose.dev.yml exec backend bash
```
