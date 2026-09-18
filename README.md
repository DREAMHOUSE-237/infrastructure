# infrastructure

Orchestration, déploiement et documentation d'infrastructure de la plateforme **DREAMHOUSE237**.

## Contenu

- `docker-compose.prod.yml` — stack Docker Swarm complète (tous les microservices backend + RabbitMQ + Traefik)
- `.github/workflows/deploy.yml` — pipeline de déploiement, déclenché par `repository_dispatch` depuis chacun des repos de service (ou manuellement via `workflow_dispatch`)
- `traefik-dynamic.yml` — configuration dynamique de Traefik (certificats, routage)
- `docs/architecture.md` — runbook de bootstrap du cluster Swarm (init, join des nœuds, labels de placement, ports de sécurité requis)

## Infrastructure cible

- 2 instances **EC2** (AWS, free tier) formant un cluster **Docker Swarm** :
  - un nœud `zone=gateway` (Traefik, config-service, registry-service, RabbitMQ)
  - un nœud `zone=app` (les microservices applicatifs)
- Une instance **RDS MySQL** (connexion TLS obligatoire, certificat `global-bundle.pem`)
- **Traefik** comme reverse proxy public avec certificats Let's Encrypt (domaines `nip.io`)

Le contrainte de placement par `node.labels.zone` et le mode `host` sur certains ports (`config-service`, `registry-service`, `messagebroker-service`) permettent de contourner les limites de ressources du tier gratuit AWS tout en gardant une topologie proche d'un vrai cluster multi-nœuds — utile à des fins pédagogiques/démo, sans prétendre à une haute disponibilité réelle.

## Pipeline de déploiement

1. Un push sur `dev` dans un des repos de service déclenche son CI (tests + build + push de l'image Docker)
2. Le CI merge automatiquement `dev` → `main`
3. Un événement `repository_dispatch` (`deploy-<service>`) est envoyé à ce repo
4. Le workflow `deploy.yml` valide que tous les secrets requis sont présents, puis exécute `docker stack deploy --resolve-image always` sur le nœud manager, ce qui ne redémarre que les services dont l'image a changé.

⚠️ **Tout push sur `dev` d'un service redéploie automatiquement ce service en production.** À garder en tête avant de pousser un changement, même mineur (ex. nettoyage de fichiers), pendant une utilisation en direct de la plateforme.

## Secrets

Les secrets (identifiants DB, RabbitMQ, Cloudinary, Campay, mail, Docker Hub, `GH_PAT`) sont gérés au niveau de l'organisation GitHub (`DREAMHOUSE-237`), visibles par tous les repos publics de l'org. Ne jamais commiter de valeur réelle dans ce repo — utiliser exclusivement `${VAR}` dans `docker-compose.prod.yml`.

## Bootstrap du cluster (résumé)

Voir `docs/architecture.md` pour la procédure complète : `docker swarm init`, `docker swarm join` sur le second nœud, labellisation des nœuds (`docker node update --label-add zone=...`), et ouverture des ports de sécurité requis par Swarm (`2377`, `7946`, `4789`) ainsi que ceux de chaque service applicatif dans les Security Groups AWS.
