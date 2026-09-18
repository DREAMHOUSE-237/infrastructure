# Walkthrough — migrer vers un nouveau compte AWS

Ce document sert pour le jour où le compte AWS actuel expire (ou doit être
remplacé) et où toute l'infrastructure doit être reconstruite ailleurs. Il
détaille, dans l'ordre, tout ce qu'il faut refaire pour reproduire
l'infrastructure actuelle de DREAMHOUSE237 : 2 EC2 en cluster Docker Swarm,
1 RDS MySQL, Traefik en reverse proxy public avec HTTPS.

À suivre séquentiellement. Compter une bonne partie d'une journée la
première fois.

---

## 0. Avant de commencer

- [ ] Un nouveau compte AWS (ou le même compte réactivé) avec accès à la
      console EC2 et RDS, région `eu-north-1` (Stockholm) ou une autre de
      ton choix — si tu changes de région, toutes les URLs `*.rds.amazonaws.com`
      et `*.compute.amazonaws.com` changeront de suffixe en plus de l'IP.
- [ ] Le fichier `.pem` (clé SSH) du nouveau compte, téléchargé et
      accessible localement.
- [ ] Accès admin à l'organisation GitHub `DREAMHOUSE-237` (pour modifier
      les secrets).
- [ ] Accès au dashboard Campay (pour le webhook, étape 7).

---

## 1. Provisionner l'infrastructure AWS

### 1.1 Deux instances EC2

- Type `t3.micro` ou équivalent free tier, Ubuntu 22.04/24.04 LTS
- Une sera le nœud **gateway** (Traefik, config-service, registry-service,
  messagebroker-service), l'autre le nœud **app** (tous les microservices
  applicatifs) — voir `docs/architecture.md` pour le détail du découpage.
- Activer une IP publique pour les deux (adresse publique dynamique ou
  Elastic IP si tu veux une adresse stable).

### 1.2 Une instance RDS MySQL

- Moteur MySQL 8.x, tier gratuit (`db.t3.micro`), stockage minimal
- Accès public désactivé (ou restreint aux security groups des EC2)
- **TLS obligatoire** : noter l'endpoint RDS affiché après création
  (`<identifiant>.<id>.<region>.rds.amazonaws.com`) — il sera réutilisé
  dans plusieurs endroits (étape 5).
- Créer une base par service ayant besoin de MySQL : `auth_db`, `user_db`,
  `identity_db`, `payment_db`, `publication_db`, `commentary_db` (ou laisser
  chaque service créer la sienne via `CREATE DATABASE IF NOT EXISTS`).

### 1.3 Security Groups

Un seul security group partagé entre les deux EC2 (ou deux, avec règle
croisée) doit autoriser en entrée :

| Port(s) | Protocole | Source | Usage |
|---|---|---|---|
| 22 | TCP | Ton IP | SSH |
| 80, 443 | TCP | 0.0.0.0/0 | Traefik (HTTP→HTTPS redirect + HTTPS public) |
| 2377 | TCP | SG lui-même | Gestion cluster Swarm |
| 7946 | TCP + UDP | SG lui-même | Communication entre nœuds Swarm |
| 4789 | UDP | SG lui-même | Réseau overlay Swarm |
| 8761 | TCP | SG lui-même | Eureka (registry-service) |
| 8888 | TCP | SG lui-même | Config Server (config-service) |
| 5672, 15672 | TCP | SG lui-même | RabbitMQ (AMQP + console management) |
| 8081, 8082, 8083, 8085, 8086, 8087 | TCP | SG lui-même | Ports applicatifs (auth, user, commentary, publication, payment, identity) — nécessaires en cross-node pour le routage du gateway |

C'est l'oubli de ces règles (en particulier les ports applicatifs et
8761/8888/5672) qui avait causé la plupart des pannes lors de la dernière
migration — à ne pas sauter.

Sur le RDS : security group séparé, autorisant le port `3306` en entrée
depuis le security group des deux EC2.

---

## 2. Préparer les deux EC2

Sur **chacune** des deux instances :

```bash
ssh -i <ta-cle>.pem ubuntu@<ip-publique-ec2>

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
# se reconnecter pour que le groupe prenne effet

# Swap (les instances free tier manquent de RAM pour plusieurs JVM)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 2.1 Certificat TLS pour RDS

Sur l'EC2 **app** (celle qui exécute les services Django/Flask/Java qui se
connectent à RDS), dans le dossier où sera cloné `infrastructure` :

```bash
mkdir -p certs
curl -o certs/global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

