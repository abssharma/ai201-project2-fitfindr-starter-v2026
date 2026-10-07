# FitFindr

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

**How the query is parsed:** Regex, in `agent.py::_parse_query`. Price comes from "under/below/max $N" (or any `$N`), size from "size X" or a standalone uppercase size (XXS to XXL), and whatever is left becomes the description.

**What moves through the session:** In order: `query` → `parsed` (description, size, max_price) → `search_results` (written by `search_listings`) → `selected_item` (first result) → `outfit_suggestion` (written by `suggest_outfit`, which reads `selected_item` and `wardrobe` from the session) → `fit_card` (written by `create_fit_card`, which reads `outfit_suggestion` and `selected_item` from the session). If the search is empty, `error` is set and the run stops after `search_results`, so `suggest_outfit` and `create_fit_card` are never called.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30, size M'

Found: Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   **Outfit 1: Off-Duty Y2K**
*   **Top:** Y2K Baby Tee — Butterfly Print
*   **Bottoms:** Baggy straight-leg jeans, dark wash 
*   **Outerwear:** Black cropped zip hoodie (worn open)
*   **Shoes:** Chunky white sneakers
*   **Accessories:** Black crossbody bag

**Outfit 2: High-Low Contrast**
*   **Top:** Y2K Baby Tee — Butterfly Print
*   **Bottoms:** Wide-leg khaki trousers
*   **Belt:** Brown leather belt (threaded through the trousers)
*   **Outerwear:** Vintage black denim jacket
*   **Shoes:** Black combat boots

  Fit card: Okay wait, this little butterfly tee is *so* 2004 pop princess coded. I’m picturing it with baggy dark denim and a chunky sneaker, or dressed down a bit with khaki trousers and combat boots. Literally just dropped it on my Depop for $18 if you want to channel your inner early-2000s off-duty model.

0 model calls this session, 2 served from cache
```

```
$ python app.py ask 'designer ballgown size XXS under $5'

  No listings matched. You could raise the $5 price limit, try a different size than XXS, use broader words than 'designer ballgown'.

0 model calls this session
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print([(i['id'], i['size'], i['price']) for i in search_listings('graphic tee', 'M', 30)])"
[('lst_002', 'S/M', 18.0), ('lst_017', 'S/M', 15.0)]

$ python -c "from tools import search_listings; print(search_listings('ballgown', 'XXS', 5))"
[]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, get_empty_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe())); print('---'); print(suggest_outfit(load_listings()[0], get_empty_wardrobe()))"
**Outfit 1: Casual Streetwear**
Pair the Vintage Levi's 501 Jeans with the white ribbed tank top, layered under the oversized grey crewneck sweatshirt. Add the chunky white sneakers and the black crossbody bag for an effortless, everyday look.

**Outfit 2: Edgy Contrast**
Style the Vintage Levi's 501 Jeans with the black cropped zip hoodie and the vintage black denim jacket for a double-denim moment. Cinchthe waist with the brown leather belt and finish with the black combat boots.
---
**Styling Advice:**
Vintage 501s are the holy grail. They have zero stretch and a high waist, so lean into that rigid, classic structure. Size up and use a belt for a relaxed look, or wear true-to-size for a tailored vintage fit. 

**Two Easy Outfits:**

1. **The Classic Off-Duty:** Tuck a plain white ribbed tank or a fitted black t-shirt into the jeans. Add a leather belt, white canvas sneakers, and silver hoops. 
2. **The Streetwear Layer:** Pair them with an oversized grey crewneck sweatshirt. Let the hem peek out, cuff the jeans slightly, and finish with retro runners (like New Balance) and a baseball cap.
```

```
$ AI201_CACHE=0 python -c "from tools import create_fit_card; from utils.data_loader import load_listings; [print(create_fit_card('jeans and white sneakers', load_listings()[1]), '\n---') for _ in range(3)]"
Found the ultimate Y2K baby tee with the cutest little butterfly print. Tossed it on with my favorite beat-up jeans and white sneakers and the vibe is literally peak 2000s mall rat. Snagged it on Depop for just $18 and I’m never taking it off. 
---
Found the ultimate Y2K baby tee with the cutest butterfly print and I am obsessed. Just posted this over on my Depop for $18 and it’s giving major early 2000s mall-rat energy. Honestly gonna look so good with baggy jeans and beat-up white sneakers. 
---
Still pinching myself over finding this butterfly baby tee. It’s giving total early 2000s mall rat energy, and I am so here for it. Snagged it on Depop for just $18. Honestly just living in this with my favorite baggy jeans and beat-up white sneakers all spring. 
---
```

## How I Used AI

**Moment 1**

- *What I asked for:* I asked Claude to help debug my three tools in `tools.py`, based on my written specs.
- *What came back:* Solutions for fixing `search_listings` (filters on price and size, then ranks by how many words match) and prompts for the two model tools. When I tested them, `'graphic tee'` in size M under $30 gave me two listings, both sized `S/M`. The impossible query gave back `[]`, and the empty wardrobe gave general advice instead of crashing. My first fit card test could have been served from the cache, so I reran it with `AI201_CACHE=0` and got three different captions.
- *What I changed:* I made sure the code matched my README rules, so the spec and the tools agree. I also added `AI201_CACHE=0` to the fit card test so the captions showed real variation, and I left the model's "Cinchthe" typo in the pasted output because that's what it actually printed.

**Moment 2**

- *What I asked for:* I asked Claude to help me tighten up or loosen the reasoning behind my acceptance criteria, and to check whether my wording made sense.
- *What came back:* It pointed out that my criterion 1 reason blamed rate limits, but `generate.py` already handles those with pacing and retries, so they rarely break a run. It also said criterion 3 was too easy, because a bug that always grabs the first listing could slip through.
- *What I changed:* I dropped rate limits and named the misses I actually expect, like "tees" not matching "tee" and my hand-written parser misreading things. I made criterion 3 harder by requiring some queries with several matches and at least one where the pick isn't the first listing in the file.

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
