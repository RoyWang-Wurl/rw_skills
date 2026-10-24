# rw_skills

Skills for Cursor

Author: Roy Wang

---

## Skills

- **[/auto-hparams-tune](auto-hparams-tune/SKILL.md)**: Agent-guided hyperparameter tuning loop.
  - Runs a `run → diagnose → propose → run …` cycle that converges on a strong hyperparameter set without human intervention after kickoff
  - Disk-anchored to three files (`kickoff.json`, `auto_tune_log.jsonl`, `SKILL.md`) so the loop survives context compaction
  - Invoke when the user explicitly asks to auto-tune or search hyperparameters

- **[/llm_wiki](llm_wiki/SKILL.md)**: LLM-as-wiki-maintainer skill for the Wurl knowledge base in DataScienceResearch.
  - Handles ingesting raw sources, answering questions grounded in wiki pages, linting for broken links and coverage gaps, and filing new analysis back into the wiki
  - Activates for any query or edit touching `llm_wiki/`
  - Integrates with the full research workflow (`/generate-research-project` → `/summarize-research`) as the canonical prior-knowledge layer

- **[/agentic-peer-review](agentic-peer-review/SKILL.md)**: Turn-based peer review between two different coding agents over one frozen diff.
  - Exchanges findings and verdicts through a locked, gitignored mailbox
  - Supports explicit read-only monitoring so each agent automatically takes its next turn
  - Enforces independent verification, concrete failure scenarios, and convergence within five rounds

- **[/yo-dashboard-weekly-audit](yo-dashboard-weekly-audit/SKILL.md)**: Weekly audit of the YO Sigma dashboard into a Google Sheet tracker.
  - Runs 13 read-only Snowflake checks (lift, optimality, volume shifts, dead arms, view and config integrity) and a view-only Chrome walk of the dashboard
  - Tracks each issue on a Master tab (New / Open / Resolved), with a dated tab per run and a Rules tab that holds every flag, severity and dismissal rule
  - Check SQL, sheet ID and dashboard URL stay in a gitignored `private/` folder
  - A reviewer marks each finding (ticket, monitor or dismiss) before any ticket is created; repeated dismissals become Proposed rules
