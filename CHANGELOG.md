# Changelog

Versions track the `mvqore-sdk` plugin. Bump `version` in both
`plugins/mvqore-sdk/.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json` with every release.

## 1.2.1

Documentation only; no app release needed.

- A captcha token works once: reset the widget after a failed submission
- The SDK does not validate the form: run the theme's validation before
  attaching the SDK
- Choice values must match the merchant's definition exactly, so translate
  labels, not values

## 1.2.0

Requires the MV Qore app release of 2026-10-02, which serves the matching SDK.

- Application forms: inputs named `mvqore_form.<key>` on lead and Create Account
  forms are saved to the new customer's `mvqore_form` metafields, in the same
  step that creates the customer
- `attachCustomerFieldsForm()` for a logged-in customer: saves the same fields
  to their existing record without creating a customer, writing the answers
  before the tags
- Create Account forms take `tags`, limited to the store's allowlist (previously
  `tags` was ignored with a warning)
- Tags on every form are checked against the allowlist in MV Qore admin → More →
  Application form, once the merchant has saved one
- New error codes for field-level rejections. Errors carry `field` naming the
  input that caused them

## 1.1.0

Requires the MV Qore app release of 2026-09-21, which serves the matching SDK.

- `attachRegistrationForm()` for Create Account forms, with auto-login via `startAutoLogin()`
  (no `tags` option: the merchant's Create Account tag is applied server-side)
- `getFavoriteProducts()` for showing a referrer's favorite products
- `getSettings().referrerBypass` for an "I don't have a referrer" option
- Lead forms: `tags` are merged with `mvqore_lead`, which is always added
- `persistReferrer()` and `clearReferrer()` check that the referrer was really
  saved or cleared, and do it themselves if not. Fixes pages with a Lead OptIn or
  Free Registration block, where both silently did nothing
- `init({ themeOwnsReferrerUI })` and `forceWrite` for themes that own all
  referrer UI
- Docs say "theme block" instead of "app embed", which was inaccurate

## 1.0.0

First release of the `mvqore-sdk` skill for building MV Qore features into a
custom Shopify theme:

- Loading the SDK site-wide and capturing referral attribution from `?ref=`
- Referrer toolbar and referrer code input, using the merchant's configured copy
- Lead capture forms with customer tags, including the SDK-managed bot protection
- Cart sharing, QR codes and importing a shared cart
- Opening the theme's cart drawer
- API reference, error codes and complete example sections
