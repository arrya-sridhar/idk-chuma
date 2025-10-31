# LYNQ Agentic AI Workshop

Your goal: Build an n8n workflow that correctly identifies targets, explains why, and submits to win the flag.


## Challenge 1: Red Gate — Intrusion Detector

**Mission:** Find and lock brute-force attackers.

### What You Get

- **Data File:** `login_attempts.json` (list of login events with user_id, success status, timestamp)
- **Endpoint:** `http://[provided-url]/red-gate/submit`

### What You Do

**Step 1: Understand the data**
- Open `login_attempts.json`
- You'll see login attempts like: `{ "user_id": "alice", "success": true }` and `{ "user_id": "bob", "success": false }`

**Step 2: Build your agent in n8n**

1. **Manual Trigger** → starts the flow
2. **HTTP GET** → fetch the JSON file
3. **LLM Brain** (HTTP POST to Flowise/Langflow):
   - **Prompt:** "Analyze this list of login attempts. Find users with 3+ consecutive failed attempts. Output only clean JSON: `{ "users_to_lock": ["user_id"], "reasoning": "explanation" }`"
   - Copy the Flowise API endpoint URL into your HTTP POST node
4. **JSON Parse** → clean up the LLM output
5. **HTTP POST** → submit to the validator endpoint with body:
   ```json
   {
     "users_to_lock": ["alice", "bob"],
     "reasoning": "Users alice and bob showed brute-force patterns with 3+ failed attempts."
   }
   ```

**Step 3: Test**
- Click "Execute Workflow"
- Check the final HTTP node's output
- **Success:** Response contains `FLAG{red_gate_defended}`

---

## Challenge 2: Blue Core — Server Overload

**Mission:** Detect critical servers and restart them.

### What You Get

- **Data File:** `server_stats.json` (list of servers with CPU %, memory %, requests/sec)
- **Endpoint:** `http://[provided-url]/blue-core/submit`

### What You Do

**Step 1: Understand the data**
- Open `server_stats.json`
- You'll see entries like: `{ "server_id": "db-02", "cpu_percent": 98, "memory_percent": 95 }`
- **Rule:** If CPU > 90% AND Memory > 90% → server is CRITICAL and needs RESTART
- Otherwise → REPORT

**Step 2: Build your agent in n8n**

1. **Manual Trigger**
2. **HTTP GET** → fetch the JSON file
3. **LLM Brain** (HTTP POST to Flowise/Langflow):
   - **Prompt:** "Analyze server stats. Critical = CPU > 90% AND Memory > 90%. Output clean JSON array: `[ { "server_id": "db-02", "action": "RESTART", "justification": "CPU and memory are both critical" }, ... ]`"
4. **Loop/Split In Batches** → process each server individually
5. **IF Node** → branch on action == "RESTART"
   - **TRUE** → HTTP POST to validator
   - **FALSE** → skip (or log as REPORT)
6. **HTTP POST** (in TRUE branch) → submit to endpoint with body:
   ```json
   {
     "server_id": "db-02",
     "action": "RESTART",
     "justification": "CPU usage is 98% and memory is 95%, exceeding critical thresholds."
   }
   ```

**Step 3: Test**
- Execute the workflow
- Watch the loop process each server
- **Success:** When the RESTART decision is submitted, response contains `FLAG{blue_core_stabilized}`

---

## Challenge 3: Green Signal — Phishing Detector

**Mission:** Classify emails as phishing or legitimate and flag the phishing ones.

### What You Get

- **Data File:** `emails.json` (list of email objects with sender, subject, body)
- **Endpoint:** `http://[provided-url]/green-signal/submit`

### What You Do

**Step 1: Understand the data**
- Open `emails.json`
- You'll see entries like:
  ```json
  {
    "sender": "noreply@bank-alerts.net",
    "subject": "Urgent: Verify Your Account",
    "body": "Click here immediately to prevent suspension..."
  }
  ```
