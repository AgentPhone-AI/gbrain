---
id: agentphone-voice-brain
name: AgentPhone Voice-to-Brain
version: 0.1.0
description: Phone calls create brain pages via AgentPhone hosted voice agents. No Twilio, no ngrok, no server — 3 API calls replace 10 setup steps.
category: sense
requires: []
secrets:
  - name: AGENTPHONE_API_KEY
    description: AgentPhone API key
    where: https://agentphone.to/settings — sign in, copy your API key
health_checks:
  - "curl -sf -H \"Authorization: Bearer $AGENTPHONE_API_KEY\" https://api.agentphone.to/v1/agents > /dev/null && echo 'AgentPhone: OK' || echo 'AgentPhone: FAIL'"
setup_time: 5 min
cost_estimate: "Usage-based (~$0.10/min voice, no monthly minimums)"
---

# AgentPhone Voice-to-Brain: Phone Calls That Create Brain Pages

Call a phone number. Talk. A structured brain page appears — same result as the
Twilio voice recipe, without the infrastructure.

AgentPhone hosts the voice agent, the phone number, and the call routing. You
bring the API key. The agent creates brain pages from every call.

## Quick Comparison with DIY Recipe

|                        | twilio-voice-brain (DIY)                         | agentphone-voice-brain   |
| ---------------------- | ------------------------------------------------ | ------------------------ |
| **Setup steps**        | 10 (Twilio, ngrok, Node server, TwiML, watchdog) | 3 API calls              |
| **Accounts needed**    | Twilio + OpenAI + ngrok ($8/mo)                  | AgentPhone only          |
| **Server to host**     | Node.js voice server (port 8765)                 | None                     |
| **Tunnel**             | ngrok required                                   | Not needed               |
| **Time to first call** | 30-60 minutes                                    | Under 5 minutes          |
| **Monthly cost**       | ~$22/mo + server                                 | Usage-based, no infra    |

If you need full control over the voice pipeline (custom STT, your own TTS,
WebSocket-level audio access), use `twilio-voice-brain` instead. AgentPhone
trades that control for zero infrastructure.

## IMPORTANT: Instructions for the Agent

**You are the installer.** This recipe is written for YOU (the AI agent) to execute
on behalf of the user. Follow these instructions precisely.

**This recipe has zero infrastructure dependencies.** No ngrok, no Node.js server,
no Twilio account. AgentPhone hosts everything. Your job is:
1. Collect the API key
2. Create a voice agent
3. Buy a phone number
4. Make a test call
5. Set up call-to-brain page flow

**Do not skip steps. Verify after each step.**

## Architecture

```
Caller (phone)
  | inbound or outbound call
AgentPhone (hosted voice agent — STT + LLM + TTS)
  | call ends, transcript available via webhook
Your machine (webhook handler OR polling script)
  | writes transcript as brain page
GBrain (meetings/YYYY-MM-DD-call-{caller}.md)
  | agent enrichment
Entity propagation, cross-references, summary to messaging
```

**Key difference from DIY:** There is no server on your machine handling live audio.
AgentPhone hosts the entire voice pipeline. Your machine only receives the finished
transcript after the call ends.

## Setup Flow

### Step 1: Get AgentPhone API Key

Tell the user:
"I need your AgentPhone API key. Here's where to find it:

1. Go to https://agentphone.to/settings
2. Sign in (or create an account)
3. Copy your API key
4. Paste it to me"

Validate immediately:
```bash
curl -sf -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  https://api.agentphone.to/v1/agents \
  | grep -q '"agents"' \
  && echo "PASS: AgentPhone API connected" \
  || echo "FAIL: AgentPhone API key invalid"
```

**If validation fails:** "That didn't work. Make sure you copied the full key from
https://agentphone.to/settings. Keys start with `ap_`."

**STOP until AgentPhone validates.**

### Step 2: Create a Voice Agent

