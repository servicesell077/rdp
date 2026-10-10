# RDP Windows via GitHub Actions + Tailscale

Un bureau Windows complet, accessible à la demande en Bureau à distance (RDP), sur un runner GitHub Actions. Le port RDP n'est **jamais** exposé sur Internet : tout passe par votre réseau privé Tailscale.

> **Utilisez un dépôt privé** (ou au minimum un compte GitHub dédié). Les journaux d'un dépôt public sont lisibles par tout le monde (IP Tailscale, nom d'utilisateur).

## Ce qui a été corrigé par rapport à la version d'origine

| Problème | Correction |
|---|---|
| **RDP qui tombe après 1 à 2 h** | Tailscale est lancé avec `--unattended` : le tunnel reste actif même sans session Windows ouverte. |
| Sessions coupées par Windows | Plus aucune limite d'inactivité, de déconnexion ou de durée de session ; keep-alive RDP activé ; veille, hibernation et verrouillage d'écran désactivés. |
| Service RDP qui peut s'arrêter | Redémarrage automatique du service (`sc failure`) + surveillance toutes les 30 s. |
| Tunnel Tailscale qui peut tomber | La boucle de maintien détecte un tunnel hors ligne et le reconnecte toute seule. |
| `timeout-minutes: 3600` | Valeur invalide : GitHub limite les runners hébergés à 360 min (6 h). Remplacée par `360`, avec une durée de session réglable. |
| Mot de passe affiché dans les logs et écrit dans `GITHUB_ENV` | Supprimé : le mot de passe n'est jamais affiché ni exporté. |
| NLA et couche de sécurité désactivés | NLA + TLS activés. |
| Port 3389 ouvert à tout le monde | Autorisé uniquement depuis `100.64.0.0/10` (réseau Tailscale). |
| Tailscale 1.82.0 figé | Dernière version stable téléchargée automatiquement, avec nouvelles tentatives. |
| Machine qui reste dans la console Tailscale | `tailscale logout` en fin de job (même en cas d'échec ou d'annulation). |

## Installation

1. Créez un **dépôt privé** et placez les fichiers ainsi :
   ```
   .github/workflows/rdp.yml
   README.md
   .gitignore
   ```
2. Créez une clé d'authentification Tailscale : <https://login.tailscale.com/admin/settings/keys>
   - cochez **Reusable** ;
   - cochez **Ephemeral** (la machine disparaît toute seule du Tailnet à la fin) ;
   - copiez la clé `tskey-auth-...`.
3. Dans le dépôt : **Settings > Secrets and variables > Actions > New repository secret** :

   | Secret | Contenu |
   |---|---|
   | `TAILSCALE_AUTH_KEY` | la clé `tskey-auth-...` |
   | `RDP_USERNAME` | votre nom d'utilisateur (20 caractères max). Optionnel : `kipre10` par défaut |
   | `RDP_PASSWORD` | un mot de passe fort (8 caractères min : majuscule, minuscule, chiffre) |

## Utilisation

1. Onglet **Actions** > **RDP Tailscale (stable)** > **Run workflow**.
   - `nom_machine` : nom affiché dans Tailscale (défaut `gh-runner`).
   - `duree_minutes` : durée de la session (défaut 345, maximum 350).
2. Ouvrez l'exécution : l'adresse apparaît dans le résumé (**Summary**) et dans le journal de l'étape *Maintenir la session*.
3. Sur votre PC, Tailscale doit être lancé et connecté au même Tailnet.
4. Ouvrez *Connexion Bureau à distance*, saisissez l'IP `100.x.x.x` (ou simplement le nom `gh-runner`, grâce à MagicDNS), puis votre utilisateur et mot de passe.
5. Pour arrêter : **Cancel workflow** dans GitHub (la machine est retirée du Tailnet automatiquement).

## Limites à connaître

- **6 heures maximum** par exécution : c'est une limite de GitHub, impossible à dépasser. Relancez le workflow quand c'est fini.
- Le runner est **éphémère** : tout ce qui est installé ou enregistré disparaît à la fin. Sauvegardez vos fichiers avant d'arrêter.
- Les minutes Windows consomment votre quota Actions plus vite que Linux (multiplicateur x2 sur les dépôts privés). Vérifiez votre quota et les conditions d'utilisation de GitHub Actions.

## Dépannage

| Symptôme | Piste |
|---|---|
| « Secret ... manquant » | Vérifiez le nom exact des 3 secrets. |
| Connexion RDP impossible | Tailscale est-il connecté sur votre PC ? Utilisez bien l'IP `100.x.x.x` affichée. |
| Identifiants refusés | Mot de passe trop simple ou mal copié ; relancez après avoir corrigé le secret. |
| La machine n'apparaît pas dans Tailscale | Clé expirée ou révoquée : régénérez-la. |
| Session coupée au bout de ~6 h | Normal : limite GitHub. Relancez le workflow. |
