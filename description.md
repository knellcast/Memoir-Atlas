## Inspiration
I've lost count of the times I've tried to recall a tiny travel detail — that incredible bowl of ramen in a Tokyo alley, the name of a hidden bar in Lisbon, the exact spot where I watched a perfect sunset. Our phones hold thousands of photos and scattered notes, but none of it is searchable by *feeling*, *place*, or *flavor*. I wanted an agent that not only remembers everything I've experienced while traveling, but lets me ask for it like I'd ask a friend: "Where did I eat those amazing tacos near Coyoacán market?" That gap — between rich, unstructured human memory and rigid digital storage — is what inspired Memoir Atlas.

## What it does
Memoir Atlas is an AI travel companion powered by Google Cloud Agent Builder and Gemini 3. It captures your travel moments through natural conversation — a voice note, a quick rating, a place name — then structures them into a living, searchable memory inside MongoDB. Later, you can ask questions like:

- "What was the best seafood I had in Lisbon within walking distance of the ocean?"
- "Show me all the places I took photos of sunsets and rated over 4 stars."
- "I'm going back to Mexico City next week. What from my past trips should I revisit, and are they still open?"

The agent plans and executes a multi-step process: it geocodes fuzzy locations, enriches the memory with weather data, inserts it into MongoDB (only after your approval), and later uses MongoDB's geospatial and text search — via the MongoDB MCP server — to recall exactly what you're looking for. For planning a return trip, it can even check current opening hours for your old favorites.

## How we built it
- **Platform**: Google Cloud Agent Builder provided the low-code orchestration layer, allowing rapid prototyping with **Gemini 3 as the reasoning engine that dynamically decides which tools to call and in what order, adapting when geocoding returns ambiguous results.**
- **Partner Superpower**: The **MongoDB Atlas** MCP server is the backbone. We store every travel memory as a flexible document (geolocation, tags, ratings, free-text description, weather) and use MongoDB's `$geoNear`, text scoring, and aggregation pipelines to answer complex natural-language questions that no traditional journal app could handle.
- **Tools**: The agent is equipped with four tools:
  1. **MongoDB MCP** — for insert, find, aggregate, and text search.
  2. **Geocoding API** (Google Maps) — to convert place names into coordinates.
  3. **Weather API** (Open-Meteo) — to automatically capture the weather at the moment of the memory.
  4. **Places API** (Google Maps) — to check if a venue is still open during trip planning.
- **System Prompt**: We crafted a detailed, guard-railed prompt that enforces a strict multi-step workflow: always geocode → enrich → show the user a summary → wait for explicit confirmation → then write to MongoDB. The agent never invents data and never performs a write action without human approval.

### A custom retrieval engine (pre-computed + dynamic scoring)
We designed a hybrid ranking system that pre-computes a static quality score at capture time and then adds dynamic personalization during search.

**1. Pre-computed base score (stored inside each memory document):**

$$
S_{\text{base}} = w_1 \cdot \frac{1}{1 + e^{-(r-5)}} + w_4 \cdot e^{-\gamma \cdot t}
$$

- \\( r \\): user rating (1–10), passed through a sigmoid to separate top experiences from average ones.
- \\( t \\): age of the memory in days. The exponential decay ensures recent memories receive a gentle freshness boost. We set \\( \gamma = 0.00385 \\) so that a memory loses half its freshness after about 6 months.
- \\( w_1, w_4 \\): tunable importance weights (normalized so all four weights sum to 1).

**2. Dynamic query-time adjustment (computed via MongoDB aggregation at search):**

$$
S_{\text{final}} = S_{\text{base}} + w_2 \cdot e^{-\lambda \cdot d} + w_3 \cdot \text{sim}(q, \text{doc})
$$

- \\( d \\): distance (km) from the queried location, with decay constant \\( \lambda = 0.5 \\) (50% relevance drop after ~1.4 km).
- \\( \text{sim}(q, \text{doc}) \\): a similarity score in \\( [0, 1] \\) between the user's query \\( q \\) and the memory's tags + description.

