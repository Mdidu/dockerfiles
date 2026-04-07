# Angular 21 · Java 24 · Spring Boot · PostgreSQL

Stack full-stack conteneurisée, compatible **Windows (WSL2)** et **macOS Apple Silicon (ARM64)**.

---

## Stack technique

| Couche     | Technologie                        |
|------------|------------------------------------|
| Frontend   | Angular 21 · Node 22 · Nginx       |
| Backend    | Java 24 · Spring Boot 3.4.x · Maven|
| Base de données | PostgreSQL 17                 |
| Dev tooling | pgAdmin 4                         |

---

## Architecture : pourquoi plusieurs images ?

Ce projet utilise **3 conteneurs distincts** orchestrés via Docker Compose, conformément au principe Docker *un processus par conteneur*.

Une image unique (« fat container ») aurait posé ces problèmes :
- Rebuild du frontend = redémarrage de la base de données
- Impossible de scaler le backend indépendamment
- Image inutilement lourde (Node + JDK + PostgreSQL)
- Debug et maintenance complexes

```
┌─────────────────────────────────────────────────────────┐
│                      Docker Compose                      │
│                                                          │
│  ┌───────────┐     ┌───────────┐     ┌───────────────┐  │
│  │ frontend  │────▶│  backend  │────▶│      db       │  │
│  │ Angular   │     │  Spring   │     │  PostgreSQL   │  │
│  │ :4200/80  │     │  :8080    │     │  :5432        │  │
│  └───────────┘     └───────────┘     └───────────────┘  │
│                                                          │
│  ┌───────────┐  (dev uniquement)                        │
│  │  pgAdmin  │────▶ db                                  │
│  │  :5050    │                                          │
│  └───────────┘                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Structure des fichiers

```
.
├── frontend/
│   ├── Dockerfile.dev   # ng serve avec hot-reload
│   ├── Dockerfile.prod  # multi-stage : Node build → Nginx
│   └── nginx.conf       # routing Angular + proxy /api/
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
- WSL2 sur Windows, ou macOS avec puce Apple Silicon

---

## Mise en route

### 1. Configurer les variables d'environnement

```bash
cp .env.example .env
```

Édite le fichier `.env` et remplace les valeurs par défaut :

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

| Service         | URL                          | Environnement |
|-----------------|------------------------------|---------------|
| Angular         | http://localhost:4200        | Dev           |
| Application     | http://localhost             | Prod          |
| Spring Boot API | http://localhost:8080        | Dev           |
| pgAdmin         | http://localhost:5050        | Dev           |
| PostgreSQL      | localhost:5432               | Dev           |

> Sur **Windows**, ces URLs fonctionnent directement depuis le navigateur Windows même si Docker tourne dans WSL2 — les ports sont automatiquement exposés sur `localhost`.

---

## Fonctionnalités par environnement

### Développement (`compose.dev.yml`)

- **Hot-reload Angular** : les modifications dans `frontend/src/` sont répercutées instantanément. Le flag `--poll` assure la compatibilité du file-watcher sous WSL2.
- **Hot-reload Spring** : Spring DevTools détecte les changements dans `backend/src/`.
- **Debug distant (JDWP)** : le port `5005` est exposé pour connecter IntelliJ IDEA ou VS Code en mode debug.
- **Cache Maven** : les dépendances Maven sont stockées dans un volume nommé `maven_cache` pour ne pas être re-téléchargées à chaque rebuild.
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

- **Images multi-stage** : les images finales ne contiennent que le strict nécessaire (pas de Node, pas de JDK, pas de Maven).
- **PostgreSQL non exposé** : la base de données n'est pas accessible depuis l'extérieur du réseau Docker.
- **Nginx** sert le build Angular et proxifie les appels `/api/` vers Spring Boot.
- **Restart policies** : les services redémarrent automatiquement en cas de crash.
- **JVM optimisée** : `-XX:+UseContainerSupport` et `-XX:MaxRAMPercentage=75.0` pour respecter les limites mémoire du conteneur.

---

## Adaptations requises à ton projet

### 1. Chemin du build Angular

Dans `frontend/Dockerfile.prod`, adapte le chemin de sortie au nom de ton projet Angular :

```dockerfile
# Remplace "app" par le nom de ton projet (défini dans angular.json)
COPY --from=builder /app/dist/app/browser /usr/share/nginx/html
```

### 2. Proxy Angular en développement

Pour que `ng serve` transmette les appels API au backend, crée `frontend/proxy.conf.json` :

```json
{
  "/api": {
    "target": "http://backend:8080",
    "secure": false
  }
}
```

Puis référence-le dans `angular.json` :

```json
"serve": {
  "options": {
    "proxyConfig": "proxy.conf.json"
  }
}
```

### 3. Spring DevTools dans `pom.xml`

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

Toutes les images de base utilisées disposent de variantes **`linux/amd64`** (WSL2) et **`linux/arm64`** (Apple Silicon) :

| Image                        | amd64 | arm64 |
|------------------------------|:-----:|:-----:|
| `node:22-alpine`             | ✓     | ✓     |
| `nginx:alpine`               | ✓     | ✓     |
| `eclipse-temurin:24-jdk-jammy` | ✓   | ✓     |
| `eclipse-temurin:24-jre-jammy` | ✓   | ✓     |
| `postgres:17-alpine`         | ✓     | ✓     |
| `dpage/pgadmin4`             | ✓     | ✓     |

---

## Commandes utiles

```bash
# Arrêter les conteneurs
docker compose -f compose.dev.yml down

# Arrêter et supprimer les volumes (repart d'une DB vide)
docker compose -f compose.dev.yml down -v

# Rebuild d'un seul service
docker compose -f compose.dev.yml up --build backend

# Consulter les logs d'un service
docker compose -f compose.dev.yml logs -f backend

# Ouvrir un shell dans un conteneur
docker compose -f compose.dev.yml exec backend bash
```
