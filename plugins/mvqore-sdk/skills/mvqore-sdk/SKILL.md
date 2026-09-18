---
name: mvqore-sdk
description: Build MV Qore referral features into a custom Shopify theme — referrer toolbar, referrer code input, lead capture forms with customer tags, cart sharing, QR codes, referrer attribution. Use whenever a Shopify theme needs referral, referrer, affiliate attribution, "who referred you", lead capture, share cart or MV Qore functionality.
tags: [shopify, theme, referral, mvqore, lead-capture]
---

# MV Qore SDK

`window.MVQore` exposes MV Qore's referral functions to any Shopify theme, so a
custom theme gets the same behaviour as the app embeds while owning all of its
own markup and design.

**The division of labour: the SDK provides data and behaviour, you write the
markup.** There are no MV Qore components to drop in and no CSS to match. Build
the UI in the theme's own design system and call the SDK for everything else.

## Load it

One tag per page, before any code that uses it:

Put this in **`theme.liquid`**, once — not per template and not per section.
A referral link can land on any page (a product, the blog, the homepage), so the
SDK has to be present everywhere for attribution to be captured.

```liquid
<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({
      shop: {{ shop.permanent_domain | json }},
      locale: {{ request.locale.iso_code | default: 'en' | json }}
    });

    // Capture attribution from ?ref= on whatever page the visitor landed on.
    // getActiveReferrer() only RESOLVES a referrer -- persisting is a separate
    // call, so without this the referrer shows in the UI and is never recorded.
    // Gated on ?ref= because persistReferrer also tags the cart, and doing that
    // on every page load would fire a /cart/update.js request every time.
    if (new URLSearchParams(location.search).has("ref")) {
      const active = await MVQore.getActiveReferrer();
      if (active) MVQore.persistReferrer(active);
    }
  });
</script>
```

`{{ routes.root }}` keeps the path correct on locale-prefixed storefronts
(`/fr/apps/proxy/...`). The SDK is served by the MV Qore app through Shopify's
App Proxy, which signs every request server-side — **the theme never holds an
API key or secret.** Do not copy the SDK file into the theme's assets: it is
versioned and served centrally so fixes reach every store.

`init()` is optional (shop falls back to `Shopify.shop`) but pass it anyway.

## Always gate on status first

```js
const status = await MVQore.getStatus();
if (!status.installed || !status.configured) return;  // render nothing
```

`getStatus()` never rejects. `installed: false` means the app is not on this
store; `configured: false` means it is installed but its tenant setup is
incomplete. Render no referral UI in either case — a toolbar that cannot resolve
a referrer is worse than no toolbar.

## Recipes

### Referrer toolbar

The bar has two states: a prompt when nobody is attributed, and the merchant's
configured title when someone is. Both strings come from the merchant's admin
settings, so read them rather than hardcoding copy.

```js
const [settings, active] = await Promise.all([
  MVQore.getSettings(),
  MVQore.getActiveReferrer(),
]);

bar.innerHTML = active
  ? MVQore.formatToolbarTitle(active.referrer.name, settings)   // "YOU ARE SHOPPING WITH <span>Ana</span>"
  : MVQore.getReferrerPrompt(settings, {{ shop.name | json }}); // "Who referred you to <store>?"
```

Both helpers return HTML with the referrer name escaped. The brand colour is a
shop metafield — read it in Liquid (`shop.metafields.mvqore.brand_color`) so the
bar paints correctly on first render instead of flashing.

### Referrer code input

```js
try {
  const result = await MVQore.validateReferrer(input.value);
  MVQore.persistReferrer(result);        // writes storage + tags the cart
  render(result.referrer);               // { code, userId, name, imageUrl }
} catch (e) {
  if (e.code === "INVALID_REFERRER_CODE") showError("We couldn't find that code");
  else if (e.code === "RATE_LIMITED") showError("Too many tries — wait a minute");
  else showError("Something went wrong");
}
```

Pass `persistReferrer` the **whole result**, never `result.referrer` — the
flattened view is missing fields the rest of MV Qore reads back, and it throws
if you try. `clearReferrer()` undoes it.

