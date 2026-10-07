# Workspace with Traefik Forward Auth (OIDC/Keycloak Security)

This guide explains how to use the workspace container with Traefik reverse proxy
and OIDC authentication via Keycloak and traefik-forward-auth for secure
multi-user deployments in the DTaaS installation.

## ❓ Prerequisites

✅ Docker Engine v27 or later
✅ Sufficient system resources (at least 2GB RAM - includes Keycloak)  
✅ Port 80 available on your host machine  

## 🗒️ Overview

The `compose.traefik.secure.yml` file sets up:

- **Traefik** reverse proxy on port 80
- **Keycloak** identity provider with OIDC support
- **Oathkeeper** — JWT proxy; validates tokens and forwards authenticated
  requests to workspaces
- **login-relay** — lightweight login relay service; initiates the Keycloak
  authorization code flow and sets the `dtaas_access_token` cookie after
  successful authentication
- **client** - DTaaS web interface
- **user1** workspace using the workspace image
- **user2** workspace using the mltooling/ml-workspace-minimal image
- Two Docker networks: `dtaas-frontend` and `dtaas-users`

## ⚙️ Initial Configuration

Please follow the steps in [`CONFIGURATION.md`](CONFIGURATION.md)
for the `compose.traefik.secure.yml` composition before building
the workspace and running the setup.

## :rocket: Start Services

To start all services (Traefik, Keycloak, auth, client, and workspaces):

```bash
docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml --env-file workspaces/test/dtaas/config/.env up -d
```

This will:

1. Start the Traefik reverse proxy on port 80
2. Start Keycloak identity provider at `/auth`
3. Start Oathkeeper JWT proxy (validates tokens, routes workspace traffic)
4. Start the login-relay service at `/login-relay`
5. Start the DTaaS web client interface
6. Start workspace instances for both users

**Note**: First-time startup may take a few minutes for Keycloak to initialize.

After starting the services, Keycloak must be configured.
Follow the steps below, replacing `<DOMAIN_NAME>` with either either your
domain name if testing remotely, or `localhost` if testing locally.

### Access Keycloak Admin Console

1. Navigate to `http://<DOMAIN_NAME>/auth`
2. Login with credentials from your `.env` file (default: `admin` / `changeme`)

### Create a Realm

1. In the left sidebar, click **Manage realms**.
2. You are taken to the "Manage realms" page - click the **Create realm** button.
3. **Realm name**: `dtaas` (you can set a different name. If you do so, make so
  make sure that the value of `KEYCLOAK_REALM` in the
  [environment file `config/.env`](./config/.env) matches this name.)
4. Click **Create**

### Create the workspace Client

1. In the left sidebar, click **Clients**
2. Click **Create client**
3. Configure the client:
   - **Client type**: OpenID Connect
   - **Client ID**: `dtaas-workspace` (you can set a different id. If you do so, make so
  make sure that the value of `KEYCLOAK_CLIENT_ID` in the
  [environment file `config/.env`](./config/.env) matches this id.)
   - Click **Next**
4. Capability config:
   - **Client authentication**: **ON** (confidential client)
   - Authorization: OFF
   - Authentication flow: enable **Standard flow**
   - Click **Next**
5. Login settings:
   - **Root URL**: `http://<DOMAIN_NAME>`
   - **Valid redirect URIs**: `http://<DOMAIN_NAME>/login-relay/callback`
   - **Valid post logout redirect URIs**: `http://<DOMAIN_NAME>/*`
     *(required — login-relay redirects to `/` after logout; without this
     Keycloak will show "Invalid redirect uri")*
   - **Web origins**: `http://<DOMAIN_NAME>`
   - Click **Save**
6. Get the client secret:
   - Go to the **Credentials** tab
   - Copy the **Client secret** value
   - Update `KEYCLOAK_CLIENT_SECRET` in your [environment file `config/.env`](./config/.env) with this secret
7. Add an Audience mapper so the JWT's `aud` claim contains `dtaas-workspace`
   - Go to the **Client scopes** tab
   - Click `dtaas-workspace-dedicated`
   - Click **Configure a new mapper** → **Audience**
   - Set **Name**: `dtaas-workspace-audience`
   - Set **Included Client Audience**: `dtaas-workspace`
   - **Add to access token**: ON
   - Click **Save**

