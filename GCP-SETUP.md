# GCP Resource Setup Guide — Option 1 (Manual via Console)

> **Github Repo:** `https://github.com/kevin-naicker-dvt/study-jam-week3-monorepo`  
> **Project:** `dvt-lab-devfest-2025`  
> **Project Number:** `882266340372`  
> **Region:** `africa-south1`  
> **Console URL:** https://console.cloud.google.com/welcome?project=dvt-lab-devfest-2025

---

## Prerequisites

- A Google Cloud account with billing enabled (free $300)
- Owner or Editor role on the project
- GitHub repository connected to GCP (see Step 0)

---

## Step 0 — Connect GitHub Repository to GCP

1. Go to **Cloud Build > Repositories** in the GCP Console
2. Click **Connect Repository**
3. Select **GitHub** as the source provider
4. Authenticate with GitHub and select your repository: `study-jam-week3-monorepo`
5. Click **Connect**

---

## Step 1 — Enable Required APIs (If Brand new cloud account + project)

Go to **APIs & Services > Library** and enable:

| API | Purpose |
|-----|---------|
| Cloud Build API | CI/CD builds |
| Cloud Run API | Container hosting |
| Artifact Registry API | Docker image storage |
| Cloud SQL Admin API | Managed PostgreSQL |
| Secret Manager API | Secure secrets storage |
| Cloud Resource Manager API | Project management |

**How to enable:** Search for each API name → Click **Enable**

---

## Step 2 — Create Artifact Registry

1. Go to **Artifact Registry > Repositories**
2. Click **+ Create Repository**
3. Fill in:
   - **Name:** `studyjam-repo`
   - **Format:** Docker
   - **Mode:** Standard
   - **Location type:** Region → `africa-south1`
   - **Encryption:** Google-managed
4. Click **Create**

---

## Step 3 — Create Cloud SQL (PostgreSQL) Instance

1. Go to **SQL > Create Instance**
2. Select **PostgreSQL (Sandbox)**
3. Fill in:
   - **Instance ID:** `studyjam-db`
   - **Password:** *(set a strong password — save it for Secret Manager)*
   - **Database version:** PostgreSQL 15
   - **Region:** `africa-south1`
   - **Zone:** Single zone (for cost savings)
4. Under **Machine type:** Choose `db-f1-micro` (for dev/testing is the cheapest!)
5. Under **Connections:**
   - Enable **Private IP** (VPC: default) - 10.74.0.3
   - Disable Public IP (for security)
6. Click **Create Instance** *(takes ~5 minutes)*

### Create the Database

1. Once the instance is running, click on `studyjam-db`
2. Go to **Databases > Create Database**
   - **Name:** `studyjam`
3. Go to **Users > Add User Account**
   - **Username:** `studyjam_user`
   - **Password:** *(set a strong password — save it)*
   - **Host:** `%` (any host)

---

## Step 4 — Store Secrets in Secret Manager

1. Go to **Secret Manager > Create Secret**

### Secret 1: DB Password
- **Name:** `studyjam-db-password`
- **Secret value:** *(the DB password you set above)*
- Click **Create Secret**

### Secret 2: JWT Secret
- **Name:** `studyjam-jwt-secret`
- **Secret value:** *(generate a strong random string, e.g. 64 chars)*
- Click **Create Secret**

> **Tip:** Generate a JWT secret: `openssl rand -base64 64`

---

## Step 5 — Create Service Accounts

This project uses **two dedicated service accounts** — one for the build pipeline, one for the running app. Using dedicated accounts follows least-privilege best practices.

### 5a — Cloud Build Service Account (runs the CI/CD pipeline)

1. Go to **IAM & Admin > Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:
   - **Name:** `studyjam-cloudbuild-sa`
   - **Description:** Cloud Build pipeline service account
4. Click **Create and Continue**
5. Grant these roles:
   - `Cloud Run Admin`
   - `Artifact Registry Writer`
   - `Service Account User`
   - `Secret Manager Secret Accessor`
   - `Logs Writer`
   - `Storage Object Viewer`
6. Click **Done**

### 5b — Cloud Run Service Account (runs the deployed app)

