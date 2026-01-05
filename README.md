# Lab Observable & Résilient - Microservices

Un lab complet pour apprendre les concepts modernes de microservices : Actuator, Healthcheck Docker, Profiles Spring, Résilience avec Resilience4j, et déploiement multi-instances.

## Structure du projet

```
.
├── pricing-service/        # Service de pricing (port 8082)
├── book-service/           # Service de gestion de livres (port 8081, 8083, 8084)
├── docker-compose.yml      # Orchestration : MySQL + pricing + 3 instances book-service
└── README.md
```

## Prérequis

- JDK 21 (ou 17)
- Maven 3.9+
- Docker + Docker Compose v2
- curl (pour les tests)

## Étape 1 : Test local (sans Docker) - Profil dev avec H2

### Terminal 1 : Démarrer pricing-service

```bash
cd pricing-service
mvn spring-boot:run
```

Vérification : `curl http://localhost:8082/actuator/health`

### Terminal 2 : Démarrer book-service en profil dev (H2)

```bash
cd book-service
mvn spring-boot:run -Dspring.profiles.active=dev
```

Ou via variable d'environnement (PowerShell) :
```powershell
$env:SPRING_PROFILES_ACTIVE="dev"
mvn spring-boot:run
```

Vérification : `curl http://localhost:8081/actuator/health`

### Checkpoints locaux

**Créer un livre :**
```bash
curl -X POST http://localhost:8081/api/books `
  -H "Content-Type: application/json" `
  -d '{"title":"Dune","author":"Herbert","stock":3}'
```

**Lister les livres :**
```bash
curl http://localhost:8081/api/books
```

**Emprunter un livre :**
```bash
curl -X POST http://localhost:8081/api/books/1/borrow
```

**Instance debug :**
```bash
curl http://localhost:8081/api/debug/instance
```

---

## Étape 2 : Déploiement Docker Compose (multi-instances + MySQL)

À la racine du projet :

```bash
docker compose up -d --build
```

### Checkpoints Docker

**Santé des services :**
```bash
curl http://localhost:8082/actuator/health
curl http://localhost:8081/actuator/health
curl http://localhost:8083/actuator/health
curl http://localhost:8084/actuator/health
```

**Multi-instances (3 hostname différents) :**
```bash
curl http://localhost:8081/api/debug/instance
curl http://localhost:8083/api/debug/instance
curl http://localhost:8084/api/debug/instance
```

**Données partagées (même MySQL) :**
```bash
# Créer via instance 1
curl -X POST http://localhost:8081/api/books `
  -H "Content-Type: application/json" `
  -d '{"title":"Fondation","author":"Asimov","stock":5}'

# Lire via instance 2 et 3
curl http://localhost:8083/api/books
curl http://localhost:8084/api/books
```

### Scénarios d'apprentissage

#### Scenario 1 : Stock cohérent (verrou DB)

Emprunter 4 fois un livre avec stock=3 (3 instances + 1 de plus)

```bash
# Les 3 premiers réussissent, le 4e échoue avec HTTP 409
curl -X POST http://localhost:8081/api/books/1/borrow
curl -X POST http://localhost:8083/api/books/1/borrow
curl -X POST http://localhost:8084/api/books/1/borrow
curl -X POST http://localhost:8083/api/books/1/borrow  # doit échouer
```

#### Scenario 2 : Résilience - Pricing en panne

```bash
# Arrêter pricing-service
docker compose stop pricing-service

# Essayer d'emprunter → doit réussir avec price=0.0 (fallback)
curl -X POST http://localhost:8081/api/books/1/borrow

# Relancer pricing-service
docker compose start pricing-service

# Emprunter à nouveau → doit retourner le vrai prix
curl -X POST http://localhost:8081/api/books/1/borrow
```

#### Scenario 3 : Pricing avec panne forcée