### Lead capture form

Write whatever fields the design calls for, then hand the form to the SDK:

```js
MVQore.attachLeadForm("#lead-form", {
  tags: ["quiz", "flavor-finder"],     // customer tags, array or string
  source: "find-your-flavor",
  onSuccess: () => showThanks(),
  onError: (e) => showError(e.code === "EMAIL_IN_USE"
    ? "You're already signed up"
    : "Something went wrong"),
});
```

The SDK takes over submit, adds the honeypot, times the render, attaches the
active referrer and posts. Recognised field names are `email`, `first_name`,
`last_name`, `phone`, `country_code`, `accepts_marketing`,
`accepts_sms_marketing` — each also accepted as `customer[...]`.

For a flow with no `<form>` (a multi-step quiz), start a session when the UI
appears and submit collected answers later:

```js
const session = MVQore.createLeadSession({ tags: ["quiz"], source: "flavor-quiz" });
// ...user answers questions...
await session.submit({ email, firstName });
```

### Share cart and QR

```js
const share = await MVQore.generateShareUrl({ code: active?.referrer.code });
if (share) {                                    // null when the cart is empty
  const { dataUrl } = MVQore.generateQRCode(share.url, { size: 240, target: "#qr" });
  linkEl.href = share.url;
}
```

`target` renders the QR inline into that element; the QR encoder is built in, so
do not add a QR library. On the page that receives a share link:

```js
const result = await MVQore.importSharedCart();   // clears, then adds the items
if (result.imported) MVQore.openCartDrawer();
```

## Things that will bite you

**A lead form cannot be submitted within 5 seconds of rendering.** The server
rejects faster submissions as scripted. This is correct for humans and surprising
when testing: an auto-submit right after page load fails with `LEAD_TOO_FAST`.
Wait, or fill the form by hand.

**The captcha widget is the theme's job.** If the merchant has bot protection
enabled, something must mount Friendly Captcha and put its token in a field named
`frc-captcha-response` inside the form; the SDK sends it. `getBotProtection()`
returns `{ enabled, service, siteKey }` for mounting it. Without it, submissions
are rejected server-side.

**Do not hand-roll the lead POST.** The endpoint is fail-closed on bot signals the
SDK manages. A hand-written `fetch` to it will be rejected as "not our form".

**If the MV Qore app embed is also enabled on this theme**, the SDK defers to it:
`persistReferrer` skips its own writes so the cart is not tagged twice, and
`importSharedCart` joins the embed's guard. You do not need to detect this.

**`getActiveReferrer()` does not store anything.** It resolves who the referrer
is; `persistReferrer()` is what writes storage and tags the cart. A page that
only calls `getActiveReferrer` will display a referrer and record no attribution
— see the site-wide capture in "Load it".

**`generateShareUrl` returns `null` for an empty cart**, so check before
destructuring.

## Error codes

Every function rejects with `{ code, message }`. `getStatus`, `getSettings` and
`getBotProtection` never reject — they resolve to a safe value instead.

| Code | Meaning |
|---|---|
| `INVALID_REFERRER_CODE` | No referrer matches that code |
| `RATE_LIMITED` | Too many validation attempts; back off |
| `REFERRER_DATA_NOT_FOUND` | Referrer exists but has no usable record |
| `MISSING_EMAIL` | Lead submitted with no email |
| `LEAD_TOO_FAST` | Submitted under 5s after render (see above) |
| `REFERRER_REQUIRED` | No referrer to attribute the lead to; pass `requireReferrer: false` to allow |
| `EMAIL_IN_USE` | Already a customer — show this on the email field |
| `LEAD_REJECTED` | Rejected server-side (bot guard or captcha) |
| `INVALID_SHARE_LINK` | `share_cart` payload malformed |
| `CART_ADD_FAILED` | Shopify refused the shared line items |
| `NETWORK_ERROR` | Request never completed |

## Full API

See [reference/api.md](reference/api.md) for every function, argument and return
shape, and [reference/recipes.md](reference/recipes.md) for complete, working
section examples you can adapt.
