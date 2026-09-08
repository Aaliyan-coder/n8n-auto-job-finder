# Auto Job Finder — n8n + Local LLM

An autonomous job-hunting pipeline that runs every morning at 9 AM with **zero input**:
it finds jobs matching my resume, writes honest tailored applications with a local LLM,
and leaves ready-to-send drafts in my Gmail — CV attached.

![Workflow](workflow.png)

## How it works
Schedule (9 AM daily)
├─→ Remotive API ─┐
├─→ JSearch API ├─→ Merge → Score & Filter → Local LLM → Parse/Validate → Router
└─→ Arbeitnow API ─┘ ├─ has apply email → Gmail draft (CV attached)
└─ link only → Google Sheets to-apply list


1. **Fetch** — ~280 fresh listings pulled in parallel from Remotive, Arbeitnow, and
   JSearch (Google-for-Jobs aggregate covering Indeed/LinkedIn/Glassdoor, incl. Pakistan via `country=pk`)
2. **Score** — a Code node keyword-matches every posting against my stack
   (Python, React, AI/ML, FastAPI, Django, LLM...), dedupes, keeps the top 10
3. **Write** — Qwen3 (via Ollama, 100% local) drafts a short, honest application email per job.
   The system prompt embeds my real resume with a hard rule: *never invent experience*
4. **Validate** — LLM output is JSON-parsed defensively (think-block stripping, fence removal,
   fallbacks) — raw model output is never trusted
5. **Route** — postings containing an apply email become Gmail drafts addressed to the employer,
   with my resume PDF attached; link-only postings are appended to a Google Sheets apply-list
6. I wake up, review, and hit send — deliberate human-in-the-loop

## Stack

n8n (self-hosted, Docker) · Ollama + Qwen3 · Remotive / Arbeitnow / JSearch APIs ·
Gmail API · Google Sheets API (OAuth2)

## Design decisions

- **Local LLM** — private data (my resume) never leaves my machine; zero inference costs
- **Score before generate** — cheap keyword filtering cuts 280 jobs to 10 before any
  expensive LLM call
- **Draft, never auto-send** — the automation prepares; a human approves
- **Secrets in credentials** — the RapidAPI key lives in an n8n Header Auth credential,
  not in the exported JSON

## Run it yourself

1. n8n in Docker with a mounted files folder:

docker run -d --name n8n -p 5678:5678
-v n8n_data:/home/node/.n8n -v C:\n8n-files:/files
-e N8N_RESTRICT_FILE_ACCESS_TO="/files"
-e GENERIC_TIMEZONE="Asia/Karachi"
docker.n8n.io/n8nio/n8n

2. Ollama on the host with `OLLAMA_HOST=0.0.0.0`; n8n reaches it at `http://host.docker.internal:11434` (`ollama pull qwen3:1.7b`)
3. Import `job-finder-daily.json`
4. Attach your own credentials: Ollama, Gmail (OAuth2), Google Sheets (OAuth2), and a
   Header Auth credential named `X-RapidAPI-Key` with a free JSearch key from rapidapi.com
5. Drop your resume at `C:\n8n-files\<YourResume>.pdf` and edit the system prompt with your own resume text
6. Execute once to test, then activate — good morning, job applications ☕

## Roadmap

- [ ] Dedupe across days (Sheets lookup — skip already-seen jobs)
- [ ] Telegram notification with the morning summary
- [ ] Error workflow (alert when Ollama/API is down at 9 AM)
- [ ] Match-score threshold using the LLM analysis from my companion project
