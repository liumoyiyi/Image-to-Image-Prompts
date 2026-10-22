# Image-to-Image Prompt Engineering

Practical notes on editing existing photos with text prompts — prompt patterns per
task, model selection, and the client-side plumbing around an image upload.

This is the working notebook behind [img-img](https://img-img.net/), a free
image-to-image editor. It documents what we learned, not how our service is
built — see [Scope and boundaries](#scope-and-boundaries) before you look for an
API here.

---

## Why this repo exists

Almost every image-to-image guide stops at "describe what you want". That advice
is useless in practice, because the difference between a result you keep and one
you throw away is rarely the idea — it's whether the prompt is phrased as a
*transformation of an existing subject* or as a *description of a new scene*.

The second failure is subtler and costs more time: people burn attempts
regenerating until the person in the photo looks right, when the fix was pinning
what must not change. That distinction is the core of everything below.

If you're building anything that takes a photo as input and returns an edited
one, the patterns here should transfer regardless of which model you call.

---

## The one rule

> **Name what changes. Pin what stays.**

A prompt that only says what to change invites the model to reimagine
everything. A prompt that also states what must be preserved gives it a
constraint to work against, and constraint is what keeps the subject
recognisable across attempts.

The usual phrasing problem looks like this:

```
❌  A woman standing in a snow-covered street at dusk
✅  Add snow to this street scene and shift the light to dusk, keeping the
    same woman, her pose, and the framing unchanged
```

The first is text-to-image with a photo attached. The second is image-to-image.

---

## Prompt patterns by task

Six task families, each with the constraint that matters most. Replace the
bracketed parts with your own specifics.

### 1. Style transfer

The subject usually survives; the *composition* is what drifts.

```
Restyle this photo as {oil painting | watercolor | anime cel | pixel art},
keeping the same composition, subject placement and framing. Do not crop,
zoom, or add elements that are not in the original.
```

Failure mode to watch: a style model that "improves" a composition by
recentring the subject. If that matters, say `keep the subject off-centre where
it is`.

### 2. Portrait and headshot

Identity preservation is the whole job. Be explicit about it.

```
Relight this portrait with soft studio lighting from the upper left and put it
on a neutral dark-grey backdrop. Keep the same face, facial structure, skin
tone and expression. Do not slim, smooth, or otherwise retouch the face.
```

`Do not slim, smooth, or otherwise retouch` is doing real work — without it,
most models apply beauty-filter assumptions you never asked for. If you *do*
want retouching, ask for the specific operation (`reduce under-eye shadows`)
rather than the general outcome (`look better`).

### 3. Interior and architecture

Multiple variations from one photo is the real use case: same room, different
palette. Run it repeatedly with only the palette clause changed.

```
Change this living room to {warm Scandinavian | brutalist concrete | mid-century
modern} style. Keep the room's geometry, window positions, and camera angle
identical. Change furniture, wall colour and materials only.
```

Locking geometry matters because a re-imagined floor plan makes variations
impossible to compare — the whole value is holding the room still while
changing the look.

### 4. Product photography

E-commerce use, so the product must stay byte-for-byte believable.

```
Place this {product} on a {surface} with {lighting} and a {background}. Keep the
product's shape, label text, colour, and proportions exactly as they are. Match
perspective and shadow direction to the new scene.
```

Two clauses carry their weight here: `label text` (models love to rewrite
packaging copy into plausible-looking nonsense) and `shadow direction` (a
mismatched shadow is the tell that a product shot is composited).

### 5. Weather and season

```
Turn this {summer | daytime | autumn} scene into {winter | night | spring}.
Keep the same location, buildings, and viewpoint. Change only weather, light
colour and foliage — do not move or add structures.
```

### 6. Restoration and colourisation

Treat this as two steps, not one — they fail differently.

```
Step 1 — repair:  Remove scratches, dust and creases from this scan. Keep all
                  original detail, including grain. Do not add or invent content.
Step 2 — colour:  Colourise this photograph with natural, period-appropriate
                  tones (approx. {decade}). Keep skin tones and fabric colours
                  plausible, not saturated.
```

`Do not add or invent content` is the one that keeps a restoration from
becoming a fabrication — a face that was never in the photo is a serious problem
for archival use.

---

## Choosing between models

There's no single model that wins every row, and routing per task beats picking
a favourite. What we've observed across the models we route between:

| Task | What to look for | Typical weakness |
|---|---|---|
| Text inside the image (signage, labels, packaging) | Character accuracy, including non-Latin scripts | Rewrites words into plausible garbage |
| Scene, mood, weather, lighting | Prompt adherence without subject drift | Subject quietly becomes a different subject |
| Portrait identity | Face/structure retention | Unrequested beauty retouching |
| Restoration | Detail preservation under repair | Invents detail to fill damage |

**The trade-off worth internalising**: prompt adherence and subject preservation
pull against each other. A model that follows you precisely will also follow you
into changing the person. That's why the prompt patterns above spend so many
words on what *not* to touch — you're supplying the constraint the model won't
apply on its own.

## Iterating without losing the subject

The most common waste of attempts is starting over. Two rules help:

1. **Keep one base image and refine it.** Re-uploading the original for each
   variation resets the subject, so `attempt 5` may be a different person than
   `attempt 1`. Iterating on the same base holds identity stable.
2. **Change one variable per attempt.** Two changed clauses produce a result you
   can't attribute, and you learn nothing about which clause mattered.

---

## Client-side plumbing (copy-paste)

Model-side quality gets all the attention; half the real bugs live in what you
send. Pure-browser helpers, no server assumptions:

```js
/**
 * Validate and normalise an image before upload.
 * - rejects unsupported / oversized files with a usable message
 * - downscales the long edge, which is usually cheaper than sending the original
 * - strips EXIF (privacy) and fixes orientation first (otherwise rotated photos
 *   reach the model sideways and the prompt has to fight that)
 */
export async function prepareImage(file, { maxBytes = 10 * 1024 * 1024, maxEdge = 2048 } = {}) {
  const ok = ['image/jpeg', 'image/png', 'image/webp'];
  if (!ok.includes(file.type)) throw new Error(`Unsupported format: ${file.type || 'unknown'}`);
  if (file.size > maxBytes) throw new Error(`File is ${(file.size / 1048576).toFixed(1)}MB, limit is ${maxBytes / 1048576}MB`);

  // createImageBitmap applies EXIF orientation and does not retain metadata
  const bitmap = await createImageBitmap(file);
  const scale = Math.min(1, maxEdge / Math.max(bitmap.width, bitmap.height));
  const w = Math.round(bitmap.width * scale);
  const h = Math.round(bitmap.height * scale);

  const canvas = new OffscreenCanvas(w, h);
  canvas.getContext('2d').drawImage(bitmap, 0, 0, w, h);
  bitmap.close?.();

  const blob = await canvas.convertToBlob({ type: 'image/jpeg', quality: 0.92 });
  return { blob, width: w, height: h };
}
```

Gotchas this handles, each of which we hit in production:

- **EXIF orientation.** A phone photo can be stored landscape with a "rotate 90°"
  flag. Strip the metadata without applying the rotation and it arrives sideways.
  `createImageBitmap` fixes orientation for you; a naive canvas read does not.
- **Downscale before the call, not after.** Upload bandwidth is usually the
  slowest hop in the chain, and most edits don't need the full original.
- **Validate client-side *and* server-side.** Client checks are UX; they are not
  a security boundary.
- **Reject by MIME, then verify by decode.** A `.png` extension proves nothing;
  successful decode is the real test.

---

## If you're wiring this into your own app

The call shape is unremarkable — multipart or base64 body, a prompt string, and
a model identifier — so the interesting decisions are elsewhere:

- **Timeouts.** Image edits take seconds, not milliseconds. Set the client
  timeout above your p99, and design the loading state as part of the product
  rather than a spinner bolted on afterwards.
- **Cost per attempt.** You will pay for rejected attempts. Making refinement
  cheap enough to use (smaller inputs, same base image) usually beats making
  generation cheaper.
- **Where results live.** Decide deliberately whether generated images persist.
  For a tool handling family photos or unreleased products, "processed in memory,
  deleted after generation, never used for training" is a feature worth stating
  plainly — and worth actually implementing.
- **Failure messages.** `Error 500` for a model-side refusal trains users to
  retry blindly. Distinguishing "your prompt was rejected" from "the model
  failed" changes how people behave.

---

## Scope and boundaries

This repository is documentation and client-side code. Deliberately **not**
included:

- **No API endpoints, request schemas, or authentication details.** The service
  has a private API; its surface is not documented here, and nothing in this repo
  should be read as a spec for it.
- **No credentials, tokens, keys, or environment configuration.**
- **No rate-limit bypasses, scraping scripts, or automation against the hosted
  service.** If you want to batch-process images, run the patterns against a
  model provider you have an account with.
- **No server-side or infrastructure internals.**

If you find something here that looks like it leaks any of the above, that's a
bug — please open an issue rather than using it.

## Contributing

Corrections to the prompt patterns are especially welcome, particularly cases
where a specific model misbehaves in a way the pattern doesn't cover. Please
include the model, the prompt, and what came back.

## License

MIT for the code in this repo. The prompt text is free to reuse.

---

## Try the patterns without writing any code

Every pattern above was worked out against a real tool, and the fastest way to
see whether a prompt phrasing works is to run it. **[img-img.net](https://img-img.net/)
takes a photo and a prompt and returns the edited image in seconds — no account,
no credit card, free to use**, so you can paste any example from this README
straight into it and watch what changes.

Two things make it a decent scratchpad for this specifically: the free tier
doesn't expire, so you can come back to a half-finished comparison tomorrow; and
iteration happens on the same base image, which is exactly the workflow the
section on [iterating without losing the subject](#iterating-without-losing-the-subject)
describes. Uploaded images are processed in memory and deleted after generation,
never used for training.

If a pattern doesn't behave the way this README claims, that's useful to know —
open an issue with the prompt you used.

**[Start with an image →](https://img-img.net/)**
