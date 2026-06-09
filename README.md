# Google SSO User Service

Spring Boot microservice for Google SSO login with optional Google-based signup.

For SEOWallet and SEO HUB replacement/integration details, see [docs/integration-guide.md](/Users/meherwerali/Documents/zmms/google-sso-user-service/docs/integration-guide.md).

## Endpoints

- `GET /api/v1/auth/google/url?clientKey=web&redirectUri={uri}&signup=false`
  - Creates a Google OAuth authorization URL and stores a short-lived state.
  - Set `signup=true` when the UI is intentionally starting a signup flow.
  - `clientKey` selects the Google OAuth client configuration, for example `web`, `android`, `ios`, or `chrome-extension`.
- `POST /api/v1/auth/google/callback`
  - Exchanges the Google `code` using the client stored with the OAuth `state`, fetches the Google profile, creates the user only when signup is enabled/requested, and returns local JWT tokens.
- `POST /api/v1/auth/signup`
  - Creates a local username/email/password account and sends an email verification link.
- `POST /api/v1/auth/login`
  - Logs in with username or email and password.
- `GET /api/v1/auth/verify-email?token={token}`
- `POST /api/v1/auth/forgot-password`
- `POST /api/v1/auth/reset-password`
- `POST /api/v1/auth/change-password`
  - Requires gateway-provided `X-User-Id`; Google SSO users can set a password here and then use both login methods.
- `GET /api/v1/users`
- `GET /api/v1/users/{id}`
- `POST /api/v1/profile/complete`
  - Adds one profile type as `SEO_EXPERT`, `AGENCY`, or `BUSINESS`.
  - `AGENCY` creates an agency profile and grants `AGENCY_ADMIN`.
  - `SEO_EXPERT` grants `INDIVIDUAL_EXPERT`.
  - `BUSINESS` creates a business profile and grants `BUSINESS_OWNER`; only `businessName` is required.
  - A user can call this later for another profile type, for example adding `BUSINESS` after already setting up `AGENCY`.
- `POST /api/v1/profile/switch`
  - Switches the active profile context to a profile the user has already set up.
- `GET /api/v1/profile/me`
- `GET /api/v1/agency/members`
- `POST /api/v1/agency/members`
  - Adds an expert to an agency with an `AGENCY_*` role.
- `DELETE /api/v1/agency/members/{membershipId}`
- `GET /api/v1/business/engagements`
- `POST /api/v1/business/engagements`
  - Lets a business hire an agency or individual expert.
- `POST /api/v1/business/engagements/{engagementId}/accept`
- `DELETE /api/v1/business/engagements/{engagementId}`
- `GET /api/v1/projects/access/issued`
- `GET /api/v1/projects/access/received`
- `POST /api/v1/projects/access`
  - Shares a specific project with limited `READ_ONLY`, `COLLABORATOR`, or `FULL` project-level access.
- `DELETE /api/v1/projects/access/{grantId}`
- `POST /api/v1/roles/{targetUserId}/{roleName}`
- `DELETE /api/v1/roles/{targetUserId}/{roleName}`
  - Assigns prefixed custom roles such as `BUSINESS_ANALYST`, `BUSINESS_MANAGER`, `AGENCY_SUPPORT`, or `EXPERT_ASSISTANT`.

Role-management endpoints expect the API gateway to forward the authenticated user id in `X-User-Id`.

## Required Environment

```bash
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_ALLOWED_REDIRECT_URIS=http://localhost:3000/auth/google/callback
GOOGLE_WEB_CLIENT_ID=...
GOOGLE_WEB_CLIENT_SECRET=...
GOOGLE_WEB_ALLOWED_REDIRECT_URIS=http://localhost:3000/auth/google/callback
GOOGLE_ANDROID_CLIENT_ID=...
GOOGLE_ANDROID_ALLOWED_REDIRECT_URIS=...
GOOGLE_IOS_CLIENT_ID=...
GOOGLE_IOS_ALLOWED_REDIRECT_URIS=...
GOOGLE_CHROME_EXTENSION_CLIENT_ID=...
GOOGLE_CHROME_EXTENSION_ALLOWED_REDIRECT_URIS=...
GOOGLE_SIGNUP_ENABLED=true
JWT_SECRET=replace-with-at-least-32-random-bytes
POSTGRES_JDBC_URL=jdbc:postgresql://localhost:5432/google_sso_user_service
POSTGRES_USERNAME=zoho_rf
POSTGRES_PASSWORD=zoho_rf
JPA_DDL_AUTO=update
LOCAL_AUTH_FRONTEND_BASE_URL=http://localhost:3000
LOCAL_AUTH_EMAIL_SENDING_ENABLED=false
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
```

This uses the same PostgreSQL server and credentials from the existing manifest, but keeps the Google SSO service in its own database: `google_sso_user_service`.

Create the database once on the existing Postgres container:

```bash
docker compose exec db psql -U zoho_rf -d zoho_rf_extension -c "CREATE DATABASE google_sso_user_service OWNER zoho_rf;"
```

When running this service as a Docker container on the same Docker network as your existing Postgres service, use `jdbc:postgresql://db:5432/google_sso_user_service`. The Dockerfile sets that as the container default and does not create a new Postgres container.

## Callback Request

```json
{
  "code": "google-auth-code",
  "state": "state-from-url-endpoint",
  "redirectUri": "http://localhost:3000/auth/google/callback"
}
```

When `signup=false`, only existing users can log in. When `signup=true` and `GOOGLE_SIGNUP_ENABLED=true`, a verified Google account is created as a user if it does not already exist.

## Google OAuth Clients

This service is the centralized identity service for SEOWallet and SEO HUB. Each client app gets its own Google OAuth client registration and calls the same backend using its `clientKey`.

- `web`: SEO HUB web app.
- `android`: SEO HUB Android app.
- `ios`: SEO HUB iOS app.
- `chrome-extension`: SEOWallet Chrome extension.

The OAuth state stores the selected `clientKey`, so callback handling never trusts the frontend to choose credentials during token exchange. Public clients such as Android, iOS, and Chrome extension can leave `client-secret` blank when their Google flow does not use a client secret.

## Role Model

- Primary roles are assigned when each profile is completed: `AGENCY_ADMIN`, `INDIVIDUAL_EXPERT`, or `BUSINESS_OWNER`.
- Users can hold multiple profile types at the same time, such as `AGENCY` and `BUSINESS`, and switch active context without creating a second login.
- JWT `accountType` is the active profile context. JWT `profileTypes` lists every completed profile.
- Agencies and individual experts have full product access for systems such as SEODebate and SEOWallet.
- Businesses store `businessName`, optional `domainName`, and optional `websiteUrl`, and can hire agencies or experts through engagements.
- Custom roles must use the owning account prefix: `BUSINESS_*`, `AGENCY_*`, or `EXPERT_*`.
- Project sharing is modeled separately from primary roles so hired staff can receive limited project access without broad platform access.
# user-service-sso
