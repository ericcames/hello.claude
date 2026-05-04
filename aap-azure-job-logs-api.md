# Pulling Job Logs from AAP Azure Managed Application via REST API

## Finding Your AAP URL

For AAP deployed as an Azure Managed Application, locate the gateway URL in:

**Azure Portal → Managed Application resource → Overview → Outputs**

It is typically:
```
https://<managed-app-name>.<region>.cloudapp.azure.com
```

---

## Authentication

Create an OAuth2 token (preferred over Basic auth):

```bash
curl -s -X POST https://<AAP_HOST>/api/v2/tokens/ \
  -u admin:<password> \
  -H "Content-Type: application/json" \
  -d '{"description": "log pull", "application": null, "scope": "read"}' \
  | jq -r '.token'
```

Store the returned token value:
```bash
export TOKEN="<returned_token>"
export AAP="https://<AAP_HOST>"
```

> If SSO/Entra ID is configured, create a local service account in AAP to generate tokens via the API.

---

## Job Log Endpoints

### List Recent Jobs

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/jobs/?order_by=-finished&page_size=20" \
  | jq '.results[] | {id, name, status, finished}'
```

### Get Full stdout Log for a Job

```bash
# Plain text (most readable)
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/plain" \
  "$AAP/api/v2/jobs/<JOB_ID>/stdout/?format=txt"

# JSON format
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/jobs/<JOB_ID>/stdout/?format=json"
```

### Get Structured Job Events (Per-Task Breakdown)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/jobs/<JOB_ID>/job_events/?page_size=200" \
  | jq '.results[] | {event, task, res: .event_data.res}'
```

### Filter Jobs by Template Name

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/jobs/?job_template__name=<TEMPLATE_NAME>&order_by=-finished" \
  | jq '.results[] | {id, status, finished}'
```

---

## Workflow Job Logs

Workflows have a separate endpoint. Retrieve the child job IDs, then pull stdout for each.

```bash
# List nodes in a workflow job
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/workflow_jobs/<JOB_ID>/workflow_nodes/" \
  | jq '.results[] | {id, job: .summary_fields.job}'

# Pull stdout for a child job
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/plain" \
  "$AAP/api/v2/jobs/<CHILD_JOB_ID>/stdout/?format=txt"
```

---

## Quick One-Liners

### Dump the Last Failed Job Log

```bash
JOB_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$AAP/api/v2/jobs/?status=failed&order_by=-finished&page_size=1" \
  | jq -r '.results[0].id')

curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/plain" \
  "$AAP/api/v2/jobs/$JOB_ID/stdout/?format=txt"
```

### Save a Job Log to File

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/plain" \
  "$AAP/api/v2/jobs/<JOB_ID>/stdout/?format=txt" \
  > job_<JOB_ID>.log
```

---

## Endpoint Reference

| Purpose | Endpoint |
|---------|----------|
| List jobs | `GET /api/v2/jobs/` |
| Job details | `GET /api/v2/jobs/<id>/` |
| Full stdout log | `GET /api/v2/jobs/<id>/stdout/?format=txt` |
| Per-task events | `GET /api/v2/jobs/<id>/job_events/` |
| Workflow job nodes | `GET /api/v2/workflow_jobs/<id>/workflow_nodes/` |
| Create auth token | `POST /api/v2/tokens/` |
| Delete auth token | `DELETE /api/v2/tokens/<id>/` |

> Always delete tokens when finished: `curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "$AAP/api/v2/tokens/<TOKEN_ID>/"`

---

## Browsable API

AAP ships a browsable API UI at `https://<AAP_HOST>/api/v2/` — open it in a browser while authenticated to explore available endpoints and filter parameters interactively.
