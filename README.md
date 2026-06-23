# macavallswa11 — Azure Static Web App (BYO Functions backend)

A Vite vanilla JavaScript app deployed to **Azure Static Web Apps** (Standard tier) with a **Bring-Your-Own** Linux Function App backend running on an **S1 App Service Plan**.

Live site: https://victorious-pond-0a3c82d10.7.azurestaticapps.net
API (direct): https://macavallswa11-func.azurewebsites.net/api/hello
API (via SWA): https://victorious-pond-0a3c82d10.7.azurestaticapps.net/api/hello

---

## Architecture

| Resource | Name | Notes |
| --- | --- | --- |
| Resource Group | `macavallswa11-rg` | Region: `centralus` |
| Static Web App | `macavallswa11` | SKU: **Standard** (BYO backend) |
| App Service Plan | `macavallswa11-asp` | **S1**, Linux |
| Function App | `macavallswa11-func` | Node 24, Functions v4, Linux |
| Storage Account | `macavallswa11sa` | Standard_LRS (Function runtime storage) |
| GitHub Repo | [macavall/swamacavall11](https://github.com/macavall/swamacavall11) | CI/CD via GitHub Actions |

---

## Project layout

```
proj1/
├── .github/workflows/azure-static-web-apps.yml   # CI/CD: builds Vite, uploads to SWA
├── api/                                          # Azure Functions (Node v4 programming model)
│   ├── host.json
│   ├── package.json
│   └── src/functions/hello.js                    # GET/POST /api/hello
├── src/                                          # Vite frontend
│   ├── main.js                                   # Calls /api/hello and renders the result
│   └── style.css
├── public/
├── staticwebapp.config.json                      # SPA fallback to /index.html
├── swa-cli.config.json                           # Local SWA CLI config
├── package.json                                  # vite + @azure/static-web-apps-cli
└── README.md
```

---

## Prerequisites

- Node.js 20+ (project tested with Node 24)
- Azure CLI 2.6x+ (`az login`)
- GitHub CLI (`gh auth login`) — optional, used to set repo secrets
- An Azure subscription

---

## 1. Local development

```powershell
npm install
npm run dev                    # Vite dev server at http://localhost:5173

# Run the function locally (separate terminal)
cd api
npm install
func start                     # requires Azure Functions Core Tools v4

# Or run both together through the SWA emulator:
npx swa start --api-location api
```

---

## 2. Reproduce the Azure resources from scratch

```powershell
$RG   = "macavallswa11-rg"
$LOC  = "centralus"
$SWA  = "macavallswa11"
$ASP  = "macavallswa11-asp"
$FUNC = "macavallswa11-func"
$SA   = "macavallswa11sa"

# Resource group
az group create -n $RG -l $LOC

# Static Web App (start on Free, then upgrade to Standard for BYO backends)
az staticwebapp create -n $SWA -g $RG -l $LOC --sku Free
az staticwebapp update -n $SWA -g $RG --sku Standard

# App Service Plan (Linux, S1)
az appservice plan create -n $ASP -g $RG -l $LOC --sku S1 --is-linux

# Storage account for the Function App
az storage account create -n $SA -g $RG -l $LOC --sku Standard_LRS --kind StorageV2

# Function App on the S1 Linux plan
az functionapp create `
    -n $FUNC -g $RG `
    --plan $ASP `
    --storage-account $SA `
    --runtime node --runtime-version 24 `
    --functions-version 4 `
    --os-type Linux
```

---

## 3. Deploy the function code

```powershell
# Install runtime deps and zip the api folder
Push-Location api
npm install --omit=dev
Pop-Location

$zip = "$env:TEMP\macavallswa11-func.zip"
if (Test-Path $zip) { Remove-Item $zip -Force }
Push-Location api
$items = Get-ChildItem -Force | Where-Object { $_.Name -notin 'local.settings.json','.funcignore' }
Compress-Archive -Path $items.FullName -DestinationPath $zip -Force
Pop-Location

az functionapp deployment source config-zip -g macavallswa11-rg -n macavallswa11-func --src $zip

# Smoke test
Invoke-RestMethod "https://macavallswa11-func.azurewebsites.net/api/hello?name=world"
```

---

## 4. Link the Function App as the SWA backend

```powershell
$funcId = az functionapp show -g macavallswa11-rg -n macavallswa11-func --query id -o tsv
az staticwebapp backends link `
    -n macavallswa11 -g macavallswa11-rg `
    --backend-resource-id $funcId `
    --backend-region centralus

# Verify
az staticwebapp backends show -n macavallswa11 -g macavallswa11-rg -o table
Invoke-RestMethod "https://victorious-pond-0a3c82d10.7.azurestaticapps.net/api/hello?name=swa"
```

After linking, requests to `<swa-host>/api/*` are routed to the Function App.

---

## 5. CI/CD — GitHub Actions

The workflow [.github/workflows/azure-static-web-apps.yml](.github/workflows/azure-static-web-apps.yml) builds the Vite app and uploads the static output to the SWA on every push to `main` (and on PRs).

One-time setup:

```powershell
# Fetch the SWA deployment token and store it as a repo secret
$token = az staticwebapp secrets list -n macavallswa11 -g macavallswa11-rg --query "properties.apiKey" -o tsv
gh secret set AZURE_STATIC_WEB_APPS_API_TOKEN --repo macavall/swamacavall11 --body $token
```

Workflow key settings:

| Setting | Value |
| --- | --- |
| `app_location` | `/` |
| `output_location` | `dist` |
| `api_location` | _(empty — API is the linked BYO Function App, not a managed SWA function)_ |

To deploy:

```powershell
git push origin main
gh run watch --repo macavall/swamacavall11
```

---

## 6. Cleanup

```powershell
az group delete -n macavallswa11-rg --yes --no-wait
```

This deletes the SWA, Function App, App Service Plan, Storage Account, and Application Insights component in one shot.