```bash
# Pricing répond en erreur avec le paramètre fail=true
curl http://localhost:8082/api/prices/1?fail=true
# → Erreur 500 (normal pour pricing)

# Mais book-service gère ça avec retry + fallback
curl -X POST http://localhost:8081/api/books/1/borrow
# → Réponse OK avec price=0.0
```

#### Scenario 4 : Persistance MySQL (volume)

```bash
# Arrêter tout
docker compose down

# Relancer
docker compose up -d

# Vérifier que le livre existe toujours
curl http://localhost:8081/api/books
```

### Logs en temps réel

```bash
docker compose logs -f book-service-1
docker compose logs -f pricing-service
docker compose logs -f mysql
```

### Nettoyer

```bash
# Arrêter tout
docker compose down

# Supprimer aussi les volumes MySQL
docker compose down -v
```

---

## Concepts clés appris

### 1. **Actuator** (`Spring Boot Actuator`)
- `/actuator/health` : état de santé du service
- `/actuator/health/readiness` : prêt à recevoir du trafic ?
- `/actuator/health/liveness` : toujours vivant ?
- Utilisé par Docker Compose pour les healthchecks

### 2. **Healthcheck Docker**
```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -fsS http://localhost:8082/actuator/health || exit 1"]
  interval: 5s
  timeout: 3s
  retries: 20
```
Docker attend que le service soit `healthy` avant de lancer les dépendances.

### 3. **Profiles Spring** (`application-{profile}.yml`)
- `dev` : H2 in-memory, localhost
- `docker` : MySQL via DNS Docker, pricing-service via DNS

### 4. **MySQL Volume** (persistance)
```yaml
volumes:
  mysql_data:/var/lib/mysql
```
Les données survivent à `docker compose down`

### 5. **Wait Strategy** (`depends_on` avec `condition: service_healthy`)
Garantit que book-service démarre **après** que MySQL soit prêt.

### 6. **Résilience** (`Resilience4j`)
- **Retry** : retente 3 fois en cas d'échec
- **Circuit Breaker** : coupe l'appel après trop d'échecs pour 10s
- **Fallback** : renvoie une valeur par défaut si tout échoue
- Empêche une panne upstream (pricing) de casser book-service

### 7. **Multi-instances et Concurrence**
- `synchronized` ne protège qu'une JVM
- **Pessimistic Locking** (`@Lock(PESSIMISTIC_WRITE)`) : verrou DB
  - Quand une JVM emprunte un livre, les autres attendent
  - Garantit la cohérence du stock

---

## Fichiers importants

| Fichier | Rôle |
|---------|------|
| `pricing-service/src/main/java/.../PricingController.java` | API pricing avec simulation de panne |
| `book-service/src/main/java/.../BookService.java` | Logique métier (emprunts, transactions) |
| `book-service/src/main/java/.../PricingClient.java` | Client avec Resilience4j |
| `book-service/src/main/resources/application-dev.yml` | Config H2 pour dev |
| `book-service/src/main/resources/application-docker.yml` | Config MySQL pour Docker |
| `docker-compose.yml` | Orchestration |

---

## Troubleshooting

### `Connection refused` au lancer book-service dans Docker
→ Vérifier que `application-docker.yml` utilise `mysql:3306`, pas `localhost:3306`

### book-service démarre avant MySQL prêt
→ Vérifier le `depends_on` avec `condition: service_healthy`

### Fallback non appelé
→ Vérifier la signature de `fallbackPrice(long bookId, Throwable ex)` dans `PricingClient`

### Volume MySQL ne persiste pas
→ Vérifier que le volume est déclaré et monte correctement

---

## Variantes et approfondissements

- **Optimistic Locking** : remplacer `@Lock(PESSIMISTIC_WRITE)` par `@Version` (plus scalable)
- **Flyway/Liquibase** : remplacer `ddl-auto=update` pour la migration en production
- **Spring Cloud Config** : gérer les profils centralement
- **Observability** : ajouter Prometheus + Grafana, Jaeger pour la tracing distribuée
- **API Gateway** : ajouter Spring Cloud Gateway pour router vers les 3 instances

---



