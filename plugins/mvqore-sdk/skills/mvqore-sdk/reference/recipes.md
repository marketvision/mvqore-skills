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

## 3. Share cart button with QR

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

## 4. Receiving a shared cart

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

---

## 5. Application form with custom customer fields

`sections/mvqore-application.liquid`. This is one section for both new and
logged-in applicants. The answers are saved to `mvqore_form` customer
metafields, and the tag can trigger a Shopify Flow that emails the application
to staff.

It assumes the merchant has already created these customer metafield
definitions in Shopify admin (Settings → Custom data → Customers) and added the
tag to the allowlist in MV Qore admin (More → Application form):

| Definition | Type |
|---|---|
| `mvqore_form.license_number` | Single line text |
| `mvqore_form.years_experience` | Integer |
| `mvqore_form.specialties` | List of single line text, with choices |
| `mvqore_form.about` | Multi-line text |

```liquid
{%- capture application_fields -%}
  <label>Licence number
    <input name="mvqore_form.license_number" required>
  </label>
  <label>Years of experience
    <input name="mvqore_form.years_experience" type="number" min="0" step="1">
  </label>
  <fieldset>
    <legend>Specialties</legend>
    <label><input type="checkbox" name="mvqore_form.specialties" value="keto"> Keto</label>
    <label><input type="checkbox" name="mvqore_form.specialties" value="fitness"> Fitness</label>
  </fieldset>
  <label>About your practice
    <textarea name="mvqore_form.about" maxlength="5000"></textarea>
  </label>
  <p class="mvq-apply__consent">
    We store these answers on your customer account to review your application.
  </p>
{%- endcapture -%}

<div class="mvq-apply" data-mvq-apply hidden>
  {% if customer %}
    <form data-mvq-apply-form="existing" novalidate>
      <p>Applying as {{ customer.email | escape }}</p>
      {{ application_fields }}
      <button type="submit">Submit application</button>
      <p class="mvq-apply__error" data-mvq-apply-error hidden></p>
    </form>
  {% else %}
    <form data-mvq-apply-form="new" novalidate>
      <input type="text"  name="first_name" placeholder="First name" autocomplete="given-name" required>
      <input type="text"  name="last_name"  placeholder="Last name"  autocomplete="family-name" required>
      <input type="email" name="email"      placeholder="Email"      autocomplete="email" required>
      {{ application_fields }}
      <button type="submit">Submit application</button>
      <p class="mvq-apply__error" data-mvq-apply-error hidden></p>
    </form>
  {% endif %}
  <p class="mvq-apply__thanks" data-mvq-apply-thanks hidden>
    Thanks. We'll review your application and be in touch.
  </p>
</div>

<script>
  document.addEventListener("DOMContentLoaded", async () => {
    const root   = document.querySelector("[data-mvq-apply]");
    const form   = root.querySelector("[data-mvq-apply-form]");
    const error  = root.querySelector("[data-mvq-apply-error]");
    const thanks = root.querySelector("[data-mvq-apply-thanks]");

    const status = await MVQore.getStatus();
    if (!status.installed || !status.configured) return;  // leave it hidden
    root.hidden = false;

    const MESSAGES = {
      EMAIL_IN_USE: "You already have an account. Please log in to apply.",
      NOT_LOGGED_IN: "Your session ended. Please log in again.",
      TRY_AGAIN: "We couldn't save your application just now. Please try again.",
      RATE_LIMITED: "Too many attempts. Please wait a minute.",
      INVALID_VALUE: "Please check this answer.",
      INVALID_CHOICE: "Please pick one of the listed options.",
      VALUE_TOO_LONG: "This answer is too long.",
    };

    function showError(e) {
      form.querySelectorAll("[aria-invalid]").forEach((el) => el.removeAttribute("aria-invalid"));
      // `field` is the metafield key; mark the inputs that feed it.
      if (e.field && e.field !== "tags") {
        form.querySelectorAll(`[name="mvqore_form.${e.field}"]`)
          .forEach((el) => el.setAttribute("aria-invalid", "true"));
      }
      error.textContent = MESSAGES[e.code] || "Something went wrong. Please try again.";
      error.hidden = false;
    }

    // Registered before the SDK attaches, so it runs first. The SDK does not
    // validate: without this, an incomplete form is posted as it stands.
    form.addEventListener("submit", (e) => {
      if (!form.checkValidity()) {
        e.preventDefault();
        e.stopImmediatePropagation();
        form.reportValidity();
      }
    });

    const options = {
      tags: ["mvqore-application-submitted"],
      onSuccess: () => { form.hidden = true; thanks.hidden = false; },
      onError: (e) => {
        showError(e);
        // A captcha token works once, and this attempt used it.
        resetFriendlyCaptcha(form);
      },
    };

    if (form.dataset.mvqApplyForm === "existing") {
      // Updates the logged-in customer; creates nothing.
      MVQore.attachCustomerFieldsForm(form, options);
    } else {
      // Creates the customer with the answers and the tag in one step.
      // Mount the captcha first if the merchant has bot protection on.
      const bot = await MVQore.getBotProtection();
      if (bot.enabled && bot.siteKey) mountFriendlyCaptcha(form, bot.siteKey);
      MVQore.attachLeadForm(form, { ...options, source: "application-form", requireReferrer: false });
    }
  });
</script>
```

Notes:

- **`mountFriendlyCaptcha` and `resetFriendlyCaptcha` are yours to implement**
  with Friendly Captcha's own script. The reset matters: every submission that
  reaches the server uses up the token, even a rejected one.
- **Option values must match the definition's choices exactly.** `keto` and
  `fitness` here are the choices on `mvqore_form.specialties`. On a translated
  storefront, translate the label text, never the `value`.
- **`EMAIL_IN_USE` on the new-customer form means the applicant already has an
  account.** Ask them to log in, which switches them to the logged-in form. The
  server never adds answers to an existing customer based on an email typed into
  a form.
- **Use `attachRegistrationForm` instead of `attachLeadForm`** if a new applicant
  should get an account and be signed in. The fields and `tags` work the same
  way.
- **Keep the tag that triggers staff review separate from any tag that grants
  access or pricing.** The form adds the trigger tag only; staff, or a Flow
  after review, add the status tag.
- **Don't ask for sensitive data** such as ID numbers without the merchant's
  sign-off. Everything entered is stored on the customer record.
