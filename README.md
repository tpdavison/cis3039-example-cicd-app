# Products app — CI/CD example

A Vue and TypeScript single-page application that displays products from the Product API. GitHub Actions type-checks and builds the app, then deploys successful `main` builds to an Azure Storage static website in a separate Test environment.

## Development environment

Use your own private copy of this repository, following the course worksheet's private-copy procedure. You can choose your repository name freely. Work on `main` and check that `origin` points to your private repository:

```bash
git branch --show-current
git remote -v
```

The repository includes a [Dev Container configuration](.devcontainer/devcontainer.json) using the course image `ghcr.io/tpdavison/devops-labs:2026-latest`.

Install Docker, Visual Studio Code and the Dev Containers extension. Start Docker, open your local clone in VS Code, then select **Dev Containers: Reopen in Container** from the Command Palette. Run the commands below in the container terminal, from the repository root. The container setup also configures Git's safe-directory and file-mode settings for Windows hosts.

If working outside the container, use Node.js 22 and npm. Use the container terminal for the shell commands shown here.

## Run locally

Install the dependency versions recorded in the lockfile:

```bash
npm ci
```

For a new clone, create your local configuration file:

```bash
cp .env.example .env.local
```

If `.env.local` already exists, keep your settings. It is ignored by Git and must not be committed. The example contains:

```dotenv
VITE_API_BASE_URL=http://localhost:7071/api/
```

### Start the Product API

