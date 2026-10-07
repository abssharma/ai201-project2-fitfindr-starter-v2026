# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:** My search only matches whole words, so "tees" will miss a
listing that says "tee", and my query parser is hand-written, so it can
misread a size or price. Those are the misses I actually expect. The model
calls can also come back empty or off, but the adapter retries rate limits, so
those rarely sink a run. I'll use five differently phrased queries that each
should match something, so a phrasing problem has a chance to show up instead
of repeating five times. My target is 4 of 5 passing.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:** This path never touches the model, and there's no
randomness in it. The search either returns an empty list or it doesn't, and
the loop just checks that one thing. So if it fails even once, that's a bug in
my branch. My target is 5 of 5 passing.

---

## 3. State is carried through all three tools

Run 5 different queries that each return a result. At least 2 of them must
return several matches, and in at least 1 the selected item must not be the
first listing in the file (lst_001). For each of the 5 runs, the listing `id`
received by `suggest_outfit`, the `id` received by `create_fit_card`, and the
`id` in `session["selected_item"]` at the end of the run must all be identical
— 5 of 5 runs. (Checked by wrapping each tool and recording the item it
receives.)

**Why this target:** Passing state around is plain code with no model in it,
so nothing random can excuse a miss. If the ids differ even once, my loop is
dropping or overwriting the item. I'm using different queries, including ones
where the best match isn't the first listing in the file, because a bug like
"always grab the first listing" would pass an easier test by accident. My
target is 5 of 5 passing.

---

## 4. The fit card holds up across different items

For 5 different items (at least one with `brand` None), each fit card is two
to four sentences, contains the item's price and platform, and contains no
"None" text. At least 4 of the 5 cards must pass all three checks. Separately,
across the 5 cards, no two share the same opening sentence (5 of 5 distinct).

**Why this target:** The model sometimes ignores an instruction, and my
sentence counter is a rough split on `.`, `!` and `?` that emoji or "$24.00"
can confuse, so demanding all 5 cards pass every check would punish my counter
as much as the model. 4 of 5 leaves room for one of those. I kept the
opening-sentence check at 5 of 5 because TEMPERATURE is 0.9 and the items are
different, so two identical openings would mean my prompt is forcing a
template, and that's the exact thing I want to catch.

---

## 5. An unreachable model produces a message, not a stack trace

With an invalid `GEMINI_API_KEY` and `AI201_CACHE=0`, running the agent ends
with no Python traceback in the output, `session["error"]` set to a non-empty
string that tells the user to check their key, and `session["fit_card"]` still
`None` — in 5 of 5 tries. Tries 1–3 fail on the first model call
(`suggest_outfit`); tries 4–5 fail only on the second (`create_fit_card`).

**Why this target:** I expect to miss this the first time, because the
`ModelUnavailable` handler doesn't exist yet and a bad key currently gives a
traceback. I'm still setting it at 5 of 5. Once the handler is written, the
behavior is predictable, and since either model call can fail, a handler that
only covers one of them should fail this test. A lower target would only make
sense if the failure were random, and it isn't.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
