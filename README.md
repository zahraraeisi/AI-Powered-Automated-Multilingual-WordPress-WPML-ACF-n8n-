# Building a Fully Automated Multilingual System for WordPress — Even for Sites with Complex ACF Structures

This repository is an experiment in building an **end-to-end automated translation pipeline** for WordPress websites that use:

* Custom post types (e.g. `property`)
* Complex **ACF** field groups (repeaters, relationships, galleries, rich text, etc.)
* **WPML** for multilingual content
* **n8n** as the orchestration / workflow engine
* An LLM (e.g. OpenAI) for high-quality translation & content rewriting

> ⚠️ Status: this is **not** a finished product. It’s a working concept / prototype that currently handles **one property at a time** and proves that we can automatically:
>
> * read a Farsi (fa) property
> * translate the main content into English (en)
> * rebuild a valid ACF payload for the target language
> * create/update the translated post via the WordPress REST API

The next step is to turn this into a **looping workflow** that keeps *all* properties in sync across languages.

---

## 1. Problem this workflow tries to solve

Multilingual plugins like WPML work great for **simple pages** and posts.
But as soon as you have:

* A custom post type like `property`
* 20+ ACF fields (relations, galleries, checkboxes, textareas, etc.)
* Different languages (Farsi → English → German)
* And a lot of content to translate…

…**manual translation becomes painful**:

* You have to recreate or sync every ACF field by hand
* You must keep relationships (e.g. `city_ref`) and galleries intact
* You must avoid accidentally overwriting the original language
* You want SEO-friendly translated slugs and titles
* You want consistency across hundreds of posts

This project is a **proof of concept** of how to automate that pipeline using **n8n + WP REST + WPML REST + ACF + LLM**.

---

## 2. High-level idea

The workflow is built around one simple idea:

> “Take a property in the original language (fa), ask the LLM for a clean English version, merge the translation with the existing ACF structure, and send a single valid JSON payload to WordPress to create/update the translated post.”

At the moment, the workflow focuses on:

* **Input:** A single Farsi `property` post (ID = 318 in the tests)
* **Output:** A corresponding English `property` post with:

  * Translated `title`, `content`, `slug`
  * ACF fields rebuilt and ready to be synced in the next iterations

---

## 3. Tech stack

* **WordPress** (HomeGermany24 test instance)
* **Custom Post Type:** `property`
* **ACF Pro** for custom fields
* **WPML** (Multilingual CMS + ACF Multilingual + String Translation)
* **WordPress REST API** (`/wp-json/wp/v2/property`)
* **WPML REST API** for translation relationships
* **n8n** self-hosted (1.111.x) as the workflow engine
* **LLM** (e.g. OpenAI) for translation and content shaping

---

## 4. Current workflow architecture (MVP)

> Node names below are conceptual; they may differ slightly from the actual JSON export.

### 4.1. Get original property

* **Input:** Original property ID in Farsi (e.g. `318`)
* **Action:**

  * `GET /wp-json/wp/v2/property/318?lang=fa`
* **Output:**

  * `title_fa`
  * `content_fa`
  * `slug_fa`
  * `acf_fa` (full ACF structure of the original property)

### 4.2. Prepare data for the LLM

Code node: **Prepare Original**

* Extracts the minimal but structured JSON we want to send to the LLM:

  * Title, content, short description
  * Key ACF text fields that should be translated (e.g. `description`, maybe `district`, etc.)
* Keeps technical / structural ACF fields intact, such as:

  * `city_ref` (relationship to city CPT)
  * `gallery` (media IDs and metadata)
  * boolean flags (`Furnished`)
  * checkboxes / arrays (`amenities`, `utilities_included`, etc.)

### 4.3. LLM translation

Node: **Translate to EN (LLM)**

* System + user prompt instructs the LLM to return **valid JSON**, e.g.:

```json
{
  "title_en": "Two-bedroom Freiburg – \"Near the Park\" (Freiburg)",
  "content_en": "<p>Furnished two-bedroom near the park and tram.</p>",
  "slug_en": "two-bedroom-freiburg-near-park-frei",
  "acf_en": {
    "description": "<p>Furnished two-bedroom near the park and tram.</p>",
    "address": "",
    "district": "",
    "transport_nearby": "Tram"
  }
}
```

* The LLM only translates **textual fields**.
* It does **not** try to replicate IDs, relationships, or media: those come from the original `acf_fa`.

### 4.4. Parse and merge ACF (code node)

Node: **Prepare EN + ACF Final**

* Takes:

  * `acf_fa` (original ACF structure)
  * `acf_en` (translated text fields from LLM)
* Produces:

```json
{
  "original_id": 318,
  "title_en": "...",
  "content_en": "...",
  "slug_en": "...",
  "acf_en_final": {
    // structural fields copied from Farsi
    "city_ref": [...],
    "gallery": [...],
    "property_type": "wg",
    "status": "rent",
    "price": "1050",
    "area_sqm": "61",
    "rooms": "2",
    "Duration": "کوتاه‌مدت",
    "occupants": "چهار نفر",
    "budget_range": "low",
    "utilities_included": [...],
    "Furnished": true,
    "target_group": [...],
    "district": "Wiehre",
    "amenities": [...],
    "transport_nearby": "Tram Holzmarkt",
    "virtual_tour_embed": "...",
    "map_link": "...",
    "video_pr_url": "...",
    "ودیعه_دپوزیت": "200000",
    "cleaning_fee": "",
    // textual description replaced by EN translation (later step)
    "description": "<p>...</p>"
  }
}
```

So we keep the **structure** of the original ACF, but gradually replace **only the fields that make sense to translate**.

### 4.5. Check if EN translation already exists

Node: **Check EN Exists (IF)**