- **Task:** Classify each as PHISHING or LEGITIMATE
- Red flags: urgent language, suspicious sender domain, requests for passwords, poor grammar

**Step 2: Build your agent in n8n**

1. **Manual Trigger**
2. **HTTP GET** → fetch the JSON file
3. **Loop** → process each email individually
4. **LLM Brain** (HTTP POST to Flowise/Langflow):
   - **Prompt:** "Classify this email as PHISHING or LEGITIMATE. Look for urgent language, suspicious domains, requests for info. Output clean JSON: `{ "classification": "PHISHING", "reasoning_summary": "explanation" }`"
   - Use expressions to pass sender/subject/body from the current loop item
5. **JSON Parse** → clean up output
6. **Aggregate/Merge** → collect all classifications back into one array
7. **HTTP POST** → submit to endpoint with body:
   ```json
   [
     {
       "email": {
         "sender": "noreply@bank-alerts.net",
         "subject": "Urgent: Verify Your Account",
         "body": "..."
       },
       "classification": "PHISHING",
       "reasoning_summary": "Urgent language and suspicious sender domain."
     },
     { ... }
   ]
   ```

**Step 3: Test**
- Execute the workflow
- Watch the loop process each email
- Merge collects all into one array
- **Success:** When submitted, response contains `FLAG{green_signal_secured}`

---

## General Workflow Template (Applies to All Challenges)

```
Manual Trigger
    ↓
HTTP GET (fetch data file)
    ↓
[Optional: Loop] ← if processing multiple items
    ↓
HTTP POST (call LLM brain at Flowise/Langflow)
    ↓
JSON Parse (clean output)
    ↓
[Optional: IF Node or Aggregate]
    ↓
HTTP POST (submit to validator endpoint)
    ↓
SUCCESS: Check response for FLAG
```

---

## Critical Rules 

**Do This:**
- Test each node as you build it (click "Execute step")
- Keep LLM prompts clear and instruct it to output ONLY clean JSON
- Include reasoning in every submission (that's part of the scoring)
- Use exact field names from the challenge description
- Check the HTTP node's Request tab to verify the payload before final submission

**Don't Do This:**
- Don't hardcode answers (e.g., manually typing user_id names)
- Don't submit without reasoning
- Don't ignore the JSON structure requirements
- Don't skip the LLM brain—your agent must think, not just copy data

---

## Common Mistakes & Fixes

| Problem | Solution |
|---------|----------|
| LLM output isn't clean JSON | Rewrite prompt to say: "Output ONLY valid JSON, no extra text" |
| HTTP node returns 400 error | Check the Request tab—verify JSON structure matches expectations |
| Loop isn't iterating | Make sure Split In Batches is set to loop over the data array |
| IF node not branching | Double-check the condition syntax (e.g., `action == "RESTART"`) |
| No flag in response | Review the submission payload—field names must be exact |

---

## Troubleshooting Checklist

- [ ] Data file loads when you run HTTP GET
- [ ] LLM brain returns clean JSON (test manually in Flowise first if stuck)
- [ ] Loop/Split correctly processes multiple items (inspect one iteration)
- [ ] IF node correctly identifies your target (check condition preview)
- [ ] Submission payload matches the structure in the challenge description
- [ ] Endpoint URL is correct (copy-paste from organizers)
- [ ] Response from validator contains the FLAG

---

## Final Tips

1. **Start with Red Gate** if you're new—it's the simplest (no looping needed).
2. **Reuse your workflow pattern.** All three challenges follow the same steps; just swap the data and LLM prompt.
3. **Debug node-by-node.** Don't wire everything and hit "Execute"—test each node individually.
4. **Pay attention to JSON structure.** The validator will reject mismatched field names or types.
5. **Reason matters.** A correct guess with no explanation won't win you points—your agent must explain its logic.

---

## Good Luck! 

You've got this. Build smart, test often, and let your agent do the thinking!

---
