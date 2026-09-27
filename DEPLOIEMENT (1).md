# Civic Connect — Documentation de déploiement

Ce document explique comment lancer l'application Civic Connect (backend + frontend) avec Docker, sur n'importe quelle machine.

---

## 1. Architecture

```
Civic Connect
├── Backend  (CIVIC-CONNECT)      → FastAPI + SQLite
└── Frontend (civic-connect-frontend) → React + Vite, servi par Nginx
```

Les deux projets sont dans des dépôts GitHub distincts, et doivent être placés **côte à côte** dans un même dossier parent, avec le fichier `docker-compose.yml` à ce même niveau :

```
Documents/
├── docker-compose.yml
├── CIVIC-CONNECT/              (backend)
└── civic-connect-frontend/     (frontend)
```

---

## 2. Prérequis

- **Docker Desktop** installé et démarré (avec WSL2 activé sous Windows)
- Vérifier que Docker fonctionne :
  ```bash
  docker run hello-world
  ```

---

## 3. Configuration des variables d'environnement

Le backend a besoin d'un fichier `.env`, placé à la racine de `CIVIC-CONNECT` (jamais commité sur GitHub — il est listé dans `.gitignore`).

### Créer le fichier `.env`

1. Générer une clé secrète robuste :
   ```bash
   python -c "import secrets; print(secrets.token_hex(32))"
   ```
2. Créer `CIVIC-CONNECT/.env` avec le contenu suivant :
   ```
   SECRET_KEY=<coller la clé générée ci-dessus>
   ```

La variable `DATABASE_URL` n'a pas besoin d'être définie manuellement : elle est déjà fixée dans `docker-compose.yml` et pointe vers le volume persistant (voir section 5).

---

## 4. Lancer l'application

Depuis le dossier parent (`Documents`, ou l'équivalent qui contient `docker-compose.yml` et les deux projets) :

```bash
docker compose up --build
```

- `--build` force la reconstruction des images — nécessaire au premier lancement, ou après une modification du code.
- Pour les lancements suivants sans changement de code, `docker compose up` suffit (plus rapide, réutilise les images déjà construites).

### Arrêter l'application

```bash
docker compose down
```

Ceci arrête et supprime les conteneurs, **sans toucher aux données** (grâce au volume, voir section 5).

---

## 5. Accès à l'application

| Service | URL | Description |
|---|---|---|
| Frontend | http://localhost:8080 | Interface utilisateur |
| Backend (API) | http://localhost:8000 | API FastAPI |
| Documentation API (Swagger) | http://localhost:8000/docs | Test interactif des endpoints |

---

## 6. Persistance des données (SQLite)

La base de données SQLite est stockée dans un **volume Docker**, relié au dossier local :

```
CIVIC-CONNECT/data/civic connect.db
```

Cela garantit que les données (utilisateurs, services, signalements...) **survivent** à un arrêt/relancement des conteneurs (`docker compose down` puis `up`), ou à une reconstruction des images.

⚠️ Si ce dossier `data/` est supprimé manuellement, la base repart de zéro au prochain lancement.

---

## 7. Résolution de problèmes courants

### Le build échoue avec une erreur `UNRESOLVED_IMPORT`
Vérifier la casse exacte des noms de fichiers (`.jsx`, `.css`) : le build Docker (Linux) est sensible à la casse, contrairement à Windows. Un fichier `connectivite.css` ne correspondra pas à un import `Connectivite.css`.

### `Conflict. The container name "..." is already in use`
Un ancien conteneur du même nom existe déjà (probablement arrêté). Le supprimer avec :
```bash
docker rm <nom_du_conteneur>
```

### `services.xxx must be a mapping`
Erreur d'indentation dans `docker-compose.yml` (le YAML est très sensible à l'indentation). Vérifier qu'aucun autre fichier `docker-compose.yml` ne traîne ailleurs dans l'arborescence du projet — Docker Compose peut charger le mauvais fichier si la commande est lancée depuis le mauvais dossier.

### Erreur 404 en rafraîchissant une page du frontend (ex: `/tableau-de-bord`)
Nginx cherche un fichier physique correspondant à l'URL. Il faut configurer `nginx.conf` pour rediriger toutes les routes inconnues vers `index.html`, afin que React Router gère la navigation côté client.

### Erreur 400 sur la connexion après un changement de base de données
Normal si l'emplacement de la base SQLite a changé (nouveau volume) : les anciens comptes ne sont plus dans la nouvelle base. Créer un nouveau compte pour repartir sur la base persistante.

---

## 8. État du chantier Docker

- [x] Dockerfile backend (FastAPI + SQLite)
- [x] Dockerfile frontend (Node + Nginx, multi-stage build)
- [x] `docker-compose.yml` (orchestration backend + frontend)
- [x] Volume Docker pour la persistance SQLite
- [x] Variables d'environnement : `DATABASE_URL`, `SECRET_KEY`
- [x] Configuration Nginx pour le routing React (SPA)
- [ ] Déploiement sur un environnement autre que local (cloud, serveur) — à venir
