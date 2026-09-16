# MV Qore skills

Claude Code plugins for building Shopify themes on [MV Qore](https://mvqore.com).

## Install

```
/plugin marketplace add marketvision/mvqore-skills
/plugin install mvqore-sdk@mvqore-skills
```

Updates propagate automatically — you do not need to reinstall.

## What you get

`mvqore-sdk` teaches Claude Code how to build MV Qore referral features into a
custom Shopify theme: referrer toolbars, lead capture forms with customer tags,
cart sharing, QR codes and referrer attribution. Ask in plain language, e.g.:

- "Add an MV Qore referrer toolbar to the header"
- "Put an MV Qore lead form in this section, tagged `quiz` and `flavor-finder`"
- "Add a share-this-cart button with a QR code"

## Requirements

The MV Qore app must be installed on the store. The SDK is served by the app
itself, so nothing is copied into the theme and no API key is needed.

## Contributing

This repository is public and contains no MarketVision credentials, customer
data or private source. Keep it that way: the SDK it documents is served from
the app, and the only URLs here are public storefront paths.