Ce fichier est monté en lecture seule dans les conteneurs
(`./certs/global-bundle.pem:/certs/global-bundle.pem:ro`) — il n'est pas
versionné dans le repo, il doit être présent physiquement sur l'hôte avant
le premier déploiement.

### 2.2 Cloner le repo infrastructure

```bash
git clone https://github.com/DREAMHOUSE-237/infrastructure.git
cd infrastructure
```

---

## 3. Initialiser Docker Swarm

Voir `docs/architecture.md` pour le détail complet. Résumé :

```bash
# Sur le nœud manager (celui qui sera EC2_HOST)
docker swarm init --advertise-addr <IP_PRIVEE_DE_CETTE_EC2>

# Copier le "docker swarm join --token ..." affiché, puis sur l'autre EC2 :
docker swarm join --token <token> <ip-privee-manager>:2377

# Sur le manager
docker node ls
docker node update --label-add zone=app <NODE_ID_nœud_app>
docker node update --label-add zone=gateway <NODE_ID_nœud_gateway>
```

---

## 4. Mettre à jour les secrets GitHub

Secrets d'organisation `DREAMHOUSE-237` (Settings → Secrets and variables →
Actions → Organization secrets), visibilité "Public repositories" :

**Accès AWS / déploiement**
- `EC2_HOST` — IP ou DNS public du nœud **manager** (celui qui a fait `swarm init`)
- `EC2_USER` — `ubuntu`
- `EC2_SSH_KEY` — contenu du `.pem` de la nouvelle paire de clés
- `MYSQL_HOST` — nouvel endpoint RDS
- `MYSQL_PORT` — `3306`

**Identifiants par base**
- `USER_DB_NAME` / `USER_DB_USER` / `USER_DB_PASSWORD`
- `PAYMENT_DB_NAME` / `PAYMENT_DB_USER` / `PAYMENT_DB_PASSWORD`
- `IDENTITY_DB_NAME` / `IDENTITY_DB_USER` / `IDENTITY_DB_PASSWORD`
- `PUBLICATION_DB_USER` / `PUBLICATION_DB_PASSWORD`
- `DB_URL_AUTH`, `DB_URL_COMMENTARY` — URL de connexion complètes

**Autres services externes** (ne changent normalement pas avec la migration
AWS, mais à vérifier qu'ils sont bien présents sur le nouveau compte
GitHub/org si celui-ci change aussi)
- `RABBITMQ_USER` / `RABBITMQ_PASSWORD`
- `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET`
- `CAMPAY_TOKEN` (ou `CAMPAY_USERNAME` / `CAMPAY_PASSWORD`)
- `MAIL_USERNAME` / `MAIL_PASSWORD`
- `PAYMENT_SECRET_KEY`
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`
- `GH_PAT` — Personal Access Token avec droits *Contents + Metadata +
  Actions*, utilisé par chaque service pour déclencher le déploiement dans
  `infrastructure` après un merge sur `main`

Se référer à `infrastructure/.env.example` pour la liste vivante des
variables lues par `docker-compose.prod.yml`.

---

## 5. Adresses codées en dur à mettre à jour dans le code

Malgré la migration des secrets vers des variables d'environnement, un
certain nombre d'adresses restent codées en dur dans le code (choix fait
pour rester simple plutôt que de tout indirecter). Il faut les repasser à
la main à chaque migration :

| Fichier | Repo | Valeur à remplacer |
|---|---|---|
| `proxy-service.properties`, `publication-service.properties`, `authentification-default.properties`, `users-default.properties`, `payment-service.properties` | `config` | URL Eureka/Config Server (`eureka.client.service-url.defaultZone`, `spring.cloud.config.uri`) → nouvelle IP/DNS du nœud gateway |
| `src/main/resources/application.properties` | `registry-service`, `proxy-service`, `publication-service` | Même URL Eureka/Config Server, dupliquée **dans le repo du service lui-même** (bootstrap avant que le service sache parler à `config-service`) — ne pas oublier celle-ci, c'est elle qui avait causé des crash-loops silencieux la dernière fois |
| `docker-compose.prod.yml` (ligne `INSTANCE_HOST=...`) | `infrastructure` | IP **privée** du nœud app |
| `publication-service.properties` | `config` | IP privée du nœud app (`eureka.instance.hostname`, `eureka.instance.ip-address`, `eureka.instance.instance-id`) + endpoint RDS dans `spring.datasource.url` |
| `traefik-dynamic.yml` | `infrastructure` | Domaine du nœud gateway dans la règle de routage TLS (ex. `Host(\`ec2-<nouvelle-ip>.eu-north-1.compute.amazonaws.com\`)`) |
| `.env` | `mobile-app` | `API_URL` (nouveau DNS public du gateway) |
| Variable d'environnement frontend | Dashboard **Render** (hors repo) | URL de l'API consommée par le frontend web |
| `settings.py` (valeur de repli uniquement, jamais utilisée si `MYSQL_HOST` est bien configuré) | `auth-service` | Ancien endpoint RDS en fallback — à nettoyer par la même occasion |

