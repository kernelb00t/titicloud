# ☁️ Titicloud — Homelab Kubernetes Infrastructure

> A GitOps-driven homelab built on **k3s** and managed with **Helmfile**, featuring automated dependency updates via Renovate, a Nix dev shell, and S3-backed backups.

---

## 📦 Stack Overview

| Layer | Services |
|---|---|
| **Infra** | Traefik, cert-manager, CloudNative-PG, Velero, external-services |
| **Auth** | Authentik (SSO / ForwardAuth) |
| **Media** | Jellyfin, Radarr, Sonarr, Prowlarr, qBittorrent (VPN), Jellyseerr |
| **Documents** | Paperless-ngx |
| **Photos** | Immich |
| **Secrets** | Vaultwarden |
| **Automation** | n8n |
| **CI/CD** | ArgoCD |
| **Custom** | Trek, Sure |

---

## 🏗️ Repository Structure

```
titicloud/
├── helmfile.yaml          # Orchestration principale (toutes les releases)
├── bases/                 # Valeurs par défaut partagées (default.yaml.gotmpl)
├── releases/              # Helmfile par service, organisés par namespace
│   ├── infra/
│   ├── auth/
│   ├── media/
│   ├── immich/
│   ├── paperless/
│   ├── vaultwarden/
│   ├── n8n/
│   ├── trek/
│   ├── argocd/
│   └── sure/
├── values/                # Valeurs Helm par service (.yaml.gotmpl)
├── charts/                # Sous-modules Helm custom (ex: paperless)
├── bases/                 # Bases Helmfile réutilisables
├── nas/                   # Configurations NAS / NFS
├── scripts/               # Scripts utilitaires
├── tests/                 # Manifests de test Kubernetes
├── docs/                  # Documentation technique
├── .env.example           # Template des variables d'environnement
├── flake.nix              # Dev shell Nix (outils CLI)
└── renovate.json          # Config Renovate (mise à jour automatique des dépendances)
```

---

## 🚀 Getting Started

### Prérequis

