# CNE Deadman’s Switch — Cheapest Serverless Heartbeat Monitor

Disclaimer: This project was AI vibe coded.

A single-file AWS CloudFormation stack that deploys an HTTP endpoint you ping on a schedule.
If **no request is received for 6 minutes**, it sends an **ALARM** email; once traffic resumes, it sends an **OK**.

---

## How It Works

| Component                | Purpose                                                                                                 |
|--------------------------|---------------------------------------------------------------------------------------------------------|
| **Heartbeat API (HTTP)** | Exposes `https://<api>/<random-uuid>` — the heartbeat URL. |
| **Pings API (HTTP)**     | Exposes `https://<pings-api>/` — returns the 15 most recent hits. |
| **Ping Lambda**          | Lightweight stub: returns `200 OK` for any method on the random path.                                  |
| **Pings Lambda**         | Reads CloudWatch metrics to list recent hits (for quick debugging).                                    |
| **CloudWatch Alarm**     | `ALARM` if `AWS/ApiGateway Count < 1` in the last minute (i.e., 6 min without pings); returns to `OK` 4 min after next hit. |
| **SNS Topic**            | Sends `ALARM`/`OK` notifications (email, SMS, etc.).                                                   |

---

## Deploy

```bash
# 1. Validate (optional)
cfn-lint cne_deadmans_switch.yaml

# 2. Deploy (replace <stack-name>, <region>)
aws cloudformation deploy \
  --template-file cne_deadmans_switch.yaml \
  --stack-name <stack-name> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region <region>
```

Deployment creates all resources with zero parameters.

⸻

Subscribe to alerts

```bash
# Get the SNS topic ARN
aws cloudformation describe-stacks --stack-name <stack-name> \
  --query "Stacks[0].Outputs[?OutputKey=='SnsTopicArn'].OutputValue" --output text

# Subscribe an e‑mail address
aws sns subscribe \
  --topic-arn <topic-arn> \
  --protocol email \
  --notification-endpoint you@example.com
```
Confirm the subscription from your inbox.

⸻

Use the heartbeat
```bash
# One‑liner cron job example (every minute)
* * * * * curl -fsS --retry 3 $(aws cloudformation \
      describe-stacks --stack-name <stack-name> \
      --query "Stacks[0].Outputs[?OutputKey=='EndpointURL'].OutputValue" \
      --output text) >/dev/null
```
	•	Miss 6 minutes of pings → ALARM mail
	•	Resume pings → OK mail (≈4 minutes later)

Check recent pings
```bash
curl $(aws cloudformation \
      describe-stacks --stack-name <stack-name> \
      --query "Stacks[0].Outputs[?OutputKey=='PingsURL'].OutputValue" \
      --output text) | jq .
```
Returns an array of ISO‑8601 timestamps (local time — America/Chicago).

⸻

Clean up
```bash
aws cloudformation delete-stack --stack-name <stack-name>
```
Stack deletion removes all resources and stops all costs.

⸻

## Cost

| Service           | Steady State              | Notes                                         |
|-------------------|--------------------------|-----------------------------------------------|
| API Gateway (x2)  | $0.00 when idle          | Two HTTP APIs, both have per-request pricing  |
| Lambda            | $0.00 when idle          | 128 MB × sub‑second → free at low volume      |
| CloudWatch & SNS  | < $0.01/mo typical usage | At typical heartbeat traffic                  |

Total monthly cost is effectively $0 for most hobby workloads.

⸻

Development
```bash
pip install -r requirements.txt   # linters
cfn-lint cne_deadmans_switch.yaml # validate
yamllint cne_deadmans_switch.yaml # style
```
Both Lambda handlers are defined inline in the template; no separate source tree is required.

⸻

## Outputs

| Key               | Description                                 |
|-------------------|---------------------------------------------|
| EndpointURL       | Heartbeat URL you ping.                     |
| RandomPathSegment | The UUID segment used in the URL.           |
| PingsURL          | URL that lists the last 15 heartbeat timestamps. |
| SnsTopicArn       | ARN of the SNS topic for ALARM/OK notifications. |

⸻