1. Click **+ Create Service Account** again
2. Fill in:
   - **Name:** `studyjam-cloudrun-sa`
   - **Description:** Cloud Run runtime service account
3. Click **Create and Continue**
4. Grant these roles:
   - `Cloud SQL Client`
   - `Secret Manager Secret Accessor`
5. Click **Done**

---

## Step 6 — Verify Service Account Roles in IAM

Confirm both service accounts created in Step 5 appear in IAM with the correct roles.

1. Go to **IAM & Admin > IAM**
2. Filter by `studyjam` — you should see both accounts:

   | Service Account | Roles |
   |----------------|-------|
   | `studyjam-build-sa@dvt-lab-devfest-2025.iam.gserviceaccount.com` | Cloud Run Admin, Artifact Registry Writer, Service Account User, Secret Manager Secret Accessor, Logs Writer, Storage Object Viewer |
   | `studyjam-cloudrun-sa@dvt-lab-devfest-2025.iam.gserviceaccount.com` | Cloud SQL Client, Secret Manager Secret Accessor |

If any roles are missing, click the pencil icon on the row and add them.


## Step 7 — Create Cloud Build Trigger

1. Go to **Cloud Build > Triggers**
2. Click **+ Create Trigger**
3. Fill in:
   - **Name:** `studyjam-deploy`
   - **Event:** Push to a branch
   - **Repository:** `kevin-naicker-dvt/study-jam-week3-monorepo` (if not listed, click "Connect new repository" and authenticate with GitHub)
   - **Branch:** `^gcp/dev$` — this is our GCP build branch, do not use main
   - **Configuration:** Cloud Build configuration file (YAML)
   - **File location:** `cloudbuild.yaml`
   - **Service account:** Select `studyjam-build-sa@dvt-lab-devfest-2025.iam.gserviceaccount.com` (the dedicated build SA created in Step 5a)

   > **Note:** There are two service accounts in this project — do not confuse them:
   > - `studyjam-build-sa` → selected here, **runs the CI/CD build pipeline**
   > - `studyjam-cloudrun-sa` → **runs the deployed app** on Cloud Run (already set in `cloudbuild.yaml`, no action needed here)

4. Under **Substitution variables**, add:

   | Variable | Value |
   |----------|-------|
   | `_REGION` | `africa-south1` |
   | `_REPO_NAME` | `studyjam-repo` |
   | `_BACKEND_SERVICE` | `studyjam-backend` |
   | `_FRONTEND_SERVICE` | `studyjam-frontend` |
   | `_DB_HOST` | *(Cloud SQL private IP — see SQL instance page)* | 10.74.0.3
   | `_DB_NAME` | `studyjam` |
   | `_DB_USER` | `studyjam_user` |
   | `_DB_PASSWORD_NAME` | `studyjam-db-password` |
   | `_JWT_SECRET_NAME` | `studyjam-jwt-secret` |
   
5. Click **Create**, then run

---

## Step 8 — Trigger First Deployment

> **Do this before running migrations.** The database tables don't exist yet — migrations create them.

1. Push a commit to the **`gcp/dev`** branch of your GitHub repository (this is the branch the trigger watches, **not** `main`):
   ```bash
   git checkout -b gcp/dev   # if the branch doesn't exist yet
   git push origin gcp/dev
   ```
2. Go to **Cloud Build > History** and click the running build to watch the logs
3. Wait for **all 8 build steps** to complete successfully (takes ~5–10 minutes):

   | # | Cloud Build Step | What it does |
   |---|-----------------|--------------|
   | 1 | `build-backend` | Builds the backend Docker image |
   | 2 | `push-backend` | Pushes the image to Artifact Registry |
   | 3 | `deploy-backend` | Deploys `studyjam-backend` to Cloud Run |
   | 4 | `health-check-backend` | Polls `/health` up to 10 times — confirms backend is live |
   | 5 | `get-backend-url` | Reads the backend URL for injection into the frontend build |
   | 6 | `build-frontend` | Builds the frontend image with `VITE_API_URL` baked in |
   | 7 | `push-frontend` | Pushes the frontend image to Artifact Registry |
   | 8 | `deploy-frontend` | Deploys `studyjam-frontend` to Cloud Run |

