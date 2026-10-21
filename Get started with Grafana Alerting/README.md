# Getting Started with Grafana Alerting: Complete Tutorial

A hands-on, step-by-step guide to configuring, evaluating, triggering, and resolving alerts in Grafana using a local Docker environment and webhook notifications.

---

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start: Running the Environment](#quick-start-running-the-environment)
- [Step 1: Set Up a Contact Point (Webhook)](#step-1-set-up-a-contact-point-webhook)
- [Step 2: Create an Alert Rule](#step-2-create-an-alert-rule)
- [Step 3: Trigger the Alert](#step-3-trigger-the-alert)
- [Step 4: Resolve the Alert](#step-4-resolve-the-alert)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

---

## Overview

In this tutorial, you will learn how to:
1. Spin up a localized Grafana observability stack with Docker Compose.
2. Configure a test **Contact Point** using an external webhook endpoint.
3. Define an **Alert Rule** and configure evaluation settings (`pending period`, `evaluation interval`, and `keep firing for`).
4. Observe real-time **Firing** and **Resolved** alert notification payloads.

---

## Prerequisites

Before starting, ensure you have the following installed on your machine:
- [Git](https://git-scm.com/)
- [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/)
- An open browser tab with access to [Webhook.site](https://webhook.site/) (no account required)

---

## Quick Start: Running the Environment

### 1. Clone the repository
```bash
git clone https://github.com/grafana/tutorial-environment.git
cd tutorial-environment
```

### 2. Start the stack
```bash
docker-compose up -d
```

> **Note:** Initial startup may take a few minutes while Docker images are downloaded. 
> 
> *Port Conflict Tip:* If you already have Grafana (`3000`), Prometheus (`9090`), or Loki running locally, stop those services first to prevent port allocation errors.

### 3. Access Grafana
Open your browser and navigate to:
```
http://localhost:3000
```
*(Default login credentials if prompted: username `admin` / password `admin`)*

---

## Step 1: Set Up a Contact Point (Webhook)

Grafana Alerting uses **Contact Points** to route notifications when an alert condition fires or resolves.

1. Open [Webhook.site](https://webhook.site/) in a separate browser tab.
2. Copy your unique target URL (e.g., `https://webhook.site/#!/your-uuid...`). Keep this tab open.
3. In Grafana, navigate to the left navigation bar, hover over **Alerting** (bell icon), and select **Contact points**.
4. Click **+ New contact point**.
5. Configure the contact point:
   - **Name:** `Webhook`
   - **Integration:** `Webhook`
   - **URL:** Paste the unique URL copied from Webhook.site.
6. Click **Test**, then select **Send test notification**.
7. Return to your Webhook.site tab to verify that a `POST` request with the test payload has arrived.
8. Back in Grafana, click **Save contact point**.

---

## Step 2: Create an Alert Rule

### Understanding Evaluation Settings
- **Evaluation Interval:** How frequently the metric expression is calculated (e.g., `10s`, `1m`).
- **Pending Period (`for`):** The duration an alert condition must continuously evaluate to true before switching from `Pending` to `Firing`. Setting this to `0s` fires the alert immediately.
- **Keep Firing For:** How long the alert stays in `Firing` state after the condition ceases to be met. Helps eliminate "alert flapping" caused by noisy data.

### Setting Up the Rule
1. In Grafana, navigate to **Alerting** > **Alert rules** > **+ New alert rule**.
2. **Rule name & Folder:**
   - Enter a name (e.g., `Tutorial Test Alert`).
   - Create or choose a folder (e.g., `metric-alerts`).
3. **Query and Condition:**
   - Configure a condition designed to evaluate to true (for testing purposes, set an expression threshold lower than your test metric).
4. **Evaluation Group:**
   - **Group name:** `1m-evaluation`
   - **Evaluation interval:** `1m`
   - **Pending period:** `0s`
   - **Keep firing for:** `0s`
5. **Configure Notifications:**
   - Under **Contact point**, select `Webhook`.
6. Click **Save rule and exit** in the top right corner.

---

## Step 3: Trigger the Alert

Because the alert was configured to trigger immediately:
1. Wait for the evaluation interval (approximately 1 minute).
2. Check your **Webhook.site** dashboard.
3. Inspect the newly received `POST` request payload. It will include:
   - **Status:** `firing`
   - Metric values exceeding the threshold.
   - Deep links to silence or view the rule in the Grafana UI.

---

## Step 4: Resolve the Alert

To observe the alert transitioning to `Resolved`:

1. In Grafana, navigate to **Alerting** > **Alert rules**.
2. Expand the `metric-alerts` folder and locate your alert.
3. Click the **Edit** (pencil) icon.
4. Increment the threshold condition so that the current metric value falls below it (no longer triggers).
5. Click **Save rule and exit**.
6. Wait for the next evaluation cycle (~1 minute).
7. Return to **Webhook.site**. A new `POST` payload will appear with the status updated to:
   ```json
   "status": "resolved"
   ```

---

## Troubleshooting

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `Bind for 0.0.0.0:3000 failed: port is already allocated` | Another Grafana instance or local process is bound to port 3000. | Stop the local service or change the host port binding in `docker-compose.yml`. |
| Webhook endpoint receives no test notification | Invalid URL or firewall block. | Ensure the copied URL is the full Webhook.site target link and that your machine has internet access. |
| Alert takes too long to fire | Non-zero `Pending period` configured. | Set the `Pending period` to `0s` for immediate evaluation during testing. |

---

## Next Steps

Now that you have configured basic alert routing:
- **Multi-dimensional Alerts:** Learn how alerting labels allow a single rule to track multiple series independently.
- **Notification Policies:** Set up label matchers and routing trees to send critical alerts to PagerDuty or Slack while directing informational alerts to email or webhooks.
- **Silences:** Learn how to temporarily suppress notifications during planned maintenance windows.
