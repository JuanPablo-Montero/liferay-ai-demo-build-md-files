# Server-Side Liferay Client Extensions & PaaS Deployment Guide

This guide covers the configuration, development, and deployment of server-side Client Extensions (CX) in Liferay—specifically **Object Actions**, **Workflow Actions**, and **Proxy Objects**—hosted on Liferay Cloud PaaS.

---

## 1. Core Concepts

### Object Actions (`objectAction`)
Object Actions allow you to offload custom logic (validations, external API calls, data enrichment) to an external microservice. 
- **Trigger**: When an event occurs on a Liferay Object (e.g., `OnAfterAdd`, `OnBeforeUpdate`).
- **Payload**: Liferay POSTs a JSON payload containing the object entry data to your microservice.

### Workflow Actions (`workflowAction`)
Workflow Actions integrate external logic into Liferay's workflow engine. 
- **Trigger**: When a workflow reaches a specific action node.
- **Payload**: Liferay POSTs a JSON payload containing the task ID, entry data, and transition URLs.
- **Response**: Your service can return a transition name (e.g., `approve`, `reject`) to automatically move the workflow forward.

---

## 2. Workspace Structure & Configuration

Server-side client extensions (like Node.js backends) require a specific workspace structure to ensure they are deployed correctly as standalone LCP services, while their definitions (YAML) are provided to DXP.

### Important: Workspace Setup
1. **Exclude from Standard Build**: Server-side client extensions should be ignored by the standard Git repository and DXP build process. Add a `.gitignore` in the `client-extensions/` directory:
   ```text
   # client-extensions/.gitignore
   my-node-backend/
   ricoh-user-etc-node/
   ```
2. **Configuration Files**: Every server-side extension directory must include:
   - `client-extension.yaml`: Definition of the OAuth app and actions.
   - `LCP.json`: Liferay Cloud PaaS deployment configuration.
   - `Dockerfile`: Container image definition.
   - `application.json`: DXP-specific configuration (ports, domains).
   - `package.json` & `index.js`: Node.js source code.

### Example `client-extension.yaml`

Server-side extensions must include metadata to allow DXP to communicate with the service during local development or in Cloud.

```yaml
assemble:
  - include:
      - '**/*.js'
      - application.json
      - package.json

# OAuth Application configuration
my-node-oauth-app:
    .serviceAddress: localhost:3500
    .serviceScheme: http
    name: My Node Backend OAuth App
    type: oAuthApplicationUserAgent
    scopes:
        - Liferay.Headless.Admin.User.everything

# Object Action configuration
my-custom-object-action:
    name: My Custom Action
    type: objectAction
    oAuth2ApplicationExternalReferenceCode: my-node-oauth-app
    resourcePath: /api/actions/my-endpoint
```

**Key YAML Requirements:**
1.  **`.serviceAddress` & `.serviceScheme`**: These metadata fields must be added to the OAuth application definition. They tell DXP where to reach the service for authorization and request signing.
2.  **`assemble` Exclusions**: The `assemble` block should *only* include the application source and configuration (`.js`, `application.json`, `package.json`). Do *not* include `Dockerfile` or `LCP.json`, as these are for Liferay Cloud infrastructure, not DXP's extension registry.
3.  **`oAuth2ApplicationExternalReferenceCode`**: The action must explicitly reference the local OAuth application defined in the same file.


### Example `application.json`
```json
{
  "com.liferay.lxc.dxp.domains": "liferay:8080",
  "com.liferay.lxc.dxp.main.domain": "liferay:8080",
  "com.liferay.lxc.dxp.server.protocol": "http",
  "ready.path": "/ready",
  "server.port": "3500"
}
```

### Example `LCP.json`
Crucial for Liferay Cloud deployment. Note the `id: "__PROJECT_ID__"` pattern and the required LXC environment variables. 
**IMPORTANT Constraint:** The `id` field maps directly to Kubernetes service names in Liferay Cloud. It **MUST be strictly alphanumeric and cannot start with a number** (no hyphens or underscores). Using the `__PROJECT_ID__` placeholder is highly recommended, as the DXP Gradle build will automatically sanitize the folder name and inject a compliant ID during the build process.

```json
{
  "cpu": 1,
  "env": {
    "NODE_ENV": "production",
    "LIFERAY_SERVER_URL": "https://webserver-your-env.lfr.cloud",
    "LIFERAY_ROUTES_CLIENT_EXTENSION": "/etc/liferay/lxc/ext-init-metadata",
    "LIFERAY_ROUTES_DXP": "/etc/liferay/lxc/dxp-metadata"
  },
  "environments": {
    "infra": {
      "deploy": false
    }
  },
  "id": "__PROJECT_ID__",
  "kind": "Deployment",
  "livenessProbe": {
    "httpGet": {
      "path": "/ready",
      "port": 3500
    }
  },
  "loadBalancer": {
    "targetPort": 3500
  },
  "memory": 512,
  "readinessProbe": {
    "httpGet": {
      "path": "/ready",
      "port": 3500
    }
  },
  "scale": 1
}
```

---

## 3. Deployment Workflow (Liferay Cloud)

Server-side extensions are deployed differently than standard frontend extensions. Follow this exact sequence:

