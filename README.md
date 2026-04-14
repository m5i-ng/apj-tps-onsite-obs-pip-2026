# Pipeline Hero — Datadog Observability Pipelines Hands-on Lab

> **APJ TPS Onsite Event | 2026**
> *DeliverLah's logs are a mess! Hey! Let's fix them.*

---

## Table of Contents

- [Overview](#overview)
- [Pre-requisites](#pre-requisites)
- [Getting Started](#getting-started)
- [Lab Environment Setup](#lab-environment-setup)
- [Observation Questions](#observation-questions)
- [Build: Create Your Pipeline](#build-create-your-pipeline)
  - [Create the Pipeline in Datadog UI](#create-the-pipeline-in-datadog-ui)
  - [Start OPW](#start-opw)
  - [Route the Datadog Agent through OPW](#route-the-datadog-agent-through-opw)
- [Fix: Add Processors for Each Scenario](#fix-add-processors-for-each-scenario)
  - [S1 · PII in Structured JSON](#s1--pii-in-structured-json)
  - [S2 · Inconsistent Field Naming](#s2--inconsistent-field-naming)
  - [S3 · Null and Empty Fields](#s3--null-and-empty-fields)
  - [S4 · ddtags Array Extraction](#s4--ddtags-array-extraction)
  - [S5 · High-Frequency Duplicate Events](#s5--high-frequency-duplicate-events)
- [Validate](#validate)
- [Monitor the Pipeline](#monitor-the-pipeline)
- [Clean up](#clean-up)

---

## Overview

**DeliverLah** is a fast-growing APAC food delivery platform serving 8 cities: Singapore, Sydney, Tokyo, Seoul, Jakarta, Bangkok, Kuala Lumpur, and Manila.

Their platform is a collection of microservices built by different teams, across different years, using different languages and logging conventions. As they scale, three problems are getting worse:

- **Compliance** — PII (card numbers, SSNs, addresses, emails) is flowing into Datadog unredacted
- **Operational noise** — null fields, inconsistent field names, and split log events make alerting and dashboards unreliable
- **Cost** — high log volume from verbose and duplicated events

**Your mission:** Use Datadog Observability Pipelines to intercept DeliverLah's logs before they reach Datadog, fix the quality issues, and protect customer PII — all without touching application code.

---

## Pre-requisites

- [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
- Add the path `/opt/datadog-agent/run` under **Docker Desktop → Settings → Resources → File Sharing** *(required for Docker log collection on macOS)*
- Access to your Personal Datadog Sandbox org / "Dogfood" account

---

## Getting Started

Clone the lab repository:

```
cd ~
git clone https://github.com/m5i-ng/apj-tps-onsite-ops-pip-2026.git
```

---

## Lab Environment Setup

The lab uses three containers:

| Container | Image | Role |
|---|---|---|
| `dd-agent` | `datadog/agent:latest` | Collects container logs and forwards them |
| `deliverlah` | `min99/deliverlah:latest` | DeliverLah log simulator (stdout only) |
| `opw` | `datadog/observability-pipelines-worker:latest` | Intercepts and processes logs |

**1. Create the Docker network**

```
docker network create my-dd-network
```

**2. Set your API key**

```
vi ~/apac-onsite-event-2026/app/.env
```

```
DD_SITE=datadoghq.com
DD_API_KEY=<Your API Key>
```

**3. Start the Datadog Agent and DeliverLah app** *(OPW not yet)*

```
cd ~/apac-onsite-event-2026/app
docker-compose -f dc-app-init.yaml up -d
```

**4. Verify containers are running**

```
docker ps
```

Expected output:

```
CONTAINER ID   IMAGE                       STATUS          NAMES
xxxxxxxxxxxx   min99/deliverlah:latest    Up N minutes    deliverlah
xxxxxxxxxxxx   datadog/agent:latest        Up N minutes    dd-agent
```

**5. Check the simulator is generating logs**

```
docker logs deliverlah
```

**6. Confirm logs are arriving in Datadog**

Go to: https://app.datadoghq.com/logs

Filter by `container_name:deliverlah`. You should see a stream of logs — messy ones.

---

### Observation Questions

Before building any processors, spend a few minutes browsing the raw logs in Datadog Log Explorer. For each scenario below, open the relevant service logs and note what you see. This gives you a baseline to compare against once your processors are running.

**S1 · PII** — Filter `service:payment-service`. Open any log and expand the `payment_instrument` object. Look for `card_number` and `card_cvv` — they are in plain text. Then filter `service:user-service` and find an event with `event:profile_accessed`. Look for `ssn` and `national_id`.

**S2 · Field naming** — Filter to `api-gateway`, `order-service`, `legacy-crm`, `mobile-backend`, and `payment-service` in turn. Each service identifies the user differently: `userId`, `user_id`, `CUST_ID`, `uid`, `customer_id`. The first four also differ on HTTP status and response time field names. You cannot build a cross-service dashboard with these as-is.

**S3 · Null fields** — Filter `service:notification-service`. Expand any log event and count how many fields have values like `null`, `""`, `N/A`, `undefined`, or `-`. These are noise that pollute facets and dashboards.

**S4 · ddtags** — Filter `service:chaos-engineering`. Look at the `ddtags` field — it is an array of strings like `["env:prod", "team:sre", "version:2.1.4"]`. Try to filter on `env:prod` in Log Explorer. Notice it does not work because `env` is not a top-level attribute.

**S5 · Duplicate events** — Filter `service:delivery-service`. Look for events with `event:driver_poll` and `status:no_driver_available`. Find groups of events sharing the same `order_id` — each has `poll_count: 1` and is otherwise identical. Count how many duplicates arrive within a 10-second window for one order.


---

## Build: Create Your Pipeline

### Create the pipeline in Datadog UI

**1.** Go to: https://app.datadoghq.com/observability-pipelines

**2.** Click **"New Pipeline"** and name it `pipeline-hero`

**3.** Configure the **Source**:
   - Type: **Datadog Agent**
   - Address: `0.0.0.0:8282`
   - Click **Save**

**4.** Configure the **Destination**:
   - Type: **Datadog**
   - Click **Save**

**5.** Skip processors for now — click **"Deploy Pipeline"**

**6.** Note your **Pipeline ID** from the deployment page:
   > `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

---

### Start OPW

**1.** Set your API key and Pipeline ID:

```
vi ~/apac-onsite-event-2026/opw/.env
```

```
DD_OP_API_KEY=<Your Datadog API Key>
DD_OP_PIPELINE_ID=<Your Pipeline ID>
```

**2.** Review the bootstrap config (`~/apac-onsite-event-2026/opw/bootstrap.yaml`):

```yaml
api_key: "${DD_OP_API_KEY}"
pipeline_id: "${DD_OP_PIPELINE_ID}"
```

**3.** Start OPW:

```
cd ~/apac-onsite-event-2026/opw
docker-compose -f dc-opw.yaml up -d
```

**4.** Confirm OPW connected to Datadog:

```
docker logs opw
```

Look for:
```
INFO observability_pipelines_worker: Pipeline configuration loaded successfully
INFO observability_pipelines_worker: OPW is running and ready to receive data
```

---

### Route the Datadog Agent through OPW

**1.** Stop the agent:

```
docker stop dd-agent
```

**2.** See what changes:

```
cd ~/apac-onsite-event-2026/app
diff dc-app-init.yaml dc-app-update.yaml
```

**3.** Restart the agent with OPW routing:

```
docker-compose -f dc-app-update.yaml up -d
```

**4.** Confirm routing is active:

```
docker exec -it dd-agent agent status
```

Look for the Logs Agent section:

```
==========
Logs Agent
==========
  Reliable: Sending compressed logs in HTTP to opw on port 8282
```

---

## Fix: Add Processors for Each Scenario

Go to https://app.datadoghq.com/observability-pipelines, select `pipeline-hero`, and click **"Edit Pipeline"**.

For each scenario, add the appropriate processor(s) then click **"Deploy Pipeline"**.

> **Important:** Processors run in order, top to bottom. The order you add them matters — processors that parse or rename fields must run before processors that act on those fields.

---

### S1 · PII in Structured JSON

**Processor type:** Sensitive Data Scanner / Redact

**Where to look:** Filter by each service below and open a log event to inspect the JSON.

| Service | Object | Fields containing PII |
|---|---|---|
| `payment-service` | `payment_instrument` | `card_number`, `card_cvv`, `card_expiry` |
| `user-service` | `user` | `ssn`, `national_id`, `date_of_birth`, `address`, `email`, `phone` |
| `delivery-service` | `customer` / `driver` | `delivery_address`, `phone`, `name` |

**Questions:**

1. Which of these fields should be fully removed vs. masked (e.g., replaced with `****`)? Think about audit trail requirements vs. privacy.
2. The `ssn` field only appears on some events. How do you write a redaction rule that handles a field that may or may not be present?
3. Is it safer to redact by field name (e.g., delete `.card_number`) or by pattern (e.g., regex for `\d{4}-\d{4}-\d{4}-\d{4}`)? What are the trade-offs?

---

### S2 · Inconsistent Field Naming

**Processor type:** Remap

**Where to look:** Search across these four services and compare how each identifies the same concept:

| Concept | api-gateway | order-service | legacy-crm | mobile-backend | payment-service |
|---|---|---|---|---|---|
| User identifier | `userId` | `user_id` | `CUST_ID` | `uid` | `customer_id` |
| Request ID | `requestId` | `request_id` | `REQ_ID` | `rid` | — |
| HTTP status | `statusCode` | `status_code` | `HTTP_STATUS` | `s` | — |
| Response time | `responseTimeMs` | `response_time_ms` | `RESP_TIME` | `t` | — |

**Questions:**

1. If you create a monitor on `user_id`, which of these five services will trigger it?
2. What should the normalised field name be for each concept? Agree on a convention before writing the processor.
3. How would you handle services that already use the target name (e.g., `order-service` already has `user_id`) — do you need to guard against overwriting?
---

### S3 · Null and Empty Fields

**Processor type:** Remap — `compact`

**Where to look:** Find a log from `notification-service`. In the JSON tab, count fields with values: `null`, `""`, `"N/A"`, `"n/a"`, `"undefined"`, `"none"`, `"NONE"`, `"-"`.

**Questions:**

1. How many junk-value fields did you count in a single event? What percentage of the event's fields are noise?
2. Why does having null fields in logs cause problems for Datadog facets and dashboard widgets?

---

### S4 · ddtags Array Extraction

**Processor type:** Remap — filter ddtags array

**Where to look:** Find a log from `chaos-engineering`. The `ddtags` field is an array:

```json
"ddtags": ["env:prod", "team:sre", "service:chaos-engineering", "version:2.1.0", "pod_name:api-gateway-a1b2c-d3e4f"]
```

Try to filter logs in Log Explorer by `env:prod`. Does it work?

**Questions:**

1. Why can't you filter by `env:prod` even though the tag is present in `ddtags`?
2. After extraction, `env` and `version` are root-level reserved attributes and `team` and `pod_name` are pushed to root `.ddtags` as Datadog tags. What is the difference between a root attribute and a Datadog tag in terms of how you filter in Log Explorer?
3. Should you remove the tag from the `ddtags` array after extracting it, or keep it in both places? What are the trade-offs?

---

### S5 · High-Frequency Duplicate Events

**Processor type:** Reduce

**Where to look:** Filter `service:delivery-service`. The `delivery-service` polls for an available driver every 500ms. When no driver is found, it emits the same WARN event repeatedly for the same `order_id`. Find a group of events sharing the same `order_id` and `status:no_driver_available`. Notice that `poll_count` is `1` on every individual event.

**Questions:**

1. Without the Reduce processor, how many events does a single stuck order generate per minute? If there are 500 active deliveries at peak, what is the monthly ingest impact?
2. Which fields should you group by — `order_id` alone, or `[order_id, status]`? What happens if you group only by `order_id` and `status` changes mid-window?
3. After reducing, the merged event has `poll_count: 15`. The timestamp comes from the first event in the group. Does that matter for alerting on `no_driver_available` events?

---


## Validate

After deploying all processors, verify the results in Datadog Log Explorer.

| # | What to check | How to verify |
|---|---|---|
| S1 | PII removed | `service:payment-service` → no `card_number`, `card_cvv`; `service:user-service` → `ssn` is masked |
| S2 | Field names consistent | All 4 services show `user_id`, `http_status`, `duration_ms` |
| S3 | Nulls removed | `service:notification-service` → no `null`, `""`, `N/A` fields |
| S4 | ddtags extracted | `service:chaos-engineering` → `env` and `version` are root-level attributes; `team` and `pod_name` are tags; `env:prod` filter works |
| S5 | Duplicates collapsed | `service:delivery-service` → one event per `order_id` per window with `poll_count` set |

---

## Monitor the Pipeline

Go to https://app.datadoghq.com/observability-pipelines and select `pipeline-hero`.

- **Throughput** — events and bytes per second per component
- **Errors** — any processing failures
- **Component details** — click any processor node to see individual stats

**Bonus question:** Looking at the pipeline throughput, can you estimate roughly how many log events per second DeliverLah generates? If each event averages 1 KB, what is the monthly ingest volume?

---

## Clean up

```
cd ~/apac-onsite-event-2026/app
docker-compose -f dc-app-update.yaml down

cd ~/apac-onsite-event-2026/opw
docker-compose -f dc-opw.yaml down

docker network rm my-dd-network
```
