---
title: Referral System Design for Pooja Path
tags:
  - rails
created: 2026-03-24
updated:
status: draft
---
## Overview

This document captures the referral system we designed and implemented for Pooja Path, including the major architectural decisions, lifecycle, APIs, reward logic, and next steps.

The system supports:

- customer -> customer referrals
- partner -> partner referrals

It does **not** support cross-portal referrals.

That means:

- a customer can refer only a future customer
- a partner can refer only a future partner
- partner referrals may optionally carry a `suggested_partner_type` hint (`pandit` or `vendor`)

## Why We Chose This Design

We considered two broad approaches.

### Option 1: Classic referral code flow

Example:

- every account gets a referral code
- a new user enters that code during signup

### Option 2: Lead-first referral flow

Example:

- referrer enters `name + WhatsApp number`
- backend creates a referral lead
- backend sends invite on WhatsApp
- referred person accepts and later completes OTP/onboarding
- backend attaches conversion and rewards later

We chose **lead-first referral flow**.

## Why Lead-First Was Better

### 1. Better lead capture

We can capture a referred person before they sign up.

### 2. Better operational visibility

Ops and management can see:

- who referred whom
- whether invite was sent
- whether accepted
- whether converted
- whether reward is eligible/paid

### 3. Better fraud resistance

Rewards are not tied to raw code usage alone. They are tied to real lifecycle events:

- customer first paid booking
- partner approval

### 4. Better fit for partner onboarding

Partners do not become “fully valid” at OTP time alone. They go through onboarding and approval. A lead-based referral fits this much better than a simple code.

## Why We Restricted to Same-Portal Referrals

Initially we discussed cross referral support:

- partner -> customer
- customer -> partner

We intentionally did **not** keep that in Phase 1+.

### Reasons

- removes portal ambiguity
- avoids deciding account type too early
- avoids edge cases in OTP/account creation
- keeps reward logic much simpler
- makes attribution cleaner

If needed later, cross-portal referrals can still be added as a future extension.

---

# Final Referral Lifecycle

## Customer referral lifecycle

1. Customer creates referral
2. Invite token is generated
3. WhatsApp invite is sent
4. Referred user opens invite link
5. Referred user accepts referral
6. Referred user verifies OTP and account is attached
7. Referral becomes `converted`
8. On first paid booking, reward becomes `eligible`
9. Management settles reward
10. Referral becomes `rewarded`

## Partner referral lifecycle

1. Partner creates referral
2. Invite token is generated
3. WhatsApp invite is sent
4. Referred partner accepts referral
5. Referred partner verifies OTP and account is attached
6. Referral becomes `otp_verified`
7. Partner completes onboarding and is approved
8. Referral becomes `converted`
9. Reward becomes `eligible`
10. Management settles reward
11. Referral becomes `rewarded`

---

# Core Design Decisions

## 1. `Referral` is the source of truth

We introduced a dedicated `Referral` model instead of overloading:

- `public_id`
- onboarding data
- account creation flow

This keeps referral state independent from account identity state.

## 2. `public_id` is not the referral identifier

Why not use `public_id`?

- customer `public_id` is generated on OTP verification
- partner `public_id` is generated only on approval
- referral needs to exist before both of those

So `public_id` is a business/account ID, not a referral mechanism.

## 3. `suggested_partner_type` is optional

For partner referrals we store:

- `suggested_partner_type: pandit | vendor | nil`

This is only a hint.

Actual source of truth remains onboarding/approval.

## 4. Reward state is separate from conversion state

We intentionally separated:

- account conversion lifecycle
- reward settlement lifecycle

This gave us two axes:

### Conversion status

- `draft`
- `invited`
- `accepted`
- `otp_verified`
- `converted`
- `rewarded`
- `expired`
- `duplicate`

### Reward status

- `pending`
- `eligible`
- `paid`
- `cancelled`

This separation made the system easier to reason about.

---

# Data Model

## `referrals` table

Important fields:

