# Changelog

Versions track the `mvqore-sdk` plugin. Bump `version` in both
`plugins/mvqore-sdk/.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json` with every release.

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
