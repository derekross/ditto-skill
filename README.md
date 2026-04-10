# Ditto Skill for Shakespeare

A [Shakespeare](https://soapbox.pub/shakespeare/) plugin that teaches AI to build Nostr applications using the [Ditto](https://ditto.pub) design philosophy.

## What's Included

### `ditto` skill

Guides the AI to implement:

- **Avatar Shapes** — Emoji-based avatar masks stored in kind 0 metadata (`shape` field, [NIP-24 extension](https://github.com/nostr-protocol/nips/pull/2268))
- **Profile Themes** — User-customizable themes using kind 36767 (theme definitions) and kind 16767 (active profile theme)
- **Ditto Design Language** — Circles over boxes, endless customization, profiles as planets, the arcane aesthetic

## Install

### Via Shakespeare UI

1. Go to **Settings > AI**
2. Click **Add Plugin**
3. Enter: `https://github.com/nicksoapbox/ditto-skill.git`

### Via Discover

If you follow the publisher on Nostr, the plugin will appear in the **Discover Plugins** dialog.

## Publishing to Nostr (NIP-34)

To make this plugin discoverable via Shakespeare's Discover feature, publish a kind 30617 event:

```bash
nak event \
  --kind 30617 \
  -d ditto-skill \
  --tag name="Ditto Skill" \
  --tag description="Design Nostr apps with the Ditto philosophy — themes, avatar shapes, and visual language" \
  --tag clone=https://github.com/nicksoapbox/ditto-skill.git \
  --tag web=https://github.com/nicksoapbox/ditto-skill \
  --tag t=shakespeare-plugin \
  wss://relay.ditto.pub wss://relay.mostr.pub wss://nos.lol
```

## Learn More

- [Ditto Philosophy](https://about.ditto.pub/philosophy)
- [Shakespeare Plugins Documentation](https://soapbox.pub/shakespeare/)
- [NIP-34: Git Repository Announcements](https://github.com/nostr-protocol/nips/blob/master/34.md)
