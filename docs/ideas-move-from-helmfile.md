# Passage à ArgoCD
## Pourquoi
J'ai commencé avec Helmfile car ca me permettait de démarrer en déclarant tout, donc ne pas me perdre dans les différentes configs de Kube. Sauf qu'entre temps, j'ai aussi besoin de monitorer mes déploiements de manière plus agréable, de pouvoir voir tout ce qui fail en un clin d'oeuil et aller voir les logs, pouvoir faire un rollback en un clic... 
Bref, j'ai envie d'une interface graphique simple pour tout gérer. ArgoCD semble être le seul outil qui convient, mais cela requiert de supprimer Helmfile et de passer à un mode "App of Apps" pour que ArgoCD puisse gérer lui même l'ensemble de l'infrastructure via les différents répertoires dans apps.

## Objectifs de la transition

- Améliorer la visibilité via une interface graphique (GUI)
- Améliorer la détection de dérive (Drift)
- Supprimer les dépendances aux variables locales et passer sur des outils standards

## Comment

### 1. Principes du Moteur GitOps (ArgoCD)

    Orchestration : Passage de helmfile apply (manuel) à ArgoCD (automatique).

    Structure du Repo : Utilisation du pattern App-of-Apps ou ApplicationSet pour que le cluster se configure lui-même à partir du dossier /apps.

    Gestion des Charts : Utilisation du support natif de Helm dans ArgoCD pour les charts distants et locaux (comme trek).

### 2. Stratégie de Gestion des Secrets (Sealed Secrets)

    Abandon du .env : Remplacement des requiredEnv par des fichiers YAML scellés.

    Mécanisme : Utilisation de kubeseal pour chiffrer les secrets avant de les pousser sur GitHub.

    Avantages : Aucun plugin complexe requis dans ArgoCD et sécurité totale du dépôt Git.

### 3. Topologie des Espaces de Noms (Namespaces)

    Regroupement par Stack : Regrouper les services dépendants (ex: media pour Jellyfin/Sonarr) pour simplifier la communication interne.

    Isolation stricte : Maintenir des namespaces isolés pour les services critiques (ex: auth, vaultwarden).

    Infrastructure : Isolation des opérateurs et outils système (traefik, cert-manager, cnpg-system).

    Rester compatible avec l'ancienne topologie : La transition ne doit pas déplacer des pods entre des namespaces.