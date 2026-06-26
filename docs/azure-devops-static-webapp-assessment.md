# Azure DevOps and Static Web App implementation

This implementation keeps two concerns separate:

- Azure Static Web Apps hosts the Docusaurus site and, optionally, a protected report path.
- Azure DevOps runs the Zero Trust Assessment on a schedule and publishes the generated report as a secured pipeline artifact.

The assessment report contains tenant security posture data. Do not expose generated reports publicly. Start with pipeline artifacts, then publish to `/reports/*` only after Static Web Apps authentication or private endpoint access is confirmed.

## Branch

Work starts on the `dev` branch.

## Azure DevOps agent

The included YAML targets:

- Agent pool: `synergy`
- Agent name: `azagent2`

## Pipelines

### Static Web App deployment

Pipeline file:

```text
.azure-pipelines/azure-static-webapp-dev.yml
```

Required secret variable:

```text
AZURE_STATIC_WEB_APPS_API_TOKEN
```

This pipeline builds `src/react` with Node.js 20 and deploys the generated `src/react/build` folder to Azure Static Web Apps.

The Azure DevOps pipeline sets `BASE_URL=/` during the Docusaurus build. The default Docusaurus config still uses `/zerotrustassessment/` when `BASE_URL` is not set, so the existing GitHub Pages workflow remains compatible.

### Scheduled assessment

Pipeline file:

```text
.azure-pipelines/zero-trust-assessment-scheduled-dev.yml
```

Variable group:

```text
zta-assessment-dev
```

Required variables:

```text
ZTA_TENANT_ID
ZTA_CLIENT_ID
ZTA_CERT_SECURE_FILE
ZTA_CERT_PASSWORD
ZTA_CERT_THUMBPRINT
ZTA_REPORT_DAYS
ZTA_SIGNIN_QUERY_MINUTES
```

Mark `ZTA_CERT_PASSWORD` as secret.

Recommended starting values:

```text
ZTA_REPORT_DAYS=7
ZTA_SIGNIN_QUERY_MINUTES=30
```

Secure file:

Upload the app registration certificate `.pfx` to Azure DevOps Library > Secure files and authorize this pipeline to use it. The secure file name must match `ZTA_CERT_SECURE_FILE`.

## Authentication requirements

The pipeline uses `Connect-ZtAssessment` with certificate-based app-only authentication:

```powershell
Connect-ZtAssessment -TenantId $tenantId -ClientId $clientId -Certificate $certificate -Service Graph,Azure
```

The app registration needs:

- Certificate credential matching the uploaded `.pfx`.
- Microsoft Graph application permissions required by Zero Trust Assessment.
- Admin consent granted in the tenant.
- Azure RBAC on the assessed subscription or management group. Owner works, but Security Reader or Reader is usually enough for read-only assessment coverage.

Start with `-Service Graph,Azure` and `-Pillar Identity`. Expand to Devices or additional services only after the first scheduled run is stable.

## Schedule

The assessment YAML currently runs Mondays at `01:00 UTC`, which is `02:00` in Africa/Lagos.

```yaml
schedules:
- cron: '0 1 * * Mon'
```

Azure DevOps YAML schedules use UTC. The `always: true` setting makes the pipeline run even if no code changed.

## Optional report hosting

The Static Web App config protects `/reports/*` with built-in authentication:

```text
src/react/static/staticwebapp.config.json
```

Before publishing assessment output there, restrict the Static Web App to the correct Microsoft Entra tenant or place it behind a private endpoint. The default preconfigured Entra provider can allow broader Microsoft account sign-in if not restricted.
