---
name: mvqore-sdk
description: Build MV Qore referral features into a custom Shopify theme — referrer toolbar, referrer code input, lead capture forms with customer tags, Create Account forms with auto-login, a referrer's favorite products, cart sharing, QR codes, referrer attribution. Use whenever a Shopify theme needs referral, referrer, affiliate attribution, "who referred you", lead capture, account registration, favorite products, share cart or MV Qore functionality.
tags: [shopify, theme, referral, mvqore, lead-capture]
---

# MV Qore SDK

`window.MVQore` exposes MV Qore's referral functions to any Shopify theme, so a
custom theme gets the same behaviour as MV Qore's theme blocks while owning all of its
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
  tags: ["newsletter", "spring-promo"], // added alongside mvqore_lead
  source: "homepage-form",
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

Every lead is tagged `mvqore_lead` server-side, whatever `tags` you pass. That
tag is what puts the customer in the merchant's Leads list, so never try to
remove it.

### Create Account form

```js
MVQore.attachRegistrationForm("#register-form", {
  returnTo: "/account",
  onSuccess: (r) => {
    if (!r.autoLogin.started) location.href = "/account/login"; // no silent sign-in
  },
  onError: (e) => showError(e.code === "EMAIL_IN_USE"
    ? "You already have an account"
    : "Something went wrong"),
});
```

Same fields as the lead form, plus a `terms_accepted` checkbox. No `tags`: the
merchant's configured Create Account tag is applied for you. After a
successful registration the SDK signs the customer in and redirects them. This
works only when the store uses MV Qore's identity provider. Otherwise
`autoLogin.started` is false and the theme sends the customer to login. To show
a message before the redirect, pass `autoLogin: false`, then call
`MVQore.startAutoLogin(r.autoLoginTicket)` within two minutes.

### "I don't have a referrer" option

If the merchant enabled Referrer Bypass, offer their default code as a choice:

```js
const { referrerBypass } = (await MVQore.getSettings()) ?? {};
if (referrerBypass?.enabled) {
  skipButton.textContent = referrerBypass.label;
  skipButton.onclick = async () =>
    MVQore.persistReferrer(await MVQore.validateReferrer(referrerBypass.defaultCode));
}
```

### Favorite products

```js
const { products } = await MVQore.getFavoriteProducts();
if (!products.length) section.hidden = true;
else section.innerHTML = products.map(renderProductCard).join("");
```

This returns the active referrer's picks, filtered to the shopper's market. Build
the cards in the theme's own product-card markup.

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

**MV Qore's theme blocks can share a page with the SDK.** If the Referrer Code
block is on the page, it saves the referrer and the SDK skips its own save, so
the cart is not tagged twice. The Lead OptIn and Free Registration blocks don't
save referrers, so with those the SDK saves it. Either way the SDK checks that
the save happened, so you do not need to detect which blocks are present. If the
theme draws all of its referrer UI itself, call
`MVQore.init({ themeOwnsReferrerUI: true })` so no block reacts to it.

**`getActiveReferrer()` does not store anything.** It resolves who the referrer
is; `persistReferrer()` is what writes storage and tags the cart. A page that
only calls `getActiveReferrer` will display a referrer and record no attribution
— see the site-wide capture in "Load it".

**Load the SDK on the home page if you use auto-login.** Sign-in passes through
the home page on its way back from MV Qore's login service, and the SDK finishes
it there. Without the SDK on that page, the customer is left signed out.

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
| `REGISTRATION_TOO_FAST` | Same, for a Create Account form |
| `REFERRER_REQUIRED` | No referrer to attribute the lead to; pass `requireReferrer: false` to allow |
| `EMAIL_IN_USE` | Already a customer — show this on the email field |
| `PHONE_IN_USE` | Phone belongs to another customer — show this on the phone field |
| `LEAD_REJECTED` | Rejected server-side (bot guard or captcha) |
| `REGISTRATION_REJECTED` | Same, for a Create Account form |
| `INVALID_SHARE_LINK` | `share_cart` payload malformed |
| `CART_ADD_FAILED` | Shopify refused the shared line items |
| `FAVORITES_UNAVAILABLE` | Favorite products could not be loaded |
| `NETWORK_ERROR` | Request never completed |

## Full API

See [reference/api.md](reference/api.md) for every function, argument and return
shape, and [reference/recipes.md](reference/recipes.md) for complete, working
section examples you can adapt.
