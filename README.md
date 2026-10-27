# Resume Bot — FastAPI + Groq

A small resume Q&A bot. The resume is ~8.4 KB of text, so it fits entirely in
the model's context window: there is **no vector DB, no embeddings, no
retrieval step**. The resume is injected into the system prompt once at startup
and the model answers from it directly.

Fewer moving parts means less to break during a live demo.

```
resume-bot/
├── main.py              FastAPI app — /chat, /chat/stream, /health
├── resume.txt           the resume (swap in your own)
├── static/index.html    chat UI, streaming, no build step, no CDN
├── requirements.txt
└── .env.example
```

## Run locally

```bash
python -m venv .venv
.venv/Scripts/activate          # Windows
# source .venv/bin/activate     # Linux/macOS
pip install -r requirements.txt

cp .env.example .env            # then add your key from console.groq.com/keys
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

**2. On the instance**

```bash
sudo dnf install -y python3.11 python3.11-pip git
git clone <your-repo> resume-bot && cd resume-bot
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

echo 'GROQ_API_KEY=gsk_your_real_key' > .env
```

**3. Smoke test it in the foreground first**

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
curl localhost:8000/health
```

Then visit `http://<ec2-public-ip>:8000`.

**4. Make it survive your SSH session** — run it under systemd:

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

## Swapping in a real resume

Replace `resume.txt` and restart — nothing else to change. Plain text beats PDF
here: no parser to misread column layouts, and you control exactly what the
model sees. Two edits worth making in `static/index.html` for a different
candidate: the greeting bubble and the suggestion chips.

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
  and give the instance an IAM role to read it.
- **Plain HTTP on port 8000.** Fine for a demo. Put it behind an ALB with ACM,
  or Caddy/nginx with Let's Encrypt, before it is reachable from anywhere real.
- **No rate limiting or auth.** If you open port 8000 to `0.0.0.0/0`, anyone
  who finds the IP can spend your Groq quota.
