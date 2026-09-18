# MV Qore SDK — working examples

Complete theme sections. Adapt the markup and class names to the theme's own
design system; the SDK calls are the part to keep.

---

## 0. Site-wide setup and attribution capture

`layout/theme.liquid` — once for the whole theme. Everything else assumes this is
in place.

```liquid
<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({
      shop: {{ shop.permanent_domain | json }},
      locale: {{ request.locale.iso_code | default: 'en' | json }}
    });

    // A referral link can land on ANY page, so this belongs in the layout rather
    // than in the toolbar section. getActiveReferrer() resolves the referrer but
    // does not store it; persistReferrer() is what writes localStorage and tags
    // the cart, which is what order attribution reads later.
    //
    // Gated on ?ref= deliberately: persistReferrer posts to /cart/update.js, so
    // calling it on every page load would add a cart write to every page view.
    if (new URLSearchParams(location.search).has("ref")) {
      try {
        const active = await MVQore.getActiveReferrer();
        if (active) MVQore.persistReferrer(active);
      } catch (e) {
        // An invalid ?ref= is not worth breaking the page over.
        console.warn("[MVQore] could not capture referrer:", e);
      }
    }
  });
</script>
```

Sections then use the already-loaded `MVQore` global and do not include their own
script tag. (Including one is safe — the SDK ignores a second load — but it costs
a second parse for nothing.)

---

## 1. Referrer toolbar with code modal

`sections/mvqore-referrer-bar.liquid` — displays the current referrer and lets a
visitor enter a code. Attribution from `?ref=` is captured in the layout
(recipe 0), not here.



```liquid
{%- assign brand = shop.metafields.mvqore.brand_color.value | default: '#000000' -%}

<div class="mvq-bar" data-mvq-bar hidden>
  <button type="button" class="mvq-bar__text" data-mvq-bar-text
          style="color: {{ brand | escape }}"></button>
</div>

<dialog class="mvq-modal" data-mvq-modal>
  <form method="dialog" data-mvq-code-form>
    <label for="mvq-code">Referrer code</label>
    <input id="mvq-code" name="code" required>
    <p class="mvq-modal__error" data-mvq-error hidden></p>
    <button value="cancel">Cancel</button>
    <button value="submit">Apply</button>
  </form>
</dialog>

<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({
      shop: {{ shop.permanent_domain | json }},
      locale: {{ request.locale.iso_code | default: 'en' | json }}
    });

    const bar   = document.querySelector("[data-mvq-bar]");
    const text  = document.querySelector("[data-mvq-bar-text]");
    const modal = document.querySelector("[data-mvq-modal]");
    const error = document.querySelector("[data-mvq-error]");

    const status = await MVQore.getStatus();
    if (!status.installed || !status.configured) return;  // leave the bar hidden

    const settings = await MVQore.getSettings();

    async function paint() {
      const active = await MVQore.getActiveReferrer();
      text.innerHTML = active
        ? MVQore.formatToolbarTitle(active.referrer.name, settings)
        : MVQore.getReferrerPrompt(settings, {{ shop.name | json }});
      bar.hidden = false;
    }

    text.addEventListener("click", () => modal.showModal());

    modal.querySelector("[data-mvq-code-form]").addEventListener("submit", async (e) => {
      if (e.submitter?.value !== "submit") return;
      e.preventDefault();
      error.hidden = true;
      try {
        const result = await MVQore.validateReferrer(e.target.code.value);
        MVQore.persistReferrer(result);
        modal.close();
        paint();
      } catch (err) {
        error.textContent = err.code === "INVALID_REFERRER_CODE"
          ? "We couldn't find that code."
          : err.code === "RATE_LIMITED"
            ? "Too many attempts. Try again in a minute."
            : "Something went wrong. Please try again.";
        error.hidden = false;
      }
    });

    paint();
  });
</script>
```

---

## 2. Lead capture form with tags

`sections/mvqore-lead-form.liquid`