1. **Build the Workspace**: Run the clean build command from the root or `liferay/` directory.
   ```bash
   ./gradlew clean build
   ```
2. **Navigate to Build Artifacts**: Once the build completes, artifacts are generated in the `dist/` directory at the root.
3. **Deploy via LCP CLI**: Use the Liferay Cloud `lcp` command to deploy the specific server-side extension from the `dist/` folder.
   ```bash
   cd dist/
   lcp deploy --extension ricoh-user-etc-node.zip
   ```
   *Note: If `lcp` is not in your path, use the full path to the binary. You may also need to specify the `--project` if it's not already configured.*

---

## 4. Microservice Implementation (Node.js Default)

For demo purposes, Node.js (with Express) is the standard and preferred framework due to its lightweight nature, fast spin-up times in PaaS, and native handling of Liferay's JSON payloads.

### Handling LXC Secrets (OAuth Credentials)

When you define an `oAuthApplicationHeadlessServer` in `client-extension.yaml`, Liferay Cloud automatically creates the OAuth application and injects its credentials into your microservice container. 

**CRITICAL:** Liferay Cloud does *not* inject these credentials as a single JSON file. It mounts a directory (typically `/etc/liferay/lxc/ext-init-metadata`) containing multiple plaintext files. Your application must explicitly parse the directory and read the `.client.id` and `.client.secret` files.

```javascript
const fs = require('fs');
const METADATA_PATH = process.env.LIFERAY_ROUTES_CLIENT_EXTENSION || '/etc/liferay/lxc/ext-init-metadata';

function getOAuthCredentials() {
    if (!fs.existsSync(METADATA_PATH)) return null;
    
    // Ensure we are reading a directory
    const stats = fs.statSync(METADATA_PATH);
    if (!stats.isDirectory()) return null;

    const files = fs.readdirSync(METADATA_PATH);
    let clientId = null;
    let clientSecret = null;

    // Search for the specific credential files injected by LXC
    const clientIdFile = files.find(f => f.endsWith('.oauth2.headless.server.client.id'));
    const clientSecretFile = files.find(f => f.endsWith('.oauth2.headless.server.client.secret'));

    if (clientIdFile) clientId = fs.readFileSync(`${METADATA_PATH}/${clientIdFile}`, 'utf8').trim();
    if (clientSecretFile) clientSecret = fs.readFileSync(`${METADATA_PATH}/${clientSecretFile}`, 'utf8').trim();

    return (clientId && clientSecret) ? { clientId, clientSecret } : null;
}
```

### Handling an Object Action (Express.js) & Payload Parsing

When Liferay triggers an Object Action Client Extension, it securely POSTs a JSON payload.

**CRITICAL Payload Structure (Liferay 7.4+):** The custom fields and relationship IDs are *not* located at the root of the `objectEntry` object. They are nested inside `objectEntry.values`.

```javascript
const express = require('express');
const axios = require('axios');
const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3500;
const LIFERAY_SERVER_URL = process.env.LIFERAY_SERVER_URL;

// Health check endpoint for LCP probes
app.get('/ready', (req, res) => {
    res.status(200).json({ status: 'UP' });
});

// Action endpoint matching client-extension.yaml resourcePath
app.post('/api/actions/my-endpoint', async (req, res) => {
    const entryId = req.body.objectEntryId || req.body.userId;
    const authHeader = req.headers.authorization;
    
    let token = null;
    if (authHeader && authHeader.startsWith('Bearer ')) {
        token = authHeader.replace('Bearer ', '');
    }

    if (!token || !entryId) {
        return res.status(400).json({ error: "Missing authentication or objectEntryId" });
    }

    try {
        // Example: Fetching user data using the token and Headless Admin User API
        const liferayApiUrl = `${LIFERAY_SERVER_URL}/o/headless-admin-user/v1.0/user-accounts/${entryId}`;
        const userResp = await axios.get(liferayApiUrl, {
            headers: { 'Authorization': `Bearer ${token}` }
        });

        // Perform your logic (e.g., updating a field)
        await axios.patch(liferayApiUrl, {
            additionalName: "Bob" 
        }, {
            headers: { 'Authorization': `Bearer ${token}` }
        });

        res.status(200).json({ message: "Success" });
    } catch (error) {
        console.error(`Error: ${error.message}`);
        res.status(500).json({ error: "Failed to process action" });
    }
});

app.listen(PORT, () => {
    console.log(`Microservice listening on port ${PORT}`);
});
```

---

## 5. Troubleshooting Checklist

1. **Check Ports**: Ensure `application.json`, `LCP.json`, and `index.js` all use the same port (e.g., `3500`).
2. **Verify LXC Variables**: Ensure `LIFERAY_ROUTES_CLIENT_EXTENSION` and `LIFERAY_ROUTES_DXP` are correctly set in `LCP.json`.
3. **Internal Routing**: Confirm `com.liferay.lxc.dxp.domains` is set to `liferay:8080` in `application.json` for internal cloud routing.
4. **Logs**: Use the Liferay Cloud Console to view the logs of your backend service. If you see a `SIGTERM` or `Bad Request` during deployment, check for health check (probe) failures or configuration mismatches.
5. **GitIgnore**: Ensure the server-side directory is in `client-extensions/.gitignore` to prevent DXP from zipping it.
