# System Landscape — superbooks on SAP BTP Cloud Foundry

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  Browser / End User                                                              ║
║                                                                                  ║
║    ┌──────────────────────┐   ┌──────────────────────┐                           ║
║    │   Browse Books       │   │   Manage Books        │                          ║
║    │   (Fiori Elements)   │   │   (Fiori Elements)    │                          ║
║    └──────────┬───────────┘   └──────────┬────────────┘                          ║
║               │  HTTPS                   │  HTTPS                                ║
╚═══════════════╪══════════════════════════╪═══════════════════════════════════════╝
                │                          │
╔═══════════════╪══════════════════════════╪══════════════════════════════════════╗
║  SAP BTP — Cloud Foundry Space           │                                      ║
║                                          │                                      ║
║  ┌───────────────────────────────────────▼──────────────────────────────────┐   ║
║  │  Application Frontend Service  (superbooks-app-front)                    │   ║
║  │  • Hosts UI ZIP bundles (Browse Books, Manage Books)                     │   ║
║  │  • Routes /odata/* → srv-api destination                                 │   ║
║  └────────────────────────────┬─────────────────────────────────────────────┘   ║
║                               │ OData / REST (srv-api destination)              ║
║  ┌────────────────────────────▼─────────────────────────────────────────────┐   ║
║  │  CAP Server  (superbooks-srv)                  Node.js · gen/srv         │   ║
║  │                                                                          │   ║
║  │  ┌─────────────────────────────┐  ┌─────────────────────────────────┐    │   ║
║  │  │  CatalogService             │  │  AdminService                   │    │   ║
║  │  │  srv/cat-service.cds/.js    │  │  srv/admin-service.cds/.js      │    │   ║
║  │  │  • Browse, filter books     │  │  • Draft-enabled CRUD           │    │   ║
║  │  │  • Discount logic           │  │  • @requires admin              │    │   ║
║  │  │  • submitOrder action       │  │                                 │    │   ║
║  │  └─────────────────────────────┘  └─────────────────────────────────┘    │   ║
║  │                                                                          │   ║
║  │  CDS Model  db/schema.cds   (Books · Authors · Genres)                   │   ║
║  └───────────┬─────────────────────────────────────────┬────────────────────┘   ║
║              │ HDI container binding (X.509/MTLS)      │ IAS binding (X.509)    ║
║              │                                         │                        ║
║  ┌───────────▼──────────────────────┐  ┌───────────────▼──────────────────────┐ ║
║  │  SAP HANA Cloud                  │  │  Cloud Identity Services (IAS)       │ ║
║  │  superbooks-db  (hdi-shared)     │  │  superbooks-ias  (identity·app plan) │ ║
║  │                                  │  │                                      │ ║
║  │  • Books, Authors, Genres tables │  │  • Issues JWT tokens (OAuth 2.0)     │ ║
║  │  • HDI container manages schema  │  │  • Delegates authz to xsuaa          │ ║
║  │  • Artifacts deployed by         │  │                                      │ ║
║  │    superbooks-db-deployer        │  │                                      │ ║
║  └──────────────────────────────────┘  └───────────────┬──────────────────────┘ ║
║                                                        │ xs-security.json       ║
║                                        ┌───────────────▼──────────────────────┐ ║
║                                        │  XSUAA                               │ ║
║                                        └──────────────────────────────────────┘ ║
║                                                                                 ║
╚═════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║  Deployment-time only  (one-off CF tasks / jobs — not running apps)              ║
║                                                                                  ║
║  ┌──────────────────────────────────┐  ┌───────────────────────────────────────┐ ║
║  │  superbooks-db-deployer          │  │  superbooks-ams-policies-deployer     │ ║
║  │  type: hdb · path: gen/db        │  │  type: nodejs · path: gen/policies    │ ║
║  │                                  │  │                                       │ ║
║  │  Pushes HANA artifacts (CSN →    │  │  Pushes DCL authorization policies    │ ║
║  │  HANA tables/views) into the     │  │  to AMS via IAS API.                  │ ║
║  │  HDI container, then exits.      │  │  Runs before superbooks-srv starts.   │ ║
║  └──────────────────────────────────┘  └───────────────────────────────────────┘ ║
║                                                                                  ║
╚══════════════════════════════════════════════════════════════════════════════════╝

Legend
──────
→   HTTP / OData request flow (runtime)
│   Service binding / dependency
X.509/MTLS  Mutual TLS certificate credential (no client secrets)
HDI         HANA Deployment Infrastructure — manages schema lifecycle
```
