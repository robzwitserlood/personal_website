---
title: Estimating the carbon footprint of Dutch recipes
draft: false
date: 2021-05-01
---

Food systems account for roughly a quarter of global GHG emissions, with dairy, meat, and eggs making up 80% of most dietary footprints. Online recipes are a scalable, low-bias window into what people actually cook — but turning free-text Dutch recipe ingredients into emission estimates requires solving a surprisingly thorny NLP pipeline. This post is about building that pipeline, and specifically about the iterative process of getting the first stage right.

---

## The dataset: free labels hiding in the HTML

The data came from 24Kitchen, a Dutch cooking website. The scraper collected 7,097 recipes with 82,263 ingredient descriptions and 240,131 tokens in total. What made this dataset unusual was that the site's own HTML already encoded the annotations: each ingredient line used `<span class="ingredient">`, `<span class="amount">`, and `<span class="unit">` to mark up its tokens. The labeled training set was essentially free — no hand-labeling required.

The dataset was split into train, validation, and test sets using stratified random sampling on recipe category (main, appetizer, dessert, unknown) to avoid a category bias in the models:

<table>
  <thead>
    <tr><th></th><th>Recipes</th><th>Ingredients</th><th>Tokens</th></tr>
  </thead>
  <tbody>
    <tr><td><strong>Train</strong></td><td>5,640 (79.5%)</td><td>65,599 (79.7%)</td><td>191,965 (79.8%)</td></tr>
    <tr><td><strong>Validation</strong></td><td>726 (10.2%)</td><td>8,226 (10.0%)</td><td>24,131 (10.0%)</td></tr>
    <tr><td><strong>Test</strong></td><td>731 (10.3%)</td><td>8,438 (10.3%)</td><td>24,475 (10.2%)</td></tr>
  </tbody>
</table>

---

## The pipeline

The end goal was a recipe-level GHG footprint estimate. Getting there required four chained tasks:

1. **Tagging** — label each token in an ingredient description as `ingredient`, `quantity`, `unit`, or `other`
2. **Classification** — map the ingredient tokens to a food class in the FoodEx2 taxonomy, using an EFSA pretrained model (after translating Dutch tokens to English via Microsoft Translator)
3. **Conversion** — convert quantity and unit tokens to grams, using the Portie-online database and NEVO food coding system
4. **Emission allocation** — look up the GHG emission distribution for the predicted food class in the SHARP knowledge base, then compute a mass-weighted footprint for the full recipe

Errors compound: a wrong tag in step 1 produces a wrong ingredient string in step 2, which breaks the mass lookup in step 3. The tagger quality matters a lot.

---

## Iterative feature engineering for the tagger

The tagger is a decision tree (scikit-learn CART) trained to assign a label to each token. Rather than designing all features upfront, the approach was to start minimal and add features that addressed specific residual errors — one round at a time.

### Round 1: position and length (T_sti)

The first tagger used six binary features: whether the description has 1, 2, or 3 tokens total, and whether the current token is at position 0, 1, or 2. The idea was that ingredient descriptions have a predictable structure — quantities come first, units come second, ingredients follow.

This worked better than expected. Quantities and units were cleanly separated: not a single quantity token was tagged as unit or vice versa. Ingredient recall was the weak point at 88.6%, leaving the tagger with an F1 of 94.2%.

The main failure mode: descriptions like `olijfolie om in te bakken` ("olive oil for frying purposes"). The token `olijfolie` sits at position 0, just like a quantity would — so it got tagged as a quantity, and the filler words `om in te bakken` were tagged as ingredients.

### Round 2: add is_numerical (T_num)

One feature added: whether the token looks like a number. The hypothesis was simple — quantities are usually numerals.

The result: ingredient-tokens tagged as quantities dropped by 476 in the validation set. Back to `olijfolie om in te bakken`: `olijfolie` at position 0 is now correctly tagged as an ingredient because it isn't numerical. F1 improved to 95.9%.

The residual problem shifted to unit/ingredient confusion. Prepositions and conjunctions like `om`, `in`, `te` were still being tagged as units.

### Round 3: add POS features (T_pos)

The tokens most often misclassified by T_num were inspected by their Part-of-Speech tags. Three POS tags appeared consistently among the top errors: `adj`, `adp` (adposition/preposition), and `cconj` (coordinating conjunction). Three binary features were added — one per POS tag.

Ingredient recall jumped from 92.5% to 98.0%. `om in te bakken` tokens are now correctly tagged as ingredients since they carry adposition and conjunction tags. Final F1: 97.7%.

<table>
  <thead>
    <tr><th>Tagger</th><th>Features added</th><th>Key error fixed</th><th>F1</th></tr>
  </thead>
  <tbody>
    <tr><td><strong>T_sti</strong></td><td>Position, description length</td><td>—</td><td>94.2%</td></tr>
    <tr><td><strong>T_num</strong></td><td>+ is_numerical</td><td>Non-numeric tokens at position 0 mislabeled as quantity</td><td>95.9%</td></tr>
    <tr><td><strong>T_pos</strong></td><td>+ adj, adp, cconj POS tags</td><td>Prepositions and conjunctions mislabeled as unit</td><td>97.7%</td></tr>
  </tbody>
</table>

The cascade effect was visible downstream too. For `olijfolie om in te bakken`, T_sti classified the ingredient as "Snacks other than chips and similar", T_num improved to "Olives for oil production", and only T_pos got to the correct class: "Olive oils". A tagger error that dropped one ingredient token was enough to flip the food classification entirely.

---

## Downstream results

**Classification** was evaluated on a manually annotated subset of 10 recipes (120 ingredients). The best configuration — T_pos with classification threshold P_t = 25% — reached 77% accuracy. The threshold choice mattered: too low and kitchen utensils (e.g. `staafmixer`, hand mixer) were classified as food; too high and common ingredients like `water` and `ui` (onion) were rejected as non-food.

**Conversion** fared better. With the correct food class, the Portie-online database could look up unit-to-gram conversions even for implicit units — `2 tomaten` ("2 tomatoes") resolved to 370 grams via an items-to-mass lookup. Mass ratio reached up to 97%.

**Emission allocation** revealed the hard limit of the pipeline. The design used a conservative fallback — if an ingredient couldn't be classified, assume the worst-case GHG bounds from the knowledge base. In practice this wasn't enough. When a high-mass ingredient was misclassified, the error overwhelmed the fallback. In one recipe, `karbonades` (pork chops) was classified as veal — which has a lower bound of 38 kgCO2eq/kg instead of the correct 7 kgCO2eq/kg for pork. The conservative assumption couldn't compensate for that.

The conclusion: a proof-of-technology, not a proof-of-concept. All sub-tasks worked to a measurable degree, but end-to-end conservative footprint bounds were not reliably achieved.

---

## Lessons learned

- **Simple rule-based models can punch above their complexity class.** Ten features, a decision tree, 97.7% F1. The iterative feature engineering approach — start minimal, inspect residuals, add targeted features — got more mileage than upfront feature design would have.
- **Errors compound downstream.** The tagger's misses didn't just reduce tagging accuracy; they directly corrupted classification inputs. Understanding how a stage-1 error propagates is as important as measuring stage-1 accuracy in isolation.
- **The real bottleneck is the knowledge graph gap.** Tagging and conversion are solvable. The hard problem is the chain of ontology mappings: Dutch ingredient text → English → FoodEx2 → NEVO → SHARP KB. Each step introduces noise, and there is no validated end-to-end Dutch ingredient-to-emission dataset to calibrate against.

The code is on [GitHub](https://github.com/robzwitserlood/NL-recipe-sustainability).
