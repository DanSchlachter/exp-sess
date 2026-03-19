# Expert Session: Deploying a CAP Application to Cloud Foundry

**Duration:** ~60–90 minutes  
**Level:** Beginner  
**Application:** `superbooks` — a CAP bookshop app with two Fiori Elements UIs  
**Target Platform:** SAP BTP Cloud Foundry  

---

## What You Will Learn

By the end of this session you will be able to:

- Set up and configure an SAP BTP trial subaccount for Cloud Foundry deployment
- Provision SAP HANA Cloud as the production database
- Set up Cloud Identity Services (IAS) as the identity provider and SAP Authorization Management Service (AMS) for fine-grained authorization
- Configure the Application Frontend service to host your UI apps
- Use `cds add` to wire up HANA, IAS/AMS authentication, the Application Frontend service, and generate the `mta.yaml` descriptor in one command
- Optionally use the CAP Console desktop app as a visual alternative for deployment configuration
- Build and deploy your CAP application as a Multi-Target Application (MTA)

---

## Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Node.js | >= 22.6.0 | `node --version` |
| npm | >= 10 | `npm --version` |
| @sap/cds-dk | >= 9 | `cds --version` |
| Cloud Foundry CLI (`cf`) | >= 8 | `cf --version` |
| MTA Build Tool (`mbt`) | >= 1.2 | `mbt --version` |
| Application Frontend CLI | latest | `afctl --version` |

Install the tools you are missing:

```bash
# MTA Build Tool
npm install -g mbt

# Application Frontend CLI
npm install -g @sap/appfront-cli

# CF CLI — download from https://github.com/cloudfoundry/cli/releases
```

> **Starting point:** Clone or open the `superbooks` project. Run `npm install` and then `cds watch` to verify it works locally before continuing.

---

## Part 1 — SAP BTP Trial Setup

### 1.1 Create an SAP BTP Trial Account

