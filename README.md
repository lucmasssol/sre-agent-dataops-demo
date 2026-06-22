# Azure SRE Agent Demo - Data Operations (HTTP Function)

This repository provides a clean demo scenario:

Logic App (recurrence) -> Azure Function (HTTP trigger) -> Application Insights -> Azure SRE Agent -> GitHub issue/PR -> fix.

## Demo architecture

- Data-ops service: Azure Function `process_batch`
- Orchestration: Logic App sending periodic batch payloads
- Observability: Application Insights
- Incident loop: SRE Agent + GitHub

## Repo structure

```text
.
├── .github/workflows/deploy.yml
├── infra/main.bicep
├── logic-app/workflow.json
├── src/function_http/
│   ├── host.json
│   ├── requirements.txt
│   └── process_batch/
│       ├── __init__.py
│       └── function.json
└── data/
    ├── payload-small.json
    └── payload-large.json
```

## Quick start

1. Deploy infrastructure:

```bash
az group create --name rg-sre-agent-dataops-demo --location westeurope
az deployment group create --resource-group rg-sre-agent-dataops-demo --template-file infra/main.bicep
```

2. Configure GitHub secret:
- `AZURE_FUNCTIONAPP_PUBLISH_PROFILE`

3. Push to `main` to deploy function code via GitHub Actions.

4. Configure Logic App HTTP action URL with function host key:

`https://<function-app>.azurewebsites.net/api/pipeline/run?code=<function_key>`

## Copilot prompts for live demo

### Break the service

> Modify the function to simulate a data operations incident: when rows > 50000, raise RuntimeError("memory pressure detected on large batch").

### Fix the service

> Remove the artificial failure on large batches and keep robust structured logging for batch_id and rows.

## Demo narrative (7 minutes)

1. Healthy phase: Logic App sends `rows=1000`, function returns 200.
2. Break phase: Copilot introduces bug and deploys.
3. Incident phase: requests with large rows return 500.
4. Detection phase: SRE Agent detects error spike from App Insights.
5. Remediation phase: Copilot applies fix and redeploys.
6. Recovery phase: error rate drops and incident is closed.
