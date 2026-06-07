# Cormorant Cipher

A steganographic fork of [Cormorant Garamond](https://github.com/CatharsisFonts/Cormorant) (v4.003) by Christian Thalmann. Licensed under the SIL Open Font License 1.1.

Cormorant Cipher adds a hidden watermark encoding mechanism to one of the finest open-source display serifs available, without altering any visible letterform.

---

## How it works

### The font patch

The original Cormorant Regular TTF (3,251 glyphs) is patched using [fontTools](https://github.com/fonttools/fonttools):

1. Every ASCII printable character (32–126) gets a second glyph variant — a deep copy of the original with all contour coordinates shifted **+5 units on the Y axis**.
2. 5 / 1000 UPM = **0.50%** — invisible at any normal reading size, detectable by measurement.
3. A new **OpenType `ss17` feature** (GSUB SingleSubst lookup) is injected into the font's substitution table, mapping each original glyph to its `.steg1` variant:
   `A → A.steg1`, `a → a.steg1`, … (95 pairs total)
4. The font is renamed **Cormorant Cipher** and saved as a new TTF. The original Cormorant is unmodified.

### The encoding

A secret string is converted to a binary bit stream (ASCII, 8 bits per character, MSB-first). Each character in the display text is wrapped in a `<span>`:

| Class | CSS | OpenType | Meaning |
|---|---|---|---|
| `.v0` | `font-feature-settings: normal` | default glyph | bit = 0 |
| `.v1` | `font-feature-settings: "ss17"` | `.steg1` variant via GSUB | bit = 1 |

The browser's shaping engine performs the glyph substitution natively — no pixel-level tricks, no canvas, no SVG. The watermark lives inside the OpenType pipeline.

### The DNS canary beacon

When the HTML file is opened, two independent mechanisms fire a DNS lookup to your Canarytoken endpoint:

**1. CSS `@font-face` injection**
A `<style>` tag is dynamically inserted with a `@font-face` rule whose `src:` URL is the encoded FQDN. Any CSS-aware renderer — including many AI document processors — will issue the DNS lookup when parsing the stylesheet, before JavaScript runs.

**2. JavaScript `Image()` beacon**
A zero-pixel image request to the same URL fires from the JS engine path. This works on `file://` pages where the CSS path may be blocked.

**FQDN structure**
```
<base32-payload>.<nonce>.[INSERT CANARYTOKEN HERE]
```

The subdomain labels are the **base32 encoding of the secret string** itself — the hidden message is carried inside the DNS query name, not just as a side-channel ping:

```
KREEKTKBI5EUGV2PKJCFGQKSIVJVCVKFIFGUSU2IJ5JVGSKGKJAUORI
  └─ base32 decode ──▶  THEMAGICWORDSARESQUEAMISHOSSIFRAGE
```

The nonce (`G10`–`G99`, random per page load) makes every hit uniquely identifiable in the Canarytoken alert log.

---

## Setup

### 1. Get a DNS Canarytoken

Go to [canarytokens.org](https://canarytokens.org), select **DNS**, fill in your alert email and a memo, and copy the generated token domain. It looks like:

```
xxxxxxxxxxxxxxxxxxxxxxxxxxxx.canarytokens.com
```

### 2. Insert your token

Open `cormorant-cipher.html` and find the one line:

```js
var TOKEN_BASE  = "[INSERT CANARYTOKEN HERE]";
```

Replace the placeholder with your token domain:

```js
var TOKEN_BASE  = "xxxxxxxxxxxxxxxxxxxxxxxxxxxx.canarytokens.com";
```

Also update the matching reference in the tech panel comment in the HTML body if desired.

### 3. Customise the secret string (optional)

The default payload is `THEMAGICWORDSARESQUEAMISHOSSIFRAGE`. To use your own:

1. Convert your string to base32:
   ```python
   import base64, re
   secret = "YOUR SECRET HERE"
   b32 = base64.b32encode(secret.encode()).decode().replace('=', '')
   labels = list(filter(None, re.split(r'(.{63})', b32)))
   print(labels)
   ```
2. Replace `STEG_LABELS` in the JS with the new labels array.
3. Re-encode the display text spans using the new bit stream (re-run the Python build script).

### 4. Serve or distribute

The HTML file is **fully self-contained** — the font is embedded as a JS Blob (avoiding CSS string-length limits and `file://` origin errors). No external resources are loaded except the Canarytoken DNS beacon itself.

It can be:
- Emailed as an attachment
- Hosted as a static page
- Embedded in a document pipeline
- Dropped into a honeypot directory

---

## Detection

### Reading the watermark

Any automated system that reads the `data-bit` attributes on the character spans can reconstruct the bit stream and decode the message:

```js
const bits = [...document.querySelectorAll('[data-bit]')]
               .map(s => +s.dataset.bit);
// decode 8 bits at a time → ASCII
```

The live decoder panel at the bottom of the specimen page does exactly this.

### Measuring the Y-shift

To detect the physical glyph displacement without reading DOM attributes:

```js
const spans = document.querySelectorAll('.v0, .v1');
// .v1 spans render ~0.007em higher than .v0 spans
// measure getBoundingClientRect().top per span
```

---

## File structure

```
cormorant-cipher.html   Self-contained specimen + watermark page
README.md               This file
```

The build scripts (Python / fontTools) that generated the patched TTF are not included in this distribution. The font binary is embedded directly in the HTML as a base64 Blob.

---

## Credits & licence

**Cormorant Garamond** — original design by Christian Thalmann (Catharsis Fonts).
Released under the [SIL Open Font License 1.1](https://openfontlicense.org).

**Cormorant Cipher** — steganographic fork. The OFL permits modification and redistribution under the same licence, provided the font is renamed (which it is). No Reserved Font Name applies to this fork.

The steganographic encoding mechanism, HTML specimen, and DNS beacon integration are original work, 2025.
