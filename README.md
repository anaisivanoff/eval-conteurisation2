# TP2 – Docker Compose (API Node.js, PostgreSQL, Frontend, Adminer)

## 1. Objectif

Mettre en place un environnement complet avec Docker Compose pour une application composée de :
- une API Node.js qui expose une API REST pour gérer des messages,
- une base PostgreSQL pour stocker les messages,
- un frontend statique (HTML + JS) qui consomme l’API,
- Adminer pour consulter la base en graphique.

---

## 2. Structure du projet

Répertoire `tp2` :

```text
tp2/
├── SujetTP2/
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── index.js
│   │   └── package.json
│   └── frontend/
│       ├── Dockerfile
│       └── index.html
├── .env
└── docker-compose.yml
```

Fichier `.env` :

```env
DB_USER=tp2user
DB_PASSWORD=change-moi
```

---

## 3. Contenu du `docker-compose.yml`

```yaml
services:
  database:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: tp2db
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data

  api:
    build: ./SujetTP2/api
    environment:
      DB_HOST: database
      DB_PORT: 5432
      DB_NAME: tp2db
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      PORT: 3000
    depends_on:
      - database
    restart: on-failure
    ports:
      - "3000:3000"

  frontend:
    build: ./SujetTP2/frontend
    depends_on:
      - api
    ports:
      - "8080:80"

  adminer:
    image: adminer:latest
    environment:
      ADMINER_DEFAULT_SERVER: database
    depends_on:
      - database
    ports:
      - "8081:8080"

volumes:
  db_data:
```

Points importants :
- Les secrets (`DB_USER`, `DB_PASSWORD`) viennent de `.env` et ne sont pas écrits en dur.
- L’API parle à Postgres via le nom de service `database` sur le réseau interne Docker.
- La base est persistée dans le volume `db_data`.

---

## 4. API Node.js (`SujetTP2/api/index.js`)

L’API utilise `pg` et les variables d’environnement pour se connecter à PostgreSQL, crée la table `messages` si besoin, puis expose trois routes.

### Connexion et table

- Connexion avec : `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.
- Table créée si elle n’existe pas :

```sql
CREATE TABLE IF NOT EXISTS messages (
  id SERIAL PRIMARY KEY,
  texte TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Routes exposées

- `GET /messages`  
  Retourne la liste des messages triés par `created_at DESC` au format JSON.

- `POST /messages`  
  Corps JSON : `{ "texte": "Mon message" }`.  
  Insère en base et renvoie le message créé (JSON).

- `GET /health`  
  Retourne `{ "status": "ok" }` pour vérifier que l’API est en vie.

Les réponses sont en JSON, avec les en-têtes CORS appropriés (`Access-Control-Allow-Origin: *`).

---

## 5. Frontend (`SujetTP2/frontend/index.html`)

Le frontend est une page HTML statique servie par Nginx qui :
- affiche un formulaire pour saisir un message,
- appelle l’API pour lister et ajouter des messages.

Principes :
- `API_URL` = `window.API_URL` (si injecté par Nginx) ou `http://localhost:3000` par défaut.
- Au chargement, le JS appelle `GET /messages` pour remplir la liste.
- À la soumission du formulaire, le JS envoie `POST /messages` avec un JSON `{ texte }`, puis recharge la liste.

---

## 6. Commandes pour lancer et tester

### Lancement

À la racine de `tp2` :

```bash
# 1. Build + lancement de tous les services
docker compose up --build

# 2. Lancement en arrière-plan (après le premier build)
docker compose up -d
```

### Vérifier l’état des services

```bash
docker compose ps
```

Attendu :

```text
NAME             SERVICE    STATUS     PORTS
tp2-database-1   database   Up        5432/tcp
tp2-api-1        api        Up        0.0.0.0:3000->3000/tcp
tp2-frontend-1   frontend   Up        0.0.0.0:8080->80/tcp
tp2-adminer-1    adminer    Up        0.0.0.0:8081->8080/tcp
```

### Tests API

```bash
# Healthcheck de l'API
curl http://localhost:3000/health

# Récupérer la liste des messages
curl http://localhost:3000/messages

# Ajouter un message
curl -X POST http://localhost:3000/messages \
  -H "Content-Type: application/json" \
  -d '{"texte":"Bonjour TP2"}'
```

### Tests frontend

Dans le navigateur :

- Frontend : `http://localhost:8080/`  
  - Le bandeau doit passer à “API connectée ✓” quand l’API répond.
  - Les messages existants s’affichent, un nouveau message soumis apparaît dans la liste.

### Tests Adminer

Dans le navigateur :

- Adminer : `http://localhost:8081/`  

Connexion :

- Système : PostgreSQL  
- Serveur : `database`  
- Utilisateur : `tp2user`  
- Mot de passe : `change-moi`  
- Base : `tp2db`  

On doit voir la table `messages` contenant les messages envoyés depuis le frontend ou l’API.

### Persistance des données

1. Ajouter quelques messages depuis le frontend.
2. Arrêter la stack :

   ```bash
   docker compose down
   ```

3. Relancer en arrière-plan :

   ```bash
   docker compose up -d
   ```

4. Retourner sur `http://localhost:8080/` ou `http://localhost:3000/messages` : les messages sont toujours là grâce au volume `db_data`.

---

## 7. Arrêt des services

Pour tout arrêter et supprimer les conteneurs (volume conservé) :

```bash
docker compose down
```
