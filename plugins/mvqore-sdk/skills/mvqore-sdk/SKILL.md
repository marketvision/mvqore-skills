---
name: mvqore-sdk
description: Build MV Qore referral features into a custom Shopify theme — referrer toolbar, referrer code input, lead capture forms with customer tags, Create Account forms with auto-login, application forms that save extra answers to customer metafields, a referrer's favorite products, cart sharing, QR codes, referrer attribution, and content personalized by customer tag or referrer. Use whenever a Shopify theme needs referral, referrer, affiliate attribution, "who referred you", lead capture, account registration, an application or sign-up form with custom fields, favorite products, share cart, content or pricing shown only to tagged customers (VIP, wholesale), or MV Qore functionality.
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

Same fields as the lead form, plus a `terms_accepted` checkbox. The merchant's
configured Create Account tag is applied for you; `tags` can add more, but only
tags on the store's allowlist (see "Application form"). After a
successful registration the SDK signs the customer in and redirects them. This
works only when the store uses MV Qore's identity provider. Otherwise
`autoLogin.started` is false and the theme sends the customer to login. To show
a message before the redirect, pass `autoLogin: false`, then call
`MVQore.startAutoLogin(r.autoLoginTicket)` within two minutes.

### Application form (custom customer fields)

An application form collects answers beyond name and email: a licence number,
years of experience, specialties. Each answer is saved to a customer metafield
in the `mvqore_form` namespace, and the form can add a tag (for example to
trigger a Shopify Flow that emails the application to staff).

**Name each extra input `mvqore_form.<key>`.** The SDK collects every such input
and sends it with the submission; there is no list of fields to configure in
code.

```html
<input name="mvqore_form.license_number" required>
<input name="mvqore_form.years_experience" type="number">
<label><input type="checkbox" name="mvqore_form.specialties" value="keto"> Keto</label>
<label><input type="checkbox" name="mvqore_form.specialties" value="fitness"> Fitness</label>
```

The page has two states. Render both, and let Liquid pick:

```liquid
{% if customer %}
  {%- comment -%} Logged in: only the mvqore_form.* inputs. No name or email. {%- endcomment -%}
  <form id="apply-existing">…mvqore_form inputs…</form>
{% else %}
  {%- comment -%} New customer: the full form plus the same mvqore_form.* inputs. {%- endcomment -%}
  <form id="apply-new">…email, first_name, … and the mvqore_form inputs…</form>
{% endif %}
```

```js
const TAGS = ["mvqore-application-submitted"];

// New customer: the customer is created with the answers and the tag in one go.
// attachRegistrationForm works the same way if the applicant should get an account.
MVQore.attachLeadForm("#apply-new", { tags: TAGS, requireReferrer: false, onSuccess, onError });

// Logged-in customer: updates their existing record; no customer is created.
MVQore.attachCustomerFieldsForm("#apply-existing", { tags: TAGS, onSuccess, onError });
```

Check `e.field` in `onError` to mark the input that was rejected. The server
saves the answers before it adds the tag, so a Flow triggered by the tag always
sees them.

The merchant sets up two things in advance, outside the theme:

- **One metafield definition per field.** In Shopify admin, go to Settings →
  Custom data → Customers, and create it with namespace and key
  `mvqore_form.<key>`. An input with no definition is rejected with
  `UNKNOWN_FIELD`, and the definition's type decides what values are valid.
- **The allowed tags.** In MV Qore admin, go to More → Application form. A tag not
  on that list is rejected with `TAG_NOT_ALLOWED`.

Types a form can write: single-line and multi-line text, integer, decimal,
true/false, date, URL, and a list of single-line text. Shopify has no email
metafield type, so use single-line text for an email field. File uploads are not
supported.

How each kind of input is sent:

| Input | Value sent |
|---|---|
| Text, textarea, number, date, `<select>` | Its trimmed value |
| A single checkbox | `true` or `false` (for a true/false field) |
| Several checkboxes with the same name | The checked values as a list |
| Radio buttons | The checked value |
| `<select multiple>` | The selected values as a list |

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

**A captcha token works once.** The server checks the captcha before anything
else, so every submission that reaches it uses up the token, including one
rejected for `EMAIL_IN_USE` or a bad `mvqore_form` answer. A second attempt
with the same token fails with `LEAD_REJECTED` / `REGISTRATION_REJECTED`. Reset
the Friendly Captcha widget in `onError` so the customer gets a fresh token
before they resubmit.

**The SDK submits whatever is in the form; it does not validate it.** Its
submit handler ignores `required`, `pattern` and any check the theme runs
afterwards. Register the theme's validation on the form's `submit` event
**before** calling `attachLeadForm`, `attachRegistrationForm` or
`attachCustomerFieldsForm`, and stop the event when the form is invalid:

```js
form.addEventListener("submit", (e) => {
  if (!form.checkValidity()) {
    e.preventDefault();
    e.stopImmediatePropagation();   // keeps the SDK's handler from running
    form.reportValidity();
  }
});
MVQore.attachLeadForm(form, options);  // attached second, so it runs second
```

