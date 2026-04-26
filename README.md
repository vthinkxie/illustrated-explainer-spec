# Drill-Down Explainer Spec

A stack-agnostic spec for an AI illustrated explainer: type a topic, click anywhere on the image to drill into that spot, repeat forever — with the painting style preserved across every page.

Hand this spec to any capable coding LLM (or human). As long as every behavior in §12 passes, the technology choice is yours.

---

## 1. Product Shape

A locally-run single-page web app. The user types a topic and gets back an illustrated explainer page; clicking anywhere on the image generates the next page that "drills into" that location (zoom-in, cross-section, inner mechanism), preserving the painting style exactly. The user can drill infinitely deep, go back, and jump to any prior page.

## 2. Core Loop

```
[topic input box]
       │ submit query
       ▼
[Page 1: a single 16:9 illustration]
       │ user clicks anywhere on the image at (x, y)
       ▼
[Next page: drills into the (x, y) area of the previous page]
       │ click again
       ▼
   …infinitely
```

## 3. State Model

The whole document is an **ordered array of pages** plus a "current index" pointer. Each page is a self-contained object:

- `id` — a stable content fingerprint (see §6)
- `imageUrl` — the URL of the generated image
- `parentId` — id of the previous page (`null` for the first page)
- `parentClick` — the click coordinates on the parent page (`null` for the first page)
- `initialQuery` — the original topic string (only set on the first page; `null` otherwise)

When the user clicks the image, **truncate** all pages after the current one and append the new page. This means clicking from a middle page creates a new branch, but the UI always sees a single linear array.

## 4. Client Responsibilities (Deliberately Thin)

The client only does three things:

1. POST the topic string or click coordinates to `/api/page`, receive a page object, append it to the array.
2. Render the current page's image. On click, use `getBoundingClientRect()` to convert pixel coordinates into **normalized coordinates (0–1)** before sending — this way the client never needs to know the image's real resolution.
3. Provide: a Back button, jump-to-any-page (thumbnail strip), and a Reset button.

The client **does not** hold any prompts, does not hold any API keys, and never calls a model directly.

## 5. Single Server Endpoint

`POST /api/page`. The request body is one of two shapes:

```
{ "query": "<topic>" }                                          // first page
{ "parentId": "<id>", "parentClick": { "x": 0..1, "y": 0..1 } } // subsequent page
```

Response: `{ "page": <Page> }`.

The server must validate strictly: `query` is 1–300 chars; `parentId` matches the content-fingerprint shape; `x` and `y` are finite floats within `[0, 1]`.

## 6. Content-Addressed Caching

Page ids are not random — they are **deterministic hashes**:

- First-page id = `hash("first" + version + normalize(query))`
  - `normalize`: trim, collapse whitespace, lowercase
- Child-page id = `hash("child" + version + parentId + round(x, 2) + round(y, 2))`
  - Coordinates are rounded to 2 decimal places to prevent pixel-level jitter from fragmenting the cache

Generated images are written to `<static>/generated/<id>.png`. On every request, first check whether that file exists and is non-empty — if so, return its URL immediately without calling the model.

Effects:
- The same query always produces the same first page.
- Clicking the same spot on the same page always produces the same child.
- `Back` and thumbnail jumps are free (no regeneration).
- Bumping the `version` string invalidates all caches at once.

## 7. The Critical Trick for Child Pages (the Soul of the Project)

To make "drill into where I pointed" understandable to an image model, **do not** ask the model to parse coordinate numbers. Instead:

1. Server reads the parent PNG.
2. On a canvas, copy the original image, then composite a prominent **red ring + filled center dot** at `(x * width, y * height)` — a half-transparent ring, a high-contrast outline, and a solid inner dot, with a radius of about 4% of the image width.
3. Send this **composited image (with the red marker)** as a reference image, alongside the prompt, to the image model.
4. The prompt explicitly tells the model: "The red circle marks where the reader pointed. Generate the next page by drilling into whatever the red circle is on (zoom in, internal structure, mechanism). **Do not include the red circle in the output.** Match the painting style of the provided image exactly — same line weight, paper tone, palette, and title typography."

This step translates the abstract action of "pointing" into something image models are natively good at: looking at pictures.

## 8. Style Coherence (with Verbatim Prompts)

Define **one detailed style description string** that is the single source of truth. Both prompts below reference it by inclusion — never re-write the style across prompts.

### Shared style description

