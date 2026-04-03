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
---

Pour utiliser un registre privé, on va tout d'abord créer le registre avec l'image **registry**

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

Puis, pour utiliser le registre, on va pousser l'application sur le registre qui vient d'être crée

```bash
docker tag web_api_application 172.16.1.204:5000/web_api_application
docker push 172.16.1.204:5000/web_api_application
```

Pour finir, on va tester d'effectuer des requêtes vers le registre sur le manager1 et sur les workers

**Résultat sur le manager1**

![Résultat curl manager1 avec le registre]

**Résultat sur un worker**

![Résultat curl worker avec le registre]

#### Questions
---

##### A quoi sert un registre privé?

Un registre privé stocke l'image Docker utilisé de manière sécurisé

##### Pourquoi est-il important dans un environnement de production?

Un registre est important en production puisqu'il assure une sécurité, pas de dépendance à un service externe

## Phase 2 : Déployer l’Application sur Swarm

### Initialisation de Swarm
---

Pour déployer l'application Flask sur le cluster des 3 VMs (manager1, worker1 et worker2), on doit tout d'abord initialiser Docker Swarm
```bash
docker swarm init --advertise-addr 172.16.1.204
```

**Résultat de l'initialisation de swarm**

![Initialisation swarm]

Après avoir initialisé swarm sur notre manager, on doit faire en sorte que les workers rejoignent le cluster avec le token donné à l'initialisation et l'adresse du manager gràce à la commande donnée après l'initialisation

**Sur un des workers**

![Nouveau worker dans le cluster]

Dans le manager, on peut consulter les noeuds présents dans le cluster (nos workers) gràce à la commande suivante

![Noeuds du cluster]

On y voit le worker (1ère ligne) présent dans le cluster et notre manager (2ème ligne)

### Mettre en place un réseau overlay et une volume pour Redis
---

Pour continuer, on va maintenant créer un réseau overlay qui sera utilisé lors du déploiement du service Redis.

**Création du réseau overlay**

![Réseau secure_net]

On va également mettre en place un volume pour avoir une persistance des données

**Création du volume**

![Volume redis_data]

#### Questions
---

##### Pourquoi est-il important d’utiliser un réseau overlay pour les services de l’application?

Un réseau overlay permet de faire communiquer les différents services de l'application présents dans le cluster

##### Quels sont les avantages par rapport à un réseau bridge?

Un réseau overlay fonctionne sur un cluster contrairement à un réseau bridge

### Déployer l’Application sur Swarm
---

Maintenant que tous les outils nécessaires ont été mis en place, on va pouvoir déployer l'application sur Swarm.

On va commencer tout d'abord par déployer Redis en créant un service avec le réseau overlay créé précédemment et le volume également créé précédemment

![Déploiement de Redis sur swarm]

On peut voir ci-dessous sur quel noeud a été déployée Redis.

![Noeud du service Redis]

Et on peut tester la connexion avec redis grâce à un ping

![Test de la connexion avec Redis]

Redis a été déployé, on va maintenant déployer notre application Flask. Dans un premier temps, on va créer le service qui va contenir l'application avec le même réseau overlay utilisé sur le service Redis

![Service flask-app]

Le réseau utilisé par les services Redis et celui de l'application Flask est le même et on peut le vérifier ci-dessous

![Networks des services]

On peut de la même façon que pour le service Redis savoir sur quel noeud l'application Flask a été déployée

![Noeud du service flask-app]

Pour finir, on va maintenant scaler l'application grâce à cette commande:
```bash
docker service scale flask-app=3
```

On peut vérifier que l'application a bien été scaler

![Noeuds après avoir scale]

Pour vérifier qu'on arrive à utiliser l'application après le déploiement, on peut effectuer des requêtes HTTP tel qu'un POST comme celui ci-dessous sur le manager et sur les workers.

![POST Application scale]


[Ip serveur redis]: images/phase1/etape2/ip_redis.png
[Image Docker Flask]: images/phase1/etape2/image_web_api_app.png
[Résultat curl manager1]: images/phase1/etape2/curl_post_manager1.png
[Résultat curl worker1]: images/phase1/etape2/curl_post_worker1.png
[Résultat curl worker2]: images/phase1/etape2/curl_post_worker2.png
[Logs Serveur]: images/phase1/etape2/logs_server_web_api_application.png

[Résultat curl manager1 avec le registre]: images/phase1/etape3/curl_post_manager1.png
[Résultat curl worker avec le registre]: images/phase1/etape3/curl_post_worker1.png

[Initialisation swarm]: images/phase2/etape1/initialisation_swarm.png
[Nouveau worker dans le cluster]: images/phase2/etape1/worker_join_swarm.png
[Noeuds du cluster]: images/phase2/etape1/swarm_nodes.png

[Réseau secure_net]: images/phase2/etape2/reseau_overlay.png
[Volume redis_data]: images/phase2/etape2/volume_redis.png

[Déploiement de Redis sur swarm]: images/phase2/etape3/deploiement_redis_swarm.png
[Noeud du service Redis]: images/phase2/etape3/noeud_service_redis.png
[Test de la connexion avec Redis]: images/phase2/etape3/ping_redis.png
[Service flask-app]: images/phase2/etape3/service_flask_app.png
[Networks des services]: images/phase2/etape3/network_services.png
[Noeud du service flask-app]: images/phase2/etape3/node_flask_app.png
[Noeuds après avoir scale]: images/phase2/etape3/flask_app_scale.png
[POST Application scale]: images/phase2/etape3/curl_app_scale.png