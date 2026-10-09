# Déploiement Carte Fédé

Ce chart déploie l'API Flask et le frontend Astro statique dans deux Deployments, avec deux Services ClusterIP. PostgreSQL et le Gateway sont fournis séparément.

Le nom du chart, du Deployment backend, de son Service et de ses sélecteurs est conservé pour permettre une mise à jour depuis la version 0.1.0. Le frontend possède des sélecteurs distincts.

## Images et ports

| Composant | Image | Port conteneur | Port Service |
| --- | --- | --- | --- |
| Backend | `image.repository:image.tag` | `containerPort`, défaut 8000 | `service.port`, défaut 8000 |
| Frontend | `frontend.image.repository:frontend.image.tag` | `frontend.containerPort`, défaut 8080 | `frontend.service.port`, défaut 80 |

Les deux tags sont des SHA publiés ensemble par le workflow du dépôt d'application. Les Services ciblent le port nommé `http`, indépendamment de leur propre port. Chaque Deployment reçoit automatiquement la variable `PORT` correspondant à son port conteneur ; changer les ports via ces valeurs Helm, pas via une entrée `env` nommée `PORT`.

L'image frontend utilise Nginx uniquement pour servir les fichiers. Elle ne connaît ni l'adresse du backend ni ses secrets. Les requêtes du navigateur conservent le même domaine.

## Routage

L'HTTPRoute dirige :

- `/api` et ses sous-chemins vers le backend, sans retirer le préfixe.
- `/verify` exactement vers le backend, pour les URLs présentes dans les QR.
- `/` et les autres chemins vers le frontend.

Le défaut utilise le Gateway `public` du namespace `traefik` et le domaine `carte.fede.fpms.ac.be`. Adapter `httpRoute.parentRefs` et `httpRoute.hostnames` aux ressources de l'environnement. Sans `sectionName`, la route peut s'attacher aux listeners compatibles du Gateway. Pour ne servir que HTTPS, renseigner le nom réel de son listener HTTPS.

Le certificat et la redirection HTTP vers HTTPS sont gérés par l'infrastructure Gateway. Le Gateway doit autoriser les routes provenant du namespace de l'application via `allowedRoutes`. Les noms de domaines et les ports réseau sont des valeurs Helm, car Kubernetes les utilise avant le démarrage des conteneurs.

`httpRoute.rules` permet de personnaliser les règles. Chaque règle accepte `matches`, `filters` et `backend: backend|frontend`. En l'absence de `backend`, le Service API est conservé comme cible pour compatibilité avec l'ancien chart.

## Configuration par environnement

Le chart ne crée pas de Secret, de ConfigMap applicative ou de base PostgreSQL. Toute configuration de l'application est injectée par `env` et `envFrom`, avec prise en charge de `valueFrom`.

Le backend conserve les clés racine `env` et `envFrom`. Le frontend utilise `frontend.env` et `frontend.envFrom`, réservées aux paramètres publics. Ne pas y référencer le Secret contenant les accès PostgreSQL et SMTP.

Exemple de valeurs d'environnement, les ressources référencées doivent déjà exister :

```yaml
envFrom:
  - secretRef:
      name: carte-fede-backend-env
frontend:
  envFrom:
    - configMapRef:
        name: carte-fede-frontend-env
```

| Variables backend | Usage |
| --- | --- |
| `DATABASE_URL`, `SECRET_KEY` | Obligatoires. URL PostgreSQL SQLAlchemy et clé stable de signature des sessions/jetons. |
| `FRONTEND_BASE_URL` | Origine publique pour les emails et les QR, par exemple `https://carte.fede.fpms.ac.be`. |
| `SESSION_COOKIE_SECURE`, `REMEMBER_COOKIE_SECURE` | `true` en HTTPS. `false` uniquement pour un environnement HTTP. |
| `WEB_CONCURRENCY`, `GUNICORN_CMD_ARGS` | Nombre de workers et options supplémentaires Gunicorn. |
| `PROXY_FIX_X_FOR`, `PROXY_FIX_X_PROTO`, `PROXY_FIX_X_HOST`, `PROXY_FIX_X_PORT`, `PROXY_FIX_X_PREFIX` | Nombre de proxies de confiance pour chaque en-tête. |
| `MAIL_ADDRESS`, `MAIL_PASSWORD`, `MAIL_FROM_NAME`, `SMTP_HOST`, `SMTP_PORT`, `SMTP_USE_TLS`, `SMTP_USE_SSL` | Configuration SMTP. |
| `AUTO_CREATE_DB` | Garder `0` sur une base existante ; appliquer les migrations avant le rollout. |

Le frontend reçoit `FRONTEND_BASE_URL` au démarrage. Ses pages, sitemap et robots.txt utilisent cette origine, sans reconstruire l'image. L'origine doit correspondre au domaine de l'HTTPRoute. Les appels API restent relatifs à `/api/`.

La valeur par défaut de l'image frontend est l'origine de production. La remplacer explicitement dans les autres environnements. Le frontend doit pouvoir écrire ses fichiers de configuration et les fichiers statiques générés au démarrage ; un système de fichiers entièrement en lecture seule demande des volumes adaptés.

## Disponibilité et accès

Les probes vérifient `/api/health` côté backend et `/_health` côté frontend. Le premier vérifie que Flask répond, pas que PostgreSQL est disponible.

Les ports applicatifs restent des ClusterIP. En cas de NetworkPolicy, autoriser le Gateway vers les deux Services et les sorties backend vers PostgreSQL, SMTP et DNS. Si GHCR est privé, renseigner `imagePullSecrets`.

Les migrations SQL du dépôt d'application sont manuelles. Le chart ne modifie pas la base et ne crée pas d'administrateur.

## Limites de requêtes

`rateLimit.enabled: true` crée deux middlewares Traefik et deux règles exactes prioritaires :

- Inscription : 5 requêtes par minute, capacité de rafale 5.
- QR de paiement : 10 requêtes par minute, capacité de rafale 5.

Ces limites exigent la CRD `Middleware` de `traefik.io/v1alpha1` et le provider `kubernetesCRD` de Traefik. Voir la [documentation ExtensionRef](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/gateway-api/#using-traefik-middleware-as-httproute-filter). Elles sont locales à chaque instance Traefik, pas globales au cluster. La source par défaut est l'IP du client direct ; configurer `rateLimit.sourceCriterion` si un autre proxy de confiance précède Traefik.

Désactiver `rateLimit.enabled` si un autre mécanisme d'infrastructure applique déjà ces limites. Aucune installation de CRD ou modification du Gateway n'est effectuée par le chart.
