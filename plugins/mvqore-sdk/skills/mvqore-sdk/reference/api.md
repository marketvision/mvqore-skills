# MV Qore SDK — API reference

Every function lives on the `window.MVQore` global. Async functions reject with
`{ code, message }` unless noted, plus `field` when one form input caused the
rejection. `getStatus`, `getSettings` and
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
| `tags` | `null` | Customer tags — array or comma string. Merged with `mvqore_lead`, which the server always adds. Once the store has saved a tag allowlist, every tag must be on it |
| `source` | `"mvqore-sdk"` | Recorded as `lead_source` |
| `requireReferrer` | `true` | Reject submissions with no referrer |
| `captchaToken` | read from form | Friendly Captcha token |
| `onSuccess` / `onError` | — | Called when the SDK handles the form's own submit event. A failure with no `onError` is logged to the console rather than left as an unhandled rejection. Calling `submit()` yourself returns a promise instead. |

Returns `{ form, submit(overrides?), destroy() }`. `submit()` also works for a
custom button; `destroy()` detaches the listener. `submit({ formMetafields })`
adds answers the markup does not hold; they merge over the form's own
`mvqore_form.*` inputs.

Resolves `{ submitted: true, email, referrerCode, tags, autoLoginTicket, data }`.
Pass `autoLoginTicket` to `startAutoLogin` to sign the new customer in. Rejects with
`MISSING_EMAIL`, `LEAD_TOO_FAST`, `REFERRER_REQUIRED`, `EMAIL_IN_USE`,
`LEAD_REJECTED` or `NETWORK_ERROR`, or with one of the
[application field codes](#application-fields).

Field names read from the form (each also as `customer[...]`): `email`,
`first_name`/`firstName`, `last_name`/`lastName`, `phone`, `country_code`,
`accepts_marketing`, `accepts_sms_marketing`, `frc-captcha-response`, and any
`mvqore_form.<key>` input (see [Application fields](#application-fields)).

---

## Account creation

### attachRegistrationForm(formOrSelector, options?) → controller

The Create Account form. Works like `attachLeadForm`, with the same honeypot,
timing, captcha, referrer attribution, options and `mvqore_form.*` fields, plus:

| Option | Default | Meaning |
|---|---|---|
| `autoLogin` | `true` | Sign the new customer in straight after registering (see `startAutoLogin`) |
| `returnTo` | current page | Storefront path the customer lands on once signed in |

The server always applies the merchant's configured Create Account tag. `tags`
adds to it, but only tags on the store's allowlist: a tag not on it rejects the
submission with `TAG_NOT_ALLOWED`, and a store with no saved allowlist ignores
`tags`. `terms_accepted` is read from the form as a checkbox.

Resolves `{ submitted: true, email, referrerCode, autoLoginTicket, autoLogin, data }`,
where `autoLogin` is what `startAutoLogin` resolved. With auto-login on,
`onSuccess` can run while the page is already navigating away. Rejects with
`MISSING_EMAIL`, `REGISTRATION_TOO_FAST`, `REFERRER_REQUIRED`, `EMAIL_IN_USE`,
`PHONE_IN_USE`, `REGISTRATION_REJECTED` or `NETWORK_ERROR`, or with one of the
[application field codes](#application-fields).

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

## Application fields

Answers saved to customer metafields in the `mvqore_form` namespace. They work
on all three customer forms: `attachLeadForm` and `attachRegistrationForm` for a
new customer, and `attachCustomerFieldsForm` for one who is logged in.

### Inputs

Any input named `mvqore_form.<key>` is the answer for the customer metafield
`mvqore_form.<key>`:

| Input | Value sent |
|---|---|
| Text, textarea, number, date, `<select>` | Its trimmed value |
| A single checkbox | `true` / `false` |
| Several checkboxes with the same name | Checked values, as a list |
| Radio buttons | The checked value, or `""` |
| `<select multiple>` | Selected values, as a list |

Disabled inputs are skipped. A blank answer is not saved, so it never erases a
stored value. A list field rendered as a single checkbox is sent as `true` /
`false` and rejected; with only one option, use `<select multiple>`.

### What the server accepts

Rules are set by the merchant, not in theme code:

- **Keys:** only keys with a customer metafield definition in the store,
  created in Shopify admin → Settings → Custom data → Customers as
  `mvqore_form.<key>`. A form never creates a definition.
- **Values:** must fit the definition's type and any choices it lists. Writable
  types: `single_line_text_field` (max 255 chars), `multi_line_text_field`
  (max 5,000), `number_integer`, `number_decimal`, `boolean`, `date`
  (`YYYY-MM-DD`), `url` (http or https), and `list.single_line_text_field`
  (max 50 items). Other types, such as files, are rejected.
- **At most 20 fields per submission.**
- **Tags:** only tags on the allowlist in MV Qore admin → More → Application
  form.
- **Resubmission:** overwrites earlier answers, except fields the merchant
  marked "keep existing value" there.

### attachCustomerFieldsForm(formOrSelector, options?) → controller

For a customer who is already logged in. It updates their existing record and
creates no customer, so there is no email field and no `EMAIL_IN_USE`. The
server identifies the customer from Shopify's signed request; nothing in the
form can name a different customer. Render it inside `{% if customer %}`.

| Option | Default | Meaning |
|---|---|---|
| `tags` | `null` | Tags to add — array or comma string. Each must be on the allowlist |
| `onSuccess` / `onError` | — | As for `attachLeadForm` |

The same honeypot and 5-second minimum apply. There is no captcha, because the
customer is signed in.

Returns `{ form, submit(overrides?), destroy() }`. `submit({ formMetafields,
tags })` merges extra answers and replaces the tags for that call.

Resolves:

```js
{ submitted: true, fields: ["license_number", "years_experience"], tags: ["mvqore-application-submitted"], data }
```

`fields` lists the answers that were saved; a "keep existing value" field that
already had a value is left out. The answers are saved before the tags, so a
Shopify Flow triggered by the tag always sees them.

Rejects with `FIELDS_TOO_FAST`, `NOT_LOGGED_IN`, `RATE_LIMITED` (more than 10 in
a minute), `TRY_AGAIN`, `SHOPIFY_REJECTED`, `FIELDS_REJECTED`, `NETWORK_ERROR`,
or one of the codes below.

### Application field codes

From any of the three forms. `field` names the input when one caused it
(`"tags"` for a tag).

| Code | Meaning |
|---|---|
| `UNKNOWN_FIELD` | No `mvqore_form` definition for that key |
| `INVALID_VALUE` | Value does not fit the type |
| `INVALID_CHOICE` | Not one of the definition's choices |
| `VALUE_TOO_LONG` | Over the length or list-size limit |
| `UNSUPPORTED_TYPE` | The definition's type cannot be written by a form |
| `TOO_MANY_FIELDS` | More than 20 fields |
| `TAG_NOT_ALLOWED` | Tag not on the allowlist |
| `NOTHING_TO_SUBMIT` | No answers and no tags (`attachCustomerFieldsForm` only) |

`TRY_AGAIN` means Shopify stayed unavailable through the server's retries.
Resubmitting is safe: it writes the same answers again.

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