```ruby
t.uuid    :uuid
t.references :referrer, polymorphic: true, null: false

t.string  :referred_name, null: false
t.string  :referred_phone, null: false
t.string  :city
t.string  :suggested_partner_type

t.string  :status, null: false, default: "draft"
t.string  :invite_token

t.references :converted_account, polymorphic: true

t.datetime :invited_at
t.datetime :accepted_at
t.datetime :converted_at
t.datetime :rewarded_at

t.string  :reward_status, null: false, default: "pending"
t.integer :reward_amount_in_paisa, null: false, default: 0
t.datetime :reward_eligible_at
t.datetime :reward_paid_at
t.string  :reward_reference
```

## Key associations

```ruby
belongs_to :referrer, polymorphic: true
belongs_to :converted_account, polymorphic: true, optional: true
```

On actors:

```ruby
has_many :sent_referrals,
         as: :referrer,
         class_name: "Referral",
         dependent: :restrict_with_error
```

---

# Key State Transitions

## Invite flow

```ruby
def ensure_invite_token!
  return invite_token if invite_token.present?

  update!(invite_token: generate_unique_invite_token)
  invite_token
end

def mark_invited!
  attrs = { status: :invited }
  attrs[:invited_at] = Time.current if invited_at.blank?
  update!(attrs)
end
```

## Accept flow

```ruby
def mark_accepted!
  attrs = { status: :accepted }
  attrs[:accepted_at] = Time.current if accepted_at.blank?
  update!(attrs)
end
```

## OTP attach flow

```ruby
def mark_otp_verified!(account:)
  update!(
    status: :otp_verified,
    converted_account: account
  )
end

def mark_converted!(account: converted_account)
  update!(
    status: :converted,
    converted_account: account,
    converted_at: converted_at.presence || Time.current
  )
end
```

## Reward lifecycle

```ruby
def mark_reward_eligible!(amount_in_paisa:, occurred_at: Time.current)
  update!(
    reward_status: :eligible,
    reward_amount_in_paisa: amount_in_paisa,
    reward_eligible_at: reward_eligible_at.presence || occurred_at
  )
end

def mark_reward_paid!(reference:, occurred_at: Time.current)
  update!(
    reward_status: :paid,
    reward_paid_at: reward_paid_at.presence || occurred_at,
    reward_reference: reference,
    status: :rewarded,
    rewarded_at: rewarded_at.presence || occurred_at
  )
end
```

---

# Service Architecture

## 1. Creation

```ruby
module Referrals
  class CreateService
    def call
      Referral.create!(
        referrer: referrer,
        referred_name: params[:referred_name],
        referred_phone: params[:referred_phone],
        suggested_partner_type: params[:suggested_partner_type],
        city: params[:city],
        status: :draft
      )
    end
  end
end
```

## 2. Invite generation and delivery

```ruby
module Referrals
  class InviteService
    def call
      referral.ensure_invite_token!
      invite_url = build_invite_url

      delivered = InviteDeliveryService.deliver(
        referral: referral,
        invite_url: invite_url
      )

      referral.mark_invited! if delivered

      InviteResult.new(delivered, referral, invite_url)
    end
  end
end
```

## 3. Attach referral on OTP verification

```ruby
module Referrals
  class AttachOnOtpVerificationService
    def call
      return unless eligible_account?

      referral = matching_referral
      return unless referral

      if portal == "customer"
        referral.mark_converted!(account: account)
      else
        referral.mark_otp_verified!(account: account)
      end

      referral
    end
  end
end
```

## 4. Partner conversion on approval

```ruby
module Referrals
  class MarkPartnerConvertedService
    def call
      referral = Referral.find_by(
        converted_account: account,
        referrer_type: "PartnerAccount"
      )
      return unless referral
      return if referral.converted? || referral.rewarded?

      referral.mark_converted!(account: account)
      RewardEligibilityService.call(referral: referral, trigger: :partner_conversion)
    end
  end
end
```

