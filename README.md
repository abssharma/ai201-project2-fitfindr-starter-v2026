# FitFindr

> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.

---

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

FitFindr is a thrifting agent. A user types a request such as "a vintage graphic tee under $30, size M", and the agent searches a file of secondhand listings, picks the best match, suggests outfits that pair it with clothes from the user's wardrobe, and writes a short caption they could post. If nothing matches, it stops and tells the user what to change (the description, size, or price limit) instead of continuing.

---

## Tool Inventory

### `search_listings`

- **What it does:** Searches the listings file for items matching a keyword description, an optional size, and an optional price ceiling, and returns them best match first.
- **Inputs:** `description` (str), `size` (str or None), `max_price` (float or None)
- **Returns:** A list of at most 10 listing dicts (`config.SEARCH_RESULT_LIMIT`), best keyword match first. Each dict has `id` (str), `title` (str), `description` (str), `category` (str), `style_tags` (list), `size` (str), `condition` (str), `price` (float), `colors` (list), `brand` (str or None), `platform` (str).
- **When it has nothing:** Returns an empty list `[]`, never None and never an exception.

Rules:

1. The requested size must equal one whole token of the listing's size. Tokens come from splitting the listing's size on spaces, "/", and parentheses, and the comparison is case-insensitive. So "M" matches "M" and "S/M", but not "XL (oversized)" or "US 9".
2. `price <= max_price` is inclusive.
3. A listing scores one point per distinct description word found in its title, description, style_tags, or category, no matter how many times the word appears. Words match as whole words, case-insensitive, so "tee" does not match "steel". A score of 0 is dropped.
4. Filler words ("a", "an", "the", "under", "size", "in", "for") are ignored.
5. If size or max_price is None, that filter is skipped.
6. Results are sorted by score, highest first. Ties keep the order they have in the listings file.

### `suggest_outfit`

- **What it does:** Asks the model for one or two outfits that pair the found item with pieces from the user's wardrobe.
- **Inputs:** `new_item` (dict, one listing), `wardrobe` (dict with an `items` key holding a list of dicts, each with `id`, `name`, `category`, `colors`, `style_tags`, `notes`)
- **Returns:** A non-empty str with one or two outfit suggestions that name wardrobe pieces by their `name`.
- **When it has nothing:** If `wardrobe['items']` is empty or the wardrobe has no `items` key, it returns a non-empty str of general styling advice for the item. It never returns "" and never raises.

### `create_fit_card`

- **What it does:** Writes a short social-media caption about the item and its outfit.
- **Inputs:** `outfit` (str, the output of suggest_outfit), `new_item` (dict, one listing)
- **Returns:** A str of two to four sentences that mentions the item, its price, and its platform once each.
- **When it has nothing:** If `outfit` is empty or whitespace, it returns the str "No outfit was provided, so no fit card was written." It never returns "" and never raises.

---

## Planning Loop

**Branch rule:** If search_listings returns an empty list, set session["error"] to a sentence naming what the user could change (description, size, or price ceiling), leave session["fit_card"] as None, and return without calling suggest_outfit. Otherwise set session["selected_item"] to the first result in session["search_results"], then call suggest_outfit with the item read back from the session.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** TODO (Milestone 5): regex / string splitting / model

**What moves through the session:** In order: `query` → `parsed` (description, size, max_price) → `search_results` (written by `search_listings`) → `selected_item` (first result) → `outfit_suggestion` (written by `suggest_outfit`, which reads `selected_item` and `wardrobe` from the session) → `fit_card` (written by `create_fit_card`, which reads `outfit_suggestion` and `selected_item` from the session). If the search is empty, `error` is set and the run stops after `search_results`, so `suggest_outfit` and `create_fit_card` are never called.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask '...'

```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"

```

```
$ python -c "from tools import suggest_outfit; ..."

```

```
$ python -c "from tools import create_fit_card; ..."

```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
