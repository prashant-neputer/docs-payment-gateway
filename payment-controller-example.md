# Payment Controller and Routes Implementation

## Unified Payment Controller

```php
// app/Http/Controllers/PaymentController.php
<?php

namespace App\Http\Controllers;

use App\Services\PaymentManager;
use App\Models\Course;
use App\Models\Transaction;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Session;
use Illuminate\Support\Facades\Validator;

class PaymentController extends Controller
{
    private PaymentManager $paymentManager;

    public function __construct(PaymentManager $paymentManager)
    {
        $this->paymentManager = $paymentManager;
    }

    public function showCheckout(Course $course)
    {
        $availableGateways = $this->paymentManager->getAvailableGateways();
        
        return view('checkout', compact('course', 'availableGateways'));
    }

    public function createPayment(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'course_slug' => 'required|exists:courses,slug',
            'payment_gateway' => 'required|string',
            'full_name' => 'required|string|min:3|max:255',
            'email' => 'required|email|max:255',
            'phone' => 'required|string|min:10|max:20',
        ]);

        if ($validator->fails()) {
            return redirect()->back()->withErrors($validator)->withInput();
        }

        $gateway = $request->input('payment_gateway');
        $course = Course::where('slug', $request->course_slug)->firstOrFail();

        // Check if gateway is available
        if (!in_array($gateway, $this->paymentManager->getAvailableGateways())) {
            return redirect()->back()->with('error', 'Selected payment method is not available.');
        }

        $paymentData = $this->buildPaymentData($course, $request, $gateway);

        try {
            $result = $this->paymentManager->gateway($gateway)->createPayment($paymentData);

            if ($result['success']) {
                Session::put('payment_data', [
                    'gateway' => $gateway,
                    'session_id' => $result['session_id'],
                    'payment_id' => $result['payment_id'],
                    'course_data' => $request->except('_token'),
                    'amount' => $course->course_cost,
                    'currency' => 'USD'
                ]);

                return redirect($result['payment_url']);
            }

            return redirect()->back()->with('error', $result['error'] ?? 'Payment initialization failed');

        } catch (\Exception $e) {
            \Log::error('Payment creation failed', [
                'gateway' => $gateway,
                'error' => $e->getMessage(),
                'course_id' => $course->id
            ]);

            return redirect()->back()->with('error', 'Payment service temporarily unavailable');
        }
    }

    public function handleSuccess(Request $request, string $gateway)
    {
        $paymentData = Session::get('payment_data');
        
        if (!$paymentData || $paymentData['gateway'] !== $gateway) {
            return redirect()->route('home')->with('error', 'Invalid payment session');
        }

        try {
            // For Stripe, use session_id from URL parameter if available
            $transactionId = $gateway === 'stripe' && $request->has('session_id') 
                ? $request->get('session_id') 
                : $paymentData['payment_id'];

            $result = $this->paymentManager->gateway($gateway)->verifyPayment($transactionId);

            if ($result['success'] && $result['status'] === 'completed') {
                $this->storeTransaction($gateway, $result, $paymentData);
                Session::forget('payment_data');
                
                return redirect()->route('home')->with('success', 'Payment completed successfully!');
            }

            return redirect()->route('home')->with('error', 'Payment verification failed');

        } catch (\Exception $e) {
            \Log::error('Payment verification failed', [
                'gateway' => $gateway,
                'error' => $e->getMessage(),
                'payment_data' => $paymentData
            ]);

            return redirect()->route('home')->with('error', 'Payment verification failed');
        }
    }

    public function handleCancel(Request $request, string $gateway)
    {
        Session::forget('payment_data');
        return redirect()->route('home')->with('error', 'Payment was cancelled');
    }

    public function handleWebhook(Request $request, string $gateway)
    {
        try {
            $result = $this->paymentManager->gateway($gateway)->handleCallback($request);
            
            if ($result['success']) {
                \Log::info("Webhook processed successfully", [
                    'gateway' => $gateway,
                    'event' => $result['event'] ?? 'unknown'
                ]);
            }

            return response('OK', 200);

        } catch (\Exception $e) {
            \Log::error('Webhook processing failed', [
                'gateway' => $gateway,
                'error' => $e->getMessage(),
                'payload' => $request->getContent()
            ]);

            return response('Error', 500);
        }
    }

    private function buildPaymentData(Course $course, Request $request, string $gateway): array
    {
        return [
            'amount' => $course->course_cost,
            'currency' => 'USD',
            'description' => "Course: {$course->title}",
            'buyer' => [
                'name' => $request->full_name,
                'email' => $request->email,
                'phone' => $request->phone,
            ],
            'shipping_address' => [
                'city' => 'Default City',
                'address' => 'Default Address',
                'zip' => '00000',
            ],
            'order' => [
                'reference_id' => "course-{$course->id}-" . time(),
                'items' => [[
                    'title' => $course->title,
                    'quantity' => 1,
                    'unit_price' => $course->course_cost,
                    'category' => 'Education'
                ]]
            ],
            'urls' => [
                'success' => route('payment.success', ['gateway' => $gateway]),
                'cancel' => route('payment.cancel', ['gateway' => $gateway]),
                'failure' => route('payment.cancel', ['gateway' => $gateway])
            ]
        ];
    }

    private function storeTransaction(string $gateway, array $paymentResult, array $paymentData): void
    {
        $course = Course::where('slug', $paymentData['course_data']['course_slug'])->first();

        Transaction::create([
            'user_id' => auth()->id() ?? 1,
            'transaction_id' => $paymentResult['raw_data']['id'],
            'payment_method_id' => $this->getPaymentMethodId($gateway),
            'amount' => $paymentResult['amount'],
            'currency' => $paymentResult['currency'],
            'metas' => json_encode($paymentResult['raw_data']),
            'course_name' => $course->title,
            'status' => 2, // Completed
            'transaction_for' => 1, // Course
            'description' => json_encode($paymentData['course_data'])
        ]);
    }

    private function getPaymentMethodId(string $gateway): int
    {
        $methodMap = [
            'stripe' => 4,
            'paypal' => 1,
            // Add more as needed
        ];

        return $methodMap[$gateway] ?? 4; // Default to Stripe
    }
}
```

