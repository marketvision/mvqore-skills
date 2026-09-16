# MV Qore SDK — API reference

Every function lives on the `window.MVQore` global. Async functions reject with
`{ code, message }` unless noted. `getStatus`, `getSettings` and
`getBotProtection` never reject.

---

## init(config)

```js
MVQore.init({ shop: "store.myshopify.com", locale: "en" })
```

Optional. `shop` defaults to `Shopify.shop`, `locale` to `"en"`. Call once per
page before other functions.

---

## Status and settings

### getStatus() → Promise

```js
{ installed: true, configured: true }
{ installed: false, configured: false }                             // app not on this store
{ installed: false, configured: false, error: { code, message } }   // could not check
```

Never rejects, always fails closed. `error.code` is `STATUS_UNAVAILABLE` or
`NETWORK_ERROR` — its presence is what separates "really not installed" from "we
could not reach the app".

### getSettings() → Promise

Returns the merchant's storefront settings (toolbar copy, display mode, cookie
and share-cart settings), or `null` if they cannot be read. One request per page
no matter how many callers ask, shared with the app embed when both are present.

### getBotProtection() → Promise

```js
{ enabled: true, service: "friendly-captcha", siteKey: "FCM…" }
```

Public fields only. Never returns the API key. Fails closed to
`{ enabled: false, service: null, siteKey: null, error }`.

---

## Referrers

### validateReferrer(code) → Promise

Validates a referrer code against the server.

```js
{ isValid: true, referrer: { code, userId, name, imageUrl }, record: {…} }
```

`record` is the full server response — pass it to `persistReferrer`. Rejects with
`INVALID_REFERRER_CODE`, `RATE_LIMITED`, `REFERRER_DATA_NOT_FOUND` or
`NETWORK_ERROR`.

### getActiveReferrer() → Promise

Resolves the referrer for this visitor: validates `?ref=` when present and not
already fresh, otherwise falls back to what is stored. Returns the same shape as
`validateReferrer`, or `null` when there is no referrer. Never rejects.

### persistReferrer(result, options?)

Stores the referrer and tags the cart. Pass the object returned by
`validateReferrer`/`getActiveReferrer`, or the raw server record — **not**
`result.referrer`, which throws, because the flattened view drops fields other MV
Qore code reads back.

Skips its own writes when the app embed is on the page (the embed does them).
Skips the cart write on `/pages/share-cart` so an in-flight import is not
clobbered. `options.extensionOwnsWrite` is for the app embed itself; themes do
not need it.

### clearReferrer()

Removes the stored referrer and clears the cart attribute.

---

## Toolbar copy

### formatToolbarTitle(referrerName, settings) → string

The merchant's configured title with `{{affiliate}}` substituted, e.g.
`YOU ARE SHOPPING WITH <span>Ana Silva</span>`. Falls back to that same wording
when the merchant configured none. The name is HTML-escaped.

### getReferrerPrompt(settings, storeName) → string

The "nobody attributed yet" prompt, e.g. `Who referred you to Prüvit?`. Prefers
the merchant's configured prompt; otherwise substitutes `storeName`.

---

## Lead capture

### attachLeadForm(formOrSelector, options?) → controller

Takes over a `<form>`: injects the honeypot, stamps render time, attaches
referrer attribution, posts on submit.

| Option | Default | Meaning |
|---|---|---|
| `tags` | `null` | Customer tags — array or comma string |
| `source` | `"mvqore-sdk"` | Recorded as `lead_source` |
| `requireReferrer` | `true` | Reject submissions with no referrer |
| `captchaToken` | read from form | Friendly Captcha token |
| `onSuccess` / `onError` | — | Called when the SDK handles the form's own submit event. A failure with no `onError` is logged to the console rather than left as an unhandled rejection. Calling `submit()` yourself returns a promise instead. |

Returns `{ form, submit(overrides?), destroy() }`. `submit()` also works for a
custom button; `destroy()` detaches the listener.

Resolves `{ submitted: true, email, referrerCode, tags, data }`. Rejects with
`MISSING_EMAIL`, `LEAD_TOO_FAST`, `REFERRER_REQUIRED`, `EMAIL_IN_USE`,
`LEAD_REJECTED` or `NETWORK_ERROR`.

Field names read from the form (each also as `customer[...]`): `email`,
`first_name`/`firstName`, `last_name`/`lastName`, `phone`, `country_code`,
`accepts_marketing`, `accepts_sms_marketing`, `frc-captcha-response`.

### createLeadSession(options?) → { submit(fields) }

Same options, for flows with no `<form>`. **Timing starts when the session is
created**, so create it when the UI appears, not at submit time. `submit()` takes
`{ email, firstName, lastName, phone, countryCode, acceptsMarketing,
acceptsSmsMarketing, tags, source, referrerCode, referrerId }`.

---

## Cart sharing

### generateShareUrl({ code, variantId, quantity }) → Promise

Builds a share link. With `variantId` + `quantity` it shares that one item;
otherwise it reads the current cart. Returns `{ url }`, or **`null` when the cart
is empty**. Line-item properties and selling plans survive the round trip.

### importSharedCart(options?) → Promise

Reads `?share_cart=` on the current page and imports it.

```js
{ imported: true, items: [...], referrerCode: "ANA10" }
{ imported: false, items: [], referrerCode: null, reason: "NO_SHARE_LINK" }
```

`reason` is `NO_SHARE_LINK` or `ALREADY_HANDLED`. Clears the cart first so the
link reproduces the sender's cart; pass `{ clear: false }` to merge instead.
Rejects with `INVALID_SHARE_LINK` or `CART_ADD_FAILED`. Does not redirect or
write any UI — the theme decides what happens next.

---

## QR codes

### generateQRCode(url, options?) → { dataUrl, markup }

| Option | Default | Meaning |
|---|---|---|
| `size` | `200` | Pixel width/height |
| `ecLevel` | `"M"` | Error correction: `L`, `M`, `Q`, `H` |
| `margin` | `4` | Quiet-zone modules |
| `target` | `null` | Element or selector to render into |

`dataUrl` suits an `<img src>`; `markup` is inline SVG. Throws on empty input, an
unknown `ecLevel`, an unsupported `format`, or a `target` selector that matches
nothing. The encoder is bundled — do not load a QR library.

---

## Cart drawer

### openCartDrawer() → boolean

Opens the theme's drawer, trying `open()`, then `show()`, then the common open/
closed classes, and firing `cart:refresh`. Returns `false` and navigates to the
cart page when the theme has no drawer.

### setCustomCartDrawerHook(fn)

Replaces all of the above with your own function.
