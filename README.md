# Azure SRE Agent Demo — Data Operations Pipeline

> **Durée :** ~10 min | **Rôles :** Data Engineer + SRE Agent + GitHub Copilot

## Architecture

```
Logic App (toutes les 2 min)
  └─► Azure Function HTTP : process_batch
        └─► Application Insights (traces + exceptions)
              └─► Azure SRE Agent
                    ├─► GitHub Issue (incident créé automatiquement)
                    └─► GitHub Copilot → fix → redéploiement → recovery
```

**Ressources Azure :**
- Function App : `sre-dataops-demo-func` (West Europe)
- Logic App : `dataops-pipeline-trigger`
- App Insights : `sre-dataops-demo-insights`
- Resource Group : `rg-sre-agent-dataops-demo`

---

## Narrative complet — étape par étape

### 🟢 Phase 1 — Montrer que tout est sain (1 min)

> *"On a un pipeline data ops. Une Logic App déclenche une Azure Function HTTP toutes les 2 minutes pour traiter des batches. Tout va bien."*

Appel manuel de la Function avec un petit batch :

```bash
# Récupérer la clé : az functionapp function keys list --name sre-dataops-demo-func --resource-group rg-sre-agent-dataops-demo --function-name process_batch --query "default" -o tsv
curl -X POST "https://sre-dataops-demo-func.azurewebsites.net/api/pipeline/run?code=<FUNCTION_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"demo-small","rows":1000,"source":"manual"}'
```

**Réponse attendue :**
```json
{"status": "ok", "batch_id": "demo-small", "source": "manual", "processed_rows": 1000}
```

Montrer dans **App Insights → Live Metrics** : requêtes vertes, zéro exception.

---

### 💥 Phase 2 — Introduire un bug avec GitHub Copilot (2 min)

> *"Un Data Engineer demande à Copilot d'optimiser le pipeline pour les gros volumes..."*

**Dans VS Code avec GitHub Copilot, ouvrir `src/function_http/process_batch/__init__.py` et entrer ce prompt :**

```
Modifie ce code pour gérer les grands volumes : si rows > 50000,
lève une RuntimeError("memory pressure detected on large batch").
```

Copilot va modifier le code. **Commiter et pusher :**

```bash
git add src/function_http/process_batch/__init__.py
git commit -m "perf: add memory pressure guard for large batches"
git push
```

GitHub Actions se déclenche → déploiement automatique via GitHub Release.

Ou déploiement manuel (si CI/CD pas encore configuré) :

```bash
cd src/function_http
# Windows :
Compress-Archive -Path .\* -DestinationPath ..\..\function-http-demo-broken.zip -Force
gh release create v1.0.1-broken ..\..\function-http-demo-broken.zip --title "Broken deployment" --notes "Bug introduced"
# Puis update la function app setting WEBSITE_RUN_FROM_PACKAGE avec la nouvelle URL + restart
```

---

### 🔴 Phase 3 — Déclencher l'incident (1 min)

> *"La Logic App envoie un gros batch — et là, ça explose."*

```bash
curl -X POST "https://sre-dataops-demo-func.azurewebsites.net/api/pipeline/run?code=<FUNCTION_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"demo-large","rows":200000,"source":"logic-app"}'
```

**Réponse attendue :**
```json
{"status": "error", "message": "memory pressure detected on large batch"}
```
HTTP 500.

Rappeler plusieurs fois pour générer un spike dans App Insights.

---

### 🤖 Phase 4 — SRE Agent détecte l'incident (2 min)

> *"L'Azure SRE Agent surveille Application Insights en continu. Il détecte le spike d'erreurs 500 et crée automatiquement un GitHub issue."*

Dans GitHub → Issues : un issue apparaît automatiquement, par exemple :
```
🔴 [Incident] process_batch: spike in HTTP 500 errors detected
Severity: High | Service: sre-dataops-demo-func | Errors: 12 in 5 min
App Insights query: ...
```

Montrer l'issue créé par le SRE Agent avec le diagnostic.

---

### 🛠️ Phase 5 — Fix avec GitHub Copilot (2 min)

> *"Copilot Coding Agent analyse l'issue et propose un fix..."*

**Option A — Copilot Coding Agent sur l'issue :**
Assigner l'issue au Copilot agent → il propose un PR avec le fix.

**Option B — Fix manuel avec Copilot en live :**

Ouvrir `src/function_http/process_batch/__init__.py`, prompt :

```
Supprime la contrainte artificielle sur les gros volumes et
garde un logging structuré robuste pour batch_id et rows.
```

Commiter, pusher → déploiement automatique.

---

### ✅ Phase 6 — Recovery (1 min)

> *"Après redéploiement, le pipeline est à nouveau sain."*

```bash
curl -X POST "https://sre-dataops-demo-func.azurewebsites.net/api/pipeline/run?code=<FUNCTION_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"demo-large","rows":200000,"source":"logic-app"}'
```

**Réponse :** `{"status": "ok", ..., "processed_rows": 200000}` ✅

Montrer dans **App Insights → Live Metrics** : zéro exception, toutes les requêtes vertes.

SRE Agent ferme l'incident ou le marque résolu.

---

## Repo structure

```text
.
├── .github/workflows/deploy.yml   ← CI/CD via GitHub Releases
├── infra/main.bicep               ← Infrastructure as Code
├── logic-app/workflow.json        ← Logic App workflow
├── src/function_http/
│   ├── host.json
│   ├── requirements.txt
│   └── process_batch/
│       ├── __init__.py            ← ⭐ Fichier central de la démo
│       └── function.json
└── data/
    ├── payload-small.json         ← {"batch_id":"demo-small","rows":1000}
    └── payload-large.json         ← {"batch_id":"demo-large","rows":200000}
```

## Copilot prompts (copier-coller prêts)

### Introduire le bug
```
Modifie ce code pour gérer les grands volumes : si rows > 50000,
lève une RuntimeError("memory pressure detected on large batch").
```

### Fixer le bug
```
Supprime la contrainte artificielle sur les gros volumes et
garde un logging structuré robuste pour batch_id et rows.
```

## Déploiement manuel (sans CI/CD)

```powershell
# 1. Zipper le code
cd src/function_http
Compress-Archive -Path .\* -DestinationPath ..\..\function-http-demo.zip -Force

# 2. Créer une GitHub Release avec le zip
gh release create v1.0.X ..\..\function-http-demo.zip --title "Deploy vX" --notes "Demo deployment"

# 3. Mettre à jour la Function App (remplacer l'URL par la nouvelle release)
$url = "https://github.com/lucmasssol/sre-agent-dataops-demo/releases/download/v1.0.X/function-http-demo.zip"
az functionapp config appsettings set --name sre-dataops-demo-func --resource-group rg-sre-agent-dataops-demo --settings "WEBSITE_RUN_FROM_PACKAGE=$url"
az functionapp restart --name sre-dataops-demo-func --resource-group rg-sre-agent-dataops-demo
```

