# fibrestats Brand System

fibrestats uses the official Celestia brand **palette and typography** as its
design tokens. This document captures those tokens and maps them onto the
app's shadcn/ui + Tailwind theme.

> **No Celestia logos or marks.** We do not use the Celestia symbol, wordmark,
> or any Celestia logo anywhere in our apps or materials. Palette and type
> conventions only — the project is an independent DTEAMTECH product, not an
> official Celestia one.

Source for the tokens: <https://celestia.org/brand/>

---

## Palette

### Anchors

Every surface is anchored by Frost (light) and Void (dark).

| Name  | Hex       | Role                     |
| ----- | --------- | ------------------------ |
| Frost | `#FDFCFF` | Light surfaces, base background |
| Void  | `#0E1014` | Dark surfaces, primary text on Frost |

### Brand carriers

| Name     | Hex       | Role                              |
| -------- | --------- | --------------------------------- |
| Indigo   | `#5640D1` | Primary brand color — key actions, brand moments |
| Amethyst | `#7C68F2` | Secondary brand color — accents, highlights |

### Extended families

Four families handle everything else (data visualization, states, subtle
surfaces). Each has five steps: 50, 200, 400, 600, 800.

#### Cool Gray (Neutral)

| Step | Hex       |
| ---- | --------- |
| 50   | `#EAECEE` |
| 200  | `#C4C8CE` |
| 400  | `#808890` |
| 600  | `#4A5058` |
| 800  | `#242830` |

#### Blue

| Step | Hex       |
| ---- | --------- |
| 50   | `#EAEDF0` |
| 200  | `#A9C2D4` |
| 400  | `#5595C3` |
| 600  | `#32688F` |
| 800  | `#254459` |

#### Purple

| Step | Hex       |
| ---- | --------- |
| 50   | `#E3E1F2` |
| 200  | `#C8C3E5` |
| 400  | `#7C68F2` |
| 600  | `#4331A5` |
| 800  | `#231860` |

(Purple 400 is Amethyst, the secondary brand carrier.)

#### Beige

| Step | Hex       |
| ---- | --------- |
| 50   | `#EDECEB` |
| 200  | `#C6BDB7` |
| 400  | `#9A897E` |
| 600  | `#6C5E55` |
| 800  | `#473E38` |

---

## Typography

| Typeface          | CSS name       | Usage                                            |
| ----------------- | -------------- | ------------------------------------------------ |
| Nuber Next        | `nuberNext`    | Brand typeface — UI text, headings               |
| Nuber Next Wide   | `nuberNextWide`| Wide cut of the brand face — display headings    |
| Roboto Mono       | —              | Code and data (hashes, heights, metrics)         |

**License note:** Nuber Next and Nuber Next Wide are commercial fonts. The
licensed font files are **not committed to any repository** and must not be
redistributed. Obtain a license and place the files locally if you have one
(`public/fonts/` in the app); otherwise fall back to a neutral grotesque
(e.g. Inter/system-ui) in the font stack.

**Label style:** the brand site sets small labels in **uppercase with wide
tracking** (letter-spacing). Use this for overlines, stat labels, table
headers, and nav items.

---

## Cells — generative visual language

"Cells" are Celestia's generative visual language: flowing fields of unique
particles that appear across the site, decks, and brand materials. Cells-
*inspired* backgrounds or loading/hero moments are acceptable as decorative
texture (never behind dense data), but do not copy Celestia's actual cell
renders or media files.

---

## Theme mapping (shadcn/ui + Tailwind)

How the brand tokens map onto the app's shadcn/Tailwind CSS variables
(implemented in `fibrestats/app/globals.css`):

| shadcn/Tailwind token      | Light mode              | Dark mode (anchored on Void) |
| -------------------------- | ----------------------- | ---------------------------- |
| `background`               | Frost `#FDFCFF`         | Void `#0E1014`               |
| `foreground`               | Void `#0E1014`          | Frost `#FDFCFF`              |
| `primary`                  | Indigo `#5640D1`        | Indigo `#5640D1`             |
| `primary-foreground`       | White / Frost `#FDFCFF` | White / Frost `#FDFCFF`      |
| `accent`                   | Amethyst `#7C68F2`      | Amethyst `#7C68F2`           |
| `accent-foreground`        | Void `#0E1014`          | Void `#0E1014`               |
| `muted`                    | Cool Gray 50 `#EAECEE`  | Cool Gray 800 `#242830`      |
| `muted-foreground`         | Cool Gray 600 `#4A5058` | Cool Gray 200 `#C4C8CE`      |
| `border` / `input`         | Cool Gray 200 `#C4C8CE` | Cool Gray 800 `#242830`      |
| `card` / `popover`         | Frost `#FDFCFF`         | Void `#0E1014` (or Cool Gray 800 for elevation) |
| `ring`                     | Indigo `#5640D1`        | Amethyst `#7C68F2`           |

Data-visualization series pull from the extended families (Blue 400/600,
Purple 400/600, Beige 400/600) so charts stay on-brand; Cool Gray steps carry
gridlines, axes, and disabled states. Dark mode is anchored on Void with Frost
text — it is a first-class mode, not an inversion.
