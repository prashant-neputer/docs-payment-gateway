# Stripe-First Payment Gateway Implementation Guide

## Starting with Stripe: The Foundation Approach

This guide demonstrates how to implement a scalable payment gateway system starting with **Stripe** as your foundation gateway, then easily adding others (PayPal, Square, etc.) later. Stripe is chosen as the primary example because it's the most widely used and well-documented payment gateway.

## Why Start with Stripe?

1. **Industry Standard**: Most widely adopted payment gateway globally
2. **Excellent Documentation**: Comprehensive API docs with clear examples
3. **Developer-Friendly**: Great SDKs, testing tools, and sandbox environment
4. **Feature-Rich**: Supports subscriptions, marketplaces, and complex payment flows
5. **Reliable**: 99.99% uptime and excellent customer support

## 1. Foundation Architecture

### Payment Gateway Interface

```php
// app/Contracts/PaymentGatewayInterface.php
<?php

namespace App\Contracts;

use Illuminate\Http\Request;

interface PaymentGatewayInterface
{
    public function createPayment(array $paymentData): array;
    public function verifyPayment(string $transactionId): array;
    public function handleCallback(Request $request): array;
    public function refundPayment(string $transactionId, float $amount): array;
    public function getPaymentStatus(string $transactionId): string;
}
```

### Payment Manager

```php
// app/Services/PaymentManager.php
<?php

namespace App\Services;

use App\Contracts\PaymentGatewayInterface;
use App\Services\Gateways\StripePaymentService;
use InvalidArgumentException;

class PaymentManager
{
    private array $gateways = [];

    public function __construct()
    {
        $this->registerGateways();
    }

    public function gateway(string $name): PaymentGatewayInterface
    {
        if (!isset($this->gateways[$name])) {
            throw new InvalidArgumentException("Payment gateway [{$name}] not found.");
        }

        return $this->gateways[$name];
    }

    public function getAvailableGateways(): array
    {
        $available = [];
        foreach ($this->gateways as $name => $gateway) {
            if (config("payments.gateways.{$name}.enabled", false)) {
                $available[] = $name;
            }
        }
        return $available;
    }

    private function registerGateways(): void
    {
        // Start with Stripe as primary gateway
        $this->gateways['stripe'] = new StripePaymentService();
        
        // Future gateways will be added here:
        // $this->gateways['paypal'] = new PaypalPaymentService();
        // $this->gateways['square'] = new SquarePaymentService();
    }
}
```

## 2. Stripe Implementation (Your Foundation Gateway)

### Stripe Payment Service

