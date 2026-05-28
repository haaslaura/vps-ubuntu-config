# Configurer un VPS

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

## Récupérer votre projet depuis Github
...
## Configurer vos serveurs
...
## Créer la base de données
...
## Préparer le fichier .env
...