### Create the DTaaS web Client

1. In the left sidebar, click **Clients**
2. Click **Create client**
3. Configure the client:
   - **Client type**: OpenID Connect
   - **Client ID**: `dtaas-client`
   - Click **Next**
4. Capability config:
   - **Client authentication**: OFF (public client — no secret)
   - **Authorization**: OFF
   - **Authentication flow**: enable **Standard flow** only
   - **Require PKCE**: ON
   - **PKCE Method**: S256
   - Click **Next**
5. Login settings:
   - **Root URL**: `http://<DOMAIN_NAME>`
   - **Valid redirect URIs**: `http://<DOMAIN_NAME>/*`
   - **Valid post logout redirect URIs**: `http://<DOMAIN_NAME>/*`
   - **Web origins**: `http://<DOMAIN_NAME>`
   - Click **Save**
6. Add a `username` claim mapper so the SPA receives the username in the token:
   - Go to the **Client Scopes** tab
   - Click **`dtaas-client-dedicated`**
   - Click **Configure a new mapper** → **User Property**
   - Fill in:
     - **Name**: `username`
     - **Property**: `username`
     - **Token Claim Name**: `username`
     - **Claim JSON Type**: `String`
     - **Add to ID token**: ON
     - **Add to access token**: ON
     - **Add to userinfo**: ON
   - Click **Save**

### Create Users

