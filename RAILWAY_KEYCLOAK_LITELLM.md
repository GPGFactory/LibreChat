# Déploiement LibreChat sur Railway (Docker) avec Keycloak et LiteLLM

## 1) Créer le service Railway (Docker)

1. Créer un nouveau projet Railway.
2. Ajouter un service depuis ce dépôt (mode Docker, `Dockerfile` racine).
3. Ajouter une base MongoDB (plugin Railway) ou connecter une MongoDB externe.
4. Configurer un domaine public (ex: `chat.example.com`).
5. Vérifier que le service expose le port `3080`.

## 2) Préparer les configurations

1. Copier `railway.env.example` dans les variables Railway (sans commiter de secrets).
2. Ajuster les valeurs:
   - `DOMAIN_SERVER` / `DOMAIN_CLIENT` avec le domaine Railway final.
   - `MONGO_URI` avec la base cible.
   - `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`.
   - variables Keycloak (`OPENID_*`).
   - variables LiteLLM (`LITELLM_BASE_URL`, `LITELLM_API_KEY`).
3. Copier `railway.librechat.example.yaml` vers `librechat.yaml` à la racine du projet avant build/deploy.

## 3) Configurer Keycloak (OIDC)

Créer/mettre a jour un client OIDC Keycloak:

- Redirect URI: `https://chat.example.com/oauth/openid/callback`
- Web Origin: `https://chat.example.com`
- Standard Flow: active
- Access Type: confidential (client secret requis)

Reporter ensuite:

- `OPENID_CLIENT_ID`
- `OPENID_CLIENT_SECRET`
- `OPENID_ISSUER` (`https://<keycloak>/realms/<realm>`)
- `OPENID_SESSION_SECRET`

## 4) Configurer LiteLLM comme gateway unique

Dans `librechat.yaml`:

- Endpoint unique `endpoints.custom[0]` nomme `litellm`
- `baseURL` = `${LITELLM_BASE_URL}` (format OpenAI-compatible, suffixe `/v1`)
- `apiKey` = `${LITELLM_API_KEY}`
- `models.fetch: true` pour synchroniser dynamiquement la liste des modeles depuis LiteLLM
- `titleModel: current_model` pour eviter tout modele statique

Verrouillage providers:

- Ne pas definir de section endpoint native (OpenAI, Anthropic, Google, etc.)
- Garder les cles API natives vides (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_KEY`, `ASSISTANTS_API_KEY`)

## 5) Validation staging (obligatoire avant prod)

1. **Auth Keycloak**
   - Ouvrir `/login` sur LibreChat.
   - Verifier la redirection Keycloak.
   - Verifier callback et creation de session.
2. **Catalogue modeles**
   - Comparer la liste de modeles UI LibreChat avec `GET /models` sur LiteLLM.
   - Confirmer qu'aucun modele/provider natif n'apparait.
3. **Inference**
   - Tester 2-3 modeles depuis LibreChat.
   - Verifier le routage dans les logs LiteLLM.
4. **Regression**
   - Verifier login/logout.
   - Verifier absence d'option email/password si SSO-only.

## 6) Notes d'exploitation

- Rotation periodique des secrets (`OPENID_CLIENT_SECRET`, `LITELLM_API_KEY`, JWT, `OPENID_SESSION_SECRET`).
- Restriction stricte des redirect URIs et origins dans Keycloak.
- Monitoring centralise des logs Railway + Keycloak + LiteLLM.