Use your private copy of the [Product service example](https://github.com/tpdavison/cis3039-example-cicd-svc) and follow its README to install dependencies and create its local settings file.

In the **backend repository's terminal**, start the service with the frontend's browser origin allowed by CORS:

```bash
npm start -- --cors http://localhost:5173
```

The `--cors` option configures allowed origins on the local Functions host; see the [Azure Functions Core Tools reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-core-tools-reference#func-start). Keep the backend running. If you already have a local backend running, stop it before restarting with this option.

### Start the frontend

In the **frontend repository's terminal**, run:

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser. When using a Dev Container, forward port **5173** in VS Code if necessary, and forward backend port **7071** from its container. Use the same browser origin you allowed in CORS.

The page should show **Products** and the backend's product cards, with prices displayed in pounds. In the browser's developer tools, open **Network** and confirm that `GET http://localhost:7071/api/products` returns HTTP 200 with a JSON array.

The app displays loading, empty-list and error states. If a request fails, correct the API or CORS configuration, then select **Retry**. Stop the frontend and backend with **Ctrl+C** in their respective terminals when finished.

## API configuration

`VITE_API_BASE_URL` is the API's base path. The app appends `products`, so use `http://localhost:7071/api/`, rather than the full `/api/products` endpoint. A missing final slash is normalised automatically. The URL must be an absolute HTTP(S) URL without credentials, a query string or a fragment.

If the setting is absent, the app defaults to `http://localhost:7071/api/`. Restart the development server after editing `.env.local`.

Vite embeds this public setting in the generated JavaScript **at build time**. Changing it for a production build requires rebuilding the app. Never put passwords, connection strings, publishing profiles or other secrets in a `VITE_*` value: browser users can read these values.

## Continuous integration and deployment

The [Build and Deploy to Test workflow](.github/workflows/build-and-deploy.yml) uses one job on `ubuntu-latest`. It checks out the repository, selects Node.js 22 with npm caching, installs dependencies using `npm ci`, and runs `npm run build`. A later step uploads `dist` to the Storage Account's `$web` container. Repository permissions are limited to `contents: read`; Azure access comes from the Storage connection-string secret.

| Trigger                       | Type-check and build | Deploy to Test           |
| ----------------------------- | -------------------- | ------------------------ |
| Push to `main`                | Yes                  | After a successful build |
| Pull request targeting `main` | Yes                  | Skipped                  |
| Manual run on `main`          | Yes                  | After a successful build |
| Manual run on another branch  | Yes                  | Skipped                  |

A type-check or build failure prevents deployment. PR validation does not use the Azure secret. The build does not contact the API, so it needs no running backend. This is build validation; there is no automated test suite in this example.

The workflow supplies the public API URL from a GitHub repository variable:

```yaml
env:
  VITE_API_BASE_URL: ${{ vars.VITE_API_BASE_URL }}
```

Set this variable to the deployed Test API URL before deploying. Your local `.env.local` is not available to GitHub Actions. The upload step fails with a configuration message if the variable or Storage connection-string secret is empty.

Run a local install and build check with:

```bash
npm ci
npm run build
```

To inspect the production build in a browser, run:

```bash
npm run preview
```

Open the URL printed by Vite. Forward the preview port if using a container, and allow that exact browser origin in backend CORS. The preview serves the existing `dist` output; it does not rebuild after configuration changes. Stop it with **Ctrl+C** when finished.

## Prepare the Azure Test environment

Use the course Dev Container's Bash terminal for these commands. You need an Azure subscription with permission to create a Storage Account, retrieve its connection string and update the backend Function App's CORS configuration. See the Week 5 deployment worksheet for the background on static hosting, CORS and Azure setup; the Week 9 publishing route below uses GitHub Actions.

### Confirm the backend and subscription

Deploy the backend using the Product service's deployment guide first. Sign in and select the intended subscription:

```bash
az login
```

```bash
az account set --subscription 'YOUR_SUBSCRIPTION_ID'
```

```bash
az account show --query '{name:name,id:id,tenantId:tenantId}' --output table
```

Replace the placeholders with the deployed backend's details:

```bash
API_BASE_URL='https://YOUR_FUNCTION_HOSTNAME/api/'
FUNCTION_APP_NAME='products-test-ab47-func'
BACKEND_GROUP='products-test-ab47-rg'
```

```bash
curl -i "${API_BASE_URL}health/live"
```

Expect HTTP 200 with `{"status":"live"}`. Also confirm product data is available:

```bash
curl -i "${API_BASE_URL}products"
```

Expect HTTP 200 with a JSON array. Use this HTTPS API base URL, including `/api/`, for the GitHub variable below.

### Provision frontend static hosting

Choose a permitted region and replace `ab47` with your own short lowercase-letter-and-digit identifier. Storage Account names must be globally unique and contain 3–24 lowercase letters and digits. These naming rules apply to Azure resources, not your GitHub repository.

```bash
LAB_REGION='YOUR_PERMITTED_REGION'
FRONTEND_GROUP='productapp-test-ab47-rg'
FRONTEND_STORAGE='productapptestab47store'
```

Keep these variables in the same terminal. In a new terminal, reselect the subscription and restore the variables before continuing. Use a fresh, dedicated disposable frontend resource group:

```bash
az group exists --name "$FRONTEND_GROUP"
```

Expect `false`. If it returns `true`, inspect the existing group and choose a different name if it contains shared, assessment or unrelated resources.

```bash
az group create --name "$FRONTEND_GROUP" --location "$LAB_REGION" --output none
```

```bash
az storage account create \
  --name "$FRONTEND_STORAGE" \
  --resource-group "$FRONTEND_GROUP" \
  --location "$LAB_REGION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --output none
```

Enable static hosting:

```bash
az storage blob service-properties update \
  --account-name "$FRONTEND_STORAGE" \
  --auth-mode key \
  --static-website \
  --index-document index.html \
  --404-document index.html \
  --output none
```

The index document serves the app's entry page. Azure Storage can serve `index.html` for missing paths as the error document, but this does not change a missing path's HTTP status to 200. This example's product page uses the root route. See Microsoft's [static website hosting guide](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website-how-to?tabs=azure-cli).

Obtain the **static website** endpoint:

```bash
SITE_URL=$(az storage account show --name "$FRONTEND_STORAGE" --resource-group "$FRONTEND_GROUP" --query primaryEndpoints.web --output tsv)
```

```bash
printf '%s\n' "$SITE_URL"
```

### Allow the website origin on the backend

CORS uses an origin without a path or trailing slash:

```bash
SITE_ORIGIN="${SITE_URL%/}"
```

```bash
az functionapp cors add \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$BACKEND_GROUP" \
  --allowed-origins "$SITE_ORIGIN" \
  --output none
```

```bash
az functionapp cors show --name "$FUNCTION_APP_NAME" --resource-group "$BACKEND_GROUP"
```

Confirm that the exact HTTPS website origin appears. Configure CORS on the Function App that receives the browser request; see the [Function App CORS reference](https://learn.microsoft.com/en-us/cli/azure/functionapp/cors). The example's website and API endpoints are public. Deployment credentials and CORS do not provide application authentication.

## Configure GitHub deployment

Ensure Actions is enabled in your private frontend repository. Under **Settings → Secrets and variables → Actions**, create:

| Kind                | Name                              | Value                                                  |
| ------------------- | --------------------------------- | ------------------------------------------------------ |
| Repository variable | `VITE_API_BASE_URL`               | Your deployed HTTPS API base URL, including `/api/`    |
| Repository secret   | `AZURE_STORAGE_CONNECTION_STRING` | The connection string for the frontend Storage Account |

Use the **Variables** tab for the public API URL and the **Secrets** tab for the deployment credential. Obtain the connection string in a private terminal:

```bash
az storage account show-connection-string \
  --name "$FRONTEND_STORAGE" \
  --resource-group "$FRONTEND_GROUP" \
  --query connectionString \
  --output tsv
```

Copy the returned connection string into the secret. Keep it out of Git, screenshots and shared logs. This is a Storage account-key credential, not a Function App publishing profile or API key. The workflow uses it to authenticate the upload without an Azure login step. See the [Storage connection-string reference](https://learn.microsoft.com/en-us/cli/azure/storage/account#az-storage-account-show-connection-string).

If upgrading your private build-only repository, copy `.github/workflows/build-and-deploy.yml` from the lecturer's `feature/deploy-to-test` branch into your own `main`. Remove the superseded `.github/workflows/build.yml`, then commit the workflow changes together. Keep working on your private `main`; do not switch to the lecturer's feature branch. This frontend workflow deploys code to the Storage Account you created above; it does not provision infrastructure.

## Deploy and verify

Push the workflow and application to `main` in your private repository:

```bash
git push origin main
```

Alternatively, select **Actions → Build and Deploy to Test → Run workflow**, choose `main`, and run it. Inspect the install, type-check/build and upload steps in the single job. A PR run should show the upload step skipped. Manual runs on other branches also skip deployment.

Open the printed `SITE_URL` in your browser. Confirm all of the following:

- The page loads over HTTPS and displays the backend's product data.
- The browser's **Network** tab shows a successful request to the deployed `/api/products` URL, rather than `localhost:7071`.
- The **Console** shows no CORS or public-configuration errors.

Make a harmless visible change, such as editing the heading in `src/views/Products.vue`, commit it and push to `main`. After Actions succeeds, reload the live website and confirm the change is visible. Publishing through Actions is sufficient; no local build or manual upload is needed for deployment. An upload succeeding does not by itself prove the browser can retrieve backend data.

Capture the successful workflow, website URL and browser/API result before cleanup. If you change `VITE_API_BASE_URL` in GitHub, run the workflow again on `main` to rebuild and redeploy; changing the variable alone does not update the existing JavaScript.

## Clean up disposable resources

After verification and evidence capture, retire the disposable Week 9 frontend environment. Keep local development and CI available for later work:

1. Remove the **Deploy to Azure Storage static website** step from the workflow. Set its build-time API URL back to `http://localhost:7071/api/` and change the displayed workflow name to **Build**. Commit and push those changes so later runs only build.
2. Remove `AZURE_STORAGE_CONNECTION_STRING` and `VITE_API_BASE_URL` from the repository's Actions settings. Confirm the build-only workflow still succeeds.
3. Check the subscription, target group and resources before deleting the dedicated frontend group:

```bash
az account show --query '{name:name,id:id}' --output table
```

```bash
az group show --name "$FRONTEND_GROUP" --query '{name:name,id:id,location:location}' --output table
```

```bash
az resource list --resource-group "$FRONTEND_GROUP" --query '[].{name:name,type:type}' --output table
```

Delete only the dedicated disposable frontend group after confirming it contains no shared, assessment or unrelated dependencies:

```bash
az group delete --name "$FRONTEND_GROUP"
```

The command asks for confirmation and waits for deletion. Verify removal:

```bash
az group exists --name "$FRONTEND_GROUP"
```

Expect `false`. Deleting the Storage Account invalidates its connection string. If keeping the backend, remove this retired website's CORS origin:

```bash
az functionapp cors remove --name "$FUNCTION_APP_NAME" --resource-group "$BACKEND_GROUP" --allowed-origins "$SITE_ORIGIN" --output none
```

Use the backend guide for cleanup of its dedicated Week 9 resources. Do not delete assessment or shared infrastructure. Later local development and build-only CI do not require either Week 9 Azure deployment.

## Troubleshooting

| Symptom                                                | Check                                                                                                                                                                                                                                           |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The app cannot start and mentions public configuration | Check `.env.local` for a valid API base URL, then restart Vite.                                                                                                                                                                                 |
| Products cannot be loaded                              | Check that the backend is running, port 7071 is forwarded if needed, and the browser's Network tab shows the expected `/api/products` URL.                                                                                                      |
| The browser reports a CORS error                       | Allow the exact frontend origin, including scheme and port, on the backend. A successful request outside the browser does not establish that CORS is configured.                                                                                |
| The API responds but the app shows an error            | Inspect the response: the app requires an array of valid products with non-empty IDs and names, and no duplicate IDs. Optional prices must be non-negative safe integers in pence; optional descriptions must be strings.                       |
| Port 5173 is occupied                                  | Stop the process already using it. This project's development server requires port 5173.                                                                                                                                                        |
| CI fails                                               | Open the failing step's log and reproduce with Node.js 22, `npm ci` and `npm run build`. Commit matching `package.json` and `package-lock.json` changes together when changing dependencies.                                                    |
| Upload reports missing configuration                   | Check the exact repository variable and secret names above. A local `.env.local` does not configure Actions.                                                                                                                                    |
| Storage upload is denied                               | Check that the connection string belongs to the frontend account and its key is valid. This example requires account-key access and a network path from the GitHub runner; subscription policy or Storage networking restrictions can block it. |
| Upload succeeds but the live app fails                 | Check the embedded API URL, deployed backend response and exact CORS origin. Rebuild after changing the GitHub API URL variable.                                                                                                                |
| The live page shows an older version                   | Check the run and commit on `main`, then reload with the browser cache disabled.                                                                                                                                                                |

## npm commands

| Command              | Purpose                                                   |
| -------------------- | --------------------------------------------------------- |
| `npm run dev`        | Start the development server on port 5173                 |
| `npm run type-check` | Check Vue and TypeScript source                           |
| `npm run build`      | Type-check and generate production files in `dist`        |
| `npm run build-only` | Generate production files without the separate type-check |
| `npm run preview`    | Serve the existing production build locally               |
| `npm run format`     | Format files under `src` with Prettier                    |

## Source structure

| Directory         | Responsibility                                                |
| ----------------- | ------------------------------------------------------------- |
| `src/domain`      | Product list model                                            |
| `src/app`         | List-products use case and Product service interface          |
| `src/infra`       | HTTP adapter and API response validation                      |
| `src/config`      | Composition root and public API URL validation                |
| `src/composables` | Dependency injection, loading/error state and retry behaviour |
| `src/views`       | Product page and price display                                |
| `src/router`      | Route configuration                                           |

Generated files in `dist`, installed dependencies in `node_modules`, and local environment files are ignored by Git. The public configuration example and dependency lockfile are committed so a new clone can reproduce the setup. Local development and build-only CI require no Azure resources.