## 5. Customer reward eligibility on first paid booking

```ruby
module Referrals
  class CustomerBookingRewardService
    def call
      return unless booking&.customer_account.present?
      return unless booking.paid?
      return unless first_paid_booking?

      referral = Referral.find_by(
        converted_account: booking.customer_account,
        referrer_type: "CustomerAccount",
        status: "converted"
      )
      return unless referral

      RewardEligibilityService.call(
        referral: referral,
        trigger: :customer_first_paid_booking
      )
    end
  end
end
```

## 6. Reward policy

```ruby
module Referrals
  class RewardPolicy
    DEFAULT_CUSTOMER_REWARD_IN_PAISA = 25_000
    DEFAULT_PARTNER_REWARD_IN_PAISA = 50_000

    def reward_amount_in_paisa
      if referral.partner_referral?
        ENV.fetch("REFERRAL_PARTNER_REWARD_IN_PAISA", DEFAULT_PARTNER_REWARD_IN_PAISA).to_i
      else
        ENV.fetch("REFERRAL_CUSTOMER_REWARD_IN_PAISA", DEFAULT_CUSTOMER_REWARD_IN_PAISA).to_i
      end
    end
  end
end
```

## 7. Management settlement

```ruby
module Referrals
  class SettleRewardService
    def call
      case reward_status
      when "paid"
        settle_paid_reward!
      when "cancelled"
        cancel_reward!
      else
        raise ArgumentError, "Unsupported reward status"
      end

      referral
    end
  end
end
```

---

# Public and Account API Design

## Account APIs

### Create referral

`POST /api/v1/account/referrals`

#### Customer example

```json
{
  "referral": {
    "referred_name": "Rahul Das",
    "referred_phone": "+919999999999"
  }
}
```

#### Partner example

```json
{
  "referral": {
    "referred_name": "Pandit Hari",
    "referred_phone": "+918888888888",
    "suggested_partner_type": "pandit",
    "city": "Bhubaneswar"
  }
}
```

#### Example response

```json
{
  "success": true,
  "message": "Referral created successfully.",
  "invite_sent": true,
  "invite_url": "https://api.example.com/api/v1/referrals/abc123token",
  "referral": {
    "uuid": "c123...",
    "referred_name": "Rahul Das",
    "referred_phone": "+919999999999",
    "status": "invited",
    "invite_token": "abc123token",
    "reward_status": "pending",
    "reward_amount": 0.0
  }
}
```

### List own referrals

`GET /api/v1/account/referrals`

#### Example response

```json
{
  "success": true,
  "referrals": [
    {
      "uuid": "ref-1",
      "referred_name": "Rahul Das",
      "status": "rewarded",
      "reward_status": "paid",
      "reward_amount": 500.0
    },
    {
      "uuid": "ref-2",
      "referred_name": "Amit",
      "status": "accepted",
      "reward_status": "pending",
      "reward_amount": 0.0
    }
  ],
  "summary": {
    "total_referrals": 2,
    "accepted_referrals": 2,
    "converted_referrals": 1,
    "eligible_rewards_count": 1,
    "paid_rewards_count": 1,
    "total_reward_amount": 500.0,
    "eligible_reward_amount": 0.0
  }
}
```

### Show own referral

`GET /api/v1/account/referrals/:uuid`

### Re-send invite

`POST /api/v1/account/referrals/:uuid/invite`

---

## Public referral APIs

### View referral by invite token

`GET /api/v1/referrals/:invite_token`

#### Example response

```json
{
  "success": true,
  "referral": {
    "uuid": "ref-uuid",
    "referred_name": "Rahul Das",
    "referred_phone": "+919999999999",
    "status": "invited",
    "target_portal": "customer",
    "referrer_name": "CST-12345"
  }
}
```

### Accept referral

`POST /api/v1/referrals/:invite_token/accept`

#### Example response

```json
{
  "success": true,
  "message": "Referral accepted successfully.",
  "referral": {
    "uuid": "ref-uuid",
    "status": "accepted",
    "accepted_at": "2026-03-24T10:00:00Z"
  }
}
```

