# Langfuse v3 sur Railway : paramétrage et sécurité

Guide opérationnel pour déployer [Langfuse v3](https://langfuse.com/self-hosting) via le [template Railway](https://langfuse.com/self-hosting/railway), corriger les problèmes d’URL (liens internes / `NEXTAUTH_URL`) et éviter les identifiants par défaut. Documentation officielle : [configuration](https://langfuse.com/self-hosting/configuration), [initialisation headless](https://langfuse.com/self-hosting/administration/headless-initialization), [stockage S3](https://langfuse.com/self-hosting/infrastructure/blobstorage).

Pour connecter LibreChat à votre instance une fois prête, renseignez dans `.env` : `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_BASE_URL` (URL publique HTTPS de Langfuse, sans slash final superflu).

---

## 1. Secrets applicatifs (avant ou juste après le premier déploiement)

Ne pas réutiliser les valeurs d’exemple du template. Générer localement puis copier dans **Railway → Project ou Service → Variables** (ou références secrètes).

| Variable | Génération suggérée |
|----------|---------------------|
| `NEXTAUTH_SECRET` | `openssl rand -base64 32` |
| `SALT` | `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | `openssl rand -hex 32` (exactement 64 caractères hex) |

Sous Windows, `openssl` est souvent disponible via Git for Windows ou peut être utilisé depuis WSL.

Ces trois variables sont requises pour Langfuse self-hosted. Appliquez-les aux services qui exécutent l’application Langfuse (web et worker), selon le template.

---

## 2. Domaine public et `NEXTAUTH_URL` (après déploiement)

1. Sur le service **web** Langfuse : **Settings → Networking** → générer un domaine `*.up.railway.app` ou attacher un domaine personnalisé.
2. Définir **`NEXTAUTH_URL=https://<votre-domaine-exact>`** — même schéma, même host que l’URL utilisée dans le navigateur (voir [FAQ débogage](https://langfuse.com/faq/all/debug-docker-deployment)).
3. Répéter la **même** valeur sur le **worker** (et tout service du template qui embarque la même image Langfuse avec ces variables). Les références Railway du type `${{service.RAILWAY_PUBLIC_DOMAIN}}` peuvent rester vides tant que le domaine n’existe pas, ce qui produit une URL invalide du type `https://` seul et fait échouer le démarrage ([discussion Railway](https://station.railway.com/questions/langfuse-v3-template-fails-to-start-a2def377)).
4. Redéployer après modification des variables.

Si l’app n’est pas joignable : selon l’environnement, `HOSTNAME=0.0.0.0` peut être nécessaire (voir doc Langfuse).

---

## 3. Stockage S3 / MinIO (médias, exports)

- Les **URLs pré-signées** doivent pointer vers un endpoint **accessible par le navigateur et les SDK** (pas uniquement un hostname interne au réseau privé Railway).
- **Multimodal** : ajuster `LANGFUSE_S3_MEDIA_UPLOAD_*` pour que l’endpoint utilisé dans les URLs signées soit public ou exposé comme prévu (voir section MinIO dans [blob storage](https://langfuse.com/self-hosting/infrastructure/blobstorage)).
- **Exports batch** : si activés, configurer `LANGFUSE_S3_BATCH_EXPORT_*` et, si besoin, `LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT` pour les liens de téléchargement côté client.

---

## 4. Comptes et clés sans défauts faibles

**Option A — Création via l’UI**  
Premier utilisateur créé dans l’interface avec un mot de passe fort (selon les paramètres d’inscription de votre instance).

**Option B — Initialisation headless (recommandé pour l’automatisation)**  
Variables `LANGFUSE_INIT_*` : organisation, projet, utilisateur, clés API (`LANGFUSE_INIT_PROJECT_PUBLIC_KEY`, `LANGFUSE_INIT_PROJECT_SECRET_KEY` avec préfixes attendus type `lf_pk_` / `lf_sk_`). Définir `LANGFUSE_INIT_USER_PASSWORD` uniquement via les secrets Railway, pas dans un dépôt. Hiérarchie et ordre de dépendance : [Headless Initialization](https://langfuse.com/self-hosting/administration/headless-initialization). Sous Docker Compose, ne pas mettre de guillemets superflus autour des valeurs (voir doc).

**Post-installation**  
Rotater les mots de passe des services du template (Postgres, Redis, ClickHouse, MinIO, etc.) si des valeurs par défaut ont été utilisées. Optionnel : `LANGFUSE_CSP_ENFORCE_HTTPS=true` si tout est servi en HTTPS. Provisionnement avancé : [Instance Management API](https://langfuse.com/self-hosting/administration/instance-management-api).

---

## 5. Redis (template Railway)

Si le conteneur Redis ne démarre pas (image ou tag introuvable sur Docker Hub), mettre à jour l’image vers un tag maintenu (par ex. une image `redis` officielle avec tag supporté) puis redéployer ; un Redis indisponible peut empêcher la résolution correcte des autres variables.

---

## 6. Références rapides

- [Langfuse — Railway](https://langfuse.com/self-hosting/railway)
- [Variables d’environnement](https://langfuse.com/self-hosting/configuration)
- [S3 / blob storage](https://langfuse.com/self-hosting/infrastructure/blobstorage)
