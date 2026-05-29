# Configurer un VPS

J'ai récemment configuré un VPS chez [Hostinger](https://www.hostinger.com/fr). En plus de quelques notes que j’ai souhaité conserver sur le processus, ce guide pourra peut-être aider d’autres personnes. Bonne lecture !

> [!IMPORTANT]
> Ces consignes fonctionnent pour un VPS sous Ubuntu. Selon votre configuration et vos besoins, certaines commandes peuvent différer.

## 🔧 Commandes générales

### Commandes terminal

```bash
# Créer ou modifier un fichier
vi nomdufichier

# Lire un fichier sans le modifier
cat nomdufichier

# Créer un dossier
mkdir nomdudossier
```

### Raccourcis utiles dans vi

- Quitter le mode insertion : `Échap`
- Quitter le fichier et enregistrer : `:wq`
- Quitter sans enregistrer : `:qa`

## Se connecter au VPS en SSH

1. Ouvrir votre terminal préféré ([Cmder](https://cmder.app/) sous Windows, par exemple).
2. Entrer la commande suivante :

```bash
ssh root@xx.xxx.xxx.xx
```

3. Remplacez `xx.xxx.xxx.xx` par l’adresse IPv4 de votre serveur.
4. Puis saisir votre mot de passe.

## Récupérer le projet depuis GitHub

1. S’ils n’existent pas encore, créer un dossier pour vos projets (par exemple `/www`) ainsi qu’un dossier spécifique pour la production (`/prod`) et/ou le développement (`/dev`). Vous pouvez adapter ces conseils à vos besoins en matière d'organisation.
2. Se rendre dans le répertoire `/var`, puis créer les dossiers :

```bash
cd /var/
mkdir www
cd www
mkdir prod
cd prod
```

3. Si votre dépôt GitHub contient à la fois le front-end et le back-end, le cloner directement :

```bash
git clone <url-github-du-projet>
```

4. Sinon, créer un dossier frontend et un dossier backend, puis cloner les dépôts correspondants dans chacun d’eux.
5. Pour chaque projet (front et back) :

```bash
npm install
```

6. Puis si nécessaire :

```bash
npm run build
```

7. Il n’est pas nécessaire de lancer les projets pour le moment. La suite de la configuration, notamment avec PM2, permettra de s’en charger.

## Configurer vos serveurs

### Installer Nginx

**Nginx** est un logiciel libre de serveur web.

```bash
sudo apt install nginx
```

### Installer Pm2

**PM2** est un _Process Manager_ qui permet de faire tourner le serveur en arrière-plan.

```bash
npm install pm2 -g
```

### Installer Certbot

**Certbot** permet d’installer et de renouveler les certificats **SSL**.

```bash
apt install certbot python3-certbot-nginx
```

### Créer la configuration du serveur front

1. Se rendre dans le dossier du front, puis créer un fichier de configuration :

```bash
vi /etc/nginx/sites-available/front.conf
```

2. Ajouter une configuration de ce type :

```bash
server {
	listen 80;
    	listen [::]:80;

      server_name url-du-front.com;

	# Logs du trafic et des erreurs
      access_log /var/log/nginx/url-du-front.com.access.log;
      error_log /var/log/nginx/url-du-front.com.error.log;

      # Indiquez ici le port du front
      location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
	  proxy_set_header Upgrade $http_upgrade;
	  proxy_set_header Connection "upgrade";
        }        
}
```

3. Vérifier que le fichier de configuration ne comporte pas d’erreur :

```bash
nginx -t
```

4. Créer un lien symbolique :

```bash
ln -s /etc/nginx/sites-available/front.conf /etc/nginx/sites-enabled/
```

5. Recharger Nginx :

```bash
systemctl reload nginx
```

6. Installer le certificat SSL du front :

```bash
certbot --nginx
systemctl reload nginx
```

7. Lancer le serveur du front :

```bash
pm2 start npm --name "projet-front" -- run start
```

8. Prévoir une relance automatique en cas de redémarrage ou de crash du serveur :

```bash
pm2 save
```

### Créer la configuration du serveur back

1. Créer le fichier de configuration :

```bash
vi /etc/nginx/sites-available/back.conf
```

2. Ajouter une configuration de ce type :

```bash
server {
	listen 80;
	listen [::]:80;

	server_name url-du-back.com;

	# Logs du trafic et des erreurs
	access_log /var/log/nginx/url-du-back.com.access.log;
	error_log /var/log/nginx/url-du-back.com.error.log;

	# Indiquez ici le port du back
	location / {
		proxy_pass http://localhost:1337;

		# Headers standards
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		# Support WebSockets (utile pour le hot-reload/admin)
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}

	# Taille maximale des uploads (pour les images/vidéos)
	client_max_body_size 50M;

	# Headers de sécurité
	add_header X-Frame-Options "SAMEORIGIN" always;
	add_header X-Content-Type-Options "nosniff" always;
}
```

3. Vérifier que le fichier de configuration ne comporte pas d’erreur :

```bash
nginx -t
```

4. Créer un lien symbolique :

```bash
ln -s /etc/nginx/sites-available/back.conf /etc/nginx/sites-enabled/
```

5. Recharger Nginx :

```bash
systemctl reload nginx
```

6. Installer le certificat SSL du back :

```bash
certbot --nginx
systemctl reload nginx
```

7. Lancer le serveur du back :

```bash
pm2 start npm --name "projet-back" -- run start
```

8. Prévoir une relance automatique en cas de redémarrage ou de crash :

```bash
pm2 save
```

## Créer la base de données

1. Installer **PostgreSQL** en local sur votre machine, puis pousser les modifications sur le dépôt Git. Récupérer ensuite ces modifications sur le serveur (`git pull`).
2. Installer PostgreSQL sur le serveur. Lors de l’installation, un utilisateur système principal postgres est créé.

3. Pour se connecter à PostgreSQL en tant qu’utilisateur système :

```bash
sudo -i -u postgres
```

4. Une fois connecté, accéder à l’interface PostgreSQL :

```bash
psql
```

5. Créer une base de données :

```bash
CREATE DATABASE name_db;
```

6. Donner les droits à un utilisateur sur la base de données :

```bash
GRANT ALL PRIVILEGES ON DATABASE name_db TO admin_user;
GRANT ALL ON SCHEMA public TO admin_user;
```

7. Voici quelques commandes suplémentaires qui peuvent être utiles :

```bash
# Supprimer une base de données :
DROP DATABASE name_db;
# Se connecter à la base de données :
\c name_db
# Une fois dans la base de données, lister les tables :
\dt
```

## Préparer le fichier .env

1. Se placer au bon emplacement, puis créer le fichier :

```bash
vi .env
```

2. Appuyer sur `i` pour passer en mode insertion, puis ajouter :

```bash
# Commun (ou valeurs par défaut)
HOST=0.0.0.0
PORT=1337

FRONTEND_URL=https://votre-url.com
PUBLIC_URL=https://api.votre-url.com

# Si Strapi, ajouter les tokens
APP_KEYS="..., ..."
API_TOKEN_SALT=...
ADMIN_JWT_SECRET=
TRANSFER_TOKEN_SALT=
JWT_SECRET=
ENCRYPTION_KEY=

# DATABASE
DATABASE_CLIENT=postgres
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_NAME=
DATABASE_USERNAME=
DATABASE_PASSWORD=
DATABASE_SSL=false
```

3. Appuyer sur `ECHAP` puis `:wq` pour enregistrer et quitter.

## Contributions

Si vous repérez une erreur ou une amélioration possible dans ce tutoriel, n'hésitez pas à [ouvrir une issue](https://github.com/haaslaura/vps-ubuntu-config/issues/new) ou à proposer une Pull Request.

## 📬 Informations de contact

Je suis disponible via [LinkedIn](https://www.linkedin.com/in/laurahaas-developpement/) si vous souhaitez me contacter.
Merci pour vos questions ou vos feedback !
