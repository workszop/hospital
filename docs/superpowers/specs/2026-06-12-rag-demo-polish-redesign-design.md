# RAG Demo Redesign — Polish-language educational walkthrough

**Date:** 2026-06-12
**Status:** Approved design, ready for planning
**Scope:** The interactive "Live Demo" section of `index.html` (MedVector AI landing page)

## Problem

The current demo is built to teach how Retrieval-Augmented Generation (RAG)
works, for a Polish-speaking educational audience. It fails at that today:

- **It is entirely in English.** Labels, buttons, sample queries, documents,
  and generated answers are all English — a barrier for the target audience.
- **It is hard to follow.** Five buttons across two cards with no guided order;
  a redundant `Generate Vectors` button that duplicates pipeline step 0; a step
  counter (0–6) that is offset from the on-screen "Step 1…5" labels; a
  "Next Step" button that morphs into a second "Reset"; and a "Pre-indexed"
  label that contradicts the fact that indexing visibly happens in step 0.
- **The payoff is buried.** The assembled prompt and the final answer — the most
  instructive outputs — are crammed into fixed 150px-tall, ~180px-wide, 11px
  scrolling boxes.
- **It contradicts its own teaching.** The page sells "semantic meaning, not
  keywords," but the toy embedding is literal keyword matching, the off-topic
  ("weather") case is a hardcoded string check rather than low similarity, and
  the answers are hardcoded per-question lookup tables. A learner who inspects
  the source learns the wrong lesson.

## Goals

1. A single, guided, hard-to-get-lost flow through the 5 RAG stages.
2. Fully Polish UI, content, and narration.
3. Give the prompt and answer real space to be read.
4. Add genuine conceptual intuition for vector similarity (a 2-D map).
5. Make every *visible mechanism* real and internally consistent, while being
   explicit that the embedding model is a simplified teaching toy.

## Non-goals

- Calling a real LLM or a real embedding model. The demo stays offline,
  in-browser, single-file (`index.html`), vanilla JS, neobrutalist style.
- A PL/EN toggle. The demo is Polish-only.
- Changing the rest of the landing page (hero, pitch, features) beyond what is
  needed to keep it consistent with the redesigned demo.

## Approach: narrated stepper

One stage on screen at a time, driven by a single prominent control. Chosen over
a cleaned-up dashboard (still inherently busy) and a slide deck (loses the
replayable-tool feel).

## Structure

A single demo card with persistent zones, top to bottom:

1. **Query input + sample chips** (Polish). One chip is an off-topic question
   that demonstrates the "no relevant documents" case via *real* low similarity.
2. **Stepper header** — 5 labelled stages, current one highlighted:
   `1 Indeksowanie · 2 Osadzanie zapytania · 3 Wyszukiwanie · 4 Budowa promptu ·
   5 Odpowiedź`. Replaces the cramped horizontal pipeline.
3. **Stage area** (large middle) — shows only the current stage's visualization,
   with room to breathe.
4. **Narrator bar** — 1–2 sentences of plain Polish explaining what just
   happened and why, plus the controls: `Następny krok`, `Odtwórz całość`
   (auto-play), `Reset`.

Removed: the redundant `Generate Vectors` button, the `Toggle View` accordion,
the dual-purpose "Next Step → Reset" morph, and the cramped 5-box pipeline row.

## Per-stage visuals

A **2-D vector map** is introduced in Stage 1 and persists as the visual
through-line for Stages 1–3, then yields to full-width panels for Stages 4–5.

### Stage 1 — Indeksowanie (Indexing)
The 6 medical documents shown as cards. On entering the stage, each animates
text → vector and lands as a coloured dot (colour = specialty) on the vector
map. Indexing visibly *happens* (no "Pre-indexed" contradiction).
> „Każdy dokument zamieniamy na wektor — listę liczb opisującą jego znaczenie.
> Podobne tematy trafiają blisko siebie na mapie."

### Stage 2 — Osadzanie zapytania (Query embedding)
The typed question animates into the same map as a distinct **star**; its 2-D
vector appears digit by digit.
> „To samo robimy z pytaniem — zamieniamy je na wektor w tej samej przestrzeni,
> żeby móc je porównać z dokumentami."