```bash
curl -s -X POST https://api.agentphone.to/v1/agents \
  -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Brain",
    "voice_mode": "hosted",
    "system_prompt": "You are Brain, a personal AI assistant with deep knowledge about the caller'\''s life, work, and interests. Greet callers warmly. Ask what'\''s on their mind. Have natural, helpful conversations. Be concise — you'\''re on a phone call.",
    "begin_message": "Hey, it'\''s Brain. What'\''s on your mind?",
    "voice": "11labs-Brian"
  }'
```

Save the returned `agent_id`. Verify:
```bash
curl -sf -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  "https://api.agentphone.to/v1/agents/$AGENT_ID" \
  | grep -q '"name"' \
  && echo "PASS: Agent created" \
  || echo "FAIL: Agent not found"
```

Tell the user: "Created voice agent 'Brain' (ID: $AGENT_ID). It will greet
callers and have natural conversations."

### Step 3: Buy a Phone Number

Ask the user: "What area code do you want for Brain's phone number? (e.g., 415
for San Francisco, 212 for New York)"

```bash
curl -s -X POST https://api.agentphone.to/v1/numbers/buy \
  -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "country": "US",
    "area_code": "AREA_CODE",
    "agent_id": "AGENT_ID"
  }'
```

Save the returned `phone_number`. Tell the user: "Brain's number is $PHONE_NUMBER.
Anyone who calls it gets the AI voice agent."

### Step 4: Smoke Test — Call the User

Ask the user for their phone number, then:

```bash
curl -s -X POST https://api.agentphone.to/v1/calls \
  -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "AGENT_ID",
    "to_number": "USER_PHONE_NUMBER",
    "topic": "Have a natural conversation as Brain, the user'\''s personal AI assistant."
  }'
```

Tell the user: "Calling you now from Brain. Pick up and have a short conversation
(30 seconds is enough). I'll wait for you to finish, then we'll verify the
transcript came through."

**STOP and wait for the user to confirm the call worked.**

**If the call doesn't come through:** Check the phone number format (must be E.164:
+1XXXXXXXXXX for US). Verify the agent exists. Try again.

### Step 5: Set Up Call-to-Brain Page Flow

After each call, AgentPhone can send the transcript via webhook. Set up a
lightweight handler that writes transcripts as brain pages.

**Option A: MCP-native (recommended if using Claude Code or Claude Desktop)**

If the user already has GBrain as an MCP server, add AgentPhone alongside it.
The AI client can then create brain pages directly from call data using natural
language.

Add to MCP configuration:
```json
{
  "mcpServers": {
    "gbrain": {
      "command": "npx",
      "args": ["-y", "gbrain-mcp"],
      "env": { "GBRAIN_DIR": "/path/to/brain" }
    },
    "agentphone": {
      "command": "npx",
      "args": ["-y", "agentphone-mcp"],
      "env": {
        "AGENTPHONE_API_KEY": "YOUR_KEY"
      }
    }
  }
}
```

With both MCP servers active, the AI client can:
- List recent calls and their transcripts
- Create brain pages from call data
- Search the brain during calls (if agent tools are configured)

**Option B: Webhook handler (automated, no AI client needed)**

Create a webhook handler that writes transcripts as brain pages automatically:

```bash
mkdir -p call-to-brain
```

Create `call-to-brain/handler.mjs`:
```javascript
import { createServer } from "node:http";
import { writeFileSync, existsSync, mkdirSync } from "node:fs";
import { execSync } from "node:child_process";

const BRAIN_DIR = process.env.GBRAIN_DIR || process.env.HOME + "/brain";
const MEETINGS_DIR = BRAIN_DIR + "/meetings";
const PORT = 9876;

if (!existsSync(MEETINGS_DIR)) mkdirSync(MEETINGS_DIR, { recursive: true });

const server = createServer((req, res) => {
  if (req.method !== "POST") {
    res.writeHead(200);
    return res.end("ok");
  }

  let body = "";
  req.on("data", (chunk) => (body += chunk));
  req.on("end", () => {
    try {
      const payload = JSON.parse(body);
      if (payload.event === "agent.call_ended") {
        const call = payload.data;
        const caller = (call.from || "unknown").replace(/\+/g, "");
        const date = new Date().toISOString().split("T")[0];
        const duration = call.duration || "unknown";
        const transcript = call.transcript || "(no transcript)";
        const slug = `call-${caller}`;

        // Idempotent: skip if page already exists for this call
        const filename = `${date}-${slug}.md`;
        const filepath = `${MEETINGS_DIR}/${filename}`;
        if (existsSync(filepath)) {
          res.writeHead(200);
          return res.end("duplicate");
        }

        const page = `---
