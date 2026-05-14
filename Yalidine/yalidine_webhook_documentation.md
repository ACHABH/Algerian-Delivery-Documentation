# Yalidine Webhook Documentation

> Converted from the uploaded Yalidine webhook text documentation.
>
> **Security note:** secret-looking example values were replaced with placeholders such as `YOUR_WEBHOOK_SECRET_KEY`. Keep webhook secrets only in your backend environment variables.

## Contents

- [Introduction](#introduction)
- [Why Use Webhooks](#why-use-webhooks)
- [Steps to Receive Webhooks](#steps-to-receive-webhooks)
- [Best Practices](#best-practices)
- [Events Format](#events-format)
- [Supported Event Types](#supported-event-types)
- [Payload Examples](#payload-examples)
- [Retry Policy](#retry-policy)
- [Create a Webhook](#create-a-webhook)
- [Validate a Webhook](#validate-a-webhook)
- [Edit a Webhook](#edit-a-webhook)
- [Delete a Webhook](#delete-a-webhook)
- [Secure Your Webhook](#secure-your-webhook)
- [Test Your Endpoint](#test-your-endpoint)
- [Enable or Disable a Webhook](#enable-or-disable-a-webhook)
- [View the Logs](#view-the-logs)
- [Backend Integration Notes](#backend-integration-notes)

---

## Introduction

Yalidine provides a webhook service to notify your application when events occur. These notifications are almost real time and provide an alternative to polling the REST API.

Webhooks allow your application to trigger event-based behavior as soon as an event occurs, such as:

- A parcel delivery status changes.
- A parcel is created.
- A parcel is edited.
- A parcel is deleted.
- A parcel payment status changes.

This can be used to build live dashboards, synchronization flows, customer notifications, gamification mechanisms, and other event-driven features.

Using webhooks helps prevent repetitive API polling and preserves API quotas. Instead of repeatedly querying the API, Yalidine sends an event notification when something happens in your account.

---

## Why Use Webhooks

Example:

You register for the `parcel_status_updated` event.

When a Yalidine delivery agent takes a parcel out for delivery, Yalidine sends a webhook notification to your app. The event tells your app that a specific parcel tracking number has received a new status, such as:

```text
Sorti en livraison
```

After your webhook endpoint receives the `parcel_status_updated` event, your app can run a specific action, for example sending a message to the parcel recipient telling them that the parcel will be delivered today.

---

## Steps to Receive Webhooks

To receive webhook event notifications in your app:

1. Create a webhook endpoint as an HTTPS URL on your server.
2. Make sure the endpoint returns an HTTP `200` response and a valid `crc_token` during validation.
3. Create a webhook in the Yalidine Webhooks Dashboard.
4. Develop and test your webhook endpoint using the webhook test page.
5. When the webhook is ready, change its status to `active`.
6. Your application will then receive event notifications as soon as they occur.

---

## Best Practices

### `crc_token` Validation

Your endpoint must always include the `crc_token` validation logic.

Yalidine validates webhook endpoints from time to time. If your endpoint does not return a valid `crc_token`, the webhook can be disabled.

See also: [Validate a Webhook](#validate-a-webhook).

### Event Delivery

Yalidine delivers event notifications reasonably fast.

Each webhook delivery contains one or many events of the same event type. Yalidine groups events of the same type and sends them in the same notification.

See also: [Events Format](#events-format).

### Event Fields

Expect new fields to appear in event structures without notice.

Your code should tolerate additional fields. Existing fields are preserved for backward compatibility as much as possible.

### Response Time

Your endpoint must return an HTTP `200` response in less than 10 seconds.

When receiving a payload, Yalidine recommends the following flow:

1. Do not run long, time-consuming scripts inside the webhook request.
2. Store the payload in a queue or database for background processing.
3. Return the HTTP response immediately.
4. Process the received events later with another worker/script.

If your endpoint is too slow or repeatedly fails, the webhook can be disabled.

See also: [Retry Policy](#retry-policy).

### Retry Logic

Be aware of the webhook retry logic. If your endpoint does not respond correctly, Yalidine retries delivery using the retry policy described later in this document.

### Disable Webhook Logic

Yalidine may notify you by email if an endpoint has not responded with HTTP `200` for multiple days in a row. The email also states when the endpoint will be automatically disabled.

### Duplicate Events

Your endpoint might receive the same event more than once.

Each event has a unique `event_id`. To prevent duplicate processing, store processed event IDs and skip already-processed events.

### Order of Events

Yalidine does not guarantee delivery of events in the order in which they were generated.

Each event includes an `occurred_at` timestamp. Use this timestamp to preserve or reconstruct event order when needed.

### Security

Every webhook delivery contains a signature header that helps verify that the request came from Yalidine.

The signature is generated using the payload and your webhook secret key.

See also: [Secure Your Webhook](#secure-your-webhook).

### Secret Key

The webhook delivery signature is generated using:

- The raw payload.
- Your webhook secret key.

You can find or regenerate your secret key in the Yalidine Webhooks Dashboard.

---

## Events Format

Webhook payloads are sent to your endpoint as JSON using an HTTP `POST` request.

Yalidine also sends a signature in the request header to help secure the webhook.

Every payload contains two top-level elements:

| Field | Type | Description |
|---|---:|---|
| `type` | string | The event type you subscribed to. Yalidine sends one event type per request. |
| `events` | array | An array of generated events. Each item is an individual event. |

Each item in `events` contains:

| Field | Type | Description |
|---|---:|---|
| `event_id` | string | Unique identifier of the event. Use it for deduplication. |
| `occurred_at` | string | Timestamp of when the event was generated. Use it for ordering. |
| `data` | object | The resource data related to the event. The structure depends on the event type. |

---

## Supported Event Types

You can subscribe to the following event types:

| Event Type | Meaning |
|---|---|
| `parcel_created` | A parcel was created. |
| `parcel_edited` | A parcel was edited. |
| `parcel_deleted` | A parcel was deleted. |
| `parcel_status_updated` | A parcel delivery status changed. |
| `parcel_payment_updated` | A parcel payment status changed. |

---

## Payload Examples

### `parcel_created`

```json
{
  "type": "parcel_created",
  "events": [
    {
      "event_id": "yEvb0sSBG4Tu9hOo8flp2na7tLx5Pzem",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "order_id": "myfirstorder",
        "tracking": "yal-111AAA",
        "label": "https://yalidine.app/app/bordereau.php?tracking=yal-111AAA&token=EXAMPLE_TOKEN",
        "import_id": 654
      }
    },
    {
      "event_id": "SZzY2HPAoXhxsKLuVOCD3Wc0BGr5yMdm",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "order_id": "mysecondorder",
        "tracking": "yal-222BBB",
        "label": "https://yalidine.app/app/bordereau.php?tracking=yal-222BBB&token=EXAMPLE_TOKEN",
        "import_id": 964
      }
    }
  ]
}
```

### `parcel_edited`

```json
{
  "type": "parcel_edited",
  "events": [
    {
      "event_id": "4Ucabt2yYPRZrKdqzegL3fJhBVGjSoED",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-111AAA",
        "label": "https://yalidine.app/app/bordereau.php?tracking=yal-111AAA&token=EXAMPLE_TOKEN"
      }
    },
    {
      "event_id": "0Nux2QrRXCfeMgcF9Kh8UIzSjmwAYnoW",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-222BBB",
        "label": "https://yalidine.app/app/bordereau.php?tracking=yal-222BBB&token=EXAMPLE_TOKEN"
      }
    }
  ]
}
```

### `parcel_deleted`

```json
{
  "type": "parcel_deleted",
  "events": [
    {
      "event_id": "3cFveXnpiI2RGsDtHwLMUmhYCBqz0Q4J",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-111AAA"
      }
    },
    {
      "event_id": "y7iroHGImbunRfvNjTSZL9t5dUAaBlcD",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-222BBB"
      }
    }
  ]
}
```

### `parcel_status_updated`

```json
{
  "type": "parcel_status_updated",
  "events": [
    {
      "event_id": "JWUf6IkGtQu8EAov0aOjebyilYN5cKpF",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-111AAA",
        "status": "Sorti en livraison",
        "reason": null
      }
    },
    {
      "event_id": "JUtoL5qRC7gWMPTj6fhiuyOwlGA3ZbQv",
      "occurred_at": "2022-04-28 12:01:26",
      "data": {
        "tracking": "yal-222BBB",
        "status": "Tentative échouée",
        "reason": "Client ne répond pas"
      }
    }
  ]
}
```

### `parcel_payment_updated`

```json
{
  "type": "parcel_payment_updated",
  "events": [
    {
      "event_id": "KCLxYpusZ3PADq5Jtw7FOTIQG0vVNlk9",
      "occurred_at": "2022-04-28 00:01:26",
      "data": {
        "tracking": "yal-111AAA",
        "status": "ready",
        "payment_id": null
      }
    },
    {
      "event_id": "dx3vGgzat5CcLYPws4HRKEh18XSTk7yA",
      "occurred_at": "2022-04-28 12:01:26",
      "data": {
        "tracking": "yal-222BBB",
        "status": "receivable",
        "payment_id": "pmt-123456"
      }
    }
  ]
}
```

---

## Retry Policy

When Yalidine sends a webhook delivery to your endpoint, your endpoint must respond in less than 10 seconds with HTTP `200`.

If it does not, Yalidine applies a retry policy.

The retry policy consists of 7 total attempts with increasing intervals. Yalidine sends an email after each failed retry. After the last attempt, the webhook is automatically disabled.

| Attempt | Proceed After | Example Time |
|---:|---|---|
| 1 | Immediately | `2026-04-28 20:06:50` |
| 2 | 5 minutes after the previous attempt | `2026-04-28 20:11:50` |
| 3 | 15 minutes after the previous attempt | `2026-04-28 20:26:50` |
| 4 | 1 hour after the previous attempt | `2026-04-28 21:26:50` |
| 5 | 3 hours after the previous attempt | `2026-04-29 00:26:50` |
| 6 | 12 hours after the previous attempt | `2026-04-29 12:26:50` |
| 7 | 1 day after the previous attempt | `2026-04-30 12:26:50` |

> The retry intervals are counted from the date of the previous attempt, not from the event's `occurred_at` date.
>
> The retry policy runs for about 40 hours after the first attempt.

---

## Create a Webhook

Before creating the webhook subscription, create the endpoint on your server first.

The endpoint must return a valid `crc_token` during validation.

To create a webhook:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the add button.
3. Fill the form with the required values:
   - **Webhook Name:** a name that helps you recognize the webhook later.
   - **URL of reception:** an HTTPS URL for your endpoint. This endpoint must return a valid `crc_token`.
   - **Email:** an email address to receive alert notifications if event delivery fails.
   - **Event types:** one or many subscribed event types:
     - `parcel_created`
     - `parcel_edited`
     - `parcel_deleted`
     - `parcel_status_updated`
     - `parcel_payment_updated`
4. Click **Proceed**.

When you click **Proceed**, Yalidine validates the endpoint URL to make sure you own it.

The webhook is created only if it passes validation.

When the webhook is created, it is automatically disabled. You can then:

- Test it.
- Enable it.

---

## Validate a Webhook

Yalidine uses a Challenge-Response Check (CRC) to verify that you own the webhook endpoint URL.

Validation happens when:

- A webhook is created.
- A webhook is edited.
- Yalidine periodically validates already-created webhooks.

Validation flow:

1. Yalidine sends a `GET` request to your endpoint URL with two query parameters:
   - `subscribe`
   - `crc_token`
2. Your endpoint must check whether the `subscribe` query parameter is set.
3. If it is set, your endpoint must echo the received `crc_token` value.
4. The endpoint is valid only if:
   - It returns HTTP `200` in less than 10 seconds.
   - It echoes the correct `crc_token`.

### Endpoint Validation Example

```php
<?php
// validation start.

// This code must always be present to prevent your webhook from being disabled.
if (isset($_GET["subscribe"], $_GET["crc_token"])) {
    echo $_GET["crc_token"];
    exit();
}
// validation end.

/* rest of the code
    ...
*/
```

> Important: the validation code must always remain present in the endpoint. Yalidine validates webhooks from time to time, and if validation fails, the webhook can be disabled.

---

## Edit a Webhook

To edit a webhook:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the edit button in the webhook card you want to edit.
3. Make the required changes.
4. Click **Proceed**.

When you edit a webhook, Yalidine validates the URL again.

These actions are added to the webhook log.

---

## Delete a Webhook

To delete a webhook:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the delete button in the webhook card you want to delete.
3. Confirm the deletion.

When a webhook is deleted, all its log data is also deleted.

---

## Secure Your Webhook

Securing webhooks is not mandatory, but it is strongly recommended.

To secure a webhook, verify that Yalidine sent the request.

Yalidine sends a signature in every webhook delivery using the following header:

```http
X_YALIDINE_SIGNATURE: <signature>
```

The signature is generated using `hash_hmac` with:

| Input | Value |
|---|---|
| Algorithm | `sha256` |
| Data | The raw payload |
| Key | Your webhook secret key |

Before handling the payload, your endpoint should:

1. Check if `HTTP_X_YALIDINE_SIGNATURE` is present in the request headers.
2. Compute a verification hash using:
   - `sha256`
   - the raw payload
   - your webhook secret key
3. Compare the received Yalidine signature with your computed signature.
4. If they match, process the payload.
5. If they do not match, reject the request.

You can regenerate your webhook secret key in the Webhooks Dashboard.

### Secure Webhook Example

```php
<?php
// validation start.
// This code must always be present to prevent your webhook from being disabled.
if (isset($_GET["subscribe"], $_GET["crc_token"])) {
    echo $_GET["crc_token"];
    exit();
}
// validation end.

// Signature verification.
// To be sure that the event came from Yalidine, verify the signature found in the header.
if (isset($_SERVER["HTTP_X_YALIDINE_SIGNATURE"])) {
    // The signature is set, so continue.

    $secret_key = "YOUR_WEBHOOK_SECRET_KEY"; // Store this in an environment variable.
    $yalidine_signature = $_SERVER["HTTP_X_YALIDINE_SIGNATURE"];
    $payload = file_get_contents("php://input");

    // Compute the payload and the secret key with hash_hmac().
    $computed_signature = hash_hmac("sha256", $payload, $secret_key);

    // Verify if the Yalidine signature matches the computed signature.
    if ($yalidine_signature === $computed_signature) {
        // Authorization successful. The payload was sent by Yalidine.
        // Recommended: save the payload in the database or a queue,
        // then return HTTP 200 immediately and process it later.
    } else {
        header("HTTP/1.1 400 Bad Request");
        exit("invalid signature");
    }
} else {
    header("HTTP/1.1 400 Bad Request");
    exit("invalid signature");
}
```

---

## Test Your Endpoint

To test your webhook endpoint:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the **make tests** button in the webhook card you want to test.
3. You will see all event types you subscribed to.
4. Click **Send a test**.
5. Your app will receive a webhook delivery containing test data.
6. The result of the test will be shown near the button you clicked.

You can use this method to develop or test webhook endpoints.

Test deliveries are not added to the webhook log.

---

## Enable or Disable a Webhook

To change the status of a webhook:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the change status button in the webhook card you want to edit.
3. Make the required status change.
4. Click **Yes, change the status**.

You will not receive events while the webhook is disabled.

> Important: when you disable a webhook, if some events are still present in the queue, the retry policy will still be applied. These events are deleted only after they consume all possible attempts. If you enable the webhook again before deletion, you may receive them normally.

---

## View the Logs

To view webhook logs:

1. Go to the Yalidine Webhooks Dashboard.
2. Click the log button in the webhook card you want to inspect.

Webhook logs are kept for 48 hours.

When a webhook is deleted, all its logs are deleted too.

---

## Backend Integration Notes

For a production backend, especially in a NestJS/e-commerce platform, a safe webhook flow should look like this:

1. Receive the webhook request on a backend-only HTTPS endpoint.
2. Handle `GET` validation requests by returning the `crc_token` immediately.
3. For `POST` webhook events:
   - Read the raw request body.
   - Verify `X_YALIDINE_SIGNATURE` using HMAC SHA-256.
   - Reject invalid signatures.
   - Store each `event_id` in a deduplication table or cache.
   - Enqueue valid events for background processing.
   - Return HTTP `200` quickly.
4. Process the event asynchronously with a worker.
5. Update local parcels/orders using `tracking`, `status`, `payment_id`, and `occurred_at`.
6. Never rely on webhook event order alone; use `occurred_at` and idempotent updates.

Recommended environment variable:

```env
YALIDINE_WEBHOOK_SECRET=replace_with_your_webhook_secret_key
```

Recommended deduplication fields:

| Field | Purpose |
|---|---|
| `event_id` | Prevent duplicate processing. |
| `type` | Identify event category. |
| `tracking` | Link event to the local parcel/order. |
| `occurred_at` | Preserve event timeline. |
| `processed_at` | Track processing status. |