## Routes Configuration

```php
// routes/web.php

// Unified payment routes
Route::prefix('payment')->name('payment.')->group(function () {
    Route::post('create', [PaymentController::class, 'createPayment'])->name('create');
    Route::get('success/{gateway}', [PaymentController::class, 'handleSuccess'])->name('success');
    Route::get('cancel/{gateway}', [PaymentController::class, 'handleCancel'])->name('cancel');
    Route::post('webhook/{gateway}', [PaymentController::class, 'handleWebhook'])->name('webhook');
});

// Legacy routes for backward compatibility (if needed)
Route::post('stripe-process-transaction', [PaymentController::class, 'createPayment'])
    ->defaults('payment_gateway', 'stripe')
    ->name('stripeProcessTransaction');

// Additional gateway routes can be added here as needed
```

## Frontend Checkout Form

```blade
{{-- resources/views/checkout.blade.php --}}
@extends('layouts.app')

@section('content')
<div class="checkout-container">
    <form action="{{ route('payment.create') }}" method="POST">
        @csrf
        
        {{-- Course Information --}}
        <input type="hidden" name="course_slug" value="{{ $course->slug }}">
        
        {{-- Customer Information --}}
        <div class="customer-info">
            <h3>Billing Information</h3>
            <div class="form-group">
                <input type="text" name="full_name" placeholder="Full Name" 
                       value="{{ old('full_name') }}" required class="form-control">
                @error('full_name')
                    <span class="error">{{ $message }}</span>
                @enderror
            </div>
            
            <div class="form-group">
                <input type="email" name="email" placeholder="Email" 
                       value="{{ old('email') }}" required class="form-control">
                @error('email')
                    <span class="error">{{ $message }}</span>
                @enderror
            </div>
            
            <div class="form-group">
                <input type="tel" name="phone" placeholder="Phone" 
                       value="{{ old('phone') }}" required class="form-control">
                @error('phone')
                    <span class="error">{{ $message }}</span>
                @enderror
            </div>
        </div>

        {{-- Payment Gateway Selection --}}
        <div class="payment-methods">
            <h3>Select Payment Method</h3>
            @foreach($availableGateways as $gateway)
                @php
                    $config = config("payments.gateways.{$gateway}");
                @endphp
                <div class="payment-option">
                    <input type="radio" 
                           name="payment_gateway" 
                           value="{{ $gateway }}" 
                           id="gateway_{{ $gateway }}" 
                           {{ old('payment_gateway') === $gateway ? 'checked' : '' }}
                           required>
                    <label for="gateway_{{ $gateway }}" class="payment-label">
                        <div class="gateway-info">
                            <strong>{{ $config['name'] }}</strong>
                            <small>{{ $config['description'] }}</small>
                        </div>
                        <div class="gateway-logo">
                            @if($gateway === 'stripe')
                                <img src="https://logos-world.net/wp-content/uploads/2021/03/Stripe-Logo.png"
                                     alt="Stripe" class="logo-img">

                            @elseif($gateway === 'paypal')
                                <img src="https://www.paypalobjects.com/webstatic/mktg/Logo/pp-logo-100px.png"
                                     alt="PayPal" class="logo-img">
                            @else
                                <div class="generic-logo">{{ ucfirst($gateway) }}</div>
                            @endif
                        </div>
                    </label>
                </div>
            @endforeach
            
            @error('payment_gateway')
                <span class="error">{{ $message }}</span>
            @enderror
        </div>

        {{-- Order Summary --}}
        <div class="order-summary">
            <h3>Order Summary</h3>
            <div class="course-details">
                <div class="course-info">
                    <h4>{{ $course->title }}</h4>
                    <p class="course-description">{{ Str::limit($course->description, 100) }}</p>
                </div>
                <div class="pricing">
                    <span class="amount">${{ number_format($course->course_cost, 2) }}</span>
                </div>
            </div>
            <div class="total">
                <strong>Total: ${{ number_format($course->course_cost, 2) }}</strong>
            </div>
        </div>

        <button type="submit" class="btn btn-primary btn-pay">
            Complete Payment
        </button>
    </form>
</div>

<style>
.checkout-container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
}

.payment-option {
    border: 1px solid #ddd;
    border-radius: 8px;
    margin-bottom: 10px;
    overflow: hidden;
}

.payment-label {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.payment-label:hover {
    background-color: #f8f9fa;
}

.payment-option input[type="radio"]:checked + .payment-label {
    background-color: #e3f2fd;
    border-color: #2196f3;
}

.gateway-info strong {
    display: block;
    margin-bottom: 5px;
}

.gateway-info small {
    color: #666;
}



.logo-img {
    max-height: 30px;
    max-width: 80px;
}

.order-summary {
    background: #f8f9fa;
    padding: 20px;
    border-radius: 8px;
    margin: 20px 0;
}

.course-details {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
}

.btn-pay {
    width: 100%;
    padding: 15px;
    font-size: 16px;
    font-weight: bold;
}

.error {
    color: #dc3545;
    font-size: 14px;
    margin-top: 5px;
    display: block;
}
</style>
@endsection
```

## Environment Configuration

```env
# .env file

# Default payment gateway
DEFAULT_PAYMENT_GATEWAY=stripe

# Stripe Configuration (Primary Gateway)
STRIPE_ENABLED=true
STRIPE_MODE=test
STRIPE_TEST_PUBLISHABLE_KEY=pk_test_your_stripe_key
STRIPE_TEST_SECRET_KEY=sk_test_your_stripe_key
STRIPE_LIVE_PUBLISHABLE_KEY=pk_live_your_stripe_key
STRIPE_LIVE_SECRET_KEY=sk_live_your_stripe_key
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret

# Additional gateways can be configured here as needed
# Example for PayPal:
# PAYPAL_ENABLED=false
# PAYPAL_MODE=test
# PAYPAL_TEST_CLIENT_ID=your_paypal_client_id
# PAYPAL_TEST_CLIENT_SECRET=your_paypal_client_secret
```

This implementation provides a clean, scalable foundation for multiple payment gateways with Stripe as a complete example.