```liquid
<form id="mvq-lead" class="mvq-lead" novalidate>
  <input type="text"  name="first_name" placeholder="First name" autocomplete="given-name">
  <input type="email" name="email" placeholder="Email" required autocomplete="email">
  <label>
    <input type="checkbox" name="accepts_marketing"> Email me news and offers
  </label>
  <button type="submit">Get my results</button>
  <p class="mvq-lead__error" data-mvq-lead-error hidden></p>
  <p class="mvq-lead__thanks" data-mvq-lead-thanks hidden>Thanks — check your inbox.</p>
</form>

<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({ shop: {{ shop.permanent_domain | json }} });

    const form   = document.getElementById("mvq-lead");
    const error  = document.querySelector("[data-mvq-lead-error]");
    const thanks = document.querySelector("[data-mvq-lead-thanks]");

    const status = await MVQore.getStatus();
    if (!status.installed || !status.configured) { form.hidden = true; return; }

    // Mount the captcha widget when the merchant has bot protection on.
    const bot = await MVQore.getBotProtection();
    if (bot.enabled && bot.siteKey) mountFriendlyCaptcha(form, bot.siteKey);

    MVQore.attachLeadForm(form, {
      tags: ["newsletter", "spring-promo"],
      source: "homepage-form",
      onSuccess: () => { form.hidden = true; thanks.hidden = false; },
      onError: (e) => {
        error.textContent = {
          EMAIL_IN_USE: "You're already signed up with that address.",
          MISSING_EMAIL: "Please enter your email address.",
          LEAD_TOO_FAST: "Please take a moment before submitting.",
          REFERRER_REQUIRED: "Please enter a referrer code first.",
        }[e.code] || "Something went wrong. Please try again.";
        error.hidden = false;
      },
    });
  });
</script>
```

The widget must write its token into a field named `frc-captcha-response` inside
the form; `mountFriendlyCaptcha` is yours to implement with Friendly Captcha's
own script.

To submit without an active referrer, pass `requireReferrer: false`.

---

## 3. Multi-step flow (no form element)

```js
const session = MVQore.createLeadSession({ tags: ["newsletter"], source: "multi-step-signup" });
// Created when the flow opens — the anti-bot timer starts here, so the time the
// user spends working through the steps counts toward it.

const collected = {};
// ...steps collect fields...

try {
  await session.submit({ email: collected.email, firstName: collected.name });
  showConfirmation();
} catch (e) {
  showError(e.code === "EMAIL_IN_USE" ? "You're already with us" : "Try again");
}
```

---

## 4. Share cart button with QR

```liquid
<button type="button" data-mvq-share hidden>Share my cart</button>
<div data-mvq-qr></div>
<a data-mvq-share-link hidden>Copy link</a>

<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({ shop: {{ shop.permanent_domain | json }} });

    const button = document.querySelector("[data-mvq-share]");
    const link   = document.querySelector("[data-mvq-share-link]");

    const status = await MVQore.getStatus();
    if (!status.installed) return;
    button.hidden = false;

    button.addEventListener("click", async () => {
      const active = await MVQore.getActiveReferrer();
      const share  = await MVQore.generateShareUrl({ code: active?.referrer.code });
      if (!share) { button.textContent = "Your cart is empty"; return; }

      MVQore.generateQRCode(share.url, { size: 240, target: "[data-mvq-qr]" });
      link.href = share.url;
      link.hidden = false;
    });
  });
</script>
```

---

## 5. Receiving a shared cart

On the page share links point at (`/pages/share-cart` by default):

```js
const result = await MVQore.importSharedCart();

if (result.imported) {
  if (result.referrerCode) {
    const validated = await MVQore.validateReferrer(result.referrerCode);
    MVQore.persistReferrer(validated);
  }
  MVQore.openCartDrawer();
} else if (result.reason === "NO_SHARE_LINK") {
  showMessage("No shared cart found.");
}
```

`importSharedCart` replaces the current cart by default so the link reproduces
the sender's cart exactly; pass `{ clear: false }` to merge instead. It never
redirects — that is the theme's decision.
