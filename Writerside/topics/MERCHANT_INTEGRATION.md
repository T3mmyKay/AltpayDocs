# AltPay Merchant Integration Guide

## Table of Contents

- [Introduction](#introduction)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)
    - [Authentication](#authentication)
    - [Initiating Payments](#initiating-payments)
    - [Verifying Payments](#verifying-payments)
- [Webhooks](#webhooks)
    - [Setting Up Webhooks](#setting-up-webhooks)
    - [Webhook Security](#webhook-security)
    - [Webhook Events](#webhook-events)
    - [Best Practices](#best-practices)
    - [Webhook Retries](#webhook-retries)
- [Testing](#testing)
- [Sample Integration](#sample-integration)

## Introduction

AltPay provides a secure and reliable payment processing solution for businesses in Nigeria. This guide will help you integrate AltPay into your application to start accepting bank transfer payments with virtual accounts.

## Getting Started

1. Sign up for an AltPay merchant account
2. Get your API keys from the dashboard:
    - Test Secret Key: For development and testing
    - Live Secret Key: For production use
3. Set up your webhook URL
4. Start accepting payments

## API Reference

### Base URL

All API requests should be made to:

```
https://api.altpayng.com/
```

For test environment, use the same base URL with your test API keys.

### Authentication

All API requests must include your secret key in the `X-ALTPAY-KEY` header:

```http
X-ALTPAY-KEY: your_secret_key_here
```

### Initiating Payments

To initiate a payment and generate a virtual account:

```http
POST /api/v1/payments/initiate
```

Request body:

```json
{
  "amount": 500000,
  "description": "Payment for Order #123",
  "merchantReference": "ORDER_123",
  "customerEmail": "customer@example.com",
  "customerName": "John Doe",
  "customerPhone": "+2348012345678",
  "metadata": "{ \"orderId\": \"123\" }",
  "expiresInMinutes": 30
}
```

> **Amount Validation Rules**:
>
> - Amount must be provided in kobo (₦1 = 100 kobo)
> - Must be a whole number
> - Minimum: 100 kobo (₦1)
> - Maximum: 100,000,000,000 kobo (₦1,000,000)
> - Invalid amounts will return a 400 Bad Request

> **Metadata Limits**:
>
> - Maximum size: 4KB
> - Must be valid JSON
> - Sensitive data should not be included
> - Common uses: order IDs, customer references, etc.

Response:

```json
{
  "accountNumber": "1234567890",
  "accountName": "JOHN DOE/ALTPAY",
  "bankName": "Globus Bank",
  "amountInKobo": 500000,
  "amount": 5000.00,
  "expiresAt": "2024-01-04T12:30:00Z"
}
```

### Verifying Payments

To check the status of a payment:

```http
GET /api/v1/payments/status/{reference}
```

> **Note**: You can use either your secret key or public key for payment verification.

The reference can be either:

- Your merchant reference (e.g., your order ID)
- The AltPay payment reference

Response:

```json
{
  "reference": "PAY_20240104_abc123",
  "merchantReference": "ORDER_123",
  "amount": 5000.00,
  "currency": "NGN",
  "status": "completed",
  "message": "Payment completed successfully",
  "initiatedAt": "2024-01-04T12:00:00Z",
  "completedAt": "2024-01-04T12:15:00Z",
  "paymentChannel": "bank_transfer",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "phone": "+2348012345678"
  },
  "metadata": {
    "orderId": "123"
  }
}
```

**Error Responses:**

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Not Found",
  "status": 404,
  "detail": "Payment with reference ORDER_123 not found",
  "traceId": "00-123abc..."
}
```

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.3",
  "title": "Forbidden",
  "status": 403,
  "detail": "Business ID not found in API key claims",
  "traceId": "00-456def..."
}
```

**Example Responses for Different Statuses:**

1. Pending Payment:

```json
{
  "reference": "PAY_20240104_abc123",
  "merchantReference": "ORDER_123",
  "amount": 5000.00,
  "currency": "NGN",
  "status": "pending",
  "message": "Payment initiated",
  "initiatedAt": "2024-01-04T12:00:00Z",
  "completedAt": null,
  "paymentChannel": "bank_transfer",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "phone": "+2348012345678"
  },
  "metadata": {
    "orderId": "123"
  }
}
```

2. Awaiting Payment:

```json
{
  "reference": "PAY_20240104_abc123",
  "merchantReference": "ORDER_123",
  "amount": 5000.00,
  "currency": "NGN",
  "status": "awaiting_payment",
  "message": "Virtual account assigned, awaiting transfer",
  "initiatedAt": "2024-01-04T12:00:00Z",
  "completedAt": null,
  "paymentChannel": "bank_transfer",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "phone": "+2348012345678"
  },
  "metadata": {
    "orderId": "123"
  }
}
```

3. Failed Payment:

```json
{
  "reference": "PAY_20240104_abc123",
  "merchantReference": "ORDER_123",
  "amount": 5000.00,
  "currency": "NGN",
  "status": "failed",
  "message": "Payment failed: Amount mismatch",
  "initiatedAt": "2024-01-04T12:00:00Z",
  "completedAt": null,
  "paymentChannel": "bank_transfer",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "phone": "+2348012345678"
  },
  "metadata": {
    "orderId": "123"
  }
}
```

4. Expired Payment:

```json
{
  "reference": "PAY_20240104_abc123",
  "merchantReference": "ORDER_123",
  "amount": 5000.00,
  "currency": "NGN",
  "status": "expired",
  "message": "Payment window expired",
  "initiatedAt": "2024-01-04T12:00:00Z",
  "completedAt": null,
  "paymentChannel": "bank_transfer",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "phone": "+2348012345678"
  },
  "metadata": {
    "orderId": "123"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `reference` | string | The payment reference number |
| `merchantReference` | string? | The merchant's reference number if provided |
| `amount` | decimal | The amount paid in the smallest currency unit |
| `currency` | string | The currency of the payment (e.g., NGN) |
| `status` | string | The current status of the payment |
| `message` | string | A user-friendly message explaining the current payment status |
| `initiatedAt` | datetime | When the payment was initiated |
| `completedAt` | datetime? | When the payment was completed (if successful) |
| `paymentChannel` | string | The payment channel used (e.g., bank_transfer) |
| `customer` | object | Customer information if provided |
| `metadata` | object? | Any additional metadata stored with the payment |

### Payment Statuses

Each payment can be in one of the following states:

| Status | Description |
|--------|-------------|
| `pending` | Payment has been initiated |
| `awaiting_payment` | Virtual account assigned, waiting for transfer |
| `processing` | Transfer received and being processed |
| `completed` | Payment successfully processed |
| `failed` | Payment failed (e.g., wrong amount) |
| `expired` | Payment window expired |
| `unknown` | Payment status cannot be determined |

**Status Flow:**

1. `pending` → `awaiting_payment`: When virtual account is assigned
2. `awaiting_payment` → `processing`: When transfer is detected
3. `processing` → `completed`: When payment is confirmed
4. `processing` → `failed`: If any issues during processing
5. `awaiting_payment` → `expired`: When payment window expires

**Status Webhook Events:**

- `payment.awaiting`: When status changes to `awaiting_payment`
- `payment.processing`: When status changes to `processing`
- `payment.successful`: When status changes to `completed`
- `payment.failed`: When status changes to `failed`
- `payment.expired`: When status changes to `expired`

## Webhooks

### Setting Up Webhooks

1. Log in to your AltPay dashboard
2. Navigate to Settings > Webhooks
3. Add your webhook URL
4. Save your webhook secret key securely

### Webhook Security

All webhook requests from AltPay include a signature header that you should validate to ensure the request is genuine. When you set up your webhook URL in the AltPay dashboard, you'll be automatically provided with a webhook secret key. This key is different from your API keys and should be used exclusively for webhook signature verification.

You'll receive different webhook secrets for test mode and live mode. Make sure to use the correct secret based on the environment.

#### Headers

| Header | Description |
|--------|-------------|
| `X-AltPay-Signature` | HMAC-SHA512 signature of the event payload |
| `X-AltPay-Event` | The type of event (e.g., "payment.successful") |
| `User-Agent` | "AltPay-Webhook/1.0" |

#### Verifying Webhook Signatures

When you receive a webhook, you should:

1. Get the webhook secret key from your dashboard (not your API key)
2. Get the signature from the `X-AltPay-Signature` header
3. Verify the signature using your webhook secret key

Here's how to verify webhook signatures in C#:

```C#
public bool IsValidWebhook(
    string payload,
    string signatureHeader,
    string webhookSecretKey) // Get this from your AltPay dashboard
{
    try
    {
        // Generate signature using webhook secret key
        using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(webhookSecretKey));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
        var computedSignature = BitConverter.ToString(hash).Replace("-", string.Empty).ToLower();

        // Compare signatures
        return signatureHeader == computedSignature;
    }
    catch
    {
        return false;
    }
}
```

Example usage in an ASP.NET Core webhook endpoint:

```C#
[HttpPost("webhook")]
public async Task<IActionResult> HandleWebhook(
    [FromServices] IConfiguration config)
{
    // Read the raw request body
    using var reader = new StreamReader(Request.Body);
    var payload = await reader.ReadToEndAsync();

    // Get signature header
    var signatureHeader = Request.Headers["X-AltPay-Signature"].ToString();
    
    // Get your webhook secret key from configuration
    var webhookSecretKey = config["AltPay:WebhookSecretKey"];
    
    // Verify signature
    if (!IsValidWebhook(payload, signatureHeader, webhookSecretKey))
    {
        return Unauthorized("Invalid webhook signature");
    }

    // Process the webhook...
    var webhookEvent = JsonSerializer.Deserialize<WebhookEvent>(payload);
    await ProcessWebhookAsync(webhookEvent);

    return Ok();
}
```

> **Security Note**: Your webhook secret key is different from your API keys and is automatically generated when you first set up your webhook URL. Never use your API keys for webhook verification. The webhook secret key should be stored securely and used only for verifying webhook signatures.

> **Environment Note**: You'll receive different webhook secrets for test mode and live mode. Make sure to use the correct secret based on the `environment` field in the webhook payload.

#### Webhook Response Tracking

AltPay tracks all webhook delivery attempts and responses. For each attempt, we store:

- HTTP Status Code
- Response body (truncated to 4KB)
- Response time
- Headers
- Timestamp

You can view webhook delivery history in your dashboard under Settings > Webhooks > Delivery History.

### Webhook Events

#### payment.successful

Sent when a payment is successfully completed.

```json
{
  "id": "evt_123",
  "type": "payment.successful",
  "created": "2024-01-04T12:15:00Z",
  "data": {
    "reference": "PAY_20240104_abc123",
    "amount": 5000.00,
    "currency": "NGN",
    "channel": "bank_transfer",
    "status": "successful",
    "metadata": {
      "sender_name": "John Doe",
      "sender_bank": "Globus Bank",
      "narration": "Payment for Order #123"
    },
    "payment_initiation": {
      "reference": "PAY_20240104_abc123",
      "merchant_reference": "ORDER_123",
      "customer": {
        "email": "john@example.com",
        "name": "John Doe",
        "phone": "+2348012345678"
      }
    },
    "business": {
      "id": "bus_123",
      "name": "My Store"
    },
    "environment": "live"
  }
}
```

#### payment.failed

Sent when a payment fails (e.g., wrong amount).

```json
{
  "id": "evt_124",
  "type": "payment.failed",
  "created": "2024-01-04T12:15:00Z",
  "data": {
    "reference": "PAY_20240104_abc123",
    "amount": 4500.00,
    "currency": "NGN",
    "channel": "bank_transfer",
    "status": "failed",
    "failure_reason": "Payment underpaid: Expected 5000.00, received 4500.00",
    "metadata": {
      "sender_name": "John Doe",
      "sender_bank": "Globus Bank",
      "narration": "Payment for Order #123"
    },
    "payment_initiation": {
      "reference": "PAY_20240104_abc123",
      "merchant_reference": "ORDER_123",
      "customer": {
        "email": "john@example.com",
        "name": "John Doe",
        "phone": "+2348012345678"
      }
    },
    "business": {
      "id": "bus_123",
      "name": "My Store"
    },
    "environment": "live"
  }
}
```

### Best Practices

1. **Verify Signatures**: Always verify webhook signatures to ensure requests are from AltPay.
2. **Return 2xx Quickly**: Respond to webhook requests quickly with a 2xx status code.
3. **Process Asynchronously**: Handle webhook processing in the background.
4. **Handle Duplicates**: Store processed webhook IDs to prevent duplicate processing.
5. **Implement Retries**: Be prepared to handle webhook retries from AltPay.

### Webhook Retries

If your endpoint fails to respond with a 2xx status code, AltPay implements an exponential backoff retry mechanism:

| Attempt | Delay | Time from first attempt |
|---------|-------|------------------------|
| 1st retry | 5 seconds | 5 seconds |
| 2nd retry | 25 seconds | 30 seconds |
| 3rd retry | 125 seconds | ~2.5 minutes |
| 4th retry | 625 seconds | ~13 minutes |
| 5th retry | 3125 seconds | ~65 minutes |

**Important Notes about Retries:**

- Maximum 5 retry attempts
- Total retry period: ~80 minutes
- Retries stop on successful delivery (2xx response)
- Each retry includes original headers and signature
- Failed webhooks after all retries require manual investigation
- View retry status in dashboard under Webhook Logs

### API Key Types and Usage

AltPay provides two types of API keys:

1. **Secret Key**
    - Used for all server-side API calls
    - Required for payment initiation
    - Required for payment verification
    - Never expose in client-side code

2. **Public Key**
    - Used for client-side operations
    - Safe to expose in frontend code
    - Limited to non-sensitive operations

**Key Usage Matrix:**

| Operation | Secret Key | Public Key |
|-----------|------------|------------|
| Initiate Payment | ✅ | ❌ |
| Verify Payment | ✅ | ✅ |
| Fetch Payment Status | ✅ | ✅ |
| Webhook Verification | ❌ | ❌ |

> **Security Note**: Webhook signatures use a separate secret key specific to webhooks. Never use your API keys for webhook verification.

## Testing

1. Use test API keys for development
2. Test webhooks using the dashboard's webhook tester
3. Verify signature validation with test payloads

## Sample Integration

Here's a complete example of integrating AltPay in a .NET application:

```C#
public class AltPayService
{
    private readonly HttpClient _httpClient;
    private readonly string _secretKey;
    private readonly string _webhookSecret;

    public AltPayService(HttpClient httpClient, IConfiguration config)
    {
        _httpClient = httpClient;
        _secretKey = config["AltPay:SecretKey"];
        _webhookSecret = config["AltPay:WebhookSecret"];
        
        _httpClient.BaseAddress = new Uri(config["AltPay:BaseUrl"]);
        _httpClient.DefaultRequestHeaders.Add("X-ALTPAY-KEY", _secretKey);
    }

    public async Task<PaymentResponse> InitiatePaymentAsync(PaymentRequest request)
    {
        // Convert amount to kobo if needed
        var amountInKobo = (long)Math.Round(request.Amount * 100);

        var response = await _httpClient.PostAsJsonAsync("payments/initiate", new
        {
            amount = amountInKobo, // Amount in kobo
            description = request.Description,
            merchantReference = request.OrderId,
            customerEmail = request.CustomerEmail,
            customerName = request.CustomerName,
            customerPhone = request.CustomerPhone,
            metadata = JsonSerializer.Serialize(new { orderId = request.OrderId }),
            expiresInMinutes = 30
        });

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResponse>();
    }

    public async Task<PaymentStatusResponse> VerifyPaymentAsync(string reference)
    {
        var response = await _httpClient.GetAsync($"payments/status/{reference}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentStatusResponse>();
    }

    [HttpPost("webhook")]
    public async Task<IActionResult> HandleWebhook()
    {
        using var reader = new StreamReader(Request.Body);
        var payload = await reader.ReadToEndAsync();
        
        var signatureHeader = Request.Headers["X-AltPay-Signature"].ToString();
        if (!IsValidWebhook(payload, signatureHeader, _webhookSecret))
        {
            return Unauthorized();
        }

        var webhookEvent = JsonSerializer.Deserialize<WebhookEvent>(payload);
        
        // Process webhook asynchronously
        _ = Task.Run(async () =>
        {
            try
            {
                await ProcessWebhookAsync(webhookEvent);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing webhook {EventId}", webhookEvent.Id);
            }
        });

        return Ok();
    }

    private async Task ProcessWebhookAsync(WebhookEvent webhookEvent)
    {
        switch (webhookEvent.Type)
        {
            case "payment.successful":
                await HandleSuccessfulPayment(webhookEvent.Data);
                break;
            case "payment.failed":
                await HandleFailedPayment(webhookEvent.Data);
                break;
        }
    }
}
```