type: meeting
source: agentphone
source_id: ${call.id || "unknown"}
title: "Phone Call - ${call.from || "unknown"}"
date: ${date}
duration: ${duration}s
tags: [call, agentphone]
---

# Phone Call - ${call.from || "unknown"}

Duration: ${duration}s

## Transcript

${transcript}
`;
        writeFileSync(filepath, page);
        console.log(`Wrote ${filename}`);

        // Git commit + sync
        try {
          execSync(`git add "${filename}" && git commit -m "call: ${date} ${call.from || "unknown"}"`, {
            cwd: BRAIN_DIR,
            stdio: "pipe",
          });
        } catch (_) {
          // git commit may fail if nothing staged — that's fine
        }
      }
      res.writeHead(200);
      res.end("ok");
    } catch (err) {
      console.error("Webhook error:", err.message);
      res.writeHead(400);
      res.end("bad request");
    }
  });
});

server.listen(PORT, () => console.log(`Call-to-brain handler on :${PORT}`));
```

Start the handler:
```bash
node call-to-brain/handler.mjs &
```

If the user has ngrok or a public URL, register the webhook:
```bash
curl -s -X POST https://api.agentphone.to/v1/webhooks \
  -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://YOUR_PUBLIC_URL/webhook"}'
```

If no public URL is available, tell the user: "The handler is running locally.
For automatic transcript capture, you'll need a public URL (ngrok, Cloudflare
Tunnel, or a cloud server). Alternatively, you can poll for recent calls and
create brain pages manually using the MCP approach."

**Option C: Polling script (no webhook, no public URL)**

For users who don't want to run a server or expose a webhook:

```bash
# Poll recent calls and create brain pages
curl -sf -H "Authorization: Bearer $AGENTPHONE_API_KEY" \
  "https://api.agentphone.to/v1/calls?agent_id=$AGENT_ID&limit=10" \
  | node -e "
    const data = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    (data.calls || []).forEach(c => {
      if (c.status === 'completed' && c.transcript) {
        const date = c.created_at?.split('T')[0] || new Date().toISOString().split('T')[0];
        const caller = (c.from || 'unknown').replace(/\+/g, '');
        console.log('---');
        console.log('Call from', c.from, 'on', date, '-', c.duration + 's');
        console.log(c.transcript.substring(0, 200) + '...');
      }
    });
  "
```

This can be wrapped in a cron job (see Step 6).

### Step 6: Set Up Cron (Optional)

For Option C (polling), set up a cron job:
```bash
# Every 30 minutes during business hours
*/30 9-18 * * 1-5 cd /path/to/call-to-brain && node poll-and-create.mjs >> /tmp/agentphone-sync.log 2>&1
```

For Option B (webhook handler), ensure the handler starts on boot:
```bash
# Add to crontab
@reboot cd /path/to/call-to-brain && node handler.mjs >> /tmp/agentphone-handler.log 2>&1
```

### Step 7: Log Setup Completion

```bash
mkdir -p ~/.gbrain/integrations/agentphone-voice-brain
echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"setup_complete","source_version":"0.1.0","agent_id":"'$AGENT_ID'","phone_number":"'$PHONE_NUMBER'","status":"ok"}' >> ~/.gbrain/integrations/agentphone-voice-brain/heartbeat.jsonl
```