```php
// app/Services/Gateways/StripePaymentService.php
<?php

namespace App\Services\Gateways;

use App\Contracts\PaymentGatewayInterface;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class StripePaymentService implements PaymentGatewayInterface
{
    private string $secretKey;
    private string $publishableKey;
    private string $webhookSecret;

    public function __construct()
    {
        $mode = config('payments.gateways.stripe.mode');
        $this->secretKey = config("payments.gateways.stripe.{$mode}.secret_key");
        $this->publishableKey = config("payments.gateways.stripe.{$mode}.publishable_key");
        $this->webhookSecret = config('payments.gateways.stripe.webhook_secret');
    }

    public function createPayment(array $paymentData): array
    {
        try {
            $checkoutData = [
                'payment_method_types' => ['card'],
                'line_items' => [[
                    'price_data' => [
                        'currency' => strtolower($paymentData['currency']),
                        'product_data' => [
                            'name' => $paymentData['order']['items'][0]['title'],
                            'description' => $paymentData['description'],
                        ],
                        'unit_amount' => $paymentData['amount'] * 100, // Convert to cents
                    ],
                    'quantity' => 1,
                ]],
                'mode' => 'payment',
                'success_url' => $paymentData['urls']['success'] . '?session_id={CHECKOUT_SESSION_ID}',
                'cancel_url' => $paymentData['urls']['cancel'],
                'customer_email' => $paymentData['buyer']['email'],
                'metadata' => [
                    'order_reference' => $paymentData['order']['reference_id'],
                    'customer_name' => $paymentData['buyer']['name'],
                    'customer_phone' => $paymentData['buyer']['phone'],
                ]
            ];

            $response = Http::withHeaders([
                'Authorization' => 'Bearer ' . $this->secretKey,
                'Content-Type' => 'application/x-www-form-urlencoded',
            ])->asForm()->post('https://api.stripe.com/v1/checkout/sessions', $checkoutData);

            if ($response->successful()) {
                $data = $response->json();
                return [
                    'success' => true,
                    'payment_url' => $data['url'],
                    'session_id' => $data['id'],
                    'payment_id' => $data['payment_intent'] ?? $data['id']
                ];
            }

            Log::error('Stripe payment creation failed', [
                'response' => $response->json(),
                'status' => $response->status()
            ]);

            return [
                'success' => false,
                'error' => 'Payment initialization failed'
            ];

        } catch (\Exception $e) {
            Log::error('Stripe payment exception', [
                'message' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);

            return [
                'success' => false,
                'error' => 'Payment service unavailable'
            ];
        }
    }

    public function verifyPayment(string $transactionId): array
    {
        try {
            // If it's a session ID, retrieve the session first
            if (str_starts_with($transactionId, 'cs_')) {
                $sessionResponse = Http::withHeaders([
                    'Authorization' => 'Bearer ' . $this->secretKey,
                ])->get("https://api.stripe.com/v1/checkout/sessions/{$transactionId}");

                if (!$sessionResponse->successful()) {
                    return ['success' => false, 'error' => 'Session not found'];
                }

                $session = $sessionResponse->json();
                $paymentIntentId = $session['payment_intent'];
            } else {
                $paymentIntentId = $transactionId;
            }

            // Retrieve payment intent
            $response = Http::withHeaders([
                'Authorization' => 'Bearer ' . $this->secretKey,
            ])->get("https://api.stripe.com/v1/payment_intents/{$paymentIntentId}");

            if ($response->successful()) {
                $data = $response->json();
                return [
                    'success' => true,
                    'status' => $this->normalizeStatus($data['status']),
                    'amount' => $data['amount_received'] / 100, // Convert from cents
                    'currency' => strtoupper($data['currency']),
                    'raw_data' => $data
                ];
            }

            return ['success' => false, 'error' => 'Payment verification failed'];

        } catch (\Exception $e) {
            Log::error('Stripe verification exception', [
                'transaction_id' => $transactionId,
                'message' => $e->getMessage()
            ]);

            return ['success' => false, 'error' => 'Verification service unavailable'];
        }
    }

    public function handleCallback(Request $request): array
    {
        try {
            $payload = $request->getContent();
            $signature = $request->header('Stripe-Signature');

            // Verify webhook signature
            if (!$this->verifyWebhookSignature($payload, $signature)) {
                return ['success' => false, 'error' => 'Invalid signature'];
            }

            $event = json_decode($payload, true);

            switch ($event['type']) {
                case 'checkout.session.completed':
                    return $this->handleSuccessfulPayment($event['data']['object']);
                case 'payment_intent.payment_failed':
                    return $this->handleFailedPayment($event['data']['object']);
                default:
                    Log::info('Unhandled Stripe webhook event', ['type' => $event['type']]);
                    return ['success' => true, 'message' => 'Event acknowledged'];
            }

        } catch (\Exception $e) {
            Log::error('Stripe webhook exception', [
                'message' => $e->getMessage(),
                'payload' => $request->getContent()
            ]);

            return ['success' => false, 'error' => 'Webhook processing failed'];
        }
    }

    public function refundPayment(string $transactionId, float $amount): array
    {
        try {
            $refundData = [
                'payment_intent' => $transactionId,
                'amount' => $amount * 100, // Convert to cents
            ];

            $response = Http::withHeaders([
                'Authorization' => 'Bearer ' . $this->secretKey,
            ])->asForm()->post('https://api.stripe.com/v1/refunds', $refundData);

            if ($response->successful()) {
                $data = $response->json();
                return [
                    'success' => true,
                    'refund_id' => $data['id'],
                    'status' => $data['status'],
                    'amount' => $data['amount'] / 100
                ];
            }

            return ['success' => false, 'error' => 'Refund failed'];

        } catch (\Exception $e) {
            Log::error('Stripe refund exception', [
                'transaction_id' => $transactionId,
                'amount' => $amount,
                'message' => $e->getMessage()
            ]);

            return ['success' => false, 'error' => 'Refund service unavailable'];
        }
    }

    public function getPaymentStatus(string $transactionId): string
    {
        $result = $this->verifyPayment($transactionId);
        return $result['status'] ?? 'unknown';
    }

    private function normalizeStatus(string $stripeStatus): string
    {
        $statusMap = [
            'succeeded' => 'completed',
            'processing' => 'pending',
            'requires_payment_method' => 'failed',
            'requires_confirmation' => 'pending',
            'requires_action' => 'pending',
            'canceled' => 'cancelled',
            'requires_capture' => 'pending'
        ];

        return $statusMap[$stripeStatus] ?? 'unknown';
    }

    private function verifyWebhookSignature(string $payload, string $signature): bool
    {
        if (empty($this->webhookSecret)) {
            return true; // Skip verification if no secret configured
        }

        $elements = explode(',', $signature);
        $signatureHash = '';

        foreach ($elements as $element) {
            if (strpos($element, 'v1=') === 0) {
                $signatureHash = substr($element, 3);
                break;
            }
        }

        $expectedSignature = hash_hmac('sha256', $payload, $this->webhookSecret);
        
        return hash_equals($expectedSignature, $signatureHash);
    }

    private function handleSuccessfulPayment(array $session): array
    {
        Log::info('Stripe payment successful', ['session_id' => $session['id']]);
        
        return [
            'success' => true,
            'event' => 'payment_completed',
            'session_id' => $session['id'],
            'payment_intent' => $session['payment_intent']
        ];
    }

    private function handleFailedPayment(array $paymentIntent): array
    {
        Log::warning('Stripe payment failed', ['payment_intent' => $paymentIntent['id']]);
        
        return [
            'success' => true,
            'event' => 'payment_failed',
            'payment_intent' => $paymentIntent['id'],
            'failure_reason' => $paymentIntent['last_payment_error']['message'] ?? 'Unknown error'
        ];
    }
}
```