This two-stage design keeps the agent fast: the heavy rating/freshness math is pre-stored, and only the spatially/textually sensitive components are computed live. A vague request like *"best rainy-day café near the river"* returns exactly the right memory in under a second.

### MongoDB aggregation: running the formula live
We don't fetch all memories and score them in application code. Instead, we push the entire formula into a MongoDB aggregation pipeline, called through the MCP server. For a query like *"best rainy-day café within 2 km of Montmartre"*, the agent invokes the **MongoDB MCP server's `aggregate` tool**. Gemini extracts the query terms and coordinates, substitutes the tuned weights ( \\( w_2 = 0.25, w_3 = 0.20, \lambda = 0.5 \\) ), and passes this pipeline:

```javascript
[
  // 1. Geo-filter and compute distance (2dsphere index); distanceMultiplier converts meters → km
  { $geoNear: {
      near: { type: "Point", coordinates: [ 2.3429, 48.8867 ] },
      distanceField: "distance_km",
      distanceMultiplier: 0.001,
      maxDistance: 2000,        // meters
      spherical: true
  }},
  // 2. Lightweight text similarity: overlap between query terms and the memory's tags,
  //    normalized to [0, 1]. ($text can't be combined with $geoNear — see Challenges.)
  { $addFields: {
      text_score: {
        $min: [ 1, { $divide: [
          { $size: { $setIntersection: [ "$tags", [ "rainy-day", "café", "cozy" ] ] } },
          3
        ]}]
      }
  }},
  // 3. Compute dynamic adjustment and final score (weights substituted by the agent)
  { $addFields: {
      final_score: { $add: [
        "$base_score",
        { $multiply: [ 0.25, { $exp: { $multiply: [ -0.5, "$distance_km" ] } } ] },
        { $multiply: [ 0.20, "$text_score" ] }
      ]}
  }},
  // 4. Sort by final_score descending, limit to top results
  { $sort: { final_score: -1 } },
  { $limit: 5 },
  // 5. Return only necessary fields
  { $project: {
      title: 1, description: 1, rating: 1, location: 1, final_score: 1
  }}
]
```

## Challenges we ran into
<!-- TODO: verify these match what actually happened, and add your own -->
- **`$geoNear` and `$text` refuse to share a pipeline.** Our original design used MongoDB's text index score as the similarity term, but both `$geoNear` and `$text` must be the *first* stage of an aggregation pipeline — you can't have both. We redesigned the similarity term as a normalized tag-overlap computed with `$setIntersection`, which keeps the whole formula inside one pipeline and stays in the \\( [0,1] \\) range our weights expect.
- **Units are sneaky.** `$geoNear` returns distances in meters while our decay constant \\( \lambda \\) is calibrated per kilometer — with meters, \\( e^{-0.5 d} \\) underflows to zero for anything farther than your doorstep. `distanceMultiplier: 0.001` fixed it, but only after some very confusing "everything scores the same" debugging.
- **Keeping the agent honest.** Early versions of the agent would happily invent a rating or skip geocoding. The guard-railed system prompt (geocode → enrich → summarize → confirm → write) took many iterations before Gemini followed it reliably, especially the "never write without explicit confirmation" rule.

## What we learned
<!-- TODO: verify / personalize -->
- **Push computation to the database.** Moving the entire scoring formula into an aggregation pipeline (instead of fetching documents and scoring in the agent) cut response time dramatically and kept the agent's context small.
- **Pre-compute what doesn't change.** Splitting the score into a static part (stored at capture time) and a dynamic part (computed at query time) is a simple idea that made the whole system feel instant.
- **Agents need rails, not vibes.** A precise, step-ordered system prompt with explicit confirmation gates matters as much as the tools themselves — Gemini 3 is a strong planner, but only when the workflow contract is unambiguous.

## What's next for Memoir Atlas
<!-- TODO: fill in — e.g. photo ingestion with multimodal Gemini, Atlas Vector Search for semantic similarity instead of tag overlap, shared memories with travel companions -->
