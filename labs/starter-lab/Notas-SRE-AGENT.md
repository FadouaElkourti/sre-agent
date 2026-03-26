# 🤖 Azure SRE Agent — Chuleta Rápida

## ¿Qué es?
Un agente de IA conectado a tu infraestructura Azure que **investiga y remedia incidentes automáticamente**, 24/7, sin intervención humana.

---

## 🚀 Comandos para arrancar el lab

```bash
# 1. Ir al directorio del lab
cd ~/repos/github/sre-agent/labs/starter-lab

# 2. Ver el estado actual del lab
bash scripts/post-provision.sh --status

# 3. Romper la app (simula un incidente real)
bash scripts/break-app.sh

# 4. Ver la app Grubify en el browser
# https://ca-grubify-fe-7s44qwg6z5h3i.whiterock-9bb26847.eastus2.azurecontainerapps.io

# 5. Redeploy si algo falla
azd up

# 6. Reconfigurar el agente (knowledge base, subagentes, etc.)
bash scripts/post-provision.sh --retry
```

---

## 🌐 URLs importantes

| Recurso | URL |
|---------|-----|
| **Portal del Agente** | https://sre.azure.com |
| **Grubify Frontend** | https://ca-grubify-fe-7s44qwg6z5h3i.whiterock-9bb26847.eastus2.azurecontainerapps.io |
| **Grubify API** | https://ca-grubify-7s44qwg6z5h3i.whiterock-9bb26847.eastus2.azurecontainerapps.io |
| **Azure Portal (RG)** | https://portal.azure.com → rg-sre-lab |

---

## 💬 Prompts para el chat del agente

### Onboarding / Conocimiento
```
What do you know about the Grubify app architecture?
Summarize the HTTP errors runbook
What Azure resources are in my resource group?
How many container apps are deployed? List them with endpoints.
```

### Investigación de incidentes
```
The Grubify API is not responding - specifically the "Add to Cart" is 
failing. Can you investigate, find the root cause?
```

### Remediación
```
Can you mitigate this issue?
```

### Con GitHub (usa /incident-handler o /code-analyzer)
```
The Grubify API is failing. Investigate, find the root cause, and 
create a GitHub issue with your detailed findings.
```

---

## 🧩 Cómo funciona el agente

```
Azure Monitor Alert
       ↓
Response Plan: grubify-http-errors
       ↓
Subagente: incident-handler (Autónomo)
       ↓
┌─────────────────────────────────────┐
│  1. SearchMemory → runbooks          │
│  2. QueryLogAnalytics → KQL logs     │
│  3. az monitor metrics → gráficas   │
│  4. ExecutePythonCode → charts       │
│  5. Root cause analysis              │
│  6. CreateGithubIssue (con GitHub)   │
│  7. az containerapp restart → fix    │
└─────────────────────────────────────┘
```

---

## 🤖 Subagentes configurados

| Subagente | Para qué sirve |
|-----------|---------------|
| **incident-handler** | Investiga alertas de Azure Monitor, consulta logs/métricas, remedia |
| **code-analyzer** | Busca en código fuente el root cause con referencias file:line |
| **issue-triager** | Clasifica issues de GitHub con etiquetas y comentarios estructurados |

---

## 📚 Knowledge Base cargada

| Documento | Contenido |
|-----------|-----------|
| `grubify-architecture.md` | Arquitectura de la app, rutas API, Container Apps |
| `http-500-errors.md` | Runbook para errores HTTP 500 |
| `incident-report-template.md` | Plantilla de informe de incidente |
| `github-issue-triage.md` | Runbook para clasificar issues de GitHub |

---

## 🔧 Recursos Azure desplegados

```
rg-sre-lab
├── sre-agent-7s44qwg6z5h3i          ← El agente
├── ca-grubify-7s44qwg6z5h3i          ← API de Grubify (Container App)
├── ca-grubify-fe-7s44qwg6z5h3i       ← Frontend de Grubify
├── cae-7s44qwg6z5h3i                 ← Container Apps Environment
├── law-7s44qwg6z5h3i                 ← Log Analytics (logs KQL)
├── appi-7s44qwg6z5h3i                ← Application Insights
└── acrcagrubify7s44qwg6z5h3i         ← Container Registry
```

---

## ⚡ Flujo del incidente autónomo (sin intervención humana)

1. **Break app** → 311 errores HTTP 5xx en 1 minuto
2. **Azure Monitor** detecta spike → dispara alerta `alert-http-5xx-sre-lab` (Sev3)
3. **Response Plan** `grubify-http-errors` lo enruta a `incident-handler`
4. **Agente investiga** autónomamente:
   - 468 `OutOfMemoryException` en logs → `CartController.cs:line 30`
   - Memoria creció 65x en 4 minutos (637 KB → 41 MB)
   - Genera gráficas de evidencia con Python
5. **Agente remedia**: restart del container app → 5xx = 0
6. Aparece en **Activities → Incidents** con estado `Completed`

---

## 🔑 Credenciales y entorno

```bash
# Ver todas las variables del entorno
cat .azure/sre-lab/.env

# Subscription
az account show --query "{name:name, id:id}" -o table

# Estado del agente via API
python3 -c "
import subprocess, urllib.request, json
token = subprocess.run(['az','account','get-access-token','--resource',
  'https://azuresre.dev','--query','accessToken','-o','tsv'],
  capture_output=True, text=True).stdout.strip()
req = urllib.request.Request(
  'https://sre-agent-7s44qwg6z5h3i--8760f4cd.50e51f02.eastus2.azuresre.ai/api/v1/incidentPlayground/filters',
  headers={'Authorization': f'Bearer {token}'})
with urllib.request.urlopen(req) as r:
    [print(f['id'], '->', f.get('handlingAgent')) for f in json.loads(r.read())]
"
```
