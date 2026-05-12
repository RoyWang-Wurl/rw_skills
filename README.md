# rw_skills

Skills for Cursor

Author: Roy Wang

---

**/auto-hparams-tune**: Agent-guided hyperparameter tuning loop. Runs a `run → diagnose → propose → run …` cycle that converges on a strong hyperparameter set without human intervention after kickoff. Disk-anchored to three files (`kickoff.json`, `auto_tune_log.jsonl`, `SKILL.md`) so the loop survives context compaction. Invoke when the user explicitly asks to auto-tune or search hyperparameters.