Tell the user:
"Voice-to-brain is set up with AgentPhone.

- **Brain's number:** $PHONE_NUMBER
- **Anyone who calls** gets the AI voice agent
- **Transcripts** automatically become brain pages
- **No infrastructure** to maintain — AgentPhone hosts everything

To have your personal phone answered by Brain, forward your phone to $PHONE_NUMBER
in your carrier settings."

## What You Skip (vs. twilio-voice-brain)

The DIY voice recipe requires:

1. ~~Verify Node.js 18+ installed~~ Not needed
2. ~~Get Twilio Account SID + Auth Token~~ Not needed
3. ~~Get OpenAI API key for Realtime~~ Not needed
4. ~~Get ngrok auth token ($8/mo for fixed domain)~~ Not needed
5. ~~Launch ngrok tunnel on port 8765~~ Not needed
6. ~~Create Node.js voice server with WebSocket bridge~~ Not needed
7. ~~Configure Twilio phone number webhook~~ Not needed
8. ~~Verify health endpoint~~ Not needed
9. ~~Set up caller routing and OTP~~ Handled by AgentPhone
10. ~~Create watchdog cron job~~ Not needed

## Python Setup (Alternative)

For users who prefer scripting over agent-driven setup:

```python
from agentphone import AgentPhone
import os

client = AgentPhone(api_key=os.environ["AGENTPHONE_API_KEY"])

# 1. Create a voice agent
agent = client.agents.create(
    name="Brain",
    voice_mode="hosted",
    system_prompt=(
        "You are Brain, a personal AI assistant with deep knowledge "
        "about the caller's life, work, and interests. "
        "Greet callers warmly. Ask what's on their mind. "
        "Have natural, helpful conversations. Be concise - you're on a phone call."
    ),
    begin_message="Hey, it's Brain. What's on your mind?",
    voice="11labs-Brian",
)

# 2. Buy a phone number
number = client.numbers.buy(country="US", area_code="415", agent_id=agent.id)
print(f"Brain's number: {number.phone_number}")

# 3. Test it
call = client.calls.make_conversation(
    agent_id=agent.id,
    to_number="+14155551234",  # replace with your number
    topic="Have a natural conversation as Brain.",
)
print(f"Calling... {call.id}")
```

## Inbound Calls

Once your agent has a number, **anyone can call it**. The hosted AI picks up
automatically and follows the system prompt. No TwiML, no ngrok, no watchdog.

To have your personal phone answered by Brain, forward your phone to the agent's
number in your carrier settings.

## Troubleshooting

**Call doesn't come through:**
- Verify phone number format: must be E.164 (+1XXXXXXXXXX for US)
- Check agent exists: `curl -H "Authorization: Bearer $AGENTPHONE_API_KEY" https://api.agentphone.to/v1/agents/$AGENT_ID`
- Check number is assigned to agent: the `buy` response should show `agent_id`

**No transcript after call:**
- Transcripts are available after the call ends (not during)
- Short calls (<5 seconds) may not generate transcripts
- Check call status: `curl -H "Authorization: Bearer $AGENTPHONE_API_KEY" https://api.agentphone.to/v1/calls/$CALL_ID`

**Webhook not firing:**
- Verify webhook URL is reachable from the internet
- Check webhook registration: `curl -H "Authorization: Bearer $AGENTPHONE_API_KEY" https://api.agentphone.to/v1/webhooks`
- AgentPhone retries failed webhooks 3 times with exponential backoff

**Brain page not created:**
- Check the handler is running: `curl http://localhost:9876`
- Check GBRAIN_DIR is set correctly
- Look at handler logs: `tail -f /tmp/agentphone-handler.log`

## Cost Estimate

| Component | Cost |
|-----------|------|
| AgentPhone voice | ~$0.10/min |
| AgentPhone number | ~$2/mo |
| Infrastructure | $0 (hosted) |
| **10 min/day usage** | **~$32/mo** |
| **Occasional use (2 min/day)** | **~$8/mo** |