## 3. Configuration Setup

### Payment Configuration

```php
// config/payments.php
<?php

return [
    'default_gateway' => env('DEFAULT_PAYMENT_GATEWAY', 'stripe'),

    'gateways' => [
        'stripe' => [
            'enabled' => env('STRIPE_ENABLED', true),
            'name' => 'Credit/Debit Card',
            'description' => 'Pay securely with your credit or debit card via Stripe',
            'mode' => env('STRIPE_MODE', 'test'),
            'currencies' => ['USD', 'EUR', 'GBP', 'AUD', 'CAD', 'JPY'],
            'test' => [
                'publishable_key' => env('STRIPE_TEST_PUBLISHABLE_KEY'),
                'secret_key' => env('STRIPE_TEST_SECRET_KEY'),
            ],
            'live' => [
                'publishable_key' => env('STRIPE_LIVE_PUBLISHABLE_KEY'),
                'secret_key' => env('STRIPE_LIVE_SECRET_KEY'),
            ],
            'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
        ],

        // Future gateways will be added here
        /*
        'paypal' => [
            'enabled' => env('PAYPAL_ENABLED', false),
            'name' => 'PayPal',
            'description' => 'Pay with your PayPal account',
            // ... paypal config
        ],
        */
    ],
];
```

### Environment Variables

```env
# .env file

# Default payment gateway
DEFAULT_PAYMENT_GATEWAY=stripe

# Stripe Configuration
STRIPE_ENABLED=true
STRIPE_MODE=test

# Stripe Test Keys (get from https://dashboard.stripe.com/test/apikeys)
STRIPE_TEST_PUBLISHABLE_KEY=pk_test_51234567890abcdef...
STRIPE_TEST_SECRET_KEY=sk_test_51234567890abcdef...

# Stripe Live Keys (get from https://dashboard.stripe.com/apikeys)
STRIPE_LIVE_PUBLISHABLE_KEY=pk_live_51234567890abcdef...
STRIPE_LIVE_SECRET_KEY=sk_live_51234567890abcdef...

# Stripe Webhook Secret (get from https://dashboard.stripe.com/webhooks)
STRIPE_WEBHOOK_SECRET=whsec_1234567890abcdef...
```

## 4. Testing Your Stripe Integration

### Test Card Numbers

```php
// Use these test card numbers in Stripe's test mode:

// Successful payments:
4242424242424242  // Visa
4000056655665556  // Visa (debit)
5555555555554444  // Mastercard
2223003122003222  // Mastercard (2-series)

// Declined payments:
4000000000000002  // Generic decline
4000000000009995  // Insufficient funds
4000000000009987  // Lost card

// Use any future expiry date and any 3-digit CVC
```

### Testing Webhooks Locally

```bash
# Install Stripe CLI
stripe login

# Forward webhooks to your local server
stripe listen --forward-to localhost:8000/payment/webhook/stripe

# Test webhook delivery
stripe trigger checkout.session.completed
```

## Benefits of Starting with Stripe

1. **Proven Foundation**: Build on the most reliable payment platform
2. **Global Reach**: Accept payments from customers worldwide
3. **Easy Testing**: Excellent sandbox environment and test tools
4. **Comprehensive Features**: Support for refunds, subscriptions, and more
5. **Future-Proof**: Easy to add other gateways later without changing core logic

This Stripe-first approach gives you a solid foundation that can grow with your business needs.
