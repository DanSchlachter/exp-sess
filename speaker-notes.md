# Expert Session Speaker Notes
## Deploying a CAP Application to Cloud Foundry

**Format:** YouTube / Expert Session  
**Duration:** ~60–90 minutes  
**Audience:** Developers new to CAP deployment on SAP BTP  

---

> **HOW TO USE THESE NOTES**  
> Each section maps 1:1 to the tutorial steps. The **[SAY]** blocks are your spoken script (paraphrase freely — don't read verbatim). The **[DO]** blocks are what you should do on screen. The **[SHOW]** blocks are what should be visible for the audience.

---

## Intro (2–3 min)

**[SAY]**  
"Welcome to this expert session on deploying a CAP application to Cloud Foundry on SAP BTP. I'm [your name], and today I'll walk you through the complete end-to-end process — from a working local CAP app all the way to a live deployment with a HANA Cloud database, IAS authentication, AMS authorization, and a Fiori Launchpad hosted by the Application Frontend service.

This session is aimed at developers who are comfortable writing CAP applications but are new to deploying them. We'll go step by step, explain every file and command, and you'll leave with a deployment that you can use as a template for your own projects.

Here's what we'll cover: [pause, point to the screen or list]
- Setting up SAP BTP trial with IAS and the Application Frontend service
- Provisioning SAP HANA Cloud
- Running `cds add` to wire up everything at once
- Walking through the generated MTA descriptor
- Building and deploying with `mbt build` and `cf deploy`
- Opening the live Fiori Launchpad

Let's get started."

**[DO]** Show the project in VS Code or a terminal with `cds watch` running.

---

## Part 1 — SAP BTP Trial Setup (10–12 min)

### 1.1 – Create BTP Trial Account

**[SAY]**  
"If you don't have an SAP BTP trial account yet, head to `cockpit.hanatrial.ondemand.com`. Sign in with your SAP Universal ID. One important thing: when you're asked to choose a region, pick **US East (VA)**. That's the only region where the Application Frontend service is available on trial right now."

**[DO]** Open BTP Cockpit. Show the trial subaccount overview. Point out the Cloud Foundry space that was auto-created.

**[SAY]**  
"Notice that Cloud Foundry is already enabled — a default org and space are created automatically. That's where we'll deploy."

---

### 1.2 – Subscribe to Cloud Identity Services

**[SAY]**  
"The first thing we need is an identity provider. We're going to use SAP Cloud Identity Services, also known as IAS. This replaces the older XSUAA-only approach and gives us a proper enterprise-grade identity service.

Go to **Services > Instances and Subscriptions**, click **Create**, choose **Cloud Identity Services**, plan **default**, service type **PRODUCTIVE**, and create it."

**[DO]** Walk through the subscription steps on screen.

**[SAY]**  
"After a minute or two, you'll get an activation email. Let me open that quickly..."

**[DO]** Show the activation email and set a password for the IAS admin console.

---

### 1.3 – Establish Trust

**[SAY]**  
"Now we need to connect our BTP subaccount to IAS. This is called establishing trust. Without this, BTP doesn't know about IAS and won't accept logins from it.

Go to **Security > Trust Configuration**, click **Establish Trust**, and select your IAS tenant."

**[DO]** Walk through the trust configuration wizard.

**[SAY]**  
"Once trust is established, any user in your IAS tenant can log in to services in this BTP subaccount. We'll use this later for the Application Frontend service and for our deployed app."

---

### 1.4 – Subscribe to Application Frontend Service

**[SAY]**  
"Now let's subscribe to the Application Frontend service. This is what will host our two Fiori Elements UI apps and give us a Fiori Launchpad. Think of it as a managed HTML5 hosting and launchpad service that's much simpler to set up than an AppRouter.

Same place — Instances and Subscriptions, Create, choose **Application Frontend Service**, plan **trial**, Create."

**[DO]** Walk through the subscription.

---

### 1.5 – Grant Permissions

**[SAY]**  
"Subscriptions alone don't give you access. You need to assign the right role collection to yourself. Let's create one called **Application Frontend Developer**, add the `Application_Frontend_Developer` role, and assign it to your user — both from the default identity provider for Cockpit access, and from the business users provider for CLI access."

**[DO]** Walk through role collection creation and user assignment. Sign out and back in.

**[SAY]**  
"After logging back in, you'll see a new item in the left nav: **HTML5 > Application Frontend**. That's how you know it's working."

---

### 1.6 – Install Application Frontend CLI

**[SAY]**  
"We'll also need the CLI. Run `npm install -g @sap/appfront-cli`. This lets us deploy apps, list apps, and manage the service from the terminal."

**[DO]** Show the terminal command and the version output.

---

## Part 2 — Provision HANA Cloud (8–10 min)

### 2.1 – Create HANA Cloud Instance

**[SAY]**  
"Our CAP app uses SQLite locally — that's great for development, fast and zero setup. But for production on Cloud Foundry we need a real database. SAP HANA Cloud is the database of choice for CAP applications on BTP.

Let's provision a new instance. Go to Instances and Subscriptions, Create, choose **SAP HANA Cloud**, plan **hana**. Give it a name — I'll use `superbooks-hana`. The wizard asks you to set a HANA System password — save that somewhere safe. For the network settings on trial, choose **Allow all IP addresses** to keep things simple.

Click Create. This takes a few minutes so let's move on while it provisions."

**[DO]** Start the HANA Cloud provisioning and then switch to the next section while it runs in the background.

**[SAY]**  
"One important point: HANA Cloud trial instances are stopped automatically every 12 hours. Before you deploy, always check that your instance is running in BTP Cockpit."

---

## Part 3 — Prepare the App (8–10 min)

### 3.1 – Show the App Structure

**[SAY]**  
"Let's look at what we're deploying. This is the `superbooks` app — a classic CAP bookshop with two services: a public **CatalogService** for browsing books, and an **AdminService** for managing them. There are two Fiori Elements UIs: `browse` and `admin-books`."

**[DO]** Open the project in VS Code. Show the folder structure: `db/`, `srv/`, `app/`, `package.json`.

**[SAY]**  
"The `db/schema.cds` defines our Books, Authors, and Genres entities. The `srv/` folder has the service definitions and the JavaScript handlers. The two UI apps under `app/` are Fiori Elements apps that will be deployed to the Application Frontend service.

Notice that there's no `mta.yaml` yet — we'll generate that in a moment."

---

### 3.2 – Add Deployment Configuration with `cds add`

**[SAY]**  
"The `package.json` currently only has `@sap/cds` and the SQLite dev dependency. Instead of manually installing packages and editing YAML files, CAP has a single command that wires everything up for us:

```
cds add hana,ams,app-front,mta
```

Let me run that now."

**[DO]** Run `cds add hana,ams,app-front,mta` in the project root. Show the output — which files were created and modified.

**[SAY]**  
"That one command did quite a lot. Let me show you each piece.

`cds add hana` added `@cap-js/hana` to `package.json` — that's the HANA database adapter. When the app runs in Cloud Foundry and detects a HANA service binding, CAP automatically switches from SQLite to HANA. Locally, `cds watch` still uses SQLite — no code changes needed.

`cds add ams` sets up authentication and fine-grained authorization in one shot. It automatically pulls in `cds add ias` first — that adds `@sap/xssec`, configures `'auth': 'ias'` in `package.json`, and adds the `superbooks-ias` identity service resource to the MTA. Then `ams` additionally adds `@sap/ams` and an `ams-policies-deployer` module to `mta.yaml`. That deployer runs once during deployment and pushes our DCL authorization policy files to AMS. No `xs-security.json` involved — authorization is expressed in DCL files compiled by `cds build`.

`cds add app-front` configured the two UI apps for deployment: it updated `ui5.yaml` in each app with the `ui5-task-zipper` build task, and generated `xs-app.json` routing files that point OData calls to the CAP backend and everything else to the app-front service.

`cds add mta` generated the `mta.yaml` — the blueprint for the entire Cloud Foundry deployment. And because we ran all four together, the MTA is already fully wired up: the HANA HDI container, the IAS identity service, the AMS policies deployer, and the app-front service — all connected and correctly ordered."

**[DO]** Open `package.json` and show `@cap-js/hana`, `@sap/xssec`, `@sap/ams` in dependencies, `cds.requires["auth": "ias"]` and `["app-front"]`. Open one of the `xs-app.json` files and highlight the `app-front` route. Then open `mta.yaml`.

**[SAY]**  
"Let's walk through the `mta.yaml` in detail."

---

## Part 4 — Understand the Generated `mta.yaml` (12–15 min)

### 4.1 – Walk Through `mta.yaml`

**[SAY]**  
"At the top we have the schema version and the MTA ID. The ID `superbooks` is used by the MTA deployer to track this deployment — so don't change it once deployed, or the deployer will create a new deployment instead of updating the existing one.

Then we have **modules**. These are the deployable units — think of them like apps or jobs in CF.

The first module is `superbooks-srv` — our CAP backend. It's a Node.js app that will be pushed to CF. It requires the HANA HDI container (`superbooks-db`) and the IAS identity service (`superbooks-ias`). Notice the binding uses `credential-type: X509_GENERATED` — that's mutual TLS, no client secrets. It also exposes itself as a **destination** called `srv-api` — the UI apps call the backend through this name. The `AMS_DCL_ROOT` env var tells CAP where to find the compiled AMS authorization policy files at runtime. And `deployed-after: superbooks-ams-policies-deployer` ensures the policies are pushed to AMS before the server starts.

The second module is `superbooks-db-deployer`. This is a special HDI deployer module. When deployed, it runs once, pushes our compiled database schema to HANA Cloud, and then exits. It's not a long-running app — just a deployment job.

The third module is `superbooks-ams-policies-deployer`. This also runs as a one-off CF task — `no-route: true`, `no-start: true`. It pushes the DCL authorization policy files from `gen/policies/` to AMS, then exits. These policies define who can do what — replacing the old `xs-security.json` scopes and role templates.

Then we have the two UI modules — `superbooksbrowse` and `superbooksadminbooks`. These are `html5` type modules. Notice the `build-parameters` section: `mbt build` will automatically run `npm ci` and `npm run build` inside each app folder — we don't need to build the UI apps manually.

Then we have **resources** — the backing services that get created in CF. The `superbooks-db` resource is the HDI container on HANA. The `superbooks-ias` resource is the IAS identity service — `service: identity`, `service-plan: application`, with `authorization: enabled: true` which activates AMS policy evaluation. And `superbooks-app-front` is the Application Frontend service — added automatically by `cds add app-front`."

**[DO]** Scroll through the file while explaining. Collapse and expand sections to keep the audience focused.

---

### 4.2 – (Optional) Show the CAP Console Alternative

**[SAY]**  
"Before we move on, let me quickly show you an alternative approach for those who prefer a visual tool. The CAP Console is a native desktop app — separate from VS Code — available for Windows and macOS. You open your project in it and there's a Deployment tab with a step-by-step wizard that guides you through configuring modules, services, and generating the MTA descriptor. Same result as the CLI, just more visual."

**[DO]** Briefly open the CAP Console desktop app, show the project loaded and the Deployment tab. Then close it and return to the terminal.

---

## Part 5 — Build and Deploy (10–12 min)

### 5.1 – Build with `mbt`

**[SAY]**  
"We're ready to build. `mbt build` does everything in one go: it runs `cds build` to compile the CDS models, builds the UI apps, and packages everything into a single `.mtar` archive file. That archive is what we hand to the MTA deployer."

**[DO]** Run `mbt build`. Show the output — it should list each step.

**[SAY]**  
"The archive is in the `mtar/` folder. You can think of it like a ZIP file that contains your entire multi-component application."

---

### 5.2 – Log In to Cloud Foundry

**[SAY]**  
"Now let's log in to CF. We use `cf login` with the API URL for US10 and the `--sso` flag, which opens a browser-based login. That's important because we're using IAS."

**[DO]** Run `cf login -a https://api.cf.us10-001.hana.ondemand.com --sso`. Show the browser login prompt. Select the org and space.

---

### 5.3 – Deploy

**[SAY]**  
"Now the moment of truth. `cf deploy` takes the `.mtar` archive and handles everything: creates the CF services, runs the HDI deployer, pushes the Node.js app, uploads the UI ZIPs. Watch the output — it shows each step as it happens."

**[DO]** Run `cf deploy mtar/superbooks_1.0.0.mtar`. Show the live output. Narrate the key steps as they appear.

**[SAY]**  
"...and there it is: `Process finished`. Let's verify with `cf apps` and `cf services`."

**[DO]** Run `cf apps` and `cf services`. Show that `superbooks-srv` is running and all services are created.

---

## Part 6 — Access the Application (8–10 min)

### 6.1 – Application Frontend CLI Login

**[SAY]**  
"Let's log in to the Application Frontend service with the CLI."

**[DO]** Run `afctl login`. Show the browser login. Then run `afctl apps list`.

**[SAY]**  
"You can see our two apps — `browse` and `admin-books` — are listed. The service is hosting them and has automatically registered them in the Fiori Launchpad."

---

### 6.2 – Open the Fiori Launchpad

**[SAY]**  
"Let's open the Fiori Launchpad. Navigate to your Application Frontend URL — you can find it in BTP Cockpit under HTML5 > Application Frontend, or just open the URL from the `afctl apps list` output. You'll log in via IAS and land on the launchpad."

**[DO]** Open the Fiori Launchpad in the browser. Show the two tiles: Browse Books and Manage Books.

**[SAY]**  
"Browse Books is open to all authenticated users. Let's click it."

**[DO]** Open Browse Books. Show the book list, filter, click into a book.

**[SAY]**  
"Now let's try Manage Books. That requires the `admin` role — let's assign it first."

---

### 6.3 – Assign Admin Role and Test

**[SAY]**  
"Back in BTP Cockpit, go to Security > Role Collections. You'll see an `admin` role collection was created automatically during deployment. Open it, click Edit, add your user, save."

**[DO]** Walk through role assignment.

**[SAY]**  
"Refresh the Fiori Launchpad — and now Manage Books is accessible. Let's click it."

**[DO]** Open Manage Books. Create a new book using the draft workflow. Save.

**[SAY]**  
"And there it is — the book is saved in HANA Cloud. Full end-to-end, live on Cloud Foundry."

---

## Part 7 — Troubleshooting Tips (2–3 min)

**[SAY]**  
"Before we wrap up, let me mention the most common issues you might hit and how to fix them.

If the `cf deploy` fails at the HANA step, your HANA Cloud trial instance is probably stopped — just start it in BTP Cockpit.

If the app starts but OData calls return 503, the HDI deployer probably failed. Run `cf logs superbooks-db-deployer --recent` to see what went wrong.

If you get a login loop or redirect error, check that the IAS trust is properly established in Part 1.3.

If Manage Books gives a 403, you just need to assign the `admin` role collection to your user."

---

## Outro (2–3 min)

**[SAY]**  
"And that's a wrap! Let's recap what we did today:

We set up an SAP BTP trial with IAS and the Application Frontend service. We provisioned HANA Cloud. We ran `cds add hana,ams,app-front,mta` to wire up HANA, IAS/AMS authentication, the Application Frontend service, and generate the `mta.yaml` descriptor in one go — no manual YAML editing needed. We built the MTA archive with `mbt build` and deployed it with `cf deploy`. And we opened the live Fiori Launchpad and verified the whole thing works end to end.

The `mta.yaml` and project config we created today is a solid template — you can use it as a starting point for your own CAP projects.

If you want to go deeper, the next steps are: setting up a CI/CD pipeline so every push automatically rebuilds and redeploys; enabling multitenancy with `cds add multitenancy`; and adding audit logging.

All the files from this session are in the `superbooks` project. The tutorial document with all the commands is in `tutorial.md` at the root.

If you found this helpful, please like and subscribe, and leave a comment if you have questions. See you in the next one."

---

## Timing Checklist

| Section | Target time |
|---------|------------|
| Intro | 2–3 min |
| Part 1 — BTP Setup | 10–12 min |
| Part 2 — HANA Cloud | 8–10 min |
| Part 3 — `cds add` + Part 4 — MTA walkthrough | 15–18 min |
| Part 5 — Build & Deploy | 10–12 min |
| Part 6 — Access App | 8–10 min |
| Part 7 — Troubleshooting | 2–3 min |
| Outro | 2–3 min |
| **Total** | **~57–71 min** |

---

## Screen Layout Recommendations

- Use a **1920x1080** resolution for recording. Scale your IDE/terminal font to at least 16pt.
- Keep **VS Code** and **BTP Cockpit** open in two separate windows — use split screen or alt-tab between them.
- Use the **integrated terminal** in VS Code for all CLI commands so viewers can see code and commands side by side.
- When walking through `mta.yaml`, use **VS Code's YAML folds** to collapse and expand sections — it helps viewers focus.
- In BTP Cockpit, use the browser's **zoom to 110%** so UI elements are clearly visible.

---

## Common Questions (Q&A Prep)

**Q: Why IAS instead of XSUAA?**  
A: IAS is the strategic direction — it's an enterprise identity provider that works across SAP and third-party services, supports federation with corporate identity providers, and is the basis for SAP's Identity-as-a-Service offering. XSUAA is still supported but IAS is recommended for new projects.

**Q: Can I use this setup without Application Frontend service and use a standard AppRouter instead?**  
A: Yes. Instead of `cds add app-front`, run `cds add approuter` — it adds an AppRouter module to `mta.yaml` and configures `xs-app.json` accordingly.

**Q: Does HANA Cloud cost money on trial?**  
A: The trial includes a free HANA Cloud instance. However, it stops automatically every 12 hours and must be manually restarted.

**Q: Can I run `cds watch` with HANA locally?**  
A: Yes, with a HANA Cloud connection string in `.env`. But for development, SQLite is recommended — it's faster and requires zero setup.

**Q: What is `gen/` and where does it come from?**  
A: `gen/` is created by `cds build` (which `mbt build` runs automatically). It contains the compiled server code and HANA database artifacts — you never edit it directly.
