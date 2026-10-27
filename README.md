# Resume Bot — FastAPI + Groq

A small resume Q&A bot. The resume is ~8.4 KB of text, so it fits entirely in
the model's context window: there is **no vector DB, no embeddings, no
retrieval step**. The resume is injected into the system prompt once at startup
and the model answers from it directly.

Fewer moving parts means less to break during a live demo.

Repo: https://github.com/karthiks360/ResumeBot

```
ResumeBot/
├── main.py              FastAPI app — /chat, /chat/stream, /health
├── resume.txt           the resume (swap in your own)
├── static/index.html    full-screen chat UI, streaming, no build step, no CDN
├── requirements.txt
└── .env.example
```

## Configuration

All settings live in `.env` (copy it from `.env.example`). Nothing is
hard-coded in `main.py` — the app refuses to start if a required value is
missing.

| Variable                | Required | Example                | Notes |
| ----------------------- | -------- | ---------------------- | ----- |
| `GROQ_API_KEY`          | yes      | `gsk_...`              | from https://console.groq.com/keys |
| `GROQ_MODEL`            | yes      | `openai/gpt-oss-120b`  | any chat model your Groq account can use |
| `GROQ_REASONING_EFFORT` | no       | `low` (default)        | `low` / `medium` / `high` — for reasoning models such as gpt-oss |

To see which models your key can use:

```bash
python -c "from dotenv import load_dotenv; load_dotenv(); from groq import Groq; print('\n'.join(sorted(m.id for m in Groq().models.list().data)))"
```

> Groq retires models from time to time (`llama-3.3-70b-versatile` is gone).
> If you get `404 model_not_found`, pick a model from the list above, put it in
> `GROQ_MODEL` and restart — no code change needed.
>
> `gpt-oss` is a reasoning model: its thinking tokens count against the output
> limit, which is why the effort defaults to `low` and `max_tokens` is 1500. If
> you switch to a non-reasoning model and Groq rejects `reasoning_effort`,
> remove the `extra_body` lines in `main.py`.

## Run locally

**Windows (PowerShell)**

```powershell
git clone https://github.com/karthiks360/ResumeBot.git
cd ResumeBot
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt

Copy-Item .env.example .env     # then edit .env and add your key
uvicorn main:app --reload
```

If PowerShell blocks `Activate.ps1`, run
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, or skip activation
and call `.venv\Scripts\uvicorn main:app --reload` directly.

**Linux / macOS**

```bash
git clone https://github.com/karthiks360/ResumeBot.git
cd ResumeBot
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env            # then add your key
uvicorn main:app --reload
```

Open http://127.0.0.1:8000

## Endpoints

| Method | Path           | Purpose                                      |
| ------ | -------------- | -------------------------------------------- |
| GET    | `/`            | Chat UI                                      |
| GET    | `/health`      | Liveness probe — safe for an ALB target group |
| POST   | `/chat`        | Plain JSON in/out — the one to curl on stage |
| POST   | `/chat/stream` | SSE, token by token — what the UI uses       |
| GET    | `/docs`        | Auto-generated OpenAPI docs (free with FastAPI) |

The JSON endpoint, for the API half of your demo:

```bash
curl -s localhost:8000/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"Which certifications does the candidate hold?"}' | jq -r .reply
```

Multi-turn — pass prior turns back in `history`:

```bash
curl -s localhost:8000/chat -H 'Content-Type: application/json' -d '{
  "message": "How long was he there?",
  "history": [
    {"role":"user","content":"Where does he work now?"},
    {"role":"assistant","content":"He is a Senior Cloud Security Engineer at NorthBridge Payments."}
  ]
}' | jq -r .reply
```

## Deploy on EC2

Tested against Amazon Linux 2023, `t3.micro` is plenty.

**1. Security group** — inbound `22` from your IP only, and `8000` from your IP
(or `0.0.0.0/0` if the audience needs to hit it from their own laptops).

