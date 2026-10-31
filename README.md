# k3s-gitops

Manifests du namespace `goodfood-dev`, synchronisés sur le cluster k3s. La CI
de chaque service met à jour la ligne `image:` de son `deployment.yaml`.

## Organisation

| Dossier | Contenu |
|---|---|
| `apps/back/<service>/` | `deployment.yaml`, `service.yaml`, `configmap.yaml`, base de données, `hpa.yaml` (HPA + PDB) |
| `apps/back/rabbitmq/` | Broker d'événements (`goodfood.events`) |
| `apps/back/redis/` | État partagé entre réplicas : rate limit d'auth (db 0), Socket.IO de delivery (db 1) |
| `apps/back/minio/` | Stockage objet des avatars (bucket `avatars`, servi par l'ingress sur `/avatars`) |
| `apps/front/` | Front web, ingress `app.c-mbk.fr` / `api.c-mbk.fr`, middleware CORS |
| `apps/security/` | NetworkPolicies |
| `apps/tools/` | Outils d'administration des bases |

## Scalabilité et disponibilité

- Chaque service applicatif a des **probes** (`startup`, `readiness`,
  `liveness`), des **requests/limits** et une **anti-affinité** préférée
  (réplicas répartis sur les nœuds).
- **HPA** : 2 à 10 réplicas sur 70 % de CPU ; le champ `replicas` est retiré
  des Deployments concernés pour ne pas contredire l'HPA à chaque sync.
- **PDB** : au moins un réplica disponible pendant un drain ou une mise à jour.
- `assistant-service` reste à **1 réplica** : il garde ses réglages IA dans un
  fichier local.
- Les services sont prêts pour plusieurs réplicas : outbox avec
  `FOR UPDATE SKIP LOCKED`, workers sous verrou consultatif PostgreSQL ou bail
  Mongo, migrations sous verrou, Socket.IO via Redis, rate limit via Redis.

## Sécurité réseau

`apps/security/network-policies.yaml` : tout trafic entrant est refusé par
défaut ; Traefik (`kube-system`) atteint les services ; les services se
parlent entre eux (les routes `/internal` exigent en plus `X-Service-Key`) ;
**chaque base n'est joignable que par son service** (et les outils d'admin).

> ⚠️ Après application, vérifier que les pods passent `Ready` : le contrôleur
> de NetworkPolicy de k3s autorise le trafic du nœud (sondes kubelet), mais
> un CNI différent peut exiger une règle `ipBlock` pour le réseau des nœuds.

## Secrets à créer (non versionnés)

| Secret | Clés |
|---|---|
| `auth-service-secret` | `DATABASE_URL`, `POSTGRES_PASSWORD`, `JWT_SECRET` (≥ 32 octets), `JWT_REFRESH_SECRET`, `SERVICE_API_KEY`, `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` (fournisseur réel : Brevo, Mailjet, SES…), `ADMIN_EMAIL` / `ADMIN_PASSWORD` (premier admin du siège), OAuth |
| `order-service-secret` | `DATABASE_URL`, `POSTGRES_PASSWORD`, `JWT_SECRET`, `SERVICE_API_KEY`, `RABBITMQ_URL`, `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` (factures par email, même fournisseur qu'auth-service) |
| `payment-service-secret` | `DATABASE_URL`, `POSTGRES_PASSWORD`, `JWT_SECRET`, `SERVICE_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `RABBITMQ_URL` |
| `promo-service-secret`, `menu-service-secret`, `stock-service-secret`, `franchise-service-secret` | base, `JWT_SECRET`, `SERVICE_API_KEY` |
| `delivery-service-secret` | `MONGODB_URI`, `JWT_SECRET`, `SERVICE_API_KEY`, `RABBITMQ_URL`, `IBAN_ENCRYPTION_KEY` (32 octets base64) |
| `user-service-secret` | base, `JWT_SECRET`, `SERVICE_API_KEY`, `S3_ACCESS_KEY` / `S3_SECRET_KEY` (= identifiants MinIO) |
| `minio-secret` | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` (≥ 8 caractères) |
| `reservation-service-secret`, `analytics-service-secret`, `assistant-service-secret` | base, `JWT_SECRET` |

`SERVICE_API_KEY` doit être **identique** dans tous les services qui
s'appellent par `/internal` ; `JWT_SECRET` identique partout ;
`JWT_REFRESH_SECRET` **uniquement** dans auth-service. Pour versionner les
secrets chiffrés, utiliser Sealed Secrets ou External Secrets.

Webhook Stripe à déclarer : `https://api.c-mbk.fr/api/payments/webhooks/stripe`
(événements `payment_intent.succeeded`, `payment_intent.payment_failed`,
`payment_intent.canceled`, `charge.refunded`).

## Ordre de déploiement de la release 1.0.0

1. Secrets complétés (ci-dessus), Redis, MinIO (le Job `minio-init` crée le
   bucket). Les services tournent en `ENV=production` : payment-service
   refuse de démarrer sans clés Stripe, user-service sans stockage objet,
   et auth-service ne crée plus les comptes de démonstration (seul le premier
   admin, depuis `ADMIN_EMAIL` / `ADMIN_PASSWORD`).
2. **auth-service** en premier : les autres services exigent désormais des
   access tokens typés (`typ`, `iss`, `aud`) ; les jetons émis avant expirent
   en 15 minutes et les utilisateurs se reconnectent une fois.
3. payment-service et promo-service (nouvelles routes internes), puis
   order-service, franchise-service, delivery-service, puis le reste.
4. Front web et application mobile.
5. NetworkPolicies en dernier, puis contrôle des probes.

## Étapes suivantes recommandées

- **Bases de données** : passer à un opérateur (CloudNativePG pour
  PostgreSQL, replica set MongoDB) ou à des bases managées — réplication,
  sauvegardes, PgBouncer. Les `Deployment` + PVC actuels ne sont pas
  conservés en `StatefulSet` pour ne pas toucher aux données existantes.
- **MongoDB 4.4** n'est plus maintenu : monter de version par étapes
  (4.4 → 5.0 → 6.0 → 7.0), une version majeure à la fois.
- **RabbitMQ** en cluster (quorum queues) via l'operator RabbitMQ.
- **Géocodage et autocomplétion d'adresses** (UE) : les instances publiques
  Nominatim (1 requête/s) et Photon conviennent au lancement ; au-delà, les
  héberger dans le cluster et pointer `GEOCODER_URL` / `ADDRESS_SUGGEST_URL`
  dessus (les réponses sont déjà mises en cache par franchise-service).
- **Observabilité** : order-service expose `/metrics` (Prometheus) ;
  ajouter un `ServiceMonitor` et étendre aux autres services.