* Logic:

  * Query WP / WPML to see if the original post (`318`) already has an English translation.
  * If yes:

    * Get `en_id` (e.g. `1279`) and set `has_en = true`.
  * If not:

    * `has_en = false`, `en_id = null`.

### 4.6. Branch: create or update EN property

* **If `has_en === false`**
  Node: **Create EN Property**

  * Makes a `POST` request to `/en/wp-json/wp/v2/property` with a **clean JSON body**:

    ```json
    {
      "title": "Two-bedroom Freiburg – \"Near the Park\" (Freiburg)",
      "content": "<p>Furnished two-bedroom near the park and tram.</p>",
      "slug": "two-bedroom-freiburg-near-park-frei",
      "status": "draft",
      "acf": { ...acf_en_final... }
    }
    ```

  * The current prototype has gone through a few iterations and right now focuses on getting the **request structure** completely valid (no JSON errors, no stringified JSON inside JSON).

* **If `has_en === true`**
  Node: **Update EN Property**

  * Makes a `POST` / `PUT` request to `/en/wp-json/wp/v2/property/{en_id}` with:

    ```json
    {
      "title": "...",
      "content": "...",
      "acf": { ...acf_en_final... }
    }
    ```
  * This allows **incremental updates** if the Farsi original changes.

> At this stage of the project, most of the work has been focused on:
>
> * Getting the ACF structure right
> * Avoiding invalid JSON errors in n8n’s HTTP Request node
> * Verifying that WordPress accepts the payload and creates a valid post

---

## 5. Current limitations

This is a **work in progress**, not a ready-to-ship plugin.

Known limitations:

* ✅ Handles one property ID at a time (no bulk loop yet)
* ✅ Focused on **Farsi → English** only (German later)
* 🚧 WordPress sometimes creates empty EN posts when the JSON body is wrong — this is part of the debugging process captured in the repo
* 🚧 ACF fields are being mapped gradually; not all text fields are translated yet
* 🚧 WPML translation linkage (`trid`, `source_language_code`, etc.) needs more automation to be fully reliable
* 🚧 No front-end UI yet — everything is driven from n8n

---

## 6. Roadmap & ideas for further development

Some ideas for how this workflow can evolve:

### 6.1. Full translation loop (batch mode)

* Add a node that:

  * Fetches **all** `property` posts in Farsi that do **not** have an EN translation yet
  * Iterates over them in n8n (pagination + loop)
* For each one:

  * Run the full “translate + merge ACF + create/update” pipeline

Result: a **one-click “translate all properties to English”** button.

### 6.2. Change detection & sync

* Track when the original Farsi property is updated (`modified` date, checksum, or a custom ACF flag).
* Only re-translate when:

  * Content actually changed, or
  * A specific field is edited (e.g. `description`)
* Mark EN translations as “out of sync” when source changes.

### 6.3. Support more languages (e.g. German)

* Extend the workflow to:

  * `fa → en` (current step)
  * `fa → de` or `en → de` for German
* Use the same ACF mapping logic but different LLM prompts per language.

### 6.4. Smarter ACF field strategy

* Define a **configuration map** for fields:

  * `translate`: e.g. `description`, `district`, maybe `address`
  * `copy`: e.g. `price`, `area_sqm`, `rooms`, `map_link`
  * `copy-relationship`: e.g. `city_ref`, `gallery`
  * `ignore`: any internal or temporary fields
* Make it easy to add new fields without changing the core code node.

### 6.5. Better error handling & logging

* Central “Error Logger” node:

  * Log failed HTTP requests
  * Log invalid JSON from the LLM
  * Log ACF field mismatches
* Optionally send notifications to Slack / Telegram when a translation fails.

### 6.6. Turn it into a shareable template

* Clean up the workflow
* Replace hardcoded domain / IDs with environment variables:

  * `WP_BASE_URL`
  * `WP_USERNAME` / `WP_APP_PASSWORD`
  * `SOURCE_LANG`, `TARGET_LANG`
  * `POST_TYPE`
* Export as an **n8n template** that other WordPress + WPML users can import and adapt.

---

## 7. How to use this (conceptually)

> ⚠️ This section assumes you are comfortable with WordPress REST and n8n. The workflow JSON in this repo is meant as a starting point, not a plug-and-play solution.

1. **Clone the repo** and import the n8n workflow JSON.
2. Set your environment variables / credentials:

   * WordPress base URL (with `/en` if needed)
   * Application password or API auth
   * LLM API key
3. Adjust:

   * The source and target languages
   * The custom post type (`property`)
   * The ACF field names to match your setup
4. Start with **one test property ID**
5. Run the workflow manually and verify:

   * The EN post is created
   * The ACF structure is accepted
   * No JSON errors occur
6. Gradually move towards:

   * Full ACF mapping
   * WPML translation linking
   * Batch / loop mode

---

## 8. Why this might be useful

If you’re working with:

* Real-estate sites
* Directory / listing websites
* Any content-heavy multilingual WordPress setup with ACF
* And you want to use AI, but **not manually copy/paste every single translation**

…this pattern can help you design your own **robust translation pipeline** where:

* ACF structure stays stable and technical
* Only the text content is touched by AI
* WPML keeps the languages linked
* n8n acts as the brain to orchestrate everything

---

Feel free to fork, experiment, and adapt this workflow to your own WordPress + WPML + ACF setup.

If you build extra features (batch processing, better WPML linking, extra languages), contributions and ideas are very welcome. 🙌
---

## 👩‍💻 Author

**Zahra Raeisi**
Automation & Applied AI with n8n
GitHub: [https://github.com/zahraraeisi](https://github.com/zahraraeisi)
LinkedIn: [https://www.linkedin.com/in/zahraraeisi](https://www.linkedin.com/in/zahraraeisi)