---

# Management/Admin APIs

## List referrals for review

`GET /api/v1/management/referrals`

Supports filtering by:

- `reward_status`
- `status`
- `portal`

#### Example request

```http
GET /api/v1/management/referrals?reward_status=eligible&portal=partner
Authorization: Bearer <management-token>
```

#### Example response

```json
{
  "success": true,
  "referrals": [
    {
      "uuid": "ref-uuid",
      "referred_name": "Pandit Hari",
      "status": "converted",
      "reward_status": "eligible",
      "reward_amount_in_paisa": 50000,
      "reward_amount": 500.0,
      "actions": {
        "show": true,
        "settle": true
      }
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "count": 1,
    "pages": 1
  }
}
```

## Show referral for review

`GET /api/v1/management/referrals/:uuid`

## Settle reward

`PATCH /api/v1/management/referrals/:uuid/settle`

### Mark as paid

```json
{
  "referral": {
    "reward_status": "paid",
    "reward_reference": "UTR_REF_123"
  }
}
```

### Cancel reward

```json
{
  "referral": {
    "reward_status": "cancelled"
  }
}
```

### Example paid response

```json
{
  "success": true,
  "message": "Referral reward updated.",
  "referral": {
    "uuid": "ref-uuid",
    "status": "rewarded",
    "reward_status": "paid",
    "reward_reference": "UTR_REF_123",
    "reward_paid_at": "2026-03-24T11:00:00Z",
    "reward_amount": 500.0
  }
}
```

---

# Dashboard Reporting

## Partner dashboard

We added referral earnings into the existing partner dashboard overview.

Example item:

```json
{
  "key": "referral_earnings",
  "label": "Referral earnings",
  "value": 500.0,
  "display_value": "Rs. 500",
  "subtitle": "1 eligible referral rewards",
  "icon": "account-cash-outline",
  "accent": "primary"
}
```

This complements existing payout and assignment stats instead of creating a separate dashboard.

---

# Hook Points in Existing Lifecycle

## OTP verification hook

In `Auth::VerifyOtp`, after account creation/fetch:

```ruby
Referrals::AttachOnOtpVerificationService.call(account: account, portal: @portal)
```

This is where accepted referrals get attached.

## Partner approval hook

In `Partner::ApprovalService`:

```ruby
mark_referral_converted!
```

Which internally calls:

```ruby
Referrals::MarkPartnerConvertedService.call(account: user)
```

## Customer paid booking hook

In `Booking`:

```ruby
after_commit :mark_referral_reward_eligibility, on: :update, if: :referral_reward_eligibility_needed?
```

This is where first paid booking eligibility is detected.

---

# Important Constraints and Rules

## Same-portal only

- customer referrer => customer converted account
- partner referrer => partner converted account

## Matching strategy

Referral attachment currently matches by:

- normalized phone number
- portal type
- accepted referral
- not yet attached to another account

## Reward eligibility rules

### Customer referrals

Eligible only when:

- referral is already `converted`
- referred customer’s first booking becomes `paid`

### Partner referrals

Eligible only when:

- referral reaches `converted`
- partner approval completes

## Management settlement rules

### `paid`

Requires:

- reward currently `eligible`
- `reward_reference` present

### `cancelled`

Requires:

- reward currently `eligible`

---

# Why This Design Works Well

## Operationally clear

At any point, the team can answer:

- invite sent?
- accepted?
- OTP complete?
- converted?
- reward eligible?
- reward paid?

## Extendable

We can later add:

- cross-portal referrals
- coupon/referral code sharing
- multiple reward schemes
- fraud checks
- reward expiry
- referral leaderboards

without rewriting the current model.

## Safer than code-only referrals

A code-only model often becomes opaque and fragile. This design is explicit and auditable.

---

# Example End-to-End Scenarios

## Scenario 1: Customer referral

