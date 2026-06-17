We have 2 options:
1. To add a variant enum on package then clone and change variant as well as items, quantities, prices etc
2. PackageVariant approach.

>How it would look like in packagevariant approach , do the mgmt "add package" UI changes or not ? and what changes apps need ?

`PackageVariant` would make `Package` the parent/catalog entity and move the bookable differences into child rows. In that design, a customer does not really book a plain package anymore; they book a specific variant of a package.

A clean shape would be:

- `Package`
  - shared metadata: `name`, `slug`, `description`, `status`
  - shared presentation: categories, gallery/main images
  - optional shared flags: `recommended` at package level only if it means “whole package is promoted”
- `PackageVariant`
  - `package_id`
  - `label` or `variant` enum: `popular`, `plus`, `premium`
  - `status`
  - `recommended` or `is_default`
  - `pandit_fee`
  - `duration`
  - maybe `display_order`
- `PackageVariantItem`
  - `package_variant_id`, `item_id`, `quantity`
- `PackageVariantTax`
  - same idea as `package_taxes`, but attached to variant
- `Booking`
  - add `package_variant_id`
  - keep `package_id` too if you want easier reporting/grouping, but variant becomes the real source of pricing/items snapshot

So the mental model becomes:
- `Package` = “Satyanarayan Pooja”
- `PackageVariant` = “Popular”, “Plus”, “Premium”
- each variant has its own items, quantities, duration, taxes, fee, price

**Mgmt UI**

Would need real UI changes if you go with `PackageVariant`.

Right now the mgmt app has a single package form with package-level:
- `panditFee`
- `duration`
- `packageItemsAttributes`
- `packageImagesAttributes`
- `taxUuids`

That’s visible in:
- [schemas.ts](/Users/chankruze/personal/poojapath-workspace/poojapatha-mgmt-web/src/apis/packages/schemas.ts:1)
- [new/schema.ts](/Users/chankruze/personal/poojapath-workspace/poojapatha-mgmt-web/src/components/pages/packages/new/schema.ts:1)
- [package-form.tsx](/Users/chankruze/personal/poojapath-workspace/poojapatha-mgmt-web/src/components/pages/packages/package-form.tsx:560)

With `PackageVariant`, the add/edit UI would likely change from:
- one package
- one item list
- one fee/duration/tax setup

to:
- package basics section
- package images/categories section
- variants section
- inside each variant: label, fee, duration, taxes, items, maybe recommended/default

So instead of “Add Package”, the page becomes closer to “Add Package with Variants”, or:
1. create package basics
2. add one or more variants inside it

**API Impact**

Current package API is flat:
- `GET /packages`
- `GET /packages/:slug`
- `POST /packages`
- `PATCH /packages/:slug`

with package-level item/image/tax attributes in [packages_controller.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/controllers/api/v1/packages_controller.rb:1).

With `PackageVariant`, you’d usually change it to one of these shapes:

Option A: nested in package endpoints
```json
{
  "package": {
    "name": "Ganesh Pooja",
    "description": "...",
    "category_uuids": ["..."],
    "package_images_attributes": [...],
    "variants_attributes": [
      {
        "label": "popular",
        "pandit_fee": 500,
        "duration": 60,
        "tax_uuids": ["..."],
        "variant_items_attributes": [
          { "item_uuid": "...", "quantity": 2 }
        ]
      }
    ]
  }
}
```

Option B: separate package and variant APIs
- `POST /packages`
- `POST /packages/:slug/variants`
- `PATCH /packages/:slug/variants/:variant_id`

Option A is nicer for create/edit screens. Option B is nicer if variants get complex and need independent management.

**Read APIs**

`GET /packages` would likely return either:
- package summary with a compact `variants` array, or
- package rows denormalized per variant for customer-facing listing

Example summary shape:
```json
{
  "uuid": "...",
  "name": "Ganesh Pooja",
  "slug": "ganesh-pooja",
  "variants": [
    { "uuid": "...", "label": "popular", "price": 999, "duration": 60, "isDefault": true },
    { "uuid": "...", "label": "premium", "price": 1499, "duration": 90 }
  ]
}
```

`GET /packages/:slug` would include full variant details.

**Booking Impact**

This is the biggest backend change.

Right now booking creation receives package slug and snapshots `package.package_items`:
- [create_service.rb](/Users/chankruze/personal/poojapath-workspace/pooja_path_api/app/services/booking/create_service.rb:1)

With variants, booking payload would need `package_variant_uuid` or similar:
```json
{
  "booking": {
    "package_slug": "ganesh-pooja",
    "package_variant_uuid": "..."
  }
}
```

Then snapshot logic changes from:
- package items/taxes/fee/duration

to:
- variant items/taxes/fee/duration

This also affects:
- booking serializer
- receipt/invoice display
- payment amount calculation
- reporting
- assignment rules if partner capability differs by variant

**Partner Capability Impact**

Today partner capability is package-level:
- `partner_package_capabilities`

If variants are materially different, you’ll need to decide:
- capability remains package-level, meaning any partner for that package can do all variants
- or capability moves to variant-level, which is more precise but more work

If `premium` just has more samagri/items but same ritual skill, package-level may still be okay.
If `premium` requires a different ritual flow or senior pandit, variant-level is better.

**Would I recommend it now?**

Use `PackageVariant` now if:
- variants will definitely have different items/prices/duration
- variants will be selectable by users
- you expect more than a label very soon
- you want one canonical package page with multiple purchasable options

Stay with cloned `Package` rows + `variant` enum if:
- you want fastest delivery
- admin team is okay managing separate package records
- public/customer app can treat each variant as a separate package card for now
- you don’t want booking/API/UI refactor yet

**Practical tradeoff**

Cloned `Package` rows:
- smallest backend change
- smallest mgmt UI change
- no booking refactor
- but duplicates images/categories/description and gets messy over time

`PackageVariant`:
- better domain model
- better long-term customer UX
- but requires coordinated changes in API, mgmt web, booking flow, serializers, and probably customer app too

My honest recommendation: if we only need labels now and maybe separate content later, add `variant` enum on `Package` first. If product already knows customers will choose `Popular` vs `Premium` under the same package page, then it’s worth doing `PackageVariant` properly.