### Stage 3 — Wyszukiwanie (Retrieval) — conceptual centrepiece
Lines draw from the query star to every document, thickness/opacity ∝ cosine
similarity. Top-3 nearest light up; a side list ranks them with filled
similarity bars (e.g. `Zawał serca — 92%`). For the off-topic query all lines
are faint and the panel reports „Brak wystarczająco podobnych dokumentów" —
driven by real low similarity.
> „Mierzymy podobieństwo (kąt między wektorami). Im bliżej, tym bardziej
> dokument pasuje. Wybieramy 3 najlepsze."

### Stage 4 — Budowa promptu (Prompt building)
Full-width, readable. The prompt assembles block by block with labelled,
colour-coded sections: `SYSTEM` → `KONTEKST` (3 retrieved snippets dropping in)
→ `PYTANIE` → `INSTRUKCJA`, each fading in in sequence.
> „Z odnalezionych dowodów składamy prompt: instrukcje + kontekst + pytanie.
> To wszystko, co model dostaje na wejściu."

### Stage 5 — Odpowiedź (Answer)
Full-width answer panel, comfortable type. The answer streams in word by word;
cited document numbers `[1][2]` are highlighted to show the link back to
retrieved evidence. Prominent disclaimer: „Demonstracja edukacyjna — nie jest
poradą medyczną."
> „Model generuje odpowiedź wyłącznie na podstawie dostarczonego kontekstu —
> dlatego jest oparta na wytycznych, a nie zmyślona."

## Honest-model changes

The embedding stays a labelled teaching toy, but retrieval, similarity, the
no-match path, and answer assembly all become genuinely vector-driven.

1. **Embeddings become genuinely 2-D unit vectors.** What's on the map *is* the
   actual vector (no fake projection); cosine similarity = the visible angle.
   Documents cluster by specialty. Displayed as e.g. `[0.62, 0.79]` with caption:
   „Uproszczony wektor 2-D do celów dydaktycznych. Prawdziwe modele używają
   ~1500 wymiarów."

2. **Retrieval is purely cosine-similarity-driven, including the no-match case.**
   The hardcoded `if (includes('weather'))` branch is removed. An off-topic
   query gets a real vector far from every cluster → max similarity below a
   threshold → honest „Brak wystarczająco podobnych dokumentów". Same code path
   for every query.

3. **The answer is assembled from retrieved document text**, not a per-question
   hardcoded lookup. Template draws the cited documents' own text, so changing
   which docs are retrieved changes the answer. Labelled honestly: „W prawdziwym
   systemie ten krok wykonuje model językowy. Tutaj składamy odpowiedź z
   fragmentów dokumentów, aby pokazać zasadę."

4. **The keyword→vector shortcut is named, not hidden.** Stage 1 narration states
   that real systems learn these vectors from data; here a simplified rule
   assigns them so the demo runs offline.

## Data changes

- Documents (`documents` array) translated to Polish, with Polish clinical
  guideline content. Each gets a hand-assigned 2-D unit vector placing it in a
  sensible semantic cluster by specialty.
- Sample queries translated to Polish; the off-topic chip kept as a genuine
  low-similarity demonstrator (not a string-matched special case).

## Technical constraints

- Single file `index.html`, vanilla JS, no build step, no new runtime deps
  (lucide icons + Google Fonts already loaded are fine).
- The 2-D vector map can be rendered with inline SVG (no charting library) to
  keep the single-file, dependency-light constraint.
- Preserve the existing neobrutalist visual language (thick borders, hard
  shadows, the existing colour variables).
- Keep working on mobile: the stepper and stage area must reflow to a single
  column the way the current pipeline does.

## Testing / verification

No test harness exists (static single-file page). Verification is manual via
the browser:

1. Each sample query, including the off-topic one, runs end to end with both
   `Następny krok` and `Odtwórz całość`.
2. The off-topic query reaches "no relevant documents" through real low
   similarity (verifiable by inspecting computed similarities), not a hardcoded
   branch.
3. A custom typed Polish query retrieves sensible documents and the assembled
   answer reflects the retrieved docs.
4. `Reset` returns the demo to its initial state.
5. Layout holds at desktop and mobile widths.
6. All visible text is Polish.
