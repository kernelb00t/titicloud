# titiCloud
Ce repo contient toutes les ressources nécessaires pour faire tourner titiCloud, le homelab des Thivillons.

## Configuration VM Proxmox
### 1. Préparer la VM (GUI ou CLI Proxmox)
- OS : Linux 6.x Kernel (l'ISO NixOS Minimal convient parfaitement).
- Système :
  - Machine : q35
  - BIOS : OVMF (UEFI)
- Disque :
  - Bus : SCSI avec contrôleur VirtIO SCSI Single
  - Option Discard activée pour le TRIM de ton NVMe
- CPU : Type host pour exploiter toutes les instructions du i5-1340P.
- Réseau : Modèle VirtIO (paravirtualized).

### 2. Installation « Bootstrap »
1. Démarre sur l'ISO NixOS Minimal.
2. Partitionne le disque (schéma GPT/UEFI — 512 Mo pour EFI (`/boot`), le reste pour la racine (`/`)) :
```bash
parted /dev/sda -- mklabel gpt
parted /dev/sda -- mkpart primary fat32 1MiB 512MiB
parted /dev/sda -- set 1 esp on
parted /dev/sda -- mkpart primary ext4 512MiB 100%
```
3. Formate et monte les partitions :
```bash
mkfs.fat -F 32 -n boot /dev/sda1
mkfs.ext4 -L nixos /dev/sda2
mount /dev/disk/by-label/nixos /mnt
mkdir -p /mnt/boot
mount /dev/disk/by-label/boot /mnt/boot
```
4. Génère la configuration de base :
```bash
nixos-generate-config --root /mnt
```
5. Active le SSH et crée ton utilisateur dans `/mnt/etc/nixos/configuration.nix`.
   - Remarque : cette configuration sera remplacée par ta Flake juste après.
6. Installe NixOS (`nixos-install`), puis redémarre.

## Mise à jour des dépendances

### Helm charts
Dependabot est configuré (`.github/dependabot.yml`) pour ouvrir automatiquement des PRs
quand de nouvelles versions de charts Helm sont disponibles dans les dossiers `releases/`.
Aucune action manuelle n'est nécessaire.

### Tags d'images Docker
Les tags Docker dans les fichiers `values/*/values.yaml` **ne sont pas** mis à jour
automatiquement par Dependabot (limitation Dependabot — il ne scanne pas les fichiers
Helm values).

Pour mettre à jour les tags manuellement :
1. Recherche le tag actuel : `grep -r "tag:" values/`
2. Vérifie la dernière version sur Docker Hub / GitHub Container Registry pour chaque image.
3. Mets à jour le `tag:` dans le fichier `values/<service>/values.yaml` correspondant.
4. Ouvre une PR pour review avant de déployer.

> **Astuce** : Pour automatiser ce processus, tu peux activer
> [Renovate Bot](https://docs.renovatebot.com/modules/manager/helm-values/) qui supporte
> nativement les images Docker dans les fichiers Helm values via le manager `helm-values`.
> Ajoute un fichier `renovate.json` à la racine du repo pour l'activer.