Depuis le 18/09/2026, l'entrée publique (Traefik) utilise directement le
DNS public fourni par AWS (`ec2-<ip-tirets>.eu-north-1.compute.amazonaws.com`)
pour le certificat Let's Encrypt, **plus `nip.io`** — ce service DNS tiers
gratuit s'est révélé peu fiable (résolution DNS qui échoue ou traîne selon
le réseau du client, y compris hors VPN) et a causé des paiements bloqués
en silence côté frontend. Un seul format d'adresse (celui d'AWS, avec des
tirets) est donc utilisé partout désormais, en interne comme en externe —
plus de confusion entre deux formats différents.

---

## 6. Premier déploiement

Depuis le dossier `infrastructure` sur le nœud manager, avec les mêmes
variables d'environnement que celles exportées par `deploy.yml` (voir
`.env.example`, remplies avec les vraies valeurs) :

```bash
docker stack deploy -c docker-compose.prod.yml --resolve-image always --with-registry-auth dreamhouse
docker stack services dreamhouse
```

Ou plus simplement : pousser un commit trivial (ex. sur ce document) vers
`main` d'un des repos de service pour déclencher le pipeline CI/CD complet,
qui fera ce même déploiement automatiquement.

---

## 7. Mettre à jour le webhook Campay

Le webhook de confirmation de paiement n'est pas dans le code — il est
configuré côté [dashboard marchand Campay](https://www.campay.net) (ou
`demo.campay.net` en mode démo) :

1. Se connecter au dashboard
2. Paramètres de l'app → Webhook / Callback URL
3. Remplacer par `https://ec2-<nouvelle-ip>.eu-north-1.compute.amazonaws.com/payment-service/webhook/campay`

Si ce n'est pas fait, les paiements restent bloqués indéfiniment en
`PENDING` (Campay traite le paiement mais ne peut jamais notifier le
service).

---

## 8. Vérifications de bout en bout

```bash
# Tous les services sont Running (1/1)
docker stack services dreamhouse

# Eureka voit bien toutes les instances attendues
curl http://<ip-privee-gateway>:8761/eureka/apps -H "Accept: application/json"

# RabbitMQ répond et les queues existent
docker exec $(docker ps -q -f name=dreamhouse_messagebroker-service) rabbitmq-diagnostics -q ping

# Le gateway répond en HTTPS
curl https://ec2-<nouvelle-ip>.eu-north-1.compute.amazonaws.com/PUBLICATION-SERVICE/api/biens
```

Puis test fonctionnel complet depuis le frontend web (ou l'app mobile
rebuildée avec le nouveau `.env`) : inscription, connexion, publication
d'un bien, paiement des frais de publication (webhook Campay inclus).

---

## Checklist récapitulative

- [ ] 2 EC2 + 1 RDS provisionnées, Security Groups configurés (section 1)
- [ ] Docker + swap installés sur les deux EC2 (section 2)
- [ ] `certs/global-bundle.pem` présent sur l'EC2 app (section 2.1)
- [ ] Swarm initialisé, nœuds labellisés (section 3)
- [ ] Tous les secrets GitHub à jour (section 4)
- [ ] Toutes les adresses codées en dur mises à jour (section 5)
- [ ] Premier déploiement réussi, tous les services `1/1 Running` (section 6)
- [ ] Webhook Campay mis à jour (section 7)
- [ ] Vérifications de bout en bout passées (section 8)
