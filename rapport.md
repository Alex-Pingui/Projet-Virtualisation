# Projet Virtualisation

### Alexandre GUIHARD

## Phase 1: Préparation de l’Infrastructure et Docker Registre

### Construction de l'application Flask et de l'image
---

#### Fichiers de l'application
---
Pour pouvoir utiliser l'application, on a 3 fichiers

Un fichier **Dockerfile**
```Dockerfile
FROM python:3.9
WORKDIR /web
COPY web /web
# --progress-bar off is necessery to avoid a thread problem
RUN pip install --progress-bar off -r requirements.txt
ENTRYPOINT ["python", "app.py"]
```

Le fichier avec l'application **web/app.py** et le fichier **web/requirements.txt** avec les dépendances suivantes
```
flask
redis
```

#### Création de l'image et du container
---
Après avoir démarré le serveur redis, on récupère l'ip de ce serveur pour pouvoir l'utiliser sur notre application avec la commande suivante

![Ip serveur redis]

On va donc mettre l'ip obtenue dans l'host du serveur dans le fichier app.py.

Puis, on construit l'image docker via la commande
```bash
docker image build --tag web_api_application .
```

et lorsqu'on veux consulter les images construites, on voit l'image docker qui vient d'être construite à la dernière ligne
![Image Docker Flask]

#### Questions
---
##### Est-ce que l’application est dans les workers, pourquoi ?
L'application n'est pas présente sur les workers mais sur manager1, les workers vont utiliser l'ip de manager1 pour pouvoir accéder à l'application

##### Est-ce que l’application est accessible depuis les workers? 
Même si ce ne sont pas les workers qui contiennent l'application, ils peuvent effectuer des requêtes http avec l'ip de manager1 afin de communiquer avec l'application. De plus sur manager1, on peut voir les requêtes effectuées avec l'ip du client et sa requête

![Logs Serveur]

### Création d’un Registre Privé

Pour utiliser un registre privé, on va tout d'abord créer un container avec l'image **registry**

Etat fin seance 1: l'app a été push sur le registre: prochaine cmd: curl http://<manager1-ip>:5000/v2/_catalog


[Ip serveur redis]: images/p1/e2/ip_redis.png
[Image Docker Flask]: images/p1/e2/image_web_api_app.png
[Résultat curl manager1]: images/p1/e2/curl_post_manager1.png
[Résultat curl worker1]: images/p1/e2/curl_post_worker1.png
[Résultat curl worker2]: images/p1/e2/curl_post_worker2.png
[Logs Serveur]: images/p1/e2/logs_server_web_api_application.png