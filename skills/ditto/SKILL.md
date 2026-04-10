---
name: ditto
description: Design Nostr applications using the Ditto philosophy — profile themes (kind 36767/16767), avatar shapes (emoji masks via NIP-24 shape field), and the Ditto visual language of circles, customization, and self-expression.
---

# Ditto Design Skill

This skill teaches you to build Nostr applications that follow the Ditto design philosophy and implement Ditto's user profile enhancements: **profile themes** and **avatar shapes**.

## Ditto Design Philosophy

Ditto rejects conventional boxy, grid-based interfaces. Its design language is built on these principles:

- **Circles over boxes.** "Boxes are prisons." Favor rounded, organic shapes. Avatars, containers, and interactive elements should feel alive, not caged.
- **Profiles are planets.** Each user's profile is a world to inhabit — fully customizable with colors, fonts, backgrounds, and avatar shapes. Treat the profile as a canvas for self-expression, not a standardized template.
- **Gamification over utility.** Fun interactions get "repost-level super actions." Reward engagement with playful, delightful behaviors.
- **The arcane aesthetic.** Settings channel the mystical. Nostr is personal mythology. The UI should feel magical, not clinical.
- **Saturn symbolism.** Controlled chaos — the balance between rigid structure and boundless creativity. Opinionated design meeting chaotic whimsy.
- **Endless customization.** Easter eggs, whimsical details, highly individualized visual worlds. Every user should feel their space is uniquely theirs.
- **Experiential onboarding.** New users encounter interactive content (games, creative experiences) before blockchain concepts. Engagement first, education later.

When someone asks to "make it look like Ditto", "use the Ditto philosophy", "use Ditto styling", or "use Ditto design language", apply these principles.

---

## Avatar Shapes

### What Are Avatar Shapes?

Avatar shapes allow users to set any emoji as the shape of their avatar, replacing the default circle. The emoji is used as a CSS mask over the avatar image.

### How Shapes Are Stored (NIP-24 Extension)

The `shape` property is added to kind 0 profile metadata (NIP-24):

```json
{
  "kind": 0,
  "content": "{\"name\":\"alex\",\"picture\":\"https://example.com/avatar.jpg\",\"shape\":\"🔷\"}"
}
```

Reference: https://github.com/nostr-protocol/nips/pull/2268

### Shape Validation Rules

- Must be a single emoji (non-ASCII, short string under 20 characters)
- Detection: check for at least one non-ASCII character via `/[^\x00-\x7F]/`
- When absent or invalid, avatars render as circles (the default)

```typescript
function isEmoji(value: string): boolean {
  if (!value || value.length === 0) return false;
  if (value.length > 20) return false;
  return /[^\x00-\x7F]/.test(value);
}

function isValidAvatarShape(value: unknown): value is string {
  if (typeof value !== 'string' || value.length === 0) return false;
  return isEmoji(value);
}

function getAvatarShape(metadata: Record<string, unknown> | undefined): string | undefined {
  const raw = metadata?.shape;
  return isValidAvatarShape(raw) ? raw : undefined;
}
```

### Rendering Avatar Shapes (Emoji Mask Algorithm)

Convert the emoji into a PNG alpha mask for use as CSS `mask-image`:

1. **Draw large.** Render the emoji at 512px via `fillText` on an oversized (768x768) scratch canvas.
2. **Measure.** Scan pixels for the tight bounding box of non-transparent pixels (alpha threshold of 25/255 to ignore shadows).
3. **Square the crop.** Expand the shorter axis so the crop is square (centered), preventing stretching.
4. **Redraw.** Draw the squared crop onto a 256x256 output canvas.
5. **Convert to alpha mask.** Set every pixel to white RGB, keep original alpha. Export as PNG data-URL.
6. **Cache.** Store emoji-to-data-URL mapping in memory to avoid re-rendering.

```typescript
function getEmojiMaskUrl(emoji: string): string {
  // Check cache first
  const cached = cache.get(emoji);
  if (cached) return cached;

  const fontSize = 512;
  const scratch = fontSize * 1.5; // 768px
  const canvas = document.createElement('canvas');
  canvas.width = scratch;
  canvas.height = scratch;
  const ctx = canvas.getContext('2d');

  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.font = `${fontSize}px serif`;
  ctx.fillText(emoji, scratch / 2, scratch / 2);

  // ... measure bounding box, square it, redraw at 256x256,
  // convert to white+alpha mask, export as data URL ...

  const url = canvas.toDataURL('image/png');
  cache.set(emoji, url);
  return url;
}
```

### Applying the Shape Mask in CSS

When an avatar has a valid shape, apply the mask and remove the default `rounded-full`:

```tsx
// If the user has a shape, compute the mask URL
const maskUrl = shape ? getEmojiMaskUrl(shape) : '';

// Apply to the avatar container
<div
  className={cn(
    "relative flex shrink-0 overflow-hidden bg-muted",
    !maskUrl && "rounded-full" // Only round when no custom shape
  )}
  style={maskUrl ? {
    WebkitMaskImage: `url(${maskUrl})`,
    maskImage: `url(${maskUrl})`,
    WebkitMaskSize: 'contain',
    maskSize: 'contain',
    WebkitMaskRepeat: 'no-repeat',
    maskRepeat: 'no-repeat',
    WebkitMaskPosition: 'center',
    maskPosition: 'center',
  } : undefined}
>
  <img src={avatarUrl} className="absolute inset-0 h-full w-full object-cover" />
</div>
```

### Shape Border Effect

For shaped avatars, use `drop-shadow` filters instead of CSS `border` (which would be clipped by the mask):

```typescript
const shapedAvatarBorderStyle: React.CSSProperties = {
  filter:
    'drop-shadow(3px 0 0 hsl(var(--background)))' +
    ' drop-shadow(-3px 0 0 hsl(var(--background)))' +
    ' drop-shadow(0 3px 0 hsl(var(--background)))' +
    ' drop-shadow(0 -3px 0 hsl(var(--background)))',
};

// Wrap the masked avatar in a container with this style
<div style={shapedAvatarBorderStyle}>
  <Avatar shape={userShape}>...</Avatar>
</div>
```

### Editing the Shape (Profile Settings)

Allow users to set their avatar shape in profile settings. The shape is saved as part of the kind 0 metadata JSON:

```typescript
// Reading shape from metadata
const shape = getAvatarShape(metadata); // Returns emoji string or undefined

// Saving shape to metadata
const updatedMetadata = { ...metadata, shape: selectedEmoji };
// Publish kind 0 event with updated metadata
```

---

## Profile Themes

### Overview

Ditto's theme system lets every user customize their profile's visual appearance. When someone visits a profile, the visitor's UI temporarily transforms to show the profile owner's theme — colors, fonts, and background.

### Theme Architecture

The system uses **3 core colors** to derive all 19 Tailwind CSS tokens:

```typescript
interface CoreThemeColors {
  /** Background color (HSL string, e.g. "228 20% 10%") */
  background: string;
  /** Text/foreground color */
  text: string;
  /** Primary accent color (buttons, links, focus rings) */
  primary: string;
}
```

All other tokens (card, muted, border, destructive, etc.) are **algorithmically derived** from these 3 values. This is the key insight: you only need 3 colors to define an entire theme.

### Nostr Event Kinds

#### Kind 36767 — Theme Definition (Addressable)

A shareable, named theme. Users can publish multiple themes.

**Tags:**
| Tag | Format | Required | Description |
|-----|--------|----------|-------------|
| `d` | `['d', 'my-theme-slug']` | Yes | Unique identifier (URL-safe slug) |
| `c` | `['c', '#hex', 'role']` | Yes (x3) | Colors: roles are `background`, `text`, `primary` |
| `f` | `['f', 'Font Family', 'url', 'role']` | No | Fonts: roles are `body` or `title`. URL can be empty for system fonts |
| `bg` | `['bg', 'url ...', 'mode ...', 'm ...', 'dim ...', 'blurhash ...']` | No | Background media (imeta-style variadic) |
| `title` | `['title', 'My Theme']` | Yes | Human-readable name |
| `alt` | `['alt', 'Custom theme: My Theme']` | No | NIP-31 fallback text |
| `t` | `['t', 'theme']` | No | Topic tag for discoverability |
| `description` | `['description', '...']` | No | Optional description |

**Content:** empty string

**Example event:**
```json
{
  "kind": 36767,
  "content": "",
  "tags": [
    ["d", "midnight-galaxy"],
    ["c", "#14141e", "background"],
    ["c", "#e8eaf6", "text"],
    ["c", "#7c4dff", "primary"],
    ["f", "DM Sans", "", "body"],
    ["f", "Playfair Display", "https://fonts.example.com/playfair.woff2", "title"],
    ["bg", "url https://example.com/stars.jpg", "mode cover", "m image/jpeg", "dim 1920x1080"],
    ["title", "Midnight Galaxy"],
    ["alt", "Custom theme: Midnight Galaxy"],
    ["t", "theme"]
  ]
}
```

#### Kind 16767 — Active Profile Theme (Replaceable)

The user's currently active profile theme. One per user. This is what visitors see.

**Tags:** Same as kind 36767 except:
- No `d` tag (it's a replaceable event, not addressable)
- Optional `a` tag referencing the source theme definition: `['a', '36767:pubkey:identifier']`
- `alt` is `"Active profile theme"`

**Example event:**
```json
{
  "kind": 16767,
  "content": "",
  "tags": [
    ["c", "#14141e", "background"],
    ["c", "#e8eaf6", "text"],
    ["c", "#7c4dff", "primary"],
    ["f", "DM Sans", "", "body"],
    ["alt", "Active profile theme"],
    ["a", "36767:abc123pubkey:midnight-galaxy"]
  ]
}
```

### Color Format: HSL Strings

Internally, colors are stored as HSL strings without the `hsl()` wrapper:

```
"228 20% 10%"   // not "hsl(228, 20%, 10%)"
```

On Nostr events, colors are stored as hex in `c` tags and converted:

```typescript
// HSL string → hex (for publishing)
function hslStringToHex(hsl: string): string { /* ... */ }

// Hex → HSL string (for parsing)
function hexToHslString(hex: string): string { /* ... */ }
```

### Deriving All 19 Tokens from 3 Core Colors

The derivation algorithm determines light/dark mode from background luminance, then generates:

| Token | Derivation |
|-------|-----------|
| `background` | Core background color |
| `foreground` | Core text color |
| `card` | Lighten(bg) for dark mode, same as bg for light |
| `cardForeground` | Same as text |
| `popover` | Same as card |
| `popoverForeground` | Same as text |
| `primary` | Core primary color |
| `primaryForeground` | Text color (or white if contrast is sufficient) |
| `secondary` | Darken(bg) for light, lighten(bg) for dark |
| `secondaryForeground` | Same as text |
| `muted` | Darken(bg) for light, lighten(bg) for dark |
| `mutedForeground` | Desaturated, mid-luminance text |
| `accent` | Same as primary |
| `accentForeground` | Same as text |
| `destructive` | Standard red `0 72% 51%` |
| `destructiveForeground` | Near-white |
| `border` | Primary hue + reduced saturation |
| `input` | Same as border |
| `ring` | Same as primary |

### Applying CSS Custom Properties

Inject derived tokens as CSS custom properties on `:root`:

```css
:root {
  --background: 228 20% 10%;
  --foreground: 210 40% 98%;
  --card: 228 20% 12%;
  --card-foreground: 210 40% 98%;
  --primary: 258 70% 60%;
  --primary-foreground: 210 40% 98%;
  --secondary: 228 20% 18%;
  --muted: 228 20% 18%;
  --muted-foreground: 210 15% 55%;
  --border: 258 25% 30%;
  --ring: 258 70% 60%;
  /* ... etc */
}
```

Then use Tailwind classes: `bg-background`, `text-foreground`, `border-border`, `bg-primary`, etc.

### Displaying a Profile's Theme When Visiting

When a user visits someone's profile:

1. **Query** kind 16767 for the profile's pubkey (with 5s timeout)
2. **Check permission** — the visitor's `showCustomProfileThemes` setting gates this
3. **Resolve effective colors:**
   - Profile has a theme? Use it
   - Profile has no theme + visitor has custom theme? Fall back to system builtin (light/dark)
   - Otherwise: no theme override
4. **Derive tokens** from the 3 core colors via `deriveTokensFromCore()`
5. **Inject CSS** custom properties into a `<style>` element on `<html>`
6. **Load fonts** asynchronously (body font + title font). Fall back to Inter if missing
7. **Apply background** image if present:
   - `cover` mode: `background-size: cover; background-attachment: fixed;`
   - `tile` mode: `background-repeat: repeat; background-size: auto;`
8. **On navigation away**, restore the visitor's own theme via cleanup

```typescript
// Querying the active profile theme
function useActiveProfileTheme(pubkey: string | undefined) {
  const { nostr } = useNostr();

  return useQuery({
    queryKey: ['activeProfileTheme', pubkey],
    queryFn: async () => {
      const events = await nostr.query(
        [{ kinds: [16767], authors: [pubkey!], limit: 1 }],
        { signal: AbortSignal.timeout(5000) }
      );
      if (events.length === 0) return null;
      return parseActiveProfileTheme(events[0]);
    },
    enabled: !!pubkey,
    staleTime: 5 * 60 * 1000,
  });
}
```

**Key design pattern:** Theme application is scoped to the page navigation. When the visitor leaves the profile, their own theme is automatically restored. No global state mutation.

### Creating and Publishing Themes

To create a theme:

1. User picks 3 core colors (background, text, primary)
2. Optionally selects body font, title font, and background image
3. Preview is generated by deriving all 19 tokens in real-time

To publish:

```typescript
// Build tags for a theme definition (kind 36767)
function buildThemeDefinitionTags(
  identifier: string,  // URL-safe slug
  title: string,
  themeConfig: ThemeConfig,
  description?: string,
): string[][] {
  return [
    ['d', identifier],
    // 3 color tags (HSL → hex)
    ['c', hslStringToHex(themeConfig.colors.background), 'background'],
    ['c', hslStringToHex(themeConfig.colors.text), 'text'],
    ['c', hslStringToHex(themeConfig.colors.primary), 'primary'],
    // Font tags (if present)
    // Background tag (if present)
    ['title', title],
    ['alt', `Custom theme: ${title}`],
    ['t', 'theme'],
  ];
}

// Build tags for active profile theme (kind 16767)
function buildActiveThemeTags(
  themeConfig: ThemeConfig,
  sourceAuthor?: string,
  sourceIdentifier?: string,
): string[][] {
  return [
    ['c', hslStringToHex(themeConfig.colors.background), 'background'],
    ['c', hslStringToHex(themeConfig.colors.text), 'text'],
    ['c', hslStringToHex(themeConfig.colors.primary), 'primary'],
    ['alt', 'Active profile theme'],
    // Optional: reference source definition
    // ['a', `36767:${sourceAuthor}:${sourceIdentifier}`],
  ];
}
```

### Scoped Theme Rendering

For rendering theme previews or applying a theme to a specific section (not the whole page):

```tsx
function ScopedTheme({ colors, children }: { colors: CoreThemeColors; children: React.ReactNode }) {
  const tokens = deriveTokensFromCore(colors.background, colors.text, colors.primary);

  const cssVars = Object.fromEntries(
    Object.entries(tokens).map(([key, value]) => [toThemeVar(key), value])
  );

  return (
    <div style={cssVars}>
      {children}
    </div>
  );
}
```

### Theme Presets

Ditto ships with curated presets that demonstrate the philosophy:

| Name | Vibe | Key Choices |
|------|------|-------------|
| Pink | Soft, warm | Comfortaa font, floral background |
| Skater | Urban, bold | Rubik Maps font, skateboard background |
| Kawaii | Cute, playful | Cherry Bomb One font, pastel pinks |
| Grunge | Dark, edgy | Lacquer font, deep purples |
| Gamer | Neon, electric | Press Start 2P font, green-on-black |
| Galaxy | Cosmic, deep | DM Sans font, starfield background |
| Ocean | Calm, teal | Nunito font, beach background |
| MS Paint | Retro, nostalgic | Silkscreen font, desktop background |

Each preset provides colors, font, and optional background — showing that theming is about the full sensory experience, not just a color swap.

### Font System

- **Body font:** Applied globally via `html { font-family: ... !important; }`
- **Title font:** Applied via CSS custom property `--title-font-family` for profile display names only
- **Fallback stack:** `"Inter Variable", Inter, system-ui, sans-serif`
- **Loading:** Bundled fonts via `@fontsource/*`, remote fonts via `@font-face` injection
- Font URLs are required on Nostr events but optional locally for bundled fonts

### Background System

Two display modes:

```css
/* Cover mode — full-bleed, fixed position */
body {
  background-image: url("...");
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center;
  background-attachment: fixed;
}

/* Tile mode — repeating pattern */
body {
  background-image: url("...");
  background-repeat: repeat;
  background-size: auto;
}
```

Background metadata stored in the `bg` tag includes: URL, mode, MIME type, dimensions, and blurhash for progressive loading.

### Auto-Share

When `autoShareTheme` is enabled, any custom theme change automatically publishes to kind 16767, keeping the user's profile theme in sync without manual intervention.

---

## Implementation Checklist

When building a Ditto-style application, implement these features:

### Avatar Shapes
- [ ] Parse `shape` field from kind 0 metadata
- [ ] Validate shape is a valid emoji
- [ ] Generate emoji mask (canvas-based, with caching)
- [ ] Apply CSS `mask-image` to avatar elements
- [ ] Remove `rounded-full` when a custom shape is present
- [ ] Use `drop-shadow` for borders on shaped avatars
- [ ] Show shapes everywhere avatars appear (feed, DMs, sidebars, notifications, etc.)
- [ ] Allow users to edit their shape in profile settings

### Profile Themes
- [ ] Define `CoreThemeColors` type (background, text, primary as HSL strings)
- [ ] Implement token derivation (3 colors -> 19 CSS custom properties)
- [ ] Parse kind 16767 events (active profile theme)
- [ ] Parse kind 36767 events (theme definitions)
- [ ] Apply visitor theme override when viewing a profile
- [ ] Restore visitor's own theme on navigation away
- [ ] Support fonts (body + title) with fallback
- [ ] Support background images (cover + tile modes)
- [ ] Build theme creation UI (color picker + font selector + background upload)
- [ ] Publish kind 36767 (save theme)
- [ ] Publish kind 16767 (set active profile theme)
- [ ] Implement scoped theme rendering for previews

### Design Language
- [ ] Use rounded/circular shapes over sharp rectangles
- [ ] Enable deep customization for user profiles
- [ ] Prioritize fun, engaging interactions
- [ ] Apply mystical/arcane aesthetic to settings and navigation
- [ ] Use CSS custom properties for all theme colors
- [ ] Use Tailwind semantic classes (`bg-background`, `text-foreground`, etc.)
