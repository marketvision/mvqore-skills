# MV Qore skills

Claude Code plugins for building Shopify themes on MV Qore App.

## Install

```
/plugin marketplace add marketvision/mvqore-skills
/plugin install mvqore-sdk@mvqore-skills
```

### Turn on auto-update

Claude Code does **not** auto-update third-party marketplaces by default, so
without this step you stay on the version you first installed. Enable it once:

1. Run `/plugin` and open the **Marketplaces** tab
2. Select `mvqore-skills`
3. Choose **Enable auto-update**

Or update by hand whenever you like:

```
/plugin marketplace update mvqore-skills
```

Updates are fetched in the background after Claude Code starts; run
`/reload-plugins` when prompted to use the new version.

## What you get

`mvqore-sdk` teaches Claude Code how to build MV Qore referral features into a
custom Shopify theme: referrer toolbars, lead capture forms with customer tags,
cart sharing, QR codes and referrer attribution. Ask in plain language, e.g.:

- "Add an MV Qore referrer toolbar to the header"
- "Put an MV Qore lead form in this section, tagged `lead` and `spring-promo`"
- "Add a share-this-cart button with a QR code"

## Requirements

The MV Qore app must be installed on the store. The SDK is served by the app
itself, so nothing is copied into the theme and no API key is needed.

## Security

Please report vulnerabilities privately — see [SECURITY.md](SECURITY.md).

## Releasing

1. Update the skill under `plugins/mvqore-sdk/skills/mvqore-sdk/`
2. Bump `version` in `plugins/mvqore-sdk/.claude-plugin/plugin.json` **and**
   `.claude-plugin/marketplace.json`
3. Add an entry to [CHANGELOG.md](CHANGELOG.md)
4. Run `claude plugin validate .` before pushing

## Contributing

This repository is public and contains no MarketVision credentials, customer
data or private source. Keep it that way: the SDK it documents is served from
the app, and the only URLs here are public storefront paths.

## License

[MIT](LICENSE)