**Keep option values exactly as the merchant defined them.** When a
`mvqore_form` field has a list of choices, the server compares each answer to
that list character for character, including case. On a translated storefront,
translate the label and leave the `value` attribute alone:
`<option value="clinic">{{ 'apply.clinic' | t }}</option>`. A translated value
is rejected with `INVALID_CHOICE`.

**Mirror the merchant's field rules in the markup.** A metafield definition can
carry its own rules in Shopify: a minimum or maximum, a length limit, a pattern.
The SDK cannot see them. When an answer breaks one, the logged-in form fails
with `SHOPIFY_REJECTED` and Shopify's message, but a lead or Create Account
form fails with the generic `LEAD_REJECTED` / `REGISTRATION_REJECTED` and does
not say which field. Put the same rules on the inputs (`min`, `max`,
`maxlength`, `pattern`) so the theme's validation catches them first.

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

**Tags depend on the store's allowlist** (MV Qore admin → More → Application
form). Until the merchant saves a list:

- lead forms accept any tag, as they always have
- Create Account forms ignore `tags`
- `attachCustomerFieldsForm` rejects every tag

Once a list is saved, every form rejects a tag that is not on it with
`TAG_NOT_ALLOWED`. Before relying on a tag, confirm it is on the list.

**A blank optional answer never erases a stored one.** Empty `mvqore_form`
inputs are skipped rather than saved as empty. A resubmission overwrites the
previous answers, except for fields the merchant marked "keep existing value".

## Error codes

Every function rejects with `{ code, message }`, plus `field` when one input
caused it. `getStatus`, `getSettings` and `getBotProtection` never reject — they
resolve to a safe value instead.

| Code | Meaning |
|---|---|
| `INVALID_REFERRER_CODE` | No referrer matches that code |
| `RATE_LIMITED` | Too many validation attempts; back off |
| `REFERRER_DATA_NOT_FOUND` | Referrer exists but has no usable record |
| `UNKNOWN_ERROR` | `validateReferrer` failed and the server gave no reason; treat as "try again" |
| `MISSING_EMAIL` | Lead submitted with no email |
| `LEAD_TOO_FAST` | Submitted under 5s after render (see above) |
| `REGISTRATION_TOO_FAST` | Same, for a Create Account form |
| `REFERRER_REQUIRED` | No referrer to attribute the lead to; pass `requireReferrer: false` to allow |
| `EMAIL_IN_USE` | Already a customer — show this on the email field |
| `PHONE_IN_USE` | Phone belongs to another customer — show this on the phone field |
| `LEAD_REJECTED` | Rejected server-side: bot guard, captcha, or Shopify refusing the new customer (see `SHOPIFY_REJECTED`) |
| `REGISTRATION_REJECTED` | Same, for a Create Account form |
| `FIELDS_TOO_FAST` | Same, for `attachCustomerFieldsForm` |
| `UNKNOWN_FIELD` | A `mvqore_form.<key>` input has no metafield definition; `field` names it |
| `INVALID_VALUE` | An answer does not fit its field's type (not a number, not a date…) |
| `INVALID_CHOICE` | An answer is not one of the field's allowed choices |
| `VALUE_TOO_LONG` | Over 255 characters (single line), 5,000 (multi-line), or 50 list items |
| `UNSUPPORTED_TYPE` | The field's type is one forms cannot write, such as a file |
| `TOO_MANY_FIELDS` | More than 20 `mvqore_form` fields in one submission |
| `TAG_NOT_ALLOWED` | A tag is not on the store's allowlist |
| `NOTHING_TO_SUBMIT` | A logged-in application had no answers and no tags |
| `NOT_LOGGED_IN` | `attachCustomerFieldsForm` could not identify a logged-in customer. Almost always a visitor who is not signed in; ask them to log in |
| `SHOPIFY_REJECTED` | `attachCustomerFieldsForm` only: Shopify refused an answer, e.g. over a min/max the merchant set on the field. On a lead or Create Account form the same problem arrives as `LEAD_REJECTED` / `REGISTRATION_REJECTED`, with no `field` |
| `TRY_AGAIN` | Shopify was briefly unavailable; resubmitting is safe |
| `FIELDS_REJECTED` | Any other failure from `attachCustomerFieldsForm` |
| `INVALID_SHARE_LINK` | `share_cart` payload malformed |
| `CART_ADD_FAILED` | Shopify refused the shared line items |
| `FAVORITES_UNAVAILABLE` | Favorite products could not be loaded |
| `NETWORK_ERROR` | Request never completed |

## Full API

See [reference/api.md](reference/api.md) for every function, argument and return
shape, and [reference/recipes.md](reference/recipes.md) for complete, working
section examples you can adapt.

## Personalization

To show or tailor content by customer tag, by a customer's saved application
answers, or by who referred the visitor, read
[reference/personalization.md](reference/personalization.md) first. It covers
which of those Liquid can see and which only the SDK can, and why a price shown
in the theme is not a discount.
