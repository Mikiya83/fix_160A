# fix_160A.sh – Script de gestion Wi-Fi 5 GHz en bande 160 MHz

## Description
Ce script permet de maintenir la connexion Wi-Fi 5 GHz d'un routeur Asus (avec firmware Merlin) en bande **160 MHz**, même en cas de perturbations ou de changements de canal dus aux restrictions DFS (Dynamic Frequency Selection). Il est conçu pour être exécuté automatiquement via **cron** toutes les 30 minutes.

**Crédit initial** : Ce script est inspiré du travail de l'utilisateur [soul4kills](https://www.snbforums.com](https://www.snbforums.com/threads/script-160mhz-channel-recovery-for-wifi-6-only-routers.96964/#post-987670)/) sur le forum [SNBForums](https://www.snbforums.com/). Merci à lui !

---

## Fonctionnalités principales
- **Maintien de la bande 160 MHz** : Le script vérifie et réapplique la configuration 160 MHz si nécessaire.
- **Gestion des canaux DFS** : Il évite les canaux DFS (149-165) pour une stabilité optimale.
- **Journalisation** : Les actions et erreurs sont enregistrées dans un fichier de log pour un suivi facile.
- **Gestion automatique** : Ajout/suppression automatique des tâches cron et des entrées `init-start`.

---

## Restrictions
- **Non compatible avec les routeurs tri-bande** : Ce script est conçu uniquement pour les routeurs **dual-bande** sous firmware Merlin.
- **Firmware Merlin requis** : Le script utilise des commandes spécifiques au firmware Merlin.

---

## Installation

### Prérequis
- Un routeur Asus compatible avec le firmware **Merlin**.
- Accès SSH au routeur.
- Le script doit être placé dans `/jffs/scripts/`.

### Étapes d'installation
1. **Copier le script** :
   Téléchargez le fichier `fix_160A.sh` et placez-le dans `/jffs/scripts/` sur votre routeur.
   ```bash
   scp fix_160A.sh admin@<adresse_ip_du_routeur>:/jffs/scripts/
   ```

2. **Rendre le script exécutable** :
   ```bash
   chmod +x /jffs/scripts/fix_160A.sh
   ```

3. **Configurer cron** :
   Le script gère automatiquement l'ajout de la tâche cron. Exécutez-le une première fois pour activer la planification :
   ```bash
   /jffs/scripts/fix_160A.sh
   ```

4. **Vérifier les logs** :
   Les actions du script sont enregistrées dans `/jffs/scripts/fix_160A.log`.

---

## Configuration
- **Canaux préférés** : Modifiez la variable `PREFERRED` dans le script pour définir votre canal 160 MHz préféré (ex: `"100/160"`).
- **Canaux de secours** : Modifiez `BACKUP_FREQS` pour définir des canaux alternatifs.
- **Journalisation** : Ajustez `VERBOSE` pour plus ou moins de détails dans les logs.

---

## Utilisation
- Le script s'exécute automatiquement toutes les 30 minutes via cron.
- Pour une exécution manuelle :
  ```bash
  /jffs/scripts/fix_160A.sh
  ```

---

## Dépannage
- **Problèmes de connexion** : Vérifiez les logs dans `/jffs/scripts/fix_160A.log`.
- **Erreurs de syntaxe** : Assurez-vous que le script est bien copié et exécutable.
- **Compatibilité** : Vérifiez que votre routeur est compatible avec le firmware Merlin.

---

## Licence
Ce script est distribué sous licence **MIT**. Vous êtes libre de l'utiliser, le modifier et le partager.

---
