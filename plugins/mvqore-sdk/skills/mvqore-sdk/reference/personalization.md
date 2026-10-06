# MV Qore SDK — personalization

Tailoring what a visitor sees to who they are and who referred them. There is no
`MVQore.personalize()`: personalization is built from two sources that already
exist, and the work is knowing which one to read.

| What you know | Where it lives | Read it with |
|---|---|---|
| The logged-in customer, their tags and saved answers | Shopify, server-side | Liquid |
| The visitor's referrer | The visitor's browser | The SDK, in JavaScript |

**Customer context is Liquid; referrer context is JavaScript.** Liquid runs on
Shopify's servers and cannot see the referrer, because the referrer is stored in
the browser. JavaScript cannot see customer tags unless Liquid prints them. Pick
the side that owns the data before writing any markup.

---

## Customer tags

A tag is the usual switch: `VIP`, `wholesale`, the merchant's Create Account tag,
`mvqore_lead` on every lead, or a tag an application form added.

```liquid
{% if customer and customer.tags contains 'VIP' %}
  <div class="vip-banner">{{ 'vip.banner' | t }}</div>
{% endif %}
```

`customer.tags` is a list, so `contains` matches a whole tag, case-sensitively:
`'VIP'` does not match `vip` or `VIP-gold`. Ask which exact tag the merchant
uses instead of guessing its spelling.

Always test `customer` first. For a visitor who is not signed in, `customer` is
`nil` and the block should simply not render.

To use a tag in JavaScript, print it from Liquid as data. Never as a class name
or a string you concatenate:

```liquid
<script type="application/json" data-customer-context>
  {
    "loggedIn": {% if customer %}true{% else %}false{% endif %},
    "tags": {{ customer.tags | default: '' | json }}
  }
</script>
```

Print only what the page needs. Do not dump the whole `customer` object: email,
phone and addresses have no business in page source that analytics and
third-party scripts can read.

---

## A tag-based price is a display, not a discount

**Showing a lower price in the theme does not change what checkout charges.**
The theme draws the page; Shopify prices the order. A VIP section that prints
"20% off" with no matching discount in Shopify sends the customer to a checkout
at full price.

So a "VIP discount" is two pieces, and the theme is only the second:

1. **The merchant creates the discount in Shopify**, limited to the right
   customers: an automatic discount or a code restricted to a customer segment
   (for example `customer_tags CONTAINS 'VIP'`). This is what changes the price.
2. **The theme shows it** to the customers who qualify.

Before building, ask how the discount is set up, then match the display to it:

- **Automatic discount for the segment:** show the saving. Compute it from the
  product's real price with the same percentage or amount the merchant
  configured, and label it as applied at checkout, since the cart may show full
  price until then on some themes.
- **Discount code:** show the code and a button that applies it. Linking to
  `{{ routes.root }}discount/CODE?redirect={{ request.path | url_encode }}`
  applies the code to the session and returns the customer to the page.
- **Nothing set up yet:** say so and stop. Do not ship a section that advertises
  a price the checkout will not honour.

Never hide a product, a price or a page behind a tag check in the theme and call
it access control. Theme code is public and a product URL still works. Shopify's
own controls (segment-restricted discounts, B2B catalogs, password or
customer-account requirements) are what enforce who gets what.

---

## Saved application answers

Answers an application form saved are customer metafields in the `mvqore_form`
namespace, readable in Liquid once the customer is logged in:

```liquid
{% assign level = customer.metafields.mvqore_form.experience_level.value %}
{% if level == 'advanced' %}
  …
{% endif %}
```

Use `.value`, not the metafield itself, so list and number fields come back as
their real type. A field the customer never answered is `nil`; design the
default case first and treat the personalized one as the addition.

Compare against the option **values** the merchant defined, not their
translated labels (see "Keep option values exactly as the merchant defined
them" in SKILL.md).

---

## The referrer

Who referred the visitor is known only in the browser, so referrer-based content
is always painted by JavaScript after the page loads.

```js
const status = await MVQore.getStatus();
if (!status.installed || !status.configured) return;

const active = await MVQore.getActiveReferrer();
if (!active) return;                       // nobody attributed: keep the default

const { name, imageUrl, code } = active.referrer;
heading.textContent = `Picked for you by ${name}`;   // textContent, not innerHTML
if (imageUrl) avatar.src = imageUrl;
```

Render the default, non-personalized version in Liquid and let the script
upgrade it. Reserve the space the personalized version will take, so the page
does not jump when it arrives, and never leave an empty box behind when there is
no referrer.

Ready-made referrer personalization the SDK already provides:

- **Toolbar copy** in the merchant's own words: `formatToolbarTitle()` and
  `getReferrerPrompt()` (SKILL.md, "Referrer toolbar").
- **The referrer's product picks:** `getFavoriteProducts()` (SKILL.md, "Favorite
  products").

### Referrer contact details

For a "contact your referrer" popup, the full server record carries more than
the flattened `referrer` view:

```js
const node = active.record?.node ?? {};
const email = node.email;          // may be missing
const phone = node.phoneNumber;    // may be missing
```

Both are optional and depend on what the referrer has on file, so build the
popup to work with either, both or neither, and hide a row instead of printing
an empty one. They are free text from another system: set them with
`textContent`, and build links with `encodeURIComponent`, never by
concatenating into `innerHTML`.

```js
if (email) {
  emailLink.textContent = email;
  emailLink.href = `mailto:${encodeURIComponent(email)}`;
}
if (phone) {
  phoneLink.textContent = phone;
  phoneLink.href = `tel:${phone.replace(/[^\d+]/g, "")}`;
}
```

### Do not make decisions on the referrer in the browser

The stored referrer can be changed by anyone with the browser's developer tools
open. It is safe for display: a greeting, a photo, a product list. It is not
safe for anything with a consequence: a price, a discount, access to a page.
Attribution on the order is recorded by MV Qore server-side and does not depend
on what the theme shows.

---

## Combining both

A section that depends on the customer and the referrer renders in two steps:
Liquid decides whether the customer qualifies and prints the default, then the
script fills in the referrer's part.

```liquid
{% if customer and customer.tags contains 'VIP' %}
  <section class="vip-welcome" data-vip-welcome>
    <h2 data-vip-heading>{{ 'vip.welcome' | t: name: customer.first_name }}</h2>
  </section>
{% endif %}
```

```js
const section = document.querySelector("[data-vip-welcome]");
if (section) {                              // absent for everyone else
  const active = await MVQore.getActiveReferrer();
  if (active) {
    const note = document.createElement("p");
    note.textContent = `Your contact is ${active.referrer.name}`;
    section.append(note);
  }
}
```

The script checks that the section exists rather than repeating the tag test:
Liquid has already decided, and one place to decide is one place to get wrong.

---

## Checklist

- Signed out, signed in without the tag, signed in with the tag: all three look
  right.
- No referrer, and a referrer with no photo, email or phone: no gaps or empty
  rows.
- Any price or saving shown is backed by a discount that exists in Shopify, and
  checkout shows the same number.
- No customer email, phone or address is printed into the page source.
- With the MV Qore app not installed (`getStatus()` reports `installed: false`),
  the page still renders its default content.