4. When the build goes green, both services are live but **the database is empty** — proceed to Step 9.

---

## Step 9 — Run Database Migrations

> Migrations must run **after** Step 8 completes. They create the tables in Cloud SQL that the backend expects.

### Recommended: Cloud SQL Auth Proxy (run from your local machine)

This is the simplest and safest approach — no need to touch the running Cloud Run service.

1. [Download Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/connect-auth-proxy#install) for your OS and place it in your project root.
2. Start the proxy (it creates a local tunnel to Cloud SQL on port 5432):
   ```bash
   ./cloud-sql-proxy dvt-lab-devfest-2025:africa-south1:studyjam-db &
   ```
3. Set up your local backend `.env`:
   ```bash
   cd backend
   cp .env.example .env
   ```
   Edit `.env` with:
   ```
   DB_HOST=127.0.0.1
   DB_PORT=5432
   DB_NAME=studyjam
   DB_USER=studyjam_user
   DB_PASSWORD=<the password from Secret Manager>
   ```
4. Run migrations:
   ```bash
   npm run db:migrate
   ```
5. Stop the proxy when done: `kill %1` (or find its PID with `jobs -l`)

---

### Alternative: Cloud Run Command Override

Use this only if you cannot run migrations locally. This temporarily redeploys the backend with a different startup command.

> **Warning:** This deploys a new revision that runs the migration script instead of the normal API server. Cloud Run may flag it as unhealthy because the process exits after migration completes. You **must** redeploy again afterwards to restore normal operation.

1. Go to **Cloud Run > studyjam-backend**
2. Click **Edit & Deploy New Revision**
3. Under the **Container** tab, find **Container command** and set it to:
   ```
   node dist/database/migrate.js
   ```
   (Leave all other settings — env vars, secrets, service account — unchanged)
4. Click **Deploy** and watch the logs in **Logs** tab — wait for the migration output to confirm success
5. Once migrations complete, click **Edit & Deploy New Revision** again and **clear** the Container command field (leave it blank to restore the default `npm start` entrypoint from the Dockerfile)
6. Click **Deploy** again to bring the normal API server back online

---

## Step 10 — Access Your App

After deployment completes:

1. Go to **Cloud Run**
2. Click `studyjam-backend` → copy the URL (e.g. `https://studyjam-backend-xxxxx-uc.a.run.app`)
3. Click `studyjam-frontend` → copy the URL
4. Open the frontend URL in your browser

---

## Deployment Architecture

```
GitHub Push (gcp/dev)
       │
       ▼
 GCP Cloud Build
       │
       ├─► Build Backend Docker Image
       │         │
       │         ▼
       │   Artifact Registry
       │   (africa-south1)
       │         │
       │         ▼
       │   Cloud Run: studyjam-backend
       │   (Health Check: /health)
       │
       ├─► Build Frontend Docker Image
       │   (VITE_API_URL injected from backend URL)
       │         │
       │         ▼
       │   Artifact Registry
       │         │
       │         ▼
       └─► Cloud Run: studyjam-frontend
                 │
                 ▼
           Users Browser
                 │ API calls
                 ▼
       Cloud Run: Backend API
                 │
                 ▼
       Cloud SQL: PostgreSQL
       (africa-south1)
```

---

## Cost Estimates (africa-south1)

| Resource | Tier | Est. Monthly Cost |
|----------|------|-------------------|
| Cloud Run (backend) | min-instances=0 | ~$0-5 |
| Cloud Run (frontend) | min-instances=0 | ~$0-5 |
| Cloud SQL | db-f1-micro | ~$10-15 |
| Artifact Registry | <1GB storage | ~$0-1 |
| Cloud Build | 120 free mins/day | ~$0 |

> **Note:** Costs scale with usage. Cloud Run scales to zero when not in use.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Build fails at push step | Check Artifact Registry permissions for Cloud Build SA |
| Backend fails health check | Check Cloud SQL IP in `_DB_HOST` variable |
| Frontend shows API errors | Verify `VITE_API_URL` in Cloud Run env vars |
| 403 Forbidden on Cloud Run | Ensure `--allow-unauthenticated` flag is set |
| DB connection refused | Check Cloud SQL Client role on service account |
