Create Three Billing Alarms

## Overview

This document describes how to enable billing access, create billing alarms, and create a simple cost budget in AWS for account cost monitoring.

## Cloud Service Provider

- Amazon Web Services (AWS)

## Prerequisites

- Root user has access to all billing settings by default.
- IAM users and roles can access billing information only after billing access is explicitly enabled.[2]

## Enable IAM Access to Billing

1. Sign in to the AWS Management Console as the root user.
2. In the upper-right corner, select the account name or account ID and choose “Account” or “My Account”.
3. Scroll to the billing access section (often labeled “IAM user and role access to billing information”).
4. Enable IAM user and role access to billing information, then save changes.

After this, IAM users and roles with the correct permissions can view and manage billing and budgets.

## Create a Cost Budget

1. Sign in with a user that has billing permissions.
2. In the console search bar, search for “Billing” and open the Billing dashboard.
3. In the left navigation, choose “Budgets”.
4. Select “Create a budget”.
5. Choose a budget type (for example, “Cost budget”), then choose either a template or a custom budget.
6. Set the period (for example, monthly), the budgeted amount, and the alert thresholds, then complete the wizard to create the budget.

## Objectives

You must complete the following:

- Create a billing alarm for \$10.

At a high level, each billing alarm will:

- Use the “Total Estimated Charge” metric in CloudWatch.
- Trigger when estimated charges exceed the configured USD threshold.
- Send a notification through Amazon SNS (for example, an email).

## Key Questions

### 1. How many billing alarms are free?

You can create up to 10 billing alarms for free as part of the AWS Free Tier.

### 2. How will you know when a billing alarm is triggered?

When a billing alarm enters the ALARM state, it sends a notification using Amazon Simple Notification Service (SNS) to the configured topic.
That SNS topic can deliver alerts to endpoints such as email addresses, SMS, or HTTP/S webhooks subscribed to the topic.

### 3. Difference: billing alarm vs. CloudWatch alarm

- Billing alarm:
  - Uses the “EstimatedCharges” billing metric for your AWS account.
  - Triggers when your estimated charges cross a defined cost threshold, helping you control and monitor spending.

- CloudWatch alarm:
  - Uses operational metrics such as CPU utilization, memory (via custom metrics), request counts, or custom application metrics.
  - Triggers actions like notifications, auto scaling, or other automated responses based on resource or application performance thresholds.

## Project Author

- Carl Padilla



![budget](budget.png)