# MV Qore SDK — working examples

These reproduce the **app embed's structure and styles**, so a custom theme built
on the SDK looks the same as a store running the MV Qore app embeds. Keep the
MV Qore class names and ids exactly as written — `mvqore.css` targets them. Place
your own layout around these blocks rather than renaming what is inside them.

## Required assets

Every page using MV Qore markup needs the stylesheet, the icon font it references
and the SDK:

```liquid
<link rel="stylesheet" href="{{ routes.root }}apps/proxy/sdk/v1/mvqore.css">
<link rel="stylesheet" href="{{ routes.root }}apps/proxy/mv-qore/fa/css/all.min.css">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap">
<script src="{{ routes.root }}apps/proxy/sdk/v1/mvqore-sdk.js" defer></script>
```

**The ids below are singletons** (`#simple-modal`, `#referrerInput`,
`#referrerModalHandler`). Render each block **once per page** — a section used
twice produces duplicate ids and the second copy will not work.

---

## 1. Referrer toolbar with code modal

`sections/mvqore-referrer-bar.liquid`

```liquid
{%- assign brand = shop.metafields.mvqore.brand_color.value | default: '#000000' -%}

<div class="referrer-toolbar-handler">
  <p class="header-text text-center">
    <span id="referrerModalHandler"
          class="handle-refferer cursor-pointer text-underline"
          style="color: {{ brand | escape }}; text-align: center;"></span>
  </p>
</div>

<div id="simple-modal" class="cus_modal referrer-code-modal">
  <div class="modal-dialog modal-dialog-centered mxWdt_380">
    <div id="referrerFormBlock" class="modal-content">
      <div class="modal-header">
        <h2 class="modal_title" id="modal-title">Enter your referrer code</h2>
        <button type="button" id="close-modal" class="modal_btn_close" aria-label="Close">
          <i class="fa-regular fa-xmark"></i>
        </button>
      </div>
      <div class="modal-body pb_5">
        <h4 id="referrerSubtitle" class="label_title"></h4>
        <div class="modal_form mxwdt_220">
          <input id="referrerInput" type="text" aria-label="Referrer Code Input">
          <p id="referrerInputErrorText" class="error_text mb_2" style="display: none"></p>
          <button class="button_blue mxwdt_220" id="submitButton">Submit</button>
        </div>
        <p id="referrerErrorText" class="error_text mb_5 pb_1" style="display: none"></p>
      </div>
    </div>
  </div>
</div>

<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({
      shop: {{ shop.permanent_domain | json }},
      locale: {{ request.locale.iso_code | default: 'en' | json }}
    });

    const handler = document.getElementById("referrerModalHandler");
    const modal   = document.getElementById("simple-modal");
    const input   = document.getElementById("referrerInput");
    const error   = document.getElementById("referrerErrorText");

    const status = await MVQore.getStatus();
    if (!status.installed || !status.configured) {
      document.querySelector(".referrer-toolbar-handler").style.display = "none";
      return;
    }

    const settings = await MVQore.getSettings();

    async function paint() {
      const active = await MVQore.getActiveReferrer();
      handler.innerHTML = active
        ? MVQore.formatToolbarTitle(active.referrer.name, settings)
        : MVQore.getReferrerPrompt(settings, {{ shop.name | json }});
    }

    // The embed opens the modal by class, and locks scrolling on the body.
    const openModal  = () => { modal.classList.add("modal_show"); document.body.classList.add("scroll-lock"); };
    const closeModal = () => { modal.classList.remove("modal_show"); document.body.classList.remove("scroll-lock"); };

    handler.addEventListener("click", openModal);
    document.getElementById("close-modal").addEventListener("click", closeModal);

    document.getElementById("submitButton").addEventListener("click", async () => {
      error.style.display = "none";
      try {
        const result = await MVQore.validateReferrer(input.value);
        MVQore.persistReferrer(result);
        closeModal();
        paint();
      } catch (e) {
        error.textContent =
          e.code === "INVALID_REFERRER_CODE" ? "We couldn't find that code."
          : e.code === "RATE_LIMITED"        ? "Too many attempts. Try again in a minute."
          : "Something went wrong. Please try again.";
        error.style.display = "block";
      }
    });

    input.addEventListener("keydown", (e) => {
      if (e.key === "Enter") { e.preventDefault(); document.getElementById("submitButton").click(); }
    });

    paint();
  });
</script>
```

---

## 2. Lead capture form with tags

Mirrors the embed's form: same wrappers, same field names, same button.

`sections/mvqore-lead-form.liquid`

