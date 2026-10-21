Getting Started with Grafana Alerting: Complete Tutorial
This tutorial walks you through setting up your first alert in Grafana, watching it fire with real-time data, and receiving alert notifications via a webhook.

Tutorial Overview
In this tutorial you will:

Create a contact point

Set up an alert rule

Receive firing and resolved alert notifications in a public webhook

After completing Part 1, explore the advanced (but essential) alerting topics in Multi-dimensional alerts and how to route them.

Prerequisites: Download and Run the Tutorial Environment
To demonstrate observing data using the Grafana stack, download and run the following files.

Step 1: Clone the tutorial environment repository
bash
git clone https://github.com/grafana/tutorial-environment.git
Step 2: Change to the directory where you cloned the repository
bash
cd tutorial-environment
Step 3: Run the Grafana stack
bash
docker-compose up -d
Note: The first time you run docker-compose up -d, Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

NOTE: If you already have Grafana, Loki, or Prometheus running on your system, you might see errors, because the Docker image is trying to use ports that your local installations are already using. If this is the case, stop the services, then run the command again.

Step 1: Create a Contact Point
Besides being an open-source observability tool, Grafana has its own built-in alerting service. This means you can receive notifications whenever there is an event of interest in your data, and even see these events graphed in your visualizations.

In this step, we set up a new contact point. This contact point uses the webhook integration. In order to make this work, we also need an endpoint for our webhook integration to receive the alert. We can use Webhook.site to quickly set up that test endpoint. This way we can make sure that our alert is actually sending a notification somewhere.

Configure the webhook endpoint
In your browser, sign in to your Grafana Cloud account.

OSS users: To log in, navigate to http://localhost:3000, where Grafana is running.

In another tab, go to Webhook.site.

Copy Your unique URL.

Your webhook endpoint is now waiting for the first request.

Configure the contact point in Grafana
Next, let's configure a contact point in Grafana's Alerting UI to send notifications to our webhook endpoint.

Return to Grafana. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Contact points.

Click + New contact point.

In Name, write Webhook.

In Integration, choose Webhook.

In URL, paste the endpoint to your webhook endpoint.

Click Test, and then click Send test notification to send a test alert to your webhook endpoint.

Navigate back to Webhook.site. On the left side, there's now a POST / entry. Click it to see what information Grafana sent.

(See alerting-webhook-detail.png in the tutorial)

Return to Grafana and click Save contact point.

We have created a dummy Webhook endpoint and created a new Alerting contact point in Grafana. Now, we can create an alert rule and link it to this new integration.

Step 2: Set Up an Alert Rule — Evaluation Settings
Before configuring notifications, understand the evaluation settings:

Evaluation interval: Determines how frequently the alert rule is checked. For instance, the evaluation may occur every 10s, 30s, 1m, 10m, etc.

Pending period: How long the condition must be met to trigger the alert rule.

Keep firing for: Defines how long an alert should remain in the Firing state after the alert condition stops being true. During this time, the alert enters a Recovering state, suppressing additional notifications but keeping the alert active. It helps prevent alert flapping, where alerts rapidly switch between firing and resolved due to noisy or unstable metrics.

To set up the evaluation:
In the Evaluation group and interval, enter a name. For example: 1m-evaluation.

Choose an Evaluation interval (how often the alerts are evaluated). For example, every 1m (1 minute).

Set the pending period to 0s (zero seconds), so the alert rule fires the moment the condition is met.

Set Keep firing for to 0s, so the alert stops firing immediately after the condition is no longer true. Use this when you want alerts to be resolved as soon as the system is healthy again.

Step 3: Configure Notifications
Choose the contact point where you want to receive your alert notifications.

Under Contact point, select Webhook from the drop-down menu.

Click Save rule and exit at the top right corner.

Step 4: Trigger and Resolve an Alert
Now that the alert rule has been configured, you should receive alert notifications in the contact point whenever alerts trigger and get resolved.

Trigger an alert
Since the alert rule that you have created has been configured to always fire, once the evaluation interval has concluded, you should receive an alert notification in the Webhook endpoint.

(See alerting-webhook-firing-alert.png in the tutorial)

The alert notification details show that the alert rule state is Firing, and it includes:

The value that made the rule trigger by exceeding the threshold of the alert rule condition.

Links to see the alert rule details.

Another link to add a Silence to it.

Resolve an alert
To see how a resolved alert notification looks, you can modify the current alert rule threshold.

To edit the Alert rule:

Navigate to Alerting > Alert rules.

Click on the metric-alerts folder to display the alert that you created earlier.

Click the edit button on the right-hand side of the screen.

Increment the Threshold expression to 1.

Click Save rule and exit.

By incrementing the threshold, the condition is no longer met, and after the evaluation interval has concluded (1 minute approx.), you should receive an alert notification with status "Resolved".
