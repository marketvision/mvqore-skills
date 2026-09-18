# Security

## Reporting a vulnerability

Please do not open a public issue for a security problem. Report it privately
through GitHub instead: open this repository's **Security** tab and choose
**Report a vulnerability**. That creates a private advisory visible only to the
maintainers.

This applies to the documentation here and to the MV Qore storefront SDK it
describes (`/apps/proxy/sdk/v1/mvqore-sdk.js`).

## What this repository contains

Documentation only: a Claude Code skill, its API reference and example theme
code. It holds no credentials, customer data or private source, and none should
ever be added. The SDK is served by the MV Qore app through Shopify's App Proxy,
which signs every request server-side, so theme code never needs an API key or
secret — if an example here ever appears to need one, treat that as a bug and
report it.