1. In the left sidebar, click **Users**
2. Click **Create new user**
3. Fill in user details:
   - **Username**: `user1` (or desired username - make sure this matches the username set during [configuration](#️-initial-configuration))
   - **Email**: user's email (optional)
   - **First name** / **Last name**: optional
   - **Email verified**: OFF
4. Click **Create**
5. Set password:
   - Go to the **Credentials** tab
   - Click **Set password**
   - Enter a password
   - **Temporary**: OFF (so users don't have to change it on first login)  
   - Click **Save**
6. Repeat for additional users (e.g., `user2`)

### Restart Services

After configuring Keycloak, restart the auth services so they pick up the
new realm and client configuration.

```bash
docker compose -f workspaces/test/dtaas/compose.traefik.secure.tls.yml \
  --env-file workspaces/test/dtaas/config/.env \
  up -d --force-recreate oathkeeper login-relay
```

## :technologist: Accessing Workspaces

Once all services are running and Keycloak is configured, access them through Traefik at `http://<DOMAIN_NAME>`.

### Initial Access

1. Navigate to `http://<DOMAIN_NAME>` in your web browser
2. You will be redirected to Keycloak for authentication
3. Log in with a user you created in Keycloak
4. You will be redirected back to the DTaaS web interface

### Keycloak Admin Console

- **URL**: `http://<DOMAIN_NAME>/auth`
- Access to manage users, roles, clients, and authentication settings
- Login with `KEYCLOAK_ADMIN` credentials from `.env`

### DTaaS Web Client

- **URL**: `http://<DOMAIN_NAME>/`

### User1 Workspace

All endpoints require authentication:

- **VNC Desktop**: `http://<DOMAIN_NAME>/user1/tools/vnc`
- **VS Code**: `http://<DOMAIN_NAME>/user1/tools/vscode`
- **Jupyter Notebook**: `http://<DOMAIN_NAME>/user1`
- **Jupyter Lab**: `http://<DOMAIN_NAME>/user1/lab`

#### Service Discovery

The workspace provides a `/services` endpoint that returns a JSON list of
available services. This is intended for future dynamic service discovery
for frontend applications.

**Example**: Get service list for user1

```bash
curl http://<DOMAIN_NAME>/user1/services
```

**Response**:

```json
{
  "desktop": {
    "name": "Desktop",
    "description": "Virtual Desktop Environment",
    "endpoint": "tools/vnc"
  },
  "vscode": {
    "name": "VS Code",
    "description": "VS Code IDE",
    "endpoint": "tools/vscode"
  },
  "notebook": {
    "name": "Jupyter Notebook",
    "description": "Jupyter Notebook",
    "endpoint": ""
  },
  "lab": {
    "name": "Jupyter Lab",
    "description": "Jupyter Lab IDE",
    "endpoint": "lab"
  }
}
```

The endpoint values are dynamically populated with the user's username from the
`MAIN_USER` environment variable. This variable corresponds to `USERNAME1` of
`.env` file.

### User2 Workspace

- **VNC Desktop**: `http://<DOMAIN_NAME>/user2/tools/vnc`
- **VS Code**: `http://<DOMAIN_NAME>/user2/tools/vscode`
- **Jupyter Notebook**: `http://<DOMAIN_NAME>/user2`
- **Jupyter Lab**: `http://<DOMAIN_NAME>/user2/lab`

### Logging Out
When logged in as one of the users:

- **URL**: `https://<DOMAIN_NAME>/logout`

## 🛑 Stopping Services

To stop all services:

```bash
docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml \
  --env-file workspaces/test/dtaas/config/.env down
```

## 🔧 Customization

### Adding More Users

Follow these steps to add a third user (replace `user3` with the actual username
throughout).

**1. Add the new username to `config/.env`:**

```bash
USERNAME3=user3
# Keep WORKSPACE_USERS in sync — add the new name to the comma-separated list.
WORKSPACE_USERS=user1,user2,user3
```

**2. Add a new workspace service to `compose.traefik.secure.yml`:**

```yaml
  user3:
    image: workspace:latest
    restart: unless-stopped
    build:
      context: .
      dockerfile: ../Dockerfile.ubuntu.noble.xfce
    environment:
      - MAIN_USER=${USERNAME3:-user3}
    volumes:
      - "./files/common:/workspace/common"
      - "./files/${USERNAME3:-user3}:/workspace"
    shm_size: 512m
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.u3.entryPoints=web"
      - "traefik.http.routers.u3.rule=Host(`${SERVER_DNS}`) && PathPrefix(`/${USERNAME3:-user3}`)"
      - "traefik.http.routers.u3.middlewares=oathkeeper-auth"
    networks:
      - users
```

Add the desired `USERNAME3` variable in `.env`:

```bash
# Username Configuration
# These usernames will be used as path prefixes for user workspaces
# Example: http://localhost/user1, http://localhost/user2
USERNAME1=user1
USERNAME2=user2
USERNAME3=user3 # <--- replace "user3" with your desired username
```

**3. Add an access rule in `oathkeeper/access-rules.yml`:**

```yaml
- id: dtaas-user3-workspace
  version: "v0.36.0-beta.1"
  description: >
    Requires a valid Keycloak JWT whose username matches the path prefix
    (${USERNAME3}). The remote_json authorizer delegates the per-user check
    to login-relay, which returns 200 only when the token belongs to that user.
  match:
    url: "<^https?://[^/]*/${USERNAME3}(/.*)?$>"
    methods:
      - GET
      - HEAD
      - POST
      - PUT
      - PATCH
      - DELETE
      - OPTIONS
  authenticators:
    - handler: oauth2_introspection
      config:
        token_from:
          header: Authorization
    - handler: oauth2_introspection
      config:
        token_from:
          cookie: dtaas_access_token
  authorizer:
    handler: remote_json
    config:
      remote: http://login-relay:8080/authz/workspace/${USERNAME3}
      payload: '{"subject":{"id":{{ .Subject | toJson }},"extra":{{ .Extra | toJson }}}}'
  mutators:
    - handler: header
```

Replace `user3` in `upstream.url` with the Docker Compose service name of the
workspace container (it must match the service name, not `${USERNAME3}`, because
Oathkeeper resolves it as a DNS name at runtime).
The `${USERNAME3}` placeholders elsewhere are substituted at Oathkeeper startup
by the compose entrypoint.

**Note**: Oathkeeper v26 with `regexp` matching strategy requires that URL
patterns do not overlap. Verify no other rule's pattern matches the same URLs.

**Note**: The `remote_json` authorizer is required for per-user enforcement —
it ensures that only the user whose username matches the path prefix can access
that workspace. Using `allow` instead would let any authenticated user access
any workspace.

**4. Create the user directory:**

```bash
cp -r ./workspaces/test/dtaas/files/user1 ./workspaces/test/dtaas/files/user3
sudo chown -R 1000:100 workspaces/test/dtaas/files
```

**5. Create the user in Keycloak** (see [Configure Keycloak - Create Users](#create-users)).

**6. Redeploy:**

```bash
docker compose -f workspaces/test/dtaas/compose.traefik.secure.tls.yml \
  --env-file workspaces/test/dtaas/config/.env \
  up -d --build --force-recreate
```

## :shield: Security Considerations

### Current Setup (Development/Testing)

⚠️ **Important**: This configuration is designed for development and testing
and uses some insecure settings:

- `INSECURE_COOKIE=true` - Allows cookies over HTTP
- Traefik API is exposed (`--api.insecure=true`)
- No TLS/HTTPS encryption
- Debug logging enabled

For setting up a composition that includes TLS/HTTPS, see [TRAEFIK_TLS.md](TRAEFIK_TLS.md).

## 🔍 Troubleshooting

### Keycloak "HTTPS Required" Error

If Keycloak displays "We are sorry... HTTPS required" when accessed via HTTP:

1. This is caused by the per-realm `sslRequired` setting (defaults to `external`),
   which rejects HTTP from non-localhost clients
2. Fix it by disabling SSL requirement on the affected realm(s) — see
   [Disable Realm SSL Requirement](#disable-realm-ssl-requirement-http-only) above
3. If you previously ran the TLS composition (`compose.traefik.secure.tls.yml`),
   the `keycloak-data` volume may retain old SSL settings. Remove it and restart:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml \
     --env-file workspaces/test/dtaas/config/.env down
   docker volume rm dtaas_keycloak-data
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml \
     --env-file workspaces/test/dtaas/config/.env up -d
   ```

   Then re-apply the SSL disable steps after Keycloak starts.

### Keycloak Not Accessible

1. Check Keycloak is running:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml \
     --env-file workspaces/test/dtaas/config/.env ps keycloak
   ```

2. Check Keycloak logs:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml \
     --env-file workspaces/test/dtaas/config/.env logs keycloak
   ```

3. Wait for Keycloak to fully start (first startup can take 1-2 minutes)

### Authentication Loop

If you're stuck in an authentication loop:

1. Clear browser cookies for localhost
2. Check that `OAUTH_SECRET` is set and consistent
3. Verify Keycloak client redirect URI matches `http://localhost/_oauth/*`
4. Check traefik-forward-auth logs for errors
5. Ensure `KEYCLOAK_ISSUER_URL` is correct

### Services Not Accessible

1. Check all services are running:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml ps
   ```

2. Check Traefik logs:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml logs traefik
   ```

3. Check traefik-forward-auth logs:

   ```bash
   docker compose -f workspaces/test/dtaas/compose.traefik.secure.yml logs traefik-forward-auth
   ```

### OIDC/OAuth Errors

If you see OIDC errors:

1. Verify all environment variables in `.env` are correct
2. Check Keycloak client settings (client ID, secret, redirect URIs)
3. Ensure Keycloak realm name matches `KEYCLOAK_REALM`
4. Verify client authentication is enabled in Keycloak
5. Check that the issuer URL is accessible from the traefik-forward-auth container

### "Invalid Client" Error

- Verify `KEYCLOAK_CLIENT_SECRET` matches the value in Keycloak
- Ensure client authentication is enabled in Keycloak client settings

## 📚 Additional Resources

- [KEYCLOAK_SETUP.md](KEYCLOAK_SETUP.md) - Detailed Keycloak setup guide
- [KEYCLOAK_MIGRATION.md](KEYCLOAK_MIGRATION.md) - Migration guide from GitLab OAuth
- [CONFIGURATION.md](CONFIGURATION.md) - General configuration guide
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [traefik-forward-auth GitHub](https://github.com/thomseddon/traefik-forward-auth)
- [OIDC Specification](https://openid.net/specs/openid-connect-core-1_0.html)
- [DTaaS Documentation](https://github.com/INTO-CPS-Association/DTaaS)