**2. SSH in** (from Windows PowerShell)

```powershell
# once: Windows OpenSSH refuses a key file other users can read
icacls C:\path\to\your-key.pem /inheritance:r /grant:r "$($env:USERNAME):R"
ssh -i C:\path\to\your-key.pem ec2-user@<ec2-public-ip>
```

**3. On the instance**

```bash
sudo dnf install -y python3.11 python3.11-pip git
git clone https://github.com/karthiks360/ResumeBot.git resume-bot && cd resume-bot
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cat > .env <<'ENV'
GROQ_API_KEY=gsk_your_real_key
GROQ_MODEL=openai/gpt-oss-120b
ENV
chmod 600 .env
```

**4. Smoke test it in the foreground first**

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
curl localhost:8000/health
```

Then visit `http://<ec2-public-ip>:8000`.

**5. Make it survive your SSH session** — run it under systemd:

```bash
sudo tee /etc/systemd/system/resume-bot.service > /dev/null <<'UNIT'
[Unit]
Description=Resume Bot
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/resume-bot
EnvironmentFile=/home/ec2-user/resume-bot/.env
ExecStart=/home/ec2-user/resume-bot/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now resume-bot
systemctl status resume-bot
```

Logs: `journalctl -u resume-bot -f`

**6. Deploying updates**

```bash
cd ~/resume-bot && git pull
source .venv/bin/activate && pip install -r requirements.txt   # only if requirements changed
sudo systemctl restart resume-bot
```

On Ubuntu instead of Amazon Linux: the user is `ubuntu` (also in the service
file paths), install with `sudo apt install -y python3-venv python3-pip git`,
and use `python3` instead of `python3.11`.

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| `404 model_not_found` in the chat | The model in `GROQ_MODEL` was retired or isn't on your account. List models (see *Configuration*), update `.env`, restart. |
| App exits with `GROQ_API_KEY is not set` / `GROQ_MODEL is not set` | Missing line in `.env`. |
| Works with `curl localhost:8000/health` on the box but not from your browser | Port 8000 not open in the security group, or your IP changed. |
| systemd service won't start | `journalctl -u resume-bot -n 50` — usually a bad `.env`. |

## Customising

- **Resume:** replace `resume.txt` and restart — nothing else to change. Plain
  text beats PDF here: no parser to misread column layouts, and you control
  exactly what the model sees.
- **UI text:** for a different candidate, edit the greeting bubble and the
  suggestion chips in `static/index.html`.
- **UI colours:** the chat UI is full-screen with a navy/gold theme. All colours
  are CSS variables in the `:root` block at the top of `static/index.html`
  (`--brand`, `--gold`, …), with a matching dark-mode block below it.

## Notes for the demo

- `temperature=0.2` — you want the same answer twice if someone asks twice.
- The system prompt forbids inventing details and tells the model to say when
  something is not in the resume. Worth demonstrating: ask it something the
  resume does not cover (e.g. *"Does he know Rust?"*) and show it declines
  rather than bluffing. That is the part audiences remember.
- Conversation history is bounded to the last 8 turns, so a long demo session
  will not inflate the prompt indefinitely.
- History lives in the browser, not the server — the app is stateless, so it
  scales horizontally behind an ALB with no session affinity.

## Caveats

- **The API key sits in `.env` on the box.** Fine for a demo. For anything
  longer-lived, put it in SSM Parameter Store (SecureString) or Secrets Manager
  and give the instance an IAM role to read it. `.env` is in `.gitignore` —
  never commit it.
- **Plain HTTP on port 8000.** Fine for a demo. Put it behind an ALB with ACM,
  or Caddy/nginx with Let's Encrypt, before it is reachable from anywhere real.
- **No rate limiting or auth.** If you open port 8000 to `0.0.0.0/0`, anyone
  who finds the IP can spend your Groq quota.
