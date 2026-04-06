# Dockerfiles

Collection de configurations Docker prêtes à l'emploi.  
Chaque dossier est un projet autonome avec ses propres `Dockerfile` et `compose.yml`.

---

## Projets disponibles

### [angular21-java24-spring-postgres](./angular21-java24-spring-postgres/)

Stack full-stack conteneurisée avec :

- **Angular 21** (frontend) — dev avec hot-reload, prod servie via Nginx
- **Java 24 + Spring Boot 3.4.x** (backend) — hot-reload via Spring DevTools, debug JDWP
- **PostgreSQL 17** (base de données)
- **pgAdmin 4** (interface DB, environnement de développement uniquement)

Compatible **Windows via WSL2** et **macOS Apple Silicon (ARM64)**.

---

## Compatibilité générale

Tous les projets de ce dépôt ciblent une compatibilité :

- **Windows** via WSL2 + Docker Desktop
- **macOS** Apple Silicon (ARM64) + Docker Desktop

---

## Conventions

| Fichier              | Rôle                                      |
|----------------------|-------------------------------------------|
| `Dockerfile.dev`     | Image de développement (hot-reload, debug)|
| `Dockerfile.prod`    | Image de production (multi-stage, optimisée)|
| `compose.dev.yml`    | Orchestration en développement            |
| `compose.prod.yml`   | Orchestration en production               |
| `.env.example`       | Template des variables d'environnement    |
| `README.md`          | Documentation du projet                   |
