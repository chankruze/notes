We won't create a one-off `payment_settings` model. For our codebase, the better move is a small generic settings platform plus a payment-gateway abstraction.

Right now the payment flow is strongly PhonePe-specific in [app/services/payment/create_service.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/services/payment/create_service.rb), [app/controllers/api/v1/booking/payments_controller.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/controllers/api/v1/booking/payments_controller.rb), and [app/models/webhook_event.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/models/webhook_event.rb). The `payments` table also has no gateway discriminator yet in [db/schema.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/db/schema.rb:599). So the clean plan is:

**Settings Design**
A generic `app_settings` table, not `payment_settings`.

Recommended shape:
- `namespace:string` like `payments`, `branding`, `notifications`
- `key:string` like `active_gateway`, `primary_color`
- `value:jsonb`
- `updated_by_management_user_id:bigint`
- timestamps
- unique index on `[:namespace, :key]`

Start with these keys:
- `payments.active_gateway` => `"phonepe"` or `"razorpay"`
- later `branding.primary_color`, `branding.secondary_color`, `branding.logo_url`

We'll keep gateway secrets out of DB. Razorpay/PhonePe credentials should stay in env/credentials; only the runtime choice belongs in settings.

Add a typed registry/service so the app never scatters raw string keys:
- `AppSettings.fetch("payments.active_gateway")`
- `AppSettings.set!("payments.active_gateway", "razorpay", actor: current_management_user)`

**Payment Architecture**
Refactor around provider-specific adapters, but store the provider on every payment record.

Recommended payment changes:
- add `gateway:string` to `payments`, default existing rows to `phonepe`
- add `gateway_payment_id:string` for Razorpay `pay_xxx`
- add `gateway_signature:string` for client-confirmed Razorpay payments
- add `gateway_payload:jsonb` for provider-specific debug/audit data

Important rule:
- admin setting decides the gateway only for new payments
- old payments, retries, refunds, reconciliations, and webhooks must always use `payment.gateway`

That means the factory should support both:
- `build_for_new_payment` from `payments.active_gateway`
- `build_for_existing_payment(payment.gateway)`

**Razorpay Plan**
For mobile apps, the backend should create a Razorpay order and the app should open Razorpay checkout using that order. Then the app sends the success payload back to our backend API for verification.

Implementation sequence:
1. Add Razorpay Ruby SDK and create `Payments::RazorpayGateway`.
2. On payment create, backend creates Razorpay order, stores it in `payments.order_id`, and returns a provider-neutral response contract.
3. Add `POST /api/v1/bookings/:uuid/payment/confirm` for Razorpay success callback payload.
4. Verify `razorpay_signature` on the server using `order_id|razorpay_payment_id`.
5. Fetch payment/order from Razorpay after confirmation for best-effort reconcile.
6. Add `/api/v1/webhooks/razorpay` and treat webhook processing as final source of truth.

I’d also normalize the API response. Today [app/blueprints/payment_blueprint.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/blueprints/payment_blueprint.rb) exposes PhonePe-only fields. Change that to a common envelope like:
- `payment`: common fields
- `checkout`: `{ provider, flow, payload }`

So PhonePe and Razorpay can coexist without ugly controller branching.

**Webhook/Status Decisions**
Razorpay’s docs make two things important:
- webhook responses must return `2xx` within 5 seconds
- duplicate events should be deduped using `x-razorpay-event-id`
- webhook order is not guaranteed, so don’t assume `payment.authorized` always arrives before `payment.captured`

I’d extend `WebhookEvent` to support `razorpay`, and store the Razorpay event id for idempotency.

One important design choice: Razorpay defaults to auto-capture for Orders-based payments. I recommend you keep auto-capture on for this mobile flow, because it simplifies reconciliation. If you intentionally want manual capture, we should add a capture step for `payment.authorized` before marking a booking paid.

**Rollout Order**
1. Build generic settings model + management API.
2. Add `gateway` to `payments` and move PhonePe behind a gateway interface without changing behavior.
3. Add Razorpay order creation + client confirm endpoint.
4. Add Razorpay webhook processing.
5. Update React Native app to use provider-neutral checkout contract.
6. Add Razorpay refunds after the payment success path is stable.

One gap from the links I have checked: they are enough for checkout, server order creation, and webhooks, but not enough to confidently lock refund webhook/event handling. Before coding Razorpay refunds, I’d verify the exact refund webhook docs/events from Razorpay’s official refund docs.

Docs I used:
- [Razorpay webhook setup](https://razorpay.com/docs/webhooks/setup-edit-payments/)
- [Razorpay React Native Standard](https://razorpay.com/docs/payments/payment-gateway/react-native-integration/standard/)
- [Razorpay Ruby server integration](https://razorpay.com/docs/payments/server-integration/ruby/)
- [Create Order API](https://razorpay.com/docs/api/orders/create/)
- [Validate and test webhooks](https://razorpay.com/docs/webhooks/validate-test/)
- [Payments webhook events](https://razorpay.com/docs/webhooks/payments/)
- [Orders webhook events](https://razorpay.com/docs/webhooks/orders/)
- [Payment capture settings](https://razorpay.com/docs/payments/payments/capture-settings/)
