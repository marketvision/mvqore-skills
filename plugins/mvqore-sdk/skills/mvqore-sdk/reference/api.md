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

`themeOwnsReferrerUI` (default `false`): when `true`, `persistReferrer` and
`clearReferrer` always write from the SDK and never notify MV Qore blocks. See
`persistReferrer`.

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
no matter how many callers ask, shared with MV Qore's theme blocks when both are present.

`referrerBypass` is the merchant's Referrer Bypass setting, or `null` when the
store has none:

```js
{ enabled: true, label: "I don't have one", defaultCode: "rootacme" }
```

Offer `label` as a choice only when `enabled` is true. To use it, validate and
persist `defaultCode` like any other code.

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

**It does not persist anything.** Pass the result to `persistReferrer()` to
record the attribution — see recipe 0 for the site-wide capture.

### persistReferrer(result, options?)

Stores the referrer and tags the cart. Pass the object returned by
`validateReferrer`/`getActiveReferrer`, or the raw server record — **not**
`result.referrer`, which throws, because the flattened view drops fields other MV
Qore code reads back.

Returns `"sdk"` when the SDK saved the referrer, or `"listener"` when MV Qore's
Referrer Code block on the page saved it. The SDK announces the referrer, then
checks whether the block actually saved it. If not, the SDK saves it itself.
Other MV Qore blocks on the page cannot stop the save. A console warning says
which path ran.

Skips the cart write on `/pages/share-cart` so an in-flight import is not
clobbered.

| Option | Meaning |
|---|---|
| `forceWrite` | Always save from the SDK, and don't announce it to MV Qore blocks |
| `extensionOwnsWrite` | For the Referrer Code block itself; themes do not need it |

`init({ themeOwnsReferrerUI: true })` applies `forceWrite` to every call. Use it
when the theme draws every referrer UI and no MV Qore block should react.

### clearReferrer(options?)

Removes the stored referrer and clears the cart attribute. It works the same way
as `persistReferrer`: MV Qore blocks on the page are told first, so their UI
resets. If none of them cleared the referrer, the SDK clears it. Returns `"sdk"`
or `"listener"`. Takes `{ forceWrite }`.

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
| `tags` | `null` | Customer tags — array or comma string. Merged with `mvqore_lead`, which the server always adds |
| `source` | `"mvqore-sdk"` | Recorded as `lead_source` |
| `requireReferrer` | `true` | Reject submissions with no referrer |
| `captchaToken` | read from form | Friendly Captcha token |
| `onSuccess` / `onError` | — | Called when the SDK handles the form's own submit event. A failure with no `onError` is logged to the console rather than left as an unhandled rejection. Calling `submit()` yourself returns a promise instead. |

Returns `{ form, submit(overrides?), destroy() }`. `submit()` also works for a
custom button; `destroy()` detaches the listener.

Resolves `{ submitted: true, email, referrerCode, tags, autoLoginTicket, data }`.
Pass `autoLoginTicket` to `startAutoLogin` to sign the new customer in. Rejects with
`MISSING_EMAIL`, `LEAD_TOO_FAST`, `REFERRER_REQUIRED`, `EMAIL_IN_USE`,
`LEAD_REJECTED` or `NETWORK_ERROR`.

Field names read from the form (each also as `customer[...]`): `email`,
`first_name`/`firstName`, `last_name`/`lastName`, `phone`, `country_code`,
`accepts_marketing`, `accepts_sms_marketing`, `frc-captcha-response`.

---

## Account creation

### attachRegistrationForm(formOrSelector, options?) → controller

The Create Account form. Works like `attachLeadForm`, with the same honeypot,
timing, captcha and referrer attribution, and the same options except `tags`, plus:

| Option | Default | Meaning |
|---|---|---|
| `autoLogin` | `true` | Sign the new customer in straight after registering (see `startAutoLogin`) |
| `returnTo` | current page | Storefront path the customer lands on once signed in |

There is no `tags` option: the server applies the merchant's configured Create
Account tag, and nothing else. Passing `tags` logs a warning and has no effect.
Custom tags are only for lead forms. `terms_accepted` is read from the form as a
checkbox.

Resolves `{ submitted: true, email, referrerCode, autoLoginTicket, autoLogin, data }`,
where `autoLogin` is what `startAutoLogin` resolved. With auto-login on,
`onSuccess` can run while the page is already navigating away. Rejects with
`MISSING_EMAIL`, `REGISTRATION_TOO_FAST`, `REFERRER_REQUIRED`, `EMAIL_IN_USE`,
`PHONE_IN_USE`, `REGISTRATION_REJECTED` or `NETWORK_ERROR`.

### startAutoLogin(ticket, options?) → Promise

```js
{ started: true }                                // browser is navigating
{ started: false, reason: "IDP_DISABLED" }        // show a normal login link instead
```

Signs in a customer who has just registered, without a password. Pass
`result.autoLoginTicket` from `attachRegistrationForm` or `attachLeadForm`.
The ticket works once and expires after two minutes. Takes `{ returnTo }`.

This only works when the store uses MV Qore's identity provider for customer
accounts. It takes two redirects: to MV Qore's login service, back to the
storefront home page, then on to Shopify's customer login, which signs the
customer in silently. The SDK does the second redirect when it loads, so the SDK
must be loaded on the home page. The site-wide load in SKILL.md covers this.

Never rejects. `reason` is `NO_TICKET`, `IDP_DISABLED`, `TICKET_EXPIRED`,
`AUTO_LOGIN_UNAVAILABLE`, `STORAGE_UNAVAILABLE` or `NETWORK_ERROR`. In every
case the account exists, so fall back to a login link.

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

## Favorite products

### getFavoriteProducts(options?) → Promise

```js
{ products: [{ id, title, handle, image, imageAlt, price, availableVariant, totalVariants, label, color }], referrerCode: "ANA10" }
```

The products a referrer picked as favorites, at most 10, for the theme to render
in its own markup. By default this is the active referrer's list. With no active
referrer, it is the logged-in customer's own list.

| Option | Default | Meaning |
|---|---|---|
| `country` | `Shopify.country` | Market to filter by; products not sold there are left out |
| `forCustomer` | `false` | Return the logged-in customer's own list even when a referrer is active |

Resolves `{ products: [] }` when there is nothing to show (no referrer and no
logged-in customer, or no favorites), so hide the section on an empty list.
Rejects with `FAVORITES_UNAVAILABLE` or `NETWORK_ERROR`.

---

## Cart drawer

### openCartDrawer() → boolean

Opens the theme's drawer, trying `open()`, then `show()`, then the common open/
closed classes, and firing `cart:refresh`. Returns `false` and navigates to the
cart page when the theme has no drawer.

### setCustomCartDrawerHook(fn)

Replaces all of the above with your own function.
