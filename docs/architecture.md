# Docker Swarm — mise en place (une seule fois)

`docker-compose.prod.yml` est maintenant un fichier de stack Swarm (labels
`node.labels.zone`, `deploy.replicas`, `deploy.restart_policy`, réseau
`traefik-net` en `overlay`). Ce qui suit ne se fait qu'une fois, à la main,
sur les deux EC2 - ce ne sont pas des commandes que le pipeline CI exécute.

## 1. Security Group

Entre les deux EC2 (IP privées), autoriser :
- TCP 2377 (gestion du cluster)
- TCP + UDP 7946 (communication entre nœuds)
- UDP 4789 (réseau overlay)

## 2. Initialiser le swarm

Sur l'EC2 qui deviendra le **manager** (celle pointée par le secret
`EC2_HOST` - c'est elle que le pipeline de déploiement utilisera) :

```bash
docker swarm init --advertise-addr <IP_PRIVEE_DE_CETTE_EC2>
```

Copier la commande `docker swarm join --token ...` qu'elle affiche, puis
l'exécuter sur la deuxième EC2 (`EC2_HOST2`) pour qu'elle rejoigne le swarm
en tant que worker.

Vérifier sur le manager :

```bash
docker node ls
```

## 3. Labelliser les deux nœuds

Reprend le découpage qui existait déjà dans l'ancien `deploy.yml`
(EC2_HOST2 = config/registry/proxy/messagebroker, EC2_HOST = le reste) :

```bash
docker node update --label-add zone=app <NODE_ID_EC2_HOST>
docker node update --label-add zone=gateway <NODE_ID_EC2_HOST2>
```

`<NODE_ID_...>` vient de la colonne `ID` de `docker node ls`.

## 4. Premier déploiement

Depuis le dossier `infrastructure` sur le manager, avec les mêmes variables
d'environnement que celles exportées par `deploy.yml` :

```bash
docker stack deploy -c docker-compose.prod.yml --resolve-image always --with-registry-auth dreamhouse
docker stack services dreamhouse
```

Ensuite, chaque déclenchement CI (`repository_dispatch`) refait ce même
`docker stack deploy` sur le manager - Swarm ne met à jour que les services
dont l'image ou la définition a changé.