1. Go to [https://cockpit.hanatrial.ondemand.com](https://cockpit.hanatrial.ondemand.com).
2. Sign in with your SAP Universal ID or create one at [https://account.sap.com/core/create](https://account.sap.com/core/create).
3. When prompted, create the trial subaccount in the **US East (VA)** region (required for Application Frontend service).
4. Click **Go To Your Trial Account**.

> A Cloud Foundry environment and a default space are automatically created.

### 1.2 Subscribe to Cloud Identity Services (IAS)

The Application Frontend service requires IAS as the identity provider.

1. In your trial subaccount, navigate to **Services > Instances and Subscriptions**.
2. Click **Create** and configure:
   - **Service:** Cloud Identity Services
   - **Plan:** default
3. Click **Next**, choose **Service type: PRODUCTIVE**, then **Create**.

> You will receive an activation email. Open it and activate your IAS account by setting a password — you will use this to log in to the IAS Admin Console later.

### 1.3 Establish Trust Between BTP and IAS

1. Go to **Security > Trust Configuration**.
2. Click **Establish Trust**.
3. Select your IAS tenant (e.g. `xxxxxxxx.trial-accounts.ondemand.com`) and complete the wizard.

> This allows your BTP subaccount to delegate authentication to IAS.

### 1.4 Subscribe to Application Frontend Service

The Application Frontend service hosts your HTML5/UI5 applications.

1. Go to **Services > Instances and Subscriptions**.
2. Click **Create** and configure:
   - **Service:** Application Frontend Service
   - **Plan:** trial
3. Click **Create**.

### 1.5 Grant Permissions

1. Go to **Security > Role Collections**.
2. Click **Create** and name it `Application Frontend Developer`.
3. Open the new role collection, click **Edit**, and add the role **Application_Frontend_Developer**.
4. Add your user from the **Default identity provider** (for BTP Cockpit access) and from the **business users** identity provider (for CLI/API access).
5. Click **Save**.
6. Sign out and sign back in to BTP Cockpit to activate the new role.
7. Verify that **HTML5 > Application Frontend** appears in the left navigation.

### 1.6 Install the Application Frontend CLI

```bash
npm install -g @sap/appfront-cli
afctl --version
```

---

## Part 2 — Provision SAP HANA Cloud

### 2.1 Create a HANA Cloud Instance

1. In your BTP subaccount, navigate to **Services > Instances and Subscriptions**.
2. Click **Create** and configure:
   - **Service:** SAP HANA Cloud
   - **Plan:** hana
   - **Instance name:** e.g. `superbooks-hana`
3. Follow the wizard:
   - Set an **HANA System password** (save it securely).
   - Choose **Allow all IP addresses** for the trial (simplest option).
4. Click **Create**. Provisioning takes a few minutes.

> Once provisioned, your HANA Cloud instance will be available in the same Cloud Foundry org and space.

### 2.2 Add HANA Cloud to Your CF Space

1. Open your HANA Cloud instance in the BTP Cockpit.
2. Navigate to **SAP HANA Cloud Central** (or use the "Manage SAP HANA Cloud" link).
3. Ensure the instance is mapped to your Cloud Foundry organization and space.

---

## Part 3 — Prepare the Application for Deployment

### 3.1 Add Deployment Configuration with `cds add`

CAP's `cds add` command is the recommended way to add deployment facets to your project. A single command wires up HANA, IAS/AMS authentication, the Application Frontend service, and generates the `mta.yaml` descriptor — updating `package.json`, `ui5.yaml`, `xs-app.json`, and the MTA descriptor in one consistent pass:

```bash
cds add hana,ams,app-front,mta
```

What each facet does:

- **`hana`** — adds `@cap-js/hana` to `package.json` dependencies (the HANA database adapter), creates/updates `db/undeploy.json`, and wires the HDI container module and resource into `mta.yaml`.
- **`ams`** — sets up authentication and authorization via IAS and AMS. It automatically includes `cds add ias` (which adds `@sap/xssec`, configures `"auth": "ias"` in `package.json`, and adds the `superbooks-ias` identity service resource to `mta.yaml`), then additionally adds `@sap/ams` + `@sap/ams-dev`, and adds an `ams-policies-deployer` module to `mta.yaml` that deploys authorization policies to AMS on each deployment. No `xs-security.json` is used — authorization is expressed in DCL policy files compiled by `cds build`.
- **`app-front`** — adds `cds.requires["app-front"]: true` to `package.json`, configures the `ui5-task-zipper` build task in each UI app's `ui5.yaml`, generates `xs-app.json` routing for each UI app (OData → `srv-api`, everything else → `app-front` service), and adds the `app-front` managed service and content deployer module to `mta.yaml`.
- **`mta`** — generates the `mta.yaml` descriptor with the CAP server module, HDI deployer module, and all service bindings.

> CAP is smart about this: when `@cap-js/hana` is installed and the app runs in Cloud Foundry with a HANA service binding, it automatically switches from SQLite to HANA. Locally, `cds watch` continues to use SQLite — no code changes needed.

> The UI apps (`app/browse`, `app/admin-books`) do **not** need to be built manually before packaging. The `mta.yaml` includes `build-parameters` with the build commands, and `mbt build` runs them automatically.

### Option B — Using the CAP Console (Native Desktop App)

The **CAP Console** is a native desktop app for Windows and macOS, separate from VS Code. It provides a visual interface for local development, deployment, and monitoring of CAP projects.

To use it as an alternative to the CLI:

1. Download and install the CAP Console from [cap.cloud.sap/docs/tools/console](https://cap.cloud.sap/docs/tools/console).
2. Open your `superbooks` project in the CAP Console.
3. Switch to the **Deployment** tab.
4. The step-by-step deployment wizard guides you through configuring modules and services and applying the MTA configuration — the same result as running `cds add hana,app-front,mta` from the terminal.

> Both approaches produce the same files. The CLI is faster; the CAP Console is more visual and useful for understanding the structure of the generated configuration.

## Part 4 — Understand the Generated `mta.yaml`

After running `cds add hana,ams,app-front,mta`, your `mta.yaml` will contain the following sections. Let's walk through each one.

#### `_schema-version` and `ID`

```yaml
_schema-version: "3.3.0"
ID: superbooks
version: 1.0.0
```

The root metadata. `ID` becomes the MTA's unique identifier in Cloud Foundry.

#### The CAP Server Module (`superbooks-srv`)

```yaml
modules:
  - name: superbooks-srv
    type: nodejs
    path: gen/srv
    requires:
      - name: superbooks-db
      - name: superbooks-ias
        parameters:
          config:
            credential-type: X509_GENERATED
            app-identifier: srv
    parameters:
      routes:
        - route: "${default-url}"
        - route: "${default-host}.cert.${default-domain}"
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}
          srv-cert-url: '${protocol}://${default-host}.cert.${default-domain}'
    properties:
      AMS_DCL_ROOT: ams/dcl
    deployed-after:
      - superbooks-ams-policies-deployer
```

- **`path: gen/srv`** — points to the compiled CAP server output (generated by `cds build`).
- **`requires: superbooks-ias`** — binds the IAS identity service using an X.509 certificate credential (MTLS) — no client secrets.
- **`routes`** — two routes are registered: the standard HTTP route and a `.cert.` subdomain route for mutual TLS.
- **`AMS_DCL_ROOT: ams/dcl`** — tells CAP where to find the compiled AMS policy files at runtime.
- **`deployed-after`** — ensures the AMS policies deployer runs and finishes before the server starts.

#### The HDI Deployer Module (`superbooks-db-deployer`)

```yaml
  - name: superbooks-db-deployer
    type: hdb
    path: gen/db
    requires:
      - name: superbooks-db
```

- **`type: hdb`** — a special MTA module type that runs the HDI deployer.
- Deploys the compiled database artifacts (CSN → HANA artifacts) into the HDI container.
- Runs once on deployment and then exits — it's not a long-running process.

#### The AMS Policies Deployer Module (`superbooks-ams-policies-deployer`)

```yaml
  - name: superbooks-ams-policies-deployer
    type: javascript.nodejs
    path: gen/policies
    parameters:
      buildpack: nodejs_buildpack
      no-route: true
      no-start: true
      tasks:
        - name: deploy-dcl
          command: npm start
          memory: 512M
    requires:
      - name: superbooks-ias
        parameters:
          config:
            credential-type: X509_GENERATED
            app-identifier: ams-policy-deployer
```

- **`path: gen/policies`** — compiled AMS authorization policies (DCL files produced by `cds build --for ams`).
- **`no-route: true` / `no-start: true`** — this module is not a running app; it executes as a one-off CF task.
- **`tasks`** — the CF task that pushes the DCL policies to AMS and then exits.
- This module runs *before* `superbooks-srv` (enforced by `deployed-after` on the server module).

#### The UI Modules

```yaml
  - name: superbooksbrowse
    type: html5
    path: app/browse
    build-parameters:
      build-result: dist
      builder: custom
      commands:
        - npm ci
        - npm run build
      supported-platforms: []

  - name: superbooksadminbooks
    type: html5
    path: app/admin-books
    build-parameters:
      build-result: dist
      builder: custom
      commands:
        - npm ci
        - npm run build
      supported-platforms: []
```

- **`type: html5`** — HTML5 modules are uploaded to the Application Frontend service.
- **`build-result: dist`** — MBT looks for the built output in the `dist/` folder after running the build commands.
- **`commands: [npm ci, npm run build]`** — MBT runs these automatically during `mbt build`. You do not need to build the UI apps manually beforehand.
- **`supported-platforms: []`** — tells MBT this module produces a ZIP for upload, not a running CF app.

#### The Services

```yaml
resources:
  - name: superbooks-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: superbooks-ias
    type: org.cloudfoundry.managed-service
    parameters:
      service: identity
      service-name: superbooks-ias
      service-plan: application
      config:
        display-name: superbooks
        provided-apis:
          - name: superbooks-ias-api
            description: API exposed by the application
        oauth2-configuration:
          token-policy:
            access-token-format: jwt
        authorization:
          enabled: true
        xsuaa-cross-consumption: true

  - name: superbooks-app-front
    type: org.cloudfoundry.managed-service
    parameters:
      service: app-front
      service-plan: developer
```

- **`superbooks-db`** — the HANA HDI container. Added by `cds add hana`.
- **`superbooks-ias`** — the IAS identity service instance. Added by `cds add ams` (via its `ias` prerequisite). Key config:
  - `authorization: enabled: true` — activates AMS policy evaluation for this IAS application.
  - `xsuaa-cross-consumption: true` — allows tokens from systems still using XSUAA to be accepted (migration bridge).
  - `access-token-format: jwt` — ensures IAS issues standard JWT tokens that CAP can process.
- **`superbooks-app-front`** — the Application Frontend service instance. Added by `cds add app-front`.

---

## Part 5 — Build and Deploy

### 5.1 Build the MTA Archive

```bash
mbt build
```

This runs:
1. `cds build` — compiles CDS models to `gen/srv`, `gen/db`, and `gen/policies` (AMS DCL authorization policies)
2. `npm ci && npm run build` in each UI app folder
3. Packages everything into `mtar/superbooks_1.0.0.mtar`

Check that the archive was created:

```bash
ls mtar/
```

### 5.2 Log In to Cloud Foundry

```bash
cf login -a https://api.cf.us10-001.hana.ondemand.com --sso
```

> Use `--sso` to authenticate via your browser (works with IAS). Follow the one-time passcode prompt.

Select your org and space when prompted.

### 5.3 Deploy the MTA

```bash
cf deploy mtar/superbooks_1.0.0.mtar
```

The deployer will:
1. Create and bind CF services (HANA HDI container, IAS identity service, Application Frontend)
2. Deploy the HDI artifacts to HANA Cloud
3. Run the AMS policies deployer — pushes DCL authorization policies to AMS
4. Push the `superbooks-srv` Node.js application
5. Upload the UI ZIP files to the Application Frontend service

Monitor the progress in the terminal. A successful deployment ends with:

```
Process finished.
```

### 5.4 Verify the Deployment

```bash
cf apps
cf services
```

You should see:
- `superbooks-srv` — started, with a route like `superbooks-srv-<random>.cfapps.us10-001.hana.ondemand.com`

```bash
cf app superbooks-srv
```

Check the **routes** output and open the URL in a browser.

---

## Part 6 — Access Your Application

### 6.1 Log In via the Application Frontend CLI

```bash
afctl login
```

Follow the browser-based IAS login. After login:

```bash
afctl apps list
```

You should see your deployed UI apps: `browse` and `admin-books`.

### 6.2 Open the Fiori Launchpad

The Application Frontend service generates a Fiori Launchpad for your deployed apps. Open it at:

```
https://<your-appfront-url>/cp.portal
```

You will see two tiles:
- **Browse Books** — read-only catalog (accessible to all authenticated users)
- **Manage Books** — admin CRUD UI (requires the `admin` role)

### 6.3 Assign the `admin` Role

With AMS, authorization is managed via IAS role collections. To grant admin access:

1. Go to BTP Cockpit > **Security > Role Collections**.
2. Open the `admin` role collection (created automatically by the AMS policies deployer).
3. Click **Edit**, add your IAS user.
4. Click **Save**.
5. Refresh the Fiori Launchpad — the **Manage Books** tile becomes active.

---

## Part 7 — Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `cf deploy` fails at HANA step | HANA Cloud instance not running | Start it in BTP Cockpit > HANA Cloud Central |
| App starts but OData returns 503 | HANA HDI deployment failed | Check `cf logs superbooks-db-deployer --recent` |
| Login loop / redirect error | IAS trust not established | Re-check Part 1.3 |
| `Manage Books` returns 403 | Missing `admin` role collection assignment | Follow Part 6.3 |
| `mbt build` fails at UI step | `package.json` missing in UI folder | Ensure `cds add app-front` was run; check `app/browse/package.json` exists |
| `mbt build` fails at policies step | `cds add ams` was not run | Run `cds add ams` and retry |
| `afctl login` fails | CLI not installed or role collection missing | Check Part 1.5–1.6 |

---

## Summary

You have successfully:

1. Set up an SAP BTP trial subaccount with IAS trust and Application Frontend service
2. Provisioned a HANA Cloud database
3. Run `cds add hana,ams,app-front,mta` to wire up HANA, IAS/AMS authentication, the Application Frontend service, and generate the `mta.yaml`
4. Built the MTA archive with `mbt build`
5. Deployed to Cloud Foundry with `cf deploy`
6. Accessed the Fiori Launchpad via the Application Frontend service

### Key Files Created / Modified

| File | Purpose |
|------|---------|
| `mta.yaml` | MTA descriptor — generated and fully wired by `cds add hana,ams,app-front,mta` |
| `package.json` | `@cap-js/hana`, `@sap/xssec`, `@sap/ams`; `cds.requires["auth": "ias"]` and `["app-front": true]` |
| `app/*/ui5.yaml` | Updated by `cds add app-front` with `ui5-task-zipper` build task |
| `app/*/xs-app.json` | Generated by `cds add app-front` with OData and app-front routing |

### Next Steps

- Set up **CI/CD** with SAP Continuous Integration and Delivery service to automate builds and deploys
- Enable **multitenancy** with `cds add multitenancy`
- Add **audit logging** with `cds add audit-logging`
- Explore **remote services** and event-driven integration with SAP Event Mesh