1. Existing customer creates referral
2. Referral row created with `status = invited`
3. Invite accepted
4. OTP verified
5. Referral becomes `converted`
6. First paid booking happens
7. Reward becomes `eligible`
8. Management marks reward `paid`
9. Referral becomes `rewarded`

## Scenario 2: Partner referral

1. Existing partner creates referral with optional `suggested_partner_type = pandit`
2. Referral row created with `status = invited`
3. Invite accepted
4. OTP verified
5. Referral becomes `otp_verified`
6. Onboarding approved
7. Referral becomes `converted`
8. Reward becomes `eligible`
9. Management settles reward
10. Referral becomes `rewarded`

---

# Environment Variables

Current reward defaults can be overridden with:

```bash
REFERRAL_CUSTOMER_REWARD_IN_PAISA=25000
REFERRAL_PARTNER_REWARD_IN_PAISA=50000
```

Invite URL generation can use:

```bash
REFERRAL_PUBLIC_BASE_URL=https://api.example.com
APP_BASE_URL=https://api.example.com
```

WhatsApp template can use:

```bash
BHASHSMS_TEMPLATE_REFERRAL_INVITE=poojapath_referral_invite
REFERRAL_APP_LABEL=Pooja Path
```

---

# Testing Strategy

We covered the system in focused tests for:

- referral model validation
- invite token generation
- WhatsApp invite delivery
- public accept flow
- OTP attachment
- partner approval conversion
- customer first paid booking reward eligibility
- referral summary reporting
- dashboard referral earnings
- management settlement flow

This gives confidence across the full lifecycle without only relying on request tests.

---

# Tradeoffs We Accepted

## 1. Phone-number matching is strict but simple

We currently attach accepted referrals by normalized phone number. This is practical and reliable enough for current flows, but later we may want stronger invite-token to signup linking.

## 2. Reward settlement is manual

Management settles rewards explicitly. This is good for control now, but later we might automate payout rails.

## 3. Customer dashboard is not yet referral-focused

We exposed referral summaries in referral endpoints first. That was lower risk than building a separate customer dashboard immediately.

---

# Next Steps

## Short-term improvements

- add management search and sorting for referrals
- add export support for referral rewards
- add notes/comments on settlement
- notify referrer when reward becomes eligible
- notify referrer when reward is paid

## Medium-term improvements

- add fraud/risk rules
  - multiple referrals to same phone
  - suspicious booking/payment loops
  - abuse throttling per referrer
- add reward expiry windows
- add configurable reward campaigns by portal or city
- add management audit trail for who settled and why

## Long-term possibilities

- cross-portal referrals
- shareable public referral links from profile
- referral leaderboard and gamification
- automated reward payout integration

---

# Final Summary

We implemented a referral system that is:

- lead-first
- WhatsApp-driven
- portal-safe
- lifecycle-aware
- reward-aware
- admin-settleable

The biggest architectural win was separating:

- account lifecycle
- referral lifecycle
- reward lifecycle

That kept the system understandable, testable, and extensible.

---

## Useful Snippets

### Create referral

```bash
curl -X POST https://api.example.com/api/v1/account/referrals \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "referral": {
      "referred_name": "Rahul Das",
      "referred_phone": "+919999999999"
    }
  }'
```

### Accept referral

```bash
curl -X POST https://api.example.com/api/v1/referrals/<invite_token>/accept
```

### Settle reward as paid

```bash
curl -X PATCH https://api.example.com/api/v1/management/referrals/<uuid>/settle \
  -H "Authorization: Bearer <management-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "referral": {
      "reward_status": "paid",
      "reward_reference": "UTR_REF_123"
    }
  }'
```

### List eligible rewards

```bash
curl -X GET "https://api.example.com/api/v1/management/referrals?reward_status=eligible" \
  -H "Authorization: Bearer <management-token>"
```

If needed, this can later be split into:
- `Referral System - Product & UX`
- `Referral System - Backend Architecture`
- `Referral System - Reward Settlement Ops`