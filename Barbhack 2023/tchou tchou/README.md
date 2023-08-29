# Tchou-tchou

Voici le writeup du challenge tchoutchou proposé au ctf de la Barbhack 2023. Il a été créé par mpgn à qui on doit le crackmapexec d'aujourd'hui je tiens à le rappeler car c'est quelqu'un.

## Compréhension du challenge
Le challenge est composé d'une unique page Web :

Le but du challenge est de lire le flag.txt qui est présent dans un dossier inconnu de /home/demoniac.
Sur la page du chall, on peut remarque la commande "rails s -b 0.0.0.0 -e production"
Un peu de recherche nous permet de comprendre qu'une application web rails ruby est lancé sur l'interface local en mode "production". Ce mode active le cache des pages, ce qui rend le chargement plus rapide. On note aussi qu'il est redémarré au boit de trois minutes via une cron job.

## Les vulnérabilités à exploiter
J'ai cherché "rails server production exploit" sur google et je suis tombé sur le PoC github pour "Rails doubletap exploit" écrit par le créateur du chall (https://github.com/mpgn/Rails-doubletap-RCE).
Cet exploit est un chainage de deux CVE sur un rails server qui permet d'y exécuter une commande à distance.
La première CVE (CVE-2019-5418) permet de lire les fichiers du serveur via une injection en path traversal sur le header "Accept". En l'utilisant, on peut récupérer deux fichiers: credentials.yml.enc et master.key, qui vont nous permettre de récupérer la "SECRET KEY BASE" en déchiffrant credentials.yml.enc avec master.key. La "SECRET KEY BASE" nous permettra de lancer la 2e CVE qui utilise en gros la deserialization d'objets Ruby (j'ai pas lu les détails, je reviens vers vous quand c'est fait hein) pour exécuter une commande sur le serveur.
Evidemment, ça c'est quand le scénario est idéal et que le serveur est en mode développement (càd avec le cache des pages désactivé). Ici, le créateur a pris un malin plaisir à le mettre en mode "production".
En pratique du coup, lorsqu'on exploite la première CVE pour tester le serveur, on obtient bien le fichier :


Mais à cause de la mise en cache, il nous renvoit toujours le même lorsqu'on met autre chemin.
Heureusement, le serveur est redémarré au bout de quelques minutes et vide le cache mais ça nous posera problème par la suite.

## Exploitation
Si on reprend le PoC de mpgn, on observe qu'il va faire une requête pour chercher d'abord credentials.yml.enc puis master.key. Ensuite il exploite la deserialisation et exécuter une commande, l'output est redirige vers un fichier, qu'il faudra récupérer.
On comprend donc qu'on va être bloqué pour récupérér les deux premiers fichiers étant donné qu'ils changent à chaque redémarrage et que le cache nous empêche de lire 2 fichiers.

Quelques recherches sur "rails doubletap exploit" nous permettent de tomber sur le tips suivant (j'ai remarqué après que mpgn renseigne le tips directement dans le PoC):

Une race condition est possible et on peut obtenir deux fichiers à la fois via curl. On récupère donc les deux fichiers :


Et une fois les 2 fichiers obtenus j'ai modifié le PoC de mpgn pour qu'ils prennent mes 2 fichiers récupérés pour déchiffrer la "SECRET KEY BASE". 


Avec le résultat, il crée la payload pour la deserialization.


J'ai lancé la commande : ls /home/demoniac/ pour identifier le dossier et une fois la commande exécutée, on doit récupérer le fichier d'output

On attend donc que le serveur redémarre (pour contourner la mise en cache), et on relance la CVE-2019-5418 sur /tmp/results :


Le dossier identifié, on refait la procédure pour pouvoir faire un "cat flag.txt".


### Note sur le setup
J'ai perdu beaucoup de temps sur l'installation de Ruby dû à des problèmes d'incompatibilité avec la versions que j'avais et celle utilisée par le PoC. Je vous conseille donc de désinstaller Ruby et de réinstaller la version via "rbenv install 2.5.1" 
