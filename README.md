# Configurer un VPS (Guide en cours)

J'ai configuré récemment un VPS chez [Hostinger](https://www.hostinger.com/fr). Outre quelques notes sur le processus que j'ai souhaité conserver, il est possible que cela serve à d'autres. Bonne lecture !

Attention, ces consignes fonctionnent pour un VPS avec Ubuntu. Selon vos configurations et vos besoins, les commandes peuvent changer.

Commandes générales : 

```bash
# Créer ou modifier un fichier
vi nomdufichier

# Quitter le fichier et enregistrer
Echap
:wq

# Quitter sans enregistrer
Ctrl+C
:qa

# Lire sans modifier
cat nomdufichier

# Créer un dossier
mkdir nomdudossier
```

## Se connecter au SSH via le terminal de commande

Ouvrir son terminal préféré ([Cmder.exe](https://cmder.app/) sur Windows par exemple)

Entrez :

```bash
ssh root@xx.xxx.xxx.xx # Il s'agit de l'IPv4 de votre serveur
```
Puis votre mot de passe.

## Récupérer le projet depuis Github

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
...
## Préparer le fichier .env
...
