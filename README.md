# Configurer un VPS (guide en cours)

J'ai récemment configuré un VPS chez [Hostinger](https://www.hostinger.com/fr). En plus de quelques notes que j’ai souhaité conserver sur le processus, ce guide pourra peut-être aider d’autres personnes. Bonne lecture !

Attention : ces consignes fonctionnent pour un VPS sous Ubuntu. Selon votre configuration et vos besoins, certaines commandes peuvent différer.

## Commandes générales : 

```bash
# Créer ou modifier un fichier
vi nomdufichier
# Quitter le fichier et enregistrer
Échap
:wq
# Quitter sans enregistrer
Ctrl + C
:qa
# Lire un fichier sans le modifier
cat nomdufichier
# Créer un dossier
mkdir nomdudossier
```

## Se connecter au VPS en SSH

Ouvrir son terminal préféré ([Cmder.exe](https://cmder.app/) sur Windows par exemple)

Entrez :

```bash
ssh root@xx.xxx.xxx.xx # Il s'agit de l'IPv4 de votre serveur
```
Puis votre mot de passe.

## Récupérer le projet depuis GitHub

S’ils n’existent pas encore, créez un dossier pour vos projets (ex: /wwww) et un dossier spécifique à la /prod (et/ou au /dev, à vous de voir comment vous souhaitez vous organiser).

Se rendre dans les fichiers puis créer les dossiers :

```bash
cd /var/
mkdir www
cd www
mkdir prod
cd prod
```

Si votre projet Github contient le front et le back, récupérez-le :
```bash
git clone <url  github du projet>
```
Sinon, créez votre dossier frontend et votre dossier backend avant d’y cloner vos repo.

Pour chaque dossier (front et back) :
```bash
npm i # installez vos modules
npm run build # si nécessaire pour votre projet
```
Pas la peine de lancer les projets pour le moment. C’est avec la suite – et Pm2 - qu'on s’en chargera.

## Configurer vos serveurs
### Installer Nginx
Il s'agit d'un logiciel libre de serveur Web.
```bash
sudo apt install nginx
```
### Installer Pm2
Pm2 est le Product Process Manager qui fera tourner le server (en fond).
```bash
npm install pm2 -g
```
### Installer Certbot
Certbot permettra d’installer les certificats SSL.
```bash
apt install certbot python3-certbot-nginx
```
### Créer la config serveur du front :
Se rendre dans le dossier du front.
Puis créez un fichier pour la configuration :
```bash
vi /etc/nginx/sites-available/front.conf
```
Exemple de configuration :
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
Vérifier que le fichier de configuration ne comporte pas d’erreur :
```bash
nginx -t
```
Créer un lien symbolique :
```bash
ln -s /etc/nginx/sites-available/front.conf /etc/nginx/sites-enabled/
```
Relancer le système en fond :
```bash
systemctl reload nginx
```
Installer le certificat SSL du front :
```bash
certbot --nginx
systemctl reload nginx
```
Lancer le serveur du front :
```bash
pm2 start npm --name "projet-front" -- run start
```
Prévoir une relance automatique si le serveur tombe :
```bash
pm2 save
```
###	Créer la config server du back :
```bash
vi /etc/nginx/sites-available/back.conf
```
Exemple de configuration :
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
Vérifier que le fichier de configuration ne comporte pas d’erreur :
```bash
nginx -t
```
Créer un lien symbolique :
```bash
ln -s /etc/nginx/sites-available/back.conf /etc/nginx/sites-enabled/
```
Relancer le système en fond :
```bash
systemctl reload nginx
```
Installer le certificat SSL du back :
```bash
certbot --nginx
systemctl reload nginx
```
Lancer le serveur du back :
```bash
pm2 start npm --name "projet-back" -- run start
```
Prévoir une relance automatique si le serveur tombe :
```bash
pm2 save
```

## Créer la base de données
Installer pg en local.
Git push puis git pull le projet.
Installer postgresql sur le serveur. Il faudra un nom d’utilisateur principal et un mot de passe.
Pour se connecter à postgres en tant qu'utilisateur système, entrer :

```bash
sudo -i -u postgres
```
Une fois dans postgres :
Créer une base de données :
```bash
CREATE DATABASE name_db;
```
Donner le droit à l'utilisateur :
```bash
GRANT ALL PRIVILEGES ON DATABASE name_db TO admin_user;
GRANT ALL ON SCHEMA public TO admin_user;
```
Supprimer une base de données :
```bash
DROP DATABASE name_db;
```
On peut ensuite se rendre dans la base de données que l'on souhaite. Ex :
```bash
\c name_db
```
Puis voir nos tables :
```bash
\dt
```
## Préparer le fichier .env
Au bon emplacement, entrer :
```bash
vi .env
```
Appuyer sur i pour modifier :
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
Appuyer sur ECHAP puis :wq pour enregistrer.