```liquid
<section class="create-account__section mb_5">
  <div class="mxWdt_590 mb_5">
    <form action="javascript:void(0)" id="lead-capture-form">

      <div id="lead-form-fields">
        <div class="field_spaceBetween gap_3 pb_1">
          <input type="text" name="customer[first_name]" placeholder="First name" class="input_style" required>
          <input type="text" name="customer[last_name]"  placeholder="Last name"  class="input_style" required>
        </div>
        <div class="field_spaceBetween gap_3 pb_1">
          <input type="email" name="customer[email]" placeholder="Email" class="input_style" required>
        </div>
      </div>

      <p id="email-error-text" class="error_text mb_3" style="display: none; text-align:start;"></p>

      <div id="marketing-disclaimer-container" class="form_group my_3">
        <label class="inline_flex gap_2 font_14 cursor-pointer">
          <input type="checkbox" name="accepts_marketing" required>
          <span>Email me news and offers</span>
        </label>
      </div>

      <div id="lead-success-message" class="success-message-box" style="display: none;">
        Thanks — check your inbox.
      </div>

      <div class="form_group pt_5">
        <button style="min-height: 63px;" id="lead-capture-btn" type="submit" class="btn_circle_black">
          <span>Create account</span>
        </button>
      </div>
    </form>
  </div>
</section>

<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({ shop: {{ shop.permanent_domain | json }} });

    const form    = document.getElementById("lead-capture-form");
    const error   = document.getElementById("email-error-text");
    const success = document.getElementById("lead-success-message");

    const status = await MVQore.getStatus();
    if (!status.installed || !status.configured) { form.style.display = "none"; return; }

    // Mount the captcha widget when the merchant has bot protection enabled.
    const bot = await MVQore.getBotProtection();
    if (bot.enabled && bot.siteKey) mountFriendlyCaptcha(form, bot.siteKey);

    MVQore.attachLeadForm(form, {
      tags: ["quiz", "flavor-finder"],
      source: "find-your-flavor",
      onSuccess: () => { form.style.display = "none"; success.style.display = "block"; },
      onError: (e) => {
        error.textContent = {
          EMAIL_IN_USE: "You're already signed up with that address.",
          MISSING_EMAIL: "Please enter your email address.",
          LEAD_TOO_FAST: "Please take a moment before submitting.",
          REFERRER_REQUIRED: "Please enter a referrer code first.",
        }[e.code] || "Something went wrong. Please try again.";
        error.style.display = "block";
      },
    });
  });
</script>
```

The captcha widget must write its token into a field named `frc-captcha-response`
inside the form. Pass `requireReferrer: false` to accept leads with no referrer.

---

## 3. Multi-step quiz (no form element)

```js
const session = MVQore.createLeadSession({ tags: ["quiz"], source: "flavor-quiz" });
// Created when the quiz opens — the anti-bot timer starts here, so the time the
// user spends answering counts toward it.

try {
  await session.submit({ email: answers.email, firstName: answers.name });
  showResults();
} catch (e) {
  showError(e.code === "EMAIL_IN_USE" ? "You're already with us" : "Try again");
}
```

---

## 4. Share cart button with QR

```liquid
<div class="d_flex share-cart-message">
  <button type="button" id="mvq-share" class="button_blue mxwdt_220">Share my cart</button>
</div>
<div id="mvq-qr" class="text-center mt_5"></div>
<p class="text-center"><a id="mvq-share-link" class="color_gray text-underline" style="display:none">Copy link</a></p>

<script>
  document.addEventListener("DOMContentLoaded", async () => {
    MVQore.init({ shop: {{ shop.permanent_domain | json }} });

    const status = await MVQore.getStatus();
    if (!status.installed) return;

    document.getElementById("mvq-share").addEventListener("click", async () => {
      const active = await MVQore.getActiveReferrer();
      const share  = await MVQore.generateShareUrl({ code: active?.referrer.code });
      if (!share) { document.getElementById("mvq-share").textContent = "Your cart is empty"; return; }

      MVQore.generateQRCode(share.url, { size: 240, target: "#mvq-qr" });
      const link = document.getElementById("mvq-share-link");
      link.href = share.url;
      link.style.display = "inline";
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
    MVQore.persistReferrer(await MVQore.validateReferrer(result.referrerCode));
  }
  MVQore.openCartDrawer();
} else if (result.reason === "NO_SHARE_LINK") {
  showMessage("No shared cart found.");
}
```

`importSharedCart` replaces the current cart by default so the link reproduces the
sender's cart exactly; pass `{ clear: false }` to merge instead. It never
redirects — that stays the theme's decision.