```
Painting style (must remain consistent across every page):
- Light warm paper background with generous margins
- Clean, even dark gray or black ink outlines, consistent thin line weight
- Soft watercolor washes, pale palette: ivory, pale green, pale blue, light gray, with restrained warm accents
- A large serif title printed at the top center of the image
- Calm, well-composed scene with breathing room

Strict exclusions:
- No decorative borders, seals, parchment aging, ornate fonts, or vintage texture
- No 3D render, photorealism, neon, dark themes, or modern app UI cards
- No dense paragraphs of text, watermarks, or tiny unreadable labels
- No tourist map roads, landmarks, transit, or "traveler-guide" framing
```

### First-page prompt

`{query}` is the **only** user-controlled slot. Trim it; reject anything outside 1–300 chars before substitution.

```
{STYLE_DESCRIPTION}

Subject: {query}

Compose a single 16:9 illustrated explainer page about the subject above.
Let the scene's content (objects, layout, metaphor) be whatever best
explains the subject — cross-section, exploded view, timeline, anatomy,
flow, comparison, or scene — chosen to fit this specific topic.

Output a single PNG image, 16:9. Print the title clearly inside the image.
```

### Child-page prompt

This prompt has **no slots**. The "where the user pointed" signal is delivered visually via the red-marker reference image (see §7), not via text.

```
{STYLE_DESCRIPTION}

You are continuing an illustrated explainer book.
The provided image is the previous page. A red circle marks
the area the reader pointed at.

Generate the next page: a single 16:9 image that goes deeper
into whatever the red circle is on — zoom in, expand its inner
structure, or show its mechanism.

Critical: match the painting style of the provided image exactly
— same line weight, same paper tone, same pastel palette, same
title typography. The two pages must feel like consecutive spreads
in the same hand-drawn book.

Do NOT include the red circle or any cursor mark in the output.

Output a single PNG image, 16:9.
```

### Model-call shape

- Both prompts are sent to a multimodal image-generation model that accepts `(text, optional reference image) → image bytes`.
- For the child page, the request carries **two parts**: the prompt text **and** the parent image with the red marker composited onto it (PNG, base64 inline).
- Request the model to emit a 16:9 image. If the API exposes an aspect-ratio knob, set it to `16:9`; otherwise rely on the prompt instruction.
- Read the first inline-image part from the response; ignore any text parts.

## 9. Concurrency and Failure Handling

- The server **serializes generation requests in-process** (a simple promise tail is enough). Reason: image generation is slow and expensive, and concurrent runs often duplicate work on the same parent.
- Cache hits are synchronous — they could skip the lock — but it's simpler to put them inside it.
- Use `AbortController` + a configurable timeout when calling the model.
- Non-200 response, or no inline image in the response → return 500. The client just shows "Generation failed, try clicking elsewhere." Do not auto-retry.

## 10. Security and Boundaries

- The API key is read only from server-side environment variables; it is never sent to the browser.
- The browser can only send a query string and normalized coordinates — all prompts are hard-coded on the server. User input is never spliced into prompts outside a single explicit "topic" slot.
- Generated file paths are derived from the id; the client cannot specify file names, eliminating path-traversal risk.
- Validate three things: query length, coordinate range `[0, 1]`, and `parentId` matching the hash regex.

## 11. UI Element Inventory

- **Top bar**: app name; `X / N` page counter; Back button (disabled on the first page); Reset button (disabled when there are no pages).
- **Topic input**: single-line input + Generate button; both disabled while loading.
- **Canvas area**: full-width display of the current page's image; a translucent overlay during loading ("Generating the next page…"); on click, hand normalized coordinates up to the page state. Cursor is a pointer; clicks emit a brief ripple animation at the click point.
- **Thumbnail strip**: bottom row, one tile per generated page with its index; current page highlighted; click to jump; collapsible.
- **Error banner**: a single red line; clears automatically on the next successful generation.

## 12. Acceptance Checklist (Behavioral, Stack-Agnostic)

- Type "how volcanoes work" → a watercolor-style explainer page with the title printed inside, no map elements.
- Type "how a smartphone is built" → a same-style cross-section / exploded view, not a tourist map.
- Click a visible object on the image → the next page clearly "drills into that object," and the painting style (line weight, paper, palette) is nearly indistinguishable from the previous page.
- Drill 5 pages deep → the style stays consistent.
- Back returns to the previous page; thumbnails jump to any page **without** triggering a new generation (verify in the network panel).
- Reset clears state back to the empty topic input.
- After restarting the server, typing the same query returns instantly (disk-cache hit).
- Two rapid consecutive clicks → the second request is processed only after the first completes.

---

## License

Do whatever you want with this spec.

---

Inspired by [flipbook.page](https://flipbook.page/).