- [Nix](https://nixos.org/download/) avec flakes activés, **ou** les outils suivants installés manuellement :
  - `helmfile`, `helm`, `kubectl`, `k9s`, `velero`, `stern`

### 1. Cloner le dépôt

```bash
git clone --recurse-submodules https://github.com/kernelb00t/titicloud.git
cd titicloud
```

### 2. Entrer dans le dev shell

```bash
# Avec direnv (recommandé)
direnv allow

# Ou manuellement
nix develop
```

> Le shell exporte automatiquement `KUBECONFIG=$PWD/kubeconfig`.

### 3. Configurer l'environnement

```bash
cp .env.example .env
# Éditer .env et remplir toutes les valeurs CHANGE_ME
```

### 4. Déployer

```bash
# Voir les changements avant d'appliquer
helmfile diff

# Déployer tout
helmfile apply

# Déployer un namespace spécifique
helmfile -l namespace=media apply
```

---

## ⚙️ Configuration

Toutes les variables sensibles sont dans `.env` (ignoré par git). Voir `.env.example` pour la liste complète, organisée par service :

| Section | Variables clés |
|---|---|
| **Global** | `DOMAIN`, `KUBE_MASTER_IP` |
| **NFS / NAS** | `NFS_SERVER`, `NFS_MEDIA_PATH`, `NFS_IMMICH_PATH` |
| **S3 (Garage)** | `GARAGE_HOST`, `GARAGE_ACCESS_KEY`, `GARAGE_SECRET_KEY` |
| **Traefik** | `ACME_EMAIL`, `CLOUDFLARE_DNS_API_TOKEN` |
| **Authentik** | `AUTHENTIK_SECRET_KEY`, `AUTHENTIK_DB_PASSWORD` |
| **Immich** | `IMMICH_DB_PASSWORD` |
| **Paperless** | `PAPERLESS_SECRET_KEY`, `PAPERLESS_ADMIN_*` |
| **Vaultwarden** | `VAULTWARDEN_ADMIN_TOKEN` |
| **VPN (Gluetun)** | `PROTON_WIREGUARD_PRIVATE_KEY` |

---

## 🗒️ Notes Opérationnelles

### k3s — Installation

Désactiver Traefik intégré pour éviter les conflits avec le Traefik géré par Helmfile :

```bash
curl -sfL https://get.k3s.io | sh -s - --disable=traefik
```

### NFS — Partages Synology

Les partages NFS suivants doivent être créés sur le NAS avant le déploiement :

- `/volume1/titiflix` — Bibliothèque média (Jellyfin, Radarr, Sonarr…)
- `/volume1/immich` — Bibliothèque photos (Immich)

### Domaines Dev / Prod

Chaque service peut être basculé entre un domaine de dev et de prod via les variables `.env` :

```bash
DOMAIN_DEV=dev.thivillon.org
DOMAIN_PROD=thivillon.org
# Par service :
JELLYFIN_ENV=prod   # ou dev
```

### Stratégie de déploiement (RollingUpdate)

Kubernetes déploie les nouvelles versions en mode **RollingUpdate** : la nouvelle version démarre avant que l'ancienne soit supprimée. Si la nouvelle version effectue une **migration de base de données**, l'ancienne version peut crasher et bloquer le déploiement.

**Recommandation** : pour les services avec CNPG (Authentik, Immich), préférer une mise à jour en recréant le pod :

```bash
kubectl rollout restart deployment <app> -n <namespace>
```

---

## 🔐 Authentik & ForwardAuth

### Servarr (Radarr, Sonarr, Prowlarr)

Ces applications ignorent les variables d'environnement d'auth si un fichier `config.xml` existe déjà. Un **init container** patche automatiquement ce fichier au démarrage :

- `AuthenticationMethod` → `External`
- `AuthenticationRequired` → `DisabledForLocalAddresses`

> **⚠️ Premier déploiement** : au tout premier lancement, `config.xml` n'existe pas encore. L'application démarre avec l'auth par défaut (Basic Auth). Il faut **redémarrer le pod une fois** après le déploiement initial :
> ```bash
> kubectl rollout restart deployment <app> -n media
> ```

### Stockage Authentik

Les données Authentik sont persistées dans le PVC `authentik-data-pvc` (namespace `auth`), monté en `/media` dans le conteneur.

---

## 🤖 Renovate — Mise à jour automatique

Les dépendances (charts Helm, images Docker, flakes Nix) sont gérées automatiquement par **Renovate**.

### Images LinuxServer

Les images `lscr.io/linuxserver/*` nécessitent un pattern spécifique pour que Renovate trouve à la fois le bon tag Docker et le changelog GitHub :

```yaml
# renovate: datasource=docker depName=lscr.io/linuxserver/<app> changelogUrl=https://github.com/linuxserver/docker-<app>
repository: lscr.io/linuxserver/<app>
tag: "<version>"
```

> Ne pas utiliser `datasource=github-releases` : les tags GitHub upstream (ex: `6.1.2`) ne correspondent pas aux tags Docker linuxserver (ex: `5.20.2`).

### Mises à jour majeures de base de données

Les PR Renovate pour les montées de version majeures de **CNPG** ou **vectorchord** sont créées en **draft** et accompagnées d'un avertissement. Consulter [docs/database-major-version-upgrade.md](docs/database-major-version-upgrade.md) avant de merger.

---

## 🛠️ Outils inclus dans le dev shell (Nix)

| Outil | Rôle |
|---|---|
| `helm` + plugins | Packaging Kubernetes (diff, git, secrets, s3) |
| `helmfile` | Orchestration multi-releases |
| `kubectl` / `kubectx` | Interaction avec le cluster |
| `k9s` | Interface TUI pour Kubernetes |
| `stern` | Agrégation de logs multi-pods |
| `velero` | Sauvegarde / restauration cluster |
| `cmctl` | Gestion cert-manager |
| `kubectl-cnpg` | Plugin CloudNative-PG |
| `python3` | Scripts utilitaires |

---

## 📁 Documentation

- [docs/database-major-version-upgrade.md](docs/database-major-version-upgrade.md) — Procédure de mise à jour majeure PostgreSQL (CNPG)

---

## 📄 Licence

Usage personnel — homelab privé.