# OncoWatch-n8n-Workflow-Suite
OncoWatch is an oncology competitive‑intelligence (CI) workflow stack built on n8n. It ingests external signals, applies AI triage and routing, supports human overrides, and produces daily digests for medical affairs and CI teams.

---

## 1. Workflows in this repo

- `OncoWatch-Health-Monitor.json`  
  Scheduled ingestion and primary triage:
  - Pulls oncology signals from ClinicalTrials.gov and PubMed.  
  - Normalizes to a `signals` schema (drug, indication, sponsor, phase, status, URL).  
  - Uses AI to assign `signal_type`, `impact_level`, `competitor_flag`, `ai_summary`, and `recommended_action`.  
  - Computes an `impact_score` and stores/updates rows in `signals`.  
  - Sends HIGH/MED/LOW alerts to Slack and can trigger the Daily Digest workflow.

- `OncoWatch_AI_Triage_Enricher.json`  
  Advanced AI triage and routing:
  - Selects signals needing (re)triage from `signals`.  
  - Builds a structured “features + context” prompt and calls an LLM.  
  - Merges AI output back into the signal (updated scores, summaries, drivers/risks).  
  - Computes `route_tier`, `priority_score`, `notif_priority`, and `alert_channels`.  
  - Formats CRITICAL / high‑tier alerts for executive and CI Slack channels.

- `OncoWatch_Human_Override.json`  
  Human‑in‑the‑loop overrides:
  - Exposes a POST webhook for overrides and supports Slack slash commands / buttons.  
  - Validates actions: `APPROVE`, `ESCALATE`, `DISMISS`, `RECLASSIFY`.  
  - Logs overrides into `signal_overrides` and updates `signals` (level, score, urgency, override metadata).  
  - Posts a compact Slack confirmation summarizing the change and new risk.

- `Oncowatch_Daily_Digest.json`  
  Daily CI digest:
  - Runs on a morning schedule (e.g., 08:00).  
  - Aggregates last‑24h and last‑7‑day signals (impact distribution, competitor activity, indication “heatmap”).  
  - Builds a 5‑sentence AI executive summary.  
  - Sends a rich Slack digest to an OncoWatch digest channel.  
  - Optionally writes summary metrics into `workflow_metrics` and builds a CSV export.

- `Global-Error-Handler.json`  
  Central error handler:
  - Receives error events from other workflows.  
  - Normalizes core fields (workflow, node, message, mode).  
  - Logs into a DB table (e.g., `ci_error_log`) and posts a short error alert to an ops/CI Slack channel.  
  - Recommended to set as the error workflow for all OncoWatch pipelines.

---

## 2. Required infrastructure

- **n8n** instance (cloud or self‑hosted).  
- **PostgreSQL** with at least:
  - `signals`
  - `signal_overrides`
  - `ci_audit_log`
  - `workflow_metrics`
- **Slack** bot token with access to:
  - `#oncowatch-alerts`, `#oncowatch-digest`, `#exec-alerts`, `#medical_affair_ci`, `#oncowatch_ci_team` (and optionally ops channels).
- **LLM / HTTP APIs**  
  - Keys for your chosen LLM providers (e.g., OpenRouter / Groq / Gemini).

You can document connection details in a local `.env` or n8n credential store (not committed to Git).

---

## 3. Import & setup

1. In n8n, go to **Workflows → Import from file** and import:
   - `OncoWatch-Health-Monitor.json`  
   - `OncoWatch_AI_Triage_Enricher.json`  
   - `OncoWatch_Human_Override.json`  
   - `Oncowatch_Daily_Digest.json`  
   - `Global-Error-Handler.json`
2. Recreate credentials:
   - Postgres (used by all DB nodes).  
   - Slack API.  
   - HTTP header/API‑key auth for LLM providers and other APIs.
3. In each workflow:
   - Confirm channel names, table names, and URLs match your environment.  
   - Set **Error Workflow** to the imported Global Error Handler (where supported).
4. Enable scheduling:
   - Health Monitor: every 6 hours (or your desired cadence).  
   - AI Triage Enricher: frequent but within LLM quota.  
   - Daily Digest: daily at your preferred time.  

---

## 4. Signal lifecycle (short)

1. **Ingest** – Health Monitor pulls raw trial/publication data and inserts/updates `signals` with AI‑assisted impact and summary.  
2. **Enrich & route** – AI Triage Enricher refines scores, sets route tiers, and triggers high‑priority alerts.  
3. **Override** – Analysts use the Human Override workflow (API/Slack) to adjust scores/tiers with full audit.  
4. **Digest** – Daily Digest summarizes the portfolio of signals for leadership and CI teams.  
5. **Observe** – Global Error Handler and CI audit tables capture failures and routing decisions for governance.

