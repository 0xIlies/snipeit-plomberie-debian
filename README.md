# 🔧 Snipe-IT sur Debian 12 — Guide complet pour PME

> Gestion d'outillage open source, auto-hébergée, pour une entreprise de plomberie.
> Inventaire + QR codes + check-in/check-out depuis le téléphone.
> **Aucun abonnement. Données hébergées en local. 100% gratuit.**

---

## 📋 Contexte

Une petite entreprise de plomberie avait besoin de suivre son outillage commun entre plusieurs plombiers. Pertes, doublons, aucun suivi… J'ai mis en place **Snipe-IT** en auto-hébergement local avec Docker sur Debian 12.

**Contraintes :**
- Solution gratuite et open source
- Données hébergées en local (pas de cloud, pas d'abonnement)
- Utilisation simple pour des plombiers non techniciens
- Scan QR depuis le téléphone sans application à installer

---

## 🏗️ Architecture

```
PC physique au dépôt (Debian 12)
└── Docker
    ├── snipeit-app   → Apache + PHP + Snipe-IT (port 80)
    └── snipeit-mysql → MySQL 8.0
```

Tous les téléphones et PC connectés au Wi-Fi de l'entreprise accèdent à l'application via `http://192.168.1.X` — pas besoin d'internet.

---

## 🖥️ Prérequis

- Un PC ou serveur sous **Debian 12**
- Minimum **2 Go de RAM**
- Connecté au réseau local de l'entreprise
- Accès root ou sudo

---

## ⚙️ Installation étape par étape

### Étape 1 — Mettre à jour le système

```bash
sudo apt update && sudo apt upgrade -y
```

> `apt update` → met à jour la liste des paquets disponibles  
> `apt upgrade -y` → installe les mises à jour (-y = oui automatiquement)

---

### Étape 2 — Désactiver Apache2 système (si installé)

Debian 12 installe parfois Apache2 par défaut. Il faut le désactiver car il occupe le port 80 dont Docker a besoin.

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

> `systemctl stop apache2` → arrête Apache2 immédiatement  
> `systemctl disable apache2` → l'empêche de redémarrer au prochain démarrage du PC

Vérifier qu'il ne tourne plus :
```bash
sudo ss -tlnp | grep :80
```
> Si rien ne s'affiche → le port 80 est libre ✓

---

### Étape 3 — Installer Docker

Docker est le logiciel qui va créer et gérer les conteneurs (boîtes isolées) dans lesquels Snipe-IT et MySQL vont tourner.

```bash
curl -fsSL https://get.docker.com | sh
```

> `curl` → télécharge un fichier depuis internet  
> `-fsSL` → options : f=fail silently, s=silencieux, S=affiche erreurs, L=suit les redirections  
> `| sh` → exécute directement le script téléchargé

Ajouter votre utilisateur au groupe Docker pour ne pas avoir à taper `sudo` à chaque commande :

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> `usermod -aG docker $USER` → ajoute votre utilisateur au groupe "docker"  
> `$USER` → variable automatique qui contient votre nom d'utilisateur  
> `newgrp docker` → applique le changement sans redémarrer

---

### Étape 4 — Installer Docker Compose

Docker Compose permet de décrire et lancer plusieurs conteneurs en même temps avec un seul fichier de configuration.

```bash
sudo apt install -y docker-compose-plugin
```

Vérifier que tout est installé :

```bash
docker --version
docker compose version
```

> Ces commandes doivent afficher des numéros de version ✓

---

### Étape 5 — Créer le dossier de travail

```bash
mkdir ~/snipeit && cd ~/snipeit
```

> `mkdir` → crée un dossier nommé "snipeit" dans votre répertoire personnel  
> `cd ~/snipeit` → entre dans ce dossier  
> `~` → raccourci pour votre répertoire personnel (/home/votre-nom)

---

### Étape 6 — Créer le fichier docker-compose.yml

Ce fichier est la "recette" qui dit à Docker quoi créer, comment configurer chaque conteneur et comment les faire communiquer.

```bash
nano docker-compose.yml
```

> `nano` → éditeur de texte en ligne de commande  
> Copier-coller le contenu ci-dessous, puis **Ctrl+O** pour sauvegarder, **Ctrl+X** pour quitter

```yaml
services:
  mysql:
    image: mysql:8.0
    # "image" = le template Docker à utiliser (téléchargé depuis Docker Hub)
    container_name: snipeit-mysql
    # Nom qu'on donne à ce conteneur pour l'identifier facilement
    restart: unless-stopped
    # Redémarre automatiquement si le conteneur plante, sauf si on l'arrête manuellement
    environment:
      # Variables d'environnement = paramètres de configuration passés au conteneur
      MYSQL_ROOT_PASSWORD: VotreMotDePasseRoot
      # Mot de passe du super-admin MySQL (à changer !)
      MYSQL_DATABASE: snipeit
      # Nom de la base de données créée automatiquement
      MYSQL_USER: snipeit
      # Utilisateur dédié à Snipe-IT (accès limité à sa base uniquement)
      MYSQL_PASSWORD: VotreMotDePasseSnipeit
      # Mot de passe de cet utilisateur (à changer !)
    volumes:
      - mysql_data:/var/lib/mysql
      # Volume = dossier persistant géré par Docker
      # Les données MySQL survivent même si le conteneur est supprimé
    healthcheck:
      # Vérifie que MySQL est vraiment prêt avant de démarrer Snipe-IT
      test: ["CMD", "mysqladmin", "ping", "-u", "snipeit", "-pVotreMotDePasseSnipeit"]
      interval: 5s
      # Vérifie toutes les 5 secondes
      timeout: 5s
      retries: 10
      # Réessaie 10 fois avant de considérer MySQL comme en échec

  snipeit:
    image: snipe/snipe-it:latest
    # "latest" = toujours la dernière version de Snipe-IT
    container_name: snipeit-app
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
        # Snipe-IT ne démarre que quand MySQL est vraiment prêt (healthcheck OK)
    ports:
      - "80:80"
      # Format : "port-du-PC:port-dans-le-conteneur"
      # Le port 80 du PC pointe vers le port 80 du conteneur
      # C'est pour ça qu'on accède via http://IP sans numéro de port
    environment:
      APP_URL: http://192.168.1.X
      # Remplacer X par l'IP réelle du serveur (trouver avec : ip a)
      DB_HOST: mysql
      # "mysql" = nom du conteneur MySQL sur le réseau Docker interne
      # Docker traduit automatiquement ce nom en adresse IP interne
      DB_PORT: 3306
      # Port standard de MySQL
      DB_DATABASE: snipeit
      DB_USERNAME: snipeit
      DB_PASSWORD: VotreMotDePasseSnipeit
      # Doit être identique à MYSQL_PASSWORD ci-dessus
      APP_KEY: ""
      # Clé secrète de chiffrement - sera générée à l'étape suivante
      APP_TIMEZONE: Europe/Paris
      APP_LOCALE: fr
      MAIL_DRIVER: log
      # On n'utilise pas d'email, les logs suffisent
    volumes:
      - snipeit_data:/var/lib/snipeit
      # Stocke les fichiers uploadés, QR codes, etc.

volumes:
  mysql_data:
  snipeit_data:
  # Déclare les volumes pour que Docker les crée et les gère
```

---

### Étape 7 — Trouver l'adresse IP du serveur

```bash
ip a | grep "inet "
```

> `ip a` → affiche les interfaces réseau et leurs adresses IP  
> `| grep "inet "` → filtre pour n'afficher que les lignes avec des adresses IP

L'adresse ressemble à `192.168.1.X`. Remplacer dans le fichier docker-compose.yml la ligne `APP_URL: http://192.168.1.X` par la vraie adresse.

---

### Étape 8 — Premier démarrage et génération de la clé

```bash
docker compose up -d
```

> `docker compose up` → lit le docker-compose.yml et crée/démarre tous les conteneurs  
> `-d` → "detached" = en arrière-plan, le terminal reste disponible

Attendre 30 secondes que MySQL s'initialise, puis générer la clé de chiffrement :

```bash
docker exec snipeit-app php artisan key:generate --show
```

> `docker exec` → exécute une commande À L'INTÉRIEUR d'un conteneur qui tourne  
> `snipeit-app` → nom du conteneur ciblé  
> `php artisan key:generate` → commande Laravel (le framework PHP de Snipe-IT) qui génère une clé aléatoire  
> `--show` → affiche la clé au lieu de l'écrire directement dans un fichier

La commande affiche une ligne comme : `base64:AbCdEfGhIjKl...`

Copier cette clé et l'insérer dans le fichier :

```bash
nano docker-compose.yml
```

Remplacer `APP_KEY: ""` par `APP_KEY: "base64:VotreCléIci"`

> **Pourquoi cette clé ?** Laravel l'utilise pour chiffrer les sessions, les cookies et certaines données. Sans elle, l'application refuse de démarrer (erreur 500). Le préfixe `base64:` indique à Laravel le format de la clé.

---

### Étape 9 — Redémarrer avec la clé

```bash
docker compose down && docker compose up -d
```

> `docker compose down` → arrête et supprime les conteneurs (les données dans les volumes sont conservées)  
> `&&` → exécute la commande suivante seulement si la précédente a réussi  
> `docker compose up -d` → recrée et redémarre tout avec la nouvelle configuration

---

### Étape 10 — Finaliser via le navigateur

Ouvrir sur n'importe quel PC du réseau :
```
http://192.168.1.X/setup
```

Si erreur 500 à l'étape "Créer les tables", lancer manuellement :

```bash
docker exec snipeit-app php artisan migrate:fresh --force
```

> `php artisan migrate` → crée toutes les tables dans la base de données  
> `migrate:fresh` → supprime et recrée toutes les tables (base vierge)  
> `--force` → confirme l'action sans demander de validation

---

## 🏷️ Configuration des QR codes

1. **Admin → Labels**
2. Cocher **"Display 2D barcode"**
3. **2D Barcode Type** → laisser sur QRCODE
4. Dans **"Label visible fields"** cocher : Asset Name, Serial, Asset Tag
5. Cliquer **Save**

Pour imprimer les étiquettes :
- Fiche d'un outil → bouton **Print** → PDF généré
- Imprimer sur feuilles autocollantes **Avery L7160** (21 étiquettes/feuille)
- Pour outils exposés à l'eau/huile → étiquettes polyester plastifiées

---

## 📱 Solution mobile — sans application

Les apps tierces Snipe-IT manquent de maturité. La solution retenue est plus simple :

### Première connexion (une seule fois par téléphone)
1. Ouvrir **Chrome** sur le téléphone
2. Aller sur `http://192.168.1.X`
3. Se connecter avec son compte
4. Chrome → ⋮ → **"Ajouter à l'écran d'accueil"**

### Au quotidien
```
Ouvrir l'appareil photo natif
→ Pointer sur le QR code de l'outil
→ Taper la notification qui apparaît
→ Chrome ouvre la fiche de l'outil
→ Appuyer Checkout (sortie) ou Checkin (retour)
→ Terminé en 5 secondes
```

⚠️ **Condition :** les téléphones doivent être connectés au Wi-Fi de l'entreprise.

---

## 👥 Gestion des utilisateurs et rôles

| Rôle | Pour qui | Ce qu'il peut faire |
|------|----------|---------------------|
| **Superadmin** | Directeur / technicien IT | Accès total |
| **Assets** | Chef d'équipe | Voir et gérer tout l'inventaire |
| **Permissions custom** | Plombiers | Voir + checkin + checkout uniquement |

### Permissions recommandées pour les plombiers

Dans **People → plombier → Edit**, cocher uniquement :
- ✅ assets.view
- ✅ assets.checkin
- ✅ assets.checkout
- ✅ self.api (pour générer leur token si besoin)
- ✅ Gérer les jetons API

---

## 🗂️ Import des outils en masse

Snipe-IT accepte un fichier CSV pour importer plusieurs outils d'un coup.

Exemple de fichier `outils.csv` :
```csv
Name,Asset Tag,Serial,Model,Category,Location,Status,Notes
Perceuse Bosch GBH 2-26,OUTIL-001,SN-BOS-001,Bosch GBH 2-26,Perçage,Dépôt principal,Ready to Deploy,Perceuse SDS+
Chalumeau oxyacétylène,OUTIL-002,SN-CHA-001,Chalumeau Pro,Soudure,Dépôt principal,Ready to Deploy,
Coupe-tube Virax,OUTIL-003,SN-VIR-001,Virax 210212,Coupure,Dépôt principal,Ready to Deploy,
```

Importer via **Assets → Import → Choose File → Process → Import**

> ⚠️ Les catégories doivent exister dans Snipe-IT avant l'import

---

## 🔧 Commandes de maintenance

```bash
# Voir les conteneurs qui tournent
docker ps
# Affiche l'ID, l'image, le statut et les ports de chaque conteneur

# Voir les logs en temps réel (utile pour déboguer)
docker logs snipeit-app -f
# -f = "follow" = affiche les nouveaux logs au fur et à mesure

# Redémarrer Snipe-IT sans toucher à MySQL
docker restart snipeit-app

# Mettre à jour Snipe-IT vers la dernière version
docker compose pull && docker compose up -d
# pull = télécharge les nouvelles images
# up -d = recrée les conteneurs avec les nouvelles images

# Sauvegarder la base de données
docker exec snipeit-mysql mysqldump -h 127.0.0.1 \
  -u snipeit -pVotreMotDePasse snipeit > backup_$(date +%F).sql
# mysqldump = exporte toute la base en un fichier SQL
# $(date +%F) = insère la date du jour dans le nom du fichier (ex: backup_2026-10-02.sql)

# Tout supprimer et repartir de zéro (⚠️ supprime toutes les données)
docker compose down -v
# -v = supprime aussi les volumes (données MySQL et fichiers Snipe-IT)
```

---

## 🔒 Sécurité

- Solution **100% locale** — personne depuis internet ne peut y accéder
- Chaque utilisateur a son propre compte avec des permissions limitées
- Le code source de Snipe-IT est public et auditable sur GitHub
- Protéger le PC serveur avec un mot de passe de session
- **Sauvegarder la base de données au moins une fois par semaine**

---

## 🐛 Problèmes fréquents

| Problème | Cause | Solution |
|----------|-------|----------|
| Erreur 500 au setup | APP_KEY manquante ou mal formatée | Vérifier le préfixe `base64:` |
| Port 80 déjà utilisé | Apache2 système tourne | `sudo systemctl stop apache2` |
| Connection refused MySQL | MySQL pas encore prêt | Attendre 30s et relancer |
| Erreur "column already exists" | Tables partiellement créées | `docker compose down -v && docker compose up -d` |
| 403 depuis l'extérieur | Header Authorization bloqué | Vérifier la config Apache dans le conteneur |

---

## 📄 Ressources

- [Site officiel Snipe-IT](https://snipeitapp.com)
- [GitHub Snipe-IT](https://github.com/snipe/snipe-it)
- [Documentation API](https://snipe-it.readme.io/reference/api-overview)
- [Docker Hub Snipe-IT](https://hub.docker.com/r/snipe/snipe-it)

---

## 📄 Licence

Snipe-IT est sous licence [AGPL-3.0](https://github.com/snipe/snipe-it/blob/master/LICENSE)

---

*Guide réalisé dans le cadre d'un déploiement réel pour une PME de plomberie — Debian 12, Docker, réseau local.*
