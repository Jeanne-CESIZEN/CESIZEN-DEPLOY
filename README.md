# Déploiement CESIZen — VM Azure Linux (staging + prod)

Une **seule VM** héberge **deux environnements isolés** derrière des services
**partagés** (HTTPS Let's Encrypt). Images tirées depuis GHCR, déploiement continu
via **Watchtower** (unique et partagé, met à jour chaque conteneur selon son tag).

```
                       ┌──────────────── VM Azure ─────────────────┐
                       │  PARTAGÉ : Traefik (80/443) + Watchtower   │
Internet ──▶ Traefik ──┤                                           │
 (80/443)   (partagé)  │  stack cesizen-staging    stack cesizen-prod
                       │   front/api/postgres       front/api/postgres
                       └───────────────────────────────────────────┘
```

## Structure du dossier

```
deploy/
├── proxy/      → Traefik + Watchtower (partagés)   → docker compose up -d
├── staging/    → postgres + api:staging + front:staging
└── prod/       → postgres + api:prod + front:prod
```

| Branche Git | Tag image GHCR | Environnement | Domaines |
|-------------|----------------|---------------|----------|
| `develop`   | `:staging`     | staging       | `staging.DOMAIN` / `api.staging.DOMAIN` |
| `main`      | `:prod`        | prod          | `DOMAIN` / `api.DOMAIN` |

## 1. Pré-requis sur la VM

- VM Azure Linux (Ubuntu LTS), Docker + plugin `compose`
- DNS pointant vers l'IP publique de la VM, 4 enregistrements A :
  `DOMAIN`, `api.DOMAIN`, `staging.DOMAIN`, `api.staging.DOMAIN`

## 2. Sécurisation du serveur (à présenter dans le dossier)

```bash
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# SSH par clé uniquement (1 clé/personne) : /etc/ssh/sshd_config -> PasswordAuthentication no
sudo systemctl restart ssh

sudo apt install -y fail2ban && sudo systemctl enable --now fail2ban
```

> PostgreSQL (5432) n'est jamais exposé : chaque base reste sur le réseau Docker
> `internal` de son stack, accessible seulement par l'API du même environnement.

## 3. Authentification GHCR pour Watchtower

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u <github-user> --password-stdin
```

Crée `~/.docker/config.json`, monté en lecture seule par les Watchtower.

## 4. Réseau partagé + services partagés (une seule fois)

```bash
docker network create web

cd deploy/proxy
cp .env.example .env          # renseigner ACME_EMAIL
docker compose up -d          # démarre Traefik + Watchtower
```

## 5. Démarrage des deux environnements

```bash
# Staging
cd deploy/staging
cp .env.example .env          # renseigner domaine + secrets staging
docker compose up -d

# Prod
cd deploy/prod
cp .env.example .env          # renseigner domaine + secrets prod
docker compose up -d
```

Chaque compose fixe son nom de projet (`name:` cesizen-staging / cesizen-prod),
ce qui isole conteneurs, volumes et réseaux internes de chaque environnement.

## 6. Initialisation des bases (données de démo)

```bash
cd deploy/staging && docker compose exec api npm run db:init
cd deploy/prod    && docker compose exec api npm run db:init
```

`db:init` = `prisma db push` + seed. Permet de prouver la connexion API ↔ BDD
(afficher des données), exigence du sujet.

## 7. Déploiement continu

- Push sur `develop` → CI build `:staging` → Watchtower(staging) redéploie
- Push sur `main`    → CI build `:prod`    → Watchtower(prod) redéploie

Aucune intervention manuelle ni SSH.

## Variables / secrets GitHub à configurer

Repo **CESIZEN-FRONT** → *Settings ▸ Variables* :
- `VITE_API_URL_PROD    = https://api.DOMAIN`
- `VITE_API_URL_STAGING = https://api.staging.DOMAIN`

Le `GITHUB_TOKEN` (login GHCR en CI) est fourni automatiquement.
