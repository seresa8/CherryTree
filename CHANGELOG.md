# Changelog

All notable changes to Cherry Tree by Seresa are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [4.4.1]

### Changed
- Updated WordPress tested-up-to version to 7.1.

## [4.4.0]

### Changed
- **Report email footer — Seresa-branded credit plus host domain** — the outbound report footer now reads "Sent automatically from Cherry Tree by Seresa (https://seresa.io) hosted on `<host>`", crediting the plugin maker with a fixed seresa.io link and naming the client site's host domain, instead of "Sent automatically by Cherry Tree for `<site>` (`<url>`)". See `templates/emails/_layout.php`.
- **Seed idea stage — five quality gates added from AI-session feedback** — (1) `branch_slug` must be a baked pillar slug (no more invented slugs that create phantom branches); (2) short_answer gate raised to 80–100 words (target 80–90) so seed-final only ever trims; (3) keywords must be single hyphenated tokens (`cart-abandonment-rate` not `cart abandonment rate`) — the sanitizer now auto-hyphenates, and the pre-post gate rejects spaces in tags; (4) every seed requires ≥1 internal link (`related_post_ids`); (5) `research.citable_core` is required, not optional. See `stages/seed-ideas.md`, `src/API/class-seed-data-sanitizer.php`.

### Fixed
- **Health checker — passing API test now suppresses stale Bing/IndexNow dashboard warnings** — `check_engine_failures()` counted log errors from the last 24h but never checked whether a successful connection test had landed after the most recent error, so a passing "Verify" still showed "Bing API failing" until the logs aged out. It now mirrors the Google-auth suppression logic via a new `engine_recently_verified()` method, and the connection tester clears the health-checker cache after testing so the dashboard reflects results immediately. See `src/Indexing/Admin/class-health-checker.php`, `src/Indexing/API/class-api-connection-tester.php`.
- **Timezone bug — seeds, branches, and articles now store and display dates in the site's configured timezone** — `created_at`/`updated_at` relied on MySQL's `CURRENT_TIMESTAMP` (DB-server timezone, usually UTC) while the admin display used `wp_date(fmt, strtotime(...))` which double-shifted the offset, showing times hours behind the site's local time (e.g. Sep 10 18:14 instead of Sep 11 02:14 on SGT +8). Storage now uses `current_time('mysql')` explicitly in all three repositories (seeds, branches, queue); display now uses `mysql2date()` which correctly interprets the stored WP-local string. Affected: `trait-seed-crud.php`, `class-branch-repository.php`, `trait-queue-crud.php`, `seeds-dashboard.php`, `class-articles-view.php`.
- **Branches API — oversized `description` no longer fails the DB write with a 500** — `Branches_Controller::sanitize_branch_data` capped `slug` (100) and `name` (200) but left `description` at `0` (unlimited), yet the column is TEXT (~65,535 bytes); a large `description` on `POST /branches` overflowed the column and returned a 500. It is now capped at 12,000 characters, matching the seeds TEXT-field fix. Found via a systematic length-field sweep during API thrash testing. See `src/API/class-branches-controller.php`.
- **Seeds API — oversized `question`/`short_answer` no longer trigger a misleading, un-retryable 500** — both fields are stored in TEXT columns (~65,535 bytes) but the sanitizer capped them at `0` (unlimited), so a pathological oversized payload passed validation and then failed the database write, returning a 500 whose message told the AI client the error was *server-side* and to *retry the same request* — a retry that could never succeed. They are now capped in `Seed_Data_Sanitizer::STRING_FIELDS` (question 2000, short_answer 12000 characters, both safely under the TEXT byte-ceiling for worst-case UTF-8), so an oversized value is truncated to fit exactly as `slug`/`branch_slug` already are. Found via live API thrash testing. See `src/API/class-seed-data-sanitizer.php`.
- **Posts Controller — sanitize structured meta arrays before storage** — `faq_pairs`, `key_claims`, and `data_points` from the API request body were JSON-encoded and stored in post meta without sanitizing individual elements. `faq_pairs` now sanitizes question (plain text) and answer (wp_kses_post); `key_claims` and `data_points` are sanitized as plain text. Also escaped the reflected `$duplicate_h2` value in the validation error message and fixed an inline comment missing its full stop. See `src/API/class-posts-controller.php`.
- **Seeds Read Controller — cast `offset` param to `(int)` for type safety** — `get_items` passed the raw `offset` param (potentially `null` or string) into the repository args without an `(int)` cast, unlike `limit` which was already cast. See `src/API/class-seeds-read-controller.php`.
- **Email Service — replaced short array syntax with long `array()` for WPCS compliance** — six instances of `[]` array literals replaced with `array()` to satisfy the WordPress `Universal.Arrays.DisallowShortArraySyntax` sniff. No logic change. See `src/Email/class-email-service.php`.
- **API Response Filter — replaced short array syntax with long `array()` for WPCS compliance** — four instances of `[]` array literals replaced with `array()` to satisfy the WordPress `Universal.Arrays.DisallowShortArraySyntax` sniff. No logic change. See `src/Security/class-api-response-filter.php`.
- **Sanitizer — replaced short array syntax with `array()` for WPCS compliance** — 10 instances of `[]` array literals replaced with `array()` to satisfy the WordPress `Universal.Arrays.DisallowShortArraySyntax` sniff. No logic change. See `src/Security/class-sanitizer.php`.
- **Queue API — `GET /published` normalises `limit` before the repository call** — `Queue_Reads_Trait::get_published` passed the raw `limit` param straight into `Queue_Repository::get_published( int|string $limit )` on the default (date) branch, so a request that reached the handler without the param (defence-in-depth against the route default being dropped) would pass `null` and raise a `TypeError`. It now coerces anything that isn't numeric or `"all"` to `100`, matching the defensive normalisation the random branch already had. See `src/API/Queue_Controller/trait-queue-reads.php`.
- **Queue API — `GET /queue` now clamps `limit` to 1-500** — `Queue_Routes_Trait::get_collection_params()` sanitized `limit` with a bare `absint`, so an authenticated caller could request an unbounded number of rows straight into the repository's `LIMIT %d`. It now clamps to 1-500 in the `sanitize_callback`, matching the `/published` route. The value was always prepared, so this is a resource guard, not an injection fix. See `src/API/Queue_Controller/trait-queue-routes.php`.
- **Queue permissions trait — imported `WP_REST_Request`** — `trait-queue-permissions.php` referenced `WP_REST_Request` in every method docblock but never imported it, so the type resolved to the non-existent `Seresa\CherryTree\API\WP_REST_Request`. Added the `use WP_REST_Request;` import. See `src/API/Queue_Controller/trait-queue-permissions.php`.
- **Claim de-duplicator — article-side comparison was silently a no-op** — `Claim_Deduplicator::check` read `$article->research` from `Queue_Repository::get_published()`, but that list query deliberately omits the large `research` payload for performance, so the property was always absent and every published article was skipped. New ideas were only ever de-duplicated against published *seeds*, never against published articles. The article loop now loads the full record with `get_by_id()` before comparing, matching the data the seed path already had. Repositories are now lazily created (and optionally injectable) so the comparison logic is unit-testable without a live database. See `src/Utilities/class-claim-deduplicator.php`.
- **Claim de-duplicator — article scan called a non-existent repository method** — `Claim_Deduplicator::check` gated the published-article loop on `$queue_repo->table_exists()`, but `Queue_Repository` has no `table_exists()` method (only `Seed_Repository` and `Schema` do), so any call reaching that branch with real claims raised a fatal `Error: Call to undefined method`. The existing tests never hit it (they short-circuit on empty input). It now gates on the real static `Schema::table_exists()`, which checks exactly the queue table. See `src/Utilities/class-claim-deduplicator.php`.
- **Source liveness verifier — HTTP status compared as int** — `Source_Liveness_Verifier::check_single_url` used strict comparisons (`405 === $code`, `in_array( $code, LIVE_CODES, true )`) against a value from `wp_remote_retrieve_response_code()`, which can be returned as a string. When it was, the HEAD→GET fallback for 405-only servers never fired and every source resolved to `dead`. The retrieved code is now cast with `(int)` at both points, matching the idiom used across the Indexing/WAF clients; the returned array shapes also now document their `unit` key. See `src/Utilities/class-source-liveness-verifier.php`.
- **Queue API — `POST /queue` guards the advisory duplicate-claim warning against a null lookup** — after a successful insert, `Queue_Controller::create_item` re-fetched the row and, in the new near-duplicate advisory block, assigned `$item->duplicate_claim_warnings` without confirming the fetch returned an object; on the (rare) case where `get_by_id()` returns `null` this raised a fatal `TypeError` (assign property on null). The block is now guarded with `$item &&`. See `src/API/class-queue-controller.php`.
- **Seeds API — `PUT /seeds/{id}` with an empty or non-JSON body no longer fatals** — `Seeds_Controller::update_item` passed `get_json_params()` straight into `Seed_Data_Sanitizer::sanitize()`, which type-hints `array`, so a request with no JSON body (where `get_json_params()` returns `null`) raised an uncaught `TypeError` (HTTP 500). The body is now cast with `(array)` before sanitizing, matching the defensive cast already used one line later for `collect_ignored()`; an empty body now falls through to the existing "no updatable fields" 400. See `src/API/class-seeds-controller.php`.
- **Client — redirect guard now refuses a same-host `https`→`http` downgrade** — `Transport._request` compared only the redirect target's *hostname*, so a same-host redirect that dropped the scheme to `http` (a MITM injection or a misconfigured server) was followed, sending the live `X-API-Key` in cleartext to that host — the exact leak the method's own docstring claimed to prevent. It now also requires the target scheme to stay `https`; a cross-host redirect (still "different host") and a same-host cleartext downgrade are refused with distinct messages. Same-host `https` canonicalisations (trailing slash, `www`) are still followed, so healthy installs are unaffected. Applied byte-identically to both `transport.py` twins. See `Transport._request`.
- **CTA settings — inline JS boolean uses `wp_json_encode`** — `section-ctas.php` echoed a raw PHP ternary (`true`/`false`) into an inline `<script>` block; switched to `wp_json_encode()` for proper JS encoding. See `templates/admin/settings/section-ctas.php`.
- **Client — `claim` / `claim_next` now require `id` in the response, raising a typed `ClaimError`** — both guarded the body shape and required `claim_token`, but neither required `id`. Yet the very next step builds a `ClaimSession` whose `__post_init__` reads `item["id"]`, so a claim body missing `id` leaked a raw `KeyError` outside the `CherryTreeError` family — the exact gap the 4.1.6 shape guards were added to close. Both methods now raise `ClaimError("No id …")` at the transport boundary. Applied byte-identically to both `_queue.py` twins. See `Transport.claim` / `Transport.claim_next`.
- **Client — claim-session cleanup warnings now go to stderr, not stdout** — when a `claim_session` / `claim_next_session` block raised and the compensating `release()` itself failed, the `WARNING:` line was printed to **stdout**, where it could corrupt a stdout-parsed data stream. It now writes to `sys.stderr`, matching every other diagnostic in the client (`_http.py`, `client.py`). Applied to both `cherry_tree_client/transport.py` twins. See `Transport.claim_session` / `Transport.claim_next_session`.
- **DB version bump (1.12.0) ensures stale-seed cron is registered on update** — the `cherry_tree_cleanup_stale_seed_processing` cron event (15-minute sweep that releases seeds stuck in processing) was only scheduled during plugin activation. Installs activated before the cron was added never received it, leaving seeds permanently locked in processing with no automatic release. Bumping `CHERRY_TREE_DB_VERSION` to `1.12.0` triggers the activator re-run on next page load after update, scheduling any missing cron events — including the seed sweep — without requiring a manual deactivate/reactivate.

### Refactored
- **Queue API — split `class-queue-controller.php` into composing traits** — the controller had grown to 1038 lines, past the project's 500-line hard ceiling. The endpoint handlers moved into focused traits under `src/API/Queue_Controller/`: `trait-queue-routes.php` (route registration + collection params), `trait-queue-permissions.php` (the four permission callbacks), `trait-queue-reads.php` (`get_items`/`get_next_item`/`get_item`/`get_published`), `trait-queue-writes.php` (`create_item`/`update_item`/`delete_item` + subcategory helper), and `trait-queue-claims.php` (`update_status`/`claim_item`/`release_item`/`perform_claim`). `class-queue-controller.php` (98 lines) keeps the constructor and namespace and composes the five traits — mirroring the `Queue_Repository` trait layout. Public class, routes, and behaviour are unchanged. See `src/API/class-queue-controller.php`.
- **Client — split `transport.py` into `transport.py` + `_queue.py` + `_publish.py`** — the 533-line `transport.py` was past the project's 500-line hard ceiling. Queue operations (`list_queue`, `claim`, `claim_next`, `release`, `claim_session`, `claim_next_session`, `ClaimSession`) moved to `_queue.py` (245 lines); publish/discovery operations (`publish_article`, `list_published`, `seed_stats`, `find_published_by_keyword`) moved to `_publish.py` (131 lines). `transport.py` (229 lines) keeps the shared plumbing and boot, composes the two as mixins, and re-exports `ClaimSession` so every existing import keeps working. Applied byte-identically to both twins.

### Changed
- **Settings page — Status Flow moved to the top; API Endpoints list completed** — the shared Seed/Article Status Flow explainer now renders as the first section of the settings page (extracted to `templates/admin/settings/section-status-flow.php`, with a matching "Status Flow" bookmark) instead of being buried in the API reference cluster. The API Endpoints tables now also list the previously undocumented `/diagnostic` (WAF/connectivity check), `/help` (help desk), and `/seeds/stats` (seed counts) endpoints. See `Admin_Settings::get_api_documentation()` / `get_seeds_api_documentation()`. The WAF-obfuscated internal seed-lock/claim endpoints remain intentionally undocumented.
- **Seeds admin — Status dropdown removed from the filter bar** — the status filter is now controlled solely by the tab/stat cards (Idea, Queued, Processing, Review, Published, Rejected) above the list; the redundant "All Statuses" `<select>` was counterintuitive alongside the tabs. The active tab's status is preserved through the Branch filter via a hidden field, so the two still combine. See `templates/admin/seeds-dashboard.php`.
- **Seeds admin — Sources now break onto separate lines in the view modal** — a seed's Sources list (View, any status) rendered every URL on one run-together physical line because the newline join collapsed in HTML; each source is now escaped individually and joined with `<br>` so it sits on its own line. See `assets/js/admin/seeds/modal.js`.
- **Seed status-count cache TTL reduced from 30 minutes to 5 minutes** — the dashboard's `get_status_counts()` transient was stale long enough to show ghost numbers (e.g. "242 Ideas" when the real count was 20). The underlying query is a trivial `COUNT(*) GROUP BY status` on a small table; 5 minutes still prevents hammering while keeping the dashboard honest. The queue repository already used 60 seconds.

### Added
- **Seeds dashboard — multi-select and bulk actions** — the Seeds pipeline now has the same checkbox multi-selection as Articles. Idea, Queued, Review, and Published tabs show a select-all header checkbox and per-row checkboxes; checking one or more reveals a bulk-actions bar with status buttons (Approve/Reject for ideas, Reject for queued, Publish/Reject for review, Reject for published). Backed by a new `cherry_tree_seed_bulk_update_status` AJAX endpoint on `Seeds_Ajax_Handler`. See `templates/admin/seeds-dashboard.php`, `assets/js/admin/seeds/bulk.js`.
- **`GET /diagnostic` — WAF endpoint health check exposed via REST API** — the AI client can now run the loopback diagnostic without the owner visiting wp-admin. Returns `{ran_at, all_pass, blocked, results[]}` — if `all_pass` is true the AI says nothing; if endpoints are blocked it guides the owner through fixing them using the help system. API-key gated, read-budget, read-only (no auto-fix or settings changes are exposed — those stay admin-only). Diagnostic loopback probes are exempt from rate limiting (the `X-CT-Diagnostic` header skips `is_limited` and `record` in `Rate_Limiter`), so the whole health check costs one read unit instead of ~10. Added to the content skill boot sequence (SKILL.md step 5, runs every session start) and the setup flow (setup.md, runs before Build & Lock). Six new help-faq.json entries cover what WAF blocking is, how to fix slugs, User-Agent filtering, the auto-fix button, and why a skill rebuild is needed after a fix. See `src/API/class-diagnostic-controller.php`.
- **`GET /seeds/stats` — a single cheap, correct source for seed counts per status** — the seed API gained a status-count aggregate so a caller never has to enumerate seeds to learn how many exist. It returns `{ counts: { idea, queued, processing, review, published, rejected }, total }` from one cached `GROUP BY COUNT(*)` (already used by the admin dashboard via `Seed_Repository::get_status_counts()`), so every bucket is exact and uncapped — no rows loaded, no page limit. This fixes the setup/content AI reporting a wrong "published" figure: with no counts endpoint it fell back to listing `GET /seeds?status=published` and counting the array, which is hard-clamped to 500, so e.g. 743 published seeds read as 500. The new endpoint is a small dedicated `Seed_Stats_Controller` (the seeds controller is already over the file-size ceiling) with the same public-read API_Guardian gating as the other `/seeds` GETs; the Python client gains `seed_stats()` (both twins), and `GET /help` documents it (`api-seeds-stats`) with an explicit "never count the list" warning on `api-seeds-read`. See `src/API/class-seed-stats-controller.php`.
- **Help API now surfaces the public GitHub links, and the changelog is published publicly** — the `GET /help` index response gains a `links` block pointing the AI at the public source repo (`seresa8/CherryTree`): the repo/README, the full `CHANGELOG.md`, the releases list, the latest release, and issues — so the AI can read version history and known issues itself instead of guessing. URLs are built from `GitHub_Release_Client::OFFICIAL_REPO` so they can never drift from the update channel, and a one-line pointer is echoed in every response's `usage` block. To back the changelog link, the release process (`release.sh` → `sync_public()`) now copies the plugin's authoritative `CHANGELOG.md` to the public repo root on every release, giving it a stable browsable URL (`.../blob/main/CHANGELOG.md`) with no duplication or drift. See `src/API/class-help-controller.php`.
- **Help admin page — one-click copy of a self-contained AI help prompt** — a new **Cherry Tree → Help** menu item (below Settings) whose whole job is a single button: it copies a short few-line message the owner pastes into any AI. The message bakes in the site URL, the `GET /help` endpoint, and the API key, so it is a working help system straight off — even in a fresh chat with no skill loaded, the AI can immediately answer the owner's Cherry Tree questions from the API. It also explains that an installable "AEO skill" (the Cherry Tree Content Skill) exists and asks the AI to help the owner install it in whatever AI they use (an Agent Skill in Claude; custom instructions/system prompt elsewhere) — kept generic so the AI guides the owner without Seresa babysitting each platform. Template `templates/admin/help.php`, wired via `Admin_Menu::render_help_page()`.
- **Built-in AI help desk — a live, searchable `GET /help` API so the content skill can answer owners about the system with no external docs** — Cherry Tree now ships its own support knowledge base and serves it over REST, so the AI (the content skill, and the setup/edit assistant) can answer "how does this work / where is that setting / why isn't my page indexed" entirely from the plugin. The endpoint has four shapes from one route: `GET /help` (a compact index of every question), `GET /help?q=<plain words>` (relevance-ranked answers **plus** a cross-indexed `related` bundle of nearby entries, so the AI usually gets the specific answer and the likely next one in a single call), `GET /help?id=<id>` (one entry + its related cluster), and `GET /help?topic=<topic>`. Every response carries a verbose `usage` block and, on a rate-limit pause, states exactly how long to wait — API responses here are deliberately informative rather than terse. Architecture is a swappable seam: a `Help_Provider` interface (`src/Help/`) with a `File_Help_Provider` reading a shipped, version-controlled data file (`src/Help/help-faq.json`, 121 AI-oriented entries across Overview, Config API, Skill Runtime, Content Quality, Settings, Indexing, Seeds & API, and Notifications), a `cherry_tree_help_entries` filter so a DB or add-on can inject more later with zero endpoint change, ranked search + cross-index in `Help_Library`, and in-process + object-cache memoisation. It is served **live, not baked into the skill**, so help updates reach every install instantly with no rebuild (deliberately kept off the `context_hash` rebuild cycle). Gated behind the API key with its own generous `Help_Rate_Limiter`: a 5-minute burst window (`CHERRY_TREE_RATE_LIMIT_HELP_BURST`, 300) that lets the AI fire a fast cluster of lookups with no waiting, plus an hourly backstop (`CHERRY_TREE_RATE_LIMIT_HELP_PER_HOUR`, 900) that only reins in sustained runaway hammering — self-healing, never a hard lockout. `GET /help` is referenced from SKILL.md (Credentials + a new *Owner support* section), `invariants.md`, all four stage files, and both `ai-prompts/setup.md` / `edit.md`, so the AI knows to consult it. See `src/Help/` and `src/API/class-help-controller.php`.
- **Setup AI now walks every owner through the WordPress Settings page as a required step** — the setup prompt gains a mandatory, un-skippable walkthrough of the admin **Settings** sections, which a non-technical owner would otherwise never discover (they start blank or at defaults and cannot be reached through the config API the AI edits). A new source-of-truth reference, `ai-prompts/settings-sections.json`, describes each section's purpose and its deep-link anchor; `Prompt_Builder` renders it into a new `{{SETTINGS_SECTIONS}}` slot in both `setup.md` and `edit.md`, building a ready-to-click link per section (`admin.php?page=cherry-tree-settings#<anchor>`, anchors matching `$ct_bookmarks` in `templates/admin/settings.php`). The three critical sections — **📍 Content Destination**, **📣 Call to Action**, **🐝 Cherry Bee — Search Engine Indexing** — are tagged `[WALK THROUGH THIS]` so the AI actively covers them; the rest are offered as "here's everything else, ask me about any of it." Fail-open: a missing or malformed reference renders a plain fallback note, never a bare token. `edit.md` carries the same reference so an already-set-up owner can be answered about any section on demand.
- **Setup/edit AI now acts as a help desk for Cherry Bee search-engine indexing** — the copyable `ai-prompts/setup.md` and `ai-prompts/edit.md` prompts gain guidance so the assistant can answer indexing questions without ever touching credentials. It explains that Cherry Bee — Search Engine Indexing lives in its own WordPress form (**Cherry Tree → Settings → 🐝 Cherry Bee**), is *not* part of the API config the AI edits, and so cannot be read, set, or saved by the AI: **IndexNow is on by default and self-configures** (nothing to do), while **Google and Bing are optional and start off**, each needing a credential the owner creates in their own account. For Google it offers to walk the owner through creating a service account and hands them two Seresa guides (create-service-account and the accounts/projects/service-accounts background), then tells them exactly where the JSON goes. `setup.md` raises this once near the end (before Build & Lock, non-blocking); `edit.md` answers on demand. Both are explicit: never ask the owner to send a credential, and never try to save one through the API.
- **Settings section bookmarks — deep-link to any settings section by URL** — the Cherry Tree Settings page now opens with a "Jump to a section" bookmark box: a row of pill links, one per section (Content Destination, Call to Action, Seed Pages, Email Reports, Klaviyo, API Key, 🐝 Cherry Bee — Search Engine Indexing, Configuration Status, Protected Categories, Allowed Pillars, API Endpoints), that scroll to that section and set a shareable URL hash (e.g. `?page=cherry-tree-settings#ct-bm-cherry-bee`). The links and the per-section scroll anchors are generated from one ordered list in `templates/admin/settings.php`, so they cannot drift; anchors carry a scroll offset so a jumped-to section clears the sticky admin bar.
- **In-admin help system — contextual "?" icons that link to documentation** — a reusable, plugin-wide help system. A new `Help\Help_Registry` renders a small cherry-ringed "?" icon next to any heading or field from a stable *help code* (`section-subsection-index`, e.g. `settings-google-indexing-1`); the icon lifts on hover (the "hovercraft" float, disabled under `prefers-reduced-motion`) and opens its documentation link in a new tab, with the help text as its tooltip and accessible label. The copy and links live in an adjacent, human-editable JSON catalogue (`src/Help/help-strings.json`) rather than in PHP, so they can be revised without a code change; the file path is filterable (`cherry_tree_help_strings_file`) so a site can point at its own catalogue, and the decoded map is filterable (`cherry_tree_help_strings`). Fail-open — a missing or malformed catalogue simply renders no icons. Styling is a shared component in `assets/css/admin/polish.css` (`.cherry-tree-help-icon`). First live placement: the **Google Indexing API** heading in Settings → Cherry Bee. See [readme/SPEC-help-system.md](../readme/SPEC-help-system.md).
- **Citable core — the article's sourced spine, authored and signed off at the idea stage (Phase 1 of the citable-core architecture)** — an AEO article exists to transmit citable claims, so those claims are now authored *up front* on the idea rather than reverse-engineered out of finished prose. The idea's `research` object gains a `citable_core` — 8–12 units, each a `{question, self-contained answer, backing stat, source}` (plus an optional `howto` step list) — which becomes the spine the article is later written *around* and the source the FAQPage/HowTo/Speakable JSON-LD derives from. Three parts this release: (1) the ideas stage (`skill-template/stages/article-ideas.md`) now authors the core; (2) a real server-side gate — the new `Security\Citable_Core_Validator`, wired into `Sanitizer` — validates it on `POST`/`PUT /queue` and returns a 400 with the exact problem when a *present* core is malformed (wrong unit count, a question without `?`, an empty answer, a source missing `url`/`name`, or fewer than half the units carrying a stat); an *absent* core still passes, so ideas created before the skill produced one keep saving; (3) the admin idea view renders the core as **cards** with each claim's citation made prominent (`assets/js/admin/items/render.js` + `actions-modals.css`), so the owner reviews and signs off on the actual sourced claims instead of a raw JSON dump. Nests inside the existing `research` JSON column — no schema migration. See [readme/SPEC-citable-core.md](../readme/SPEC-citable-core.md).
- **Citable core — the article is now written *around* the core, with a pre-publish verify pass (Phase 2)** — the downstream writing stage consumes the approved core instead of composing freely: `skill-template/stages/article-final.md` maps each unit to its own section (`<h2 id>` = the unit's `question`, first sentence = its `answer`, its `stat`/`source` woven in), and `FAQ` is **derived from the units** rather than re-authored — so the on-page FAQ, the answer-first sections, and the FAQPage/HowTo/Speakable JSON-LD all trace to the same approved questions and cannot drift. Enforcement backs the instruction: `audit_stage_1` gained a 12th check, `FAQ_HEADING_MATCH` (every core question has a matching content H2), and its answer-first / question-heading checks no longer skip a content H2 that omits its `id`. A new **`verify_core_fidelity`** pass (`cherry_tree_client/verify.py`) runs before publish and fails if any core question, source URL, or stat number is missing from the rendered article, paired with an adversarial model re-read for answer-first delivery and readability. Client twins, `invariants.md`, and the spec updated in lockstep.
- **Citable core for seeds — the seed's citation is now authored and stored at the idea stage too (Phase 3)** — seeds now carry their citation the same way articles do: authored up front on the idea, not reconstructed at write time. Previously a seed idea stored no structured citation object — the sanitizer silently dropped a `research` payload, so only flat `sources` URL strings and `related_post_ids` survived and the self-citation angle was lost. Seeds gain a seed-scale `research.citable_core` (the seed-scale parallel of an article's — **one** flagship `{answer, stat, source}` claim instead of 8–12 units) plus an optional self-citation `product_angle`, authored at idea time and reused verbatim by seed-final. Parts: (1) a new `research JSON` column on the seeds table (`Schema_Installer`; `CHERRY_TREE_DB_VERSION` → `1.13.0` so the additive migration re-runs on update); (2) a real server-side gate — the new `Security\Seed_Citable_Core_Validator`, wired into `Seeds_Controller` create/update — validates a *present* core on `POST`/`PUT /seeds` and returns a 400 naming each problem (empty `answer`, a `source` missing `url`/`name`, a malformed `product_angle`); an *absent* core still passes, so legacy seeds and source-only seeds keep saving, and a `stat` is recommended-not-required so a definitional seed is never stranded; (3) `Seed_Data_Sanitizer` now accepts and recursively sanitizes the nested `research` object; (4) the ideas/final stage prompts (`skill-template/stages/seed-ideas.md`, `seed-final.md`) author and consume it. Nests in the new column — no rebuild needed for the gate.
- **Content-skill "system update" gate — the AI now knows when its own instructions are stale** — until now the skill's boot drift check only compared the owner's *settings* (`config_hash`); a release that improved the skill's own instructions (SKILL.md, the stage files, invariants, the vendored client) shipped to WordPress but never reached an already-built skill, and nothing told the owner to rebuild. `GET /info` now also returns a `context_hash` — a SHA-256 of the context layer baked into the skill (`Skill_Assembler::context_hash()`), stamped into SKILL.md at Build as `context-hash`. Any change to any baked instruction file moves the hash, so on boot the skill detects a shipped system update and leads with a big, unmissable **"System update available"** banner asking the owner to **Build & Lock** and reload before continuing (fail-open — it still offers "continue"). The context-layer twin of the existing config-drift gate.
- **Plugin Enhancement — data-sharing consent (option only)** — a new setup section, **Plugin Enhancement** (schema group `E`), with a single `data_sharing` boolean (default **on**, displayed as **ON**/**OFF**). During the initial setup query the AI asks the owner directly for permission to share basic usage data with Seresa so the plugin can be improved, and saves their answer; they can switch it off any time. Stored in the existing `cherry_tree_config` option — no new table. **This release collects the setting only: there is no data-sharing service and no notification system yet — both are planned for later.** See [readme/Plugin-Enhancement-Data-Sharing.md](../readme/Plugin-Enhancement-Data-Sharing.md).
- **Download plugin logs from Settings** — Settings → Configuration Status gains a **Download recent logs** button that bundles Cherry Tree's log files (kept for the retention period) into a single download for a site owner to hand to support, so debugging no longer needs SSH or the secret log URL. Admin-only (`manage_options`) and nonce-verified, served via `admin_post_cherry_tree_download_logs` (`Admin\Log_Export_Handler`). Zips when `ZipArchive` is available, else a concatenated text file; the private log-file secret is stripped from the bundled filenames so the export never leaks the token that protects direct log URLs. Cherry Tree keeps its own logs (not WordPress's `debug.log`), so this works regardless of `WP_DEBUG`.
- **Post-write block integrity check** — after the API creates a post, `Posts_Controller` inspects what actually landed in the DB and logs a `BLOCK_INTEGRITY` warning per problem found, so a silently-invalid block is caught at write time instead of months later when someone opens the editor. The new `Block_Integrity_Verifier` (`src/API/class-block-integrity-verifier.php`) re-derives the canonical block from its own stored attributes and flags drift — its first target is the third-party Yoast FAQ block (`wp:yoast/faq-block`), whose `save()` format Cherry Tree does not control: it flags questions missing the `id`/`question`/`answer` keys Yoast reads (catching every legacy shape, including raw schema.org JSON-LD), an unparseable delimiter (the signature of a save-time filter mangling the JSON), and a body that no longer matches its attributes. Purely observational — it never blocks or alters the write; a healthy post logs nothing.
- **CTA click tracking through inPIPE / Transmute Engine (premium only)** — when a reader clicks a Cherry Tree CTA on an install with the inPIPE premium (Transmute Engine) active, the click is sent as a first-class event through inPIPE's own collector. Every CTA always carries a stable 5-char **Key** (its ID — minted once, preserved across edits, shown on each card in Settings → Call to Action). Everything else is strictly gated on an active premium subscription and simply does not exist without one: the generated **inPIPE Event Name** (`cherry_tree_cta_` + up to three headline words, kept clean when unique and numbered by page position only when two CTAs share a headline) is derived on demand, never stored — so with no premium it is not generated, not shown in the admin, and not previewed. Three parts, each premium-gated: (1) the event name + live admin preview (`Utilities\CTA_Event_Namer`); (2) the rendered anchor carries `data-cta-*` and a small frontend script that beacons the page URL (with its UTMs), referrer, and page id/slug/title — enqueued only on premium installs; (3) a nonce-verified endpoint (`Frontend\CTA_Event_Bridge`) that refuses outright unless premium is active, else derives the CTA's event name server-side from its Key (so a client can't inject an arbitrary event) and forwards it to `InPipe_Premium_Event_Collector::track_custom_event()`, failing quietly so a tracking miss never blocks the click.
- **Video config system** — video IDs and versions are now managed in `config/videos.php` instead of being hardcoded as PHP constants. To ship a new video with a release: bump the version integer and change the YouTube ID — one file, one edit.
- **"Updated" badge** — when a new video ships, a pink "Updated" pill badge appears on every "Watch intro" trigger (setup screen, dashboard banner, proxy section) until the user watches it. The badge clears on dismiss and resurfaces banners/notices that were snoozed, driving re-engagement.
- **`Video_Config` class** (`src/Config/class-video-config.php`) — central loader with version-aware watched-state tracking per user.
- **Author roster in `GET /config`** — the config response now carries an `authors` array of `{ id, name }` for every user who can publish, so the setup AI can map an owner's plain-English author choice ("publish as Cherry Rose") to the numeric user id `default_author` requires, without a second lookup call. Scoped deliberately: only publishers are listed, and only id + display name are exposed (never email, login, or role). Names resolve through the shared, email-safe `Author_Name` resolver (`src/Utilities/class-author-name.php`), extracted from `Config_Summary` so the admin list and the roster agree.
- **Live dashboard stat cards** — the Ideas / Queued / Processing / In Review / Published / Rejected counts on the Cherry Tree Articles screen now refresh on their own while the tab stays open (on focus/visibility, plus a slow backstop poll that pauses when the tab is hidden), so an open dashboard no longer drifts from reality until a manual reload. Churn-safe by design: a version token bumped only when the counts actually change lets each poll settle with a single autoloaded-option read, and the aggregate query runs only behind its existing transient when something has moved. New endpoint `cherry_tree_get_status_counts` (`src/Admin/class-counts-ajax-handler.php`) and `Queue_Repository::get_counts_version()`.
- **Queue reconciliation with WordPress** — when an owner acts on a linked post inside WordPress itself, the queue now follows: publishing an in-review item with WordPress's own Publish button advances its row to Published, and unpublishing, trashing, or deleting a published post drops its row to Rejected so it stops inflating the Published count. Event-driven off `transition_post_status` / `deleted_post` and cheap — a genuine status change only, and every post Cherry Tree did not create resolves to one indexed lookup that returns nothing. New `src/Content/class-post-reconciler.php` and `Queue_Repository::mark_published_from_wp()` / `mark_rejected_from_wp()`.
- **Live Articles list rows** — the table rows now re-sync on their own alongside the stat cards, so a post taken live with WordPress's own Publish button (or any change from the AI pipeline or another writer) shows up without a manual reload: the row leaves the "In Review" tab and appears under "Published", indexed dates fill in, and removed items drop out. It reuses the counts version token — when the token moves, the dashboard fetches the current view's rows (same query and markup as a full page load) and reconciles the table by row id, preserving any in-progress bulk selection. New endpoint `cherry_tree_get_rows` (`src/Admin/class-rows-ajax-handler.php`), the shared `Articles_View::render_row()` renderer, and `Admin_Articles::get_items_for_view()`.
- **Sibling-product promo detection** — Cherry Tree promotes inPIPE (the free data-collector plugin) and, through it, the paid Transmute Engine service, so it now detects which of the two an install already has and exposes that as one reusable hook. New `Promo_Detector` (`src/Utilities/class-promo-detector.php`) surfaces two clear statuses — inPIPE (free) installed and active, and upgraded-to-premium with a live subscription — deferring to inPIPE's own `InPipe_Status_Manager` for the premium check and failing quietly to "not present" when inPIPE is absent. Any marketing code reads the result through the `cherry_tree_promo_status` filter (`apply_filters( 'cherry_tree_promo_status', array() )`); no promotional code consumes it yet — this release only publishes the detection seam.
- **Worked exemplars + feedback library baked into the skill (`baked/examples.md`)** — a new baked reference doc ships alongside `invariants.md`, carrying the concrete "show, don't just tell" methodology an earlier skill rewrite had thinned to spec-only: a complete worked article exemplar in the exact current element order (question headings, answer-first, Yoast FAQ, cite-wrapped references, no CTA), illustrative frontmatter values, the Cherry Rose common-errors feedback library, and the Stage-2 audit scorecard — so the model writes and audits against examples, not rules alone. Every example is built to the *current* contract (the retired italic-subtitle pattern, 3–5 FAQ, and CTA close are deliberately absent). `Skill_Assembler` remaps `examples.md` to `baked/examples.md`, the same way it already ships `invariants.md`.

### Added
- **Citable core presence gate on idea → queued** — an idea with a `research` object can no longer be approved (promoted to `queued`) without a well-formed citable core. Legacy items that predate the core (no `research` at all) are grandfathered so they are not stranded. Applies to both articles (`trait-queue-transitions.php`) and seeds (`trait-seed-status.php`), wired to the existing `Citable_Core_Validator` and `Seed_Citable_Core_Validator`.
- **Seed list endpoint strips internal fields** — `GET /seeds` now strips `research` and `claim_token` from list responses, matching the article queue's `LIST_COLUMNS` behaviour. Single-GET and claim endpoints still return them. See `Seeds_Read_Controller::compact_seed_for_list()`.
- **Seed core fidelity verification** — new `verify_seed_core_fidelity()` in `cherry_tree_client/verify.py` confirms the Quick Answer leads with the approved answer, the stat number appears in the body, and the source URL is in the Sources list. Called from `seed-final.md` (step 5b) before the two-call publish. Applied byte-identically to both twins.
- **Source liveness verifier** — new `Utilities\Source_Liveness_Verifier` checks each citable core source URL for reachability (HEAD/GET) at approval time (idea → queued) and caches per-source results in `research.source_checks`. A dead source warns (logged); it never blocks. Optional stronger tier fetches the page and confirms the stat number is present. Results feed admin card badges (✅/⚠️) in `render.js`. Works for both articles (multi-unit) and seeds (single source).
- **Freshness gate at claim time** — new `Utilities\Freshness_Checker` computes idea age and oldest source year when a seed or article is claimed. If stale (default: 90 days or 3-year-old source), a `freshness_warning` object is attached to the claim response so the writer stage can surface "re-verify these stats before writing". Advisory only — the claim always succeeds.
- **Corpus-level claim de-duplication** — new `Utilities\Claim_Deduplicator` checks each new idea's citable core answers against published articles and seeds using Jaccard word-set similarity. Near-duplicate warnings (≥60% overlap) are returned as `duplicate_claim_warnings` on the create response so the owner can review before approving ideas that overlap existing content.
- **Entity/E-E-A-T enrichment utility** — new `Utilities\Entity_Enrichment` annotates citable core units with Schema.org entity metadata (`entity.type`, `entity.name`, `entity.same_as`, `entity.description`). Accepted but not required — the existing `sanitize_recursive()` preserves the nested structure, and the validators pass enriched units through.
- **Seed stat advisory for data-shaped seeds** — `Seed_Citable_Core_Validator::warnings()` now detects comparison/data-shaped seeds (by keyword in the question) that lack a stat and returns an advisory recommendation. Surfaced as `stat_warnings` on the create response. Purely definitional seeds are exempt.
- **"Rebuild required" notice now spans every Cherry Tree admin page and carries a "Rebuild now" button** — previously the system-update (context-hash drift) notice appeared only on the Content Skill page and had no call to action. A new cross-page banner (`Skill_Rebuild_Notice`, hooked on `admin_notices`) shows on every Cherry Tree / Cherry Bee admin screen when a plugin update changed the AI's baked instruction layer since the last build, and both it and the Content Skill page's inline notice now include a **Rebuild now** button that jumps straight to the Build & Lock control (a new `#ct-build-anchor` bookmark, with a smooth scroll + brief highlight). The banner is non-dismissible and self-clears the moment a rebuild makes the hashes match again; it skips the Content Skill page itself to avoid doubling up with the inline copy.

### Changed
- **Plugin-wide admin design refresh (base layer)** — the shared admin polish layer (`assets/css/admin/polish.css`) now unifies the brand accent on a single cherry (`#b3325a`, matching the Content Skill screen) across every admin page, and adds shared radius / shadow / status tokens the per-screen styles consume. On-brand primary buttons, warm rounded form controls with a cherry focus ring, lifted cards and accented headings now apply consistently, and the **API Health** screen is brought into the styled set (previously it received neither the admin styling bundle nor the `cherry-tree-admin` body class). The plugin's own components — settings sections, stat cards, filter bars, CTA cards and list tables — are folded into one card language, with the selected/hover state now cherry instead of WordPress blue. **Cherry Bee (the Indexing module)** is brought fully onto the same design: its dashboard/logs screens now ride the shared polish layer (its own enqueue opts them into the `cherry-tree-admin` scope), and its stat cards, sections, tables and the "Index Now" action are retinted to the cherry — semantic status colours (badges, health dots) left intact. The whole admin now reads as one product from top to bottom.
- **Content Skill top block redesigned** — the header now shows a colour-coded status banner instead of a plain lede: amber **"Your profile is ready to build"** before the first build (replacing the contradictory *"complete and nothing is live yet"* wording), and green **"Your Content Skill is live and active"** once Build & Lock has run — the post-build state now clearly tells the owner the skill is active and ready to download. The "Describe a change" prompt, its button and its generated output are grouped into a single card, with the copied-prompt output revealing **inside** that card only after the button is pressed (no permanent empty box sitting at rest). The "Watch intro" link becomes a compact chip with a small round play button instead of a full-width bar, and its label now reads as plain text rather than a blue link. Primary buttons on this screen (Prepare my changes, Build & Lock) are recoloured to the Cherry accent, and the change box gains a matching focus ring. Also fixes a latent bug where the "hidden" copied-prompt output rendered as a permanent empty grey box (an author `display:block` rule was overriding the `[hidden]` attribute). Presentation only — the setup flow, AJAX wiring and element IDs are unchanged.
- **CTA card header layout** — each Call to Action card now stacks its identifiers on the left under the Enabled toggle: the inPIPE Event Name (premium only) with the stable **Key** directly beneath it, both using the same muted label with the value in a `<code>` chip. The Key label + value now show on **every** CTA (not just premium), and only the value sits in the chip; the event-name row stays premium-only. Previously the Key floated at the top-right of the card as a single greyed `Key : xxxxx` chip.
- **Content-skill instruction layer enriched — writing craft, seed craft, and research methodology restored** — a review against the pre-rewrite skill found ~15–30% of the instructional methodology (mostly examples and calibration) had been thinned; this restores the highest-value pieces without resurrecting the obsolete multi-tenant setup machinery. `invariants.md` gains E-E-A-T signals (real authorship over "Editorial Team", freshness) and, for seeds, the Quick-Answer four-beat scaffold, tag-selection rationale, and good/bad title examples. `article-ideas.md` regains YouTube comment-mining as a discovery channel, a tiered why-now timeliness framework (high/medium/low signals), content-territory funnel tiers so the queue balances across buyer intent instead of clustering, and qualitative score bands. Instruction-layer only — the context-hash gate flags installs to rebuild.
- **Competitors carry truthful comparison context — `competitors` is now a record list** — the `competitors` config field changes from a bare list of names to a list of records — `{ name (required), they_offer, gap, our_angle }` — so the writer compares a competitor **fairly and accurately** (their genuine strength, an honest limitation, and the client's on-merit angle) instead of guessing. The `-final` / `-ideas` stages gain a "truth, not trash" rule: represent competitors fairly, cite sources for any claim about them, and never fabricate a weakness — a self-promotional hit-piece is the one thing answer engines refuse to cite. Backward-compatible: a legacy list of name strings is accepted and normalised to `{ name }` on read (`Config_Repository::get_config`), leaving the stored option untouched so an install's `config_hash` never drifts until it is next saved or rebuilt; the admin settings summary renders the new fields and the setup schema (`/config/schema`) collects them. New `Config_Field_Validator::check_competitors` accepts both shapes and requires a name.
- **Articles row actions use event delegation** — the row buttons (Approve/Reject/Publish/Index/View/Edit/Delete) and the bulk-select checkboxes are now bound once via delegation on `document` (`assets/js/admin/utils/delegate.js`) instead of per element at load. This keeps rows inserted or replaced by the live refresh fully interactive. Behaviour is unchanged for a normal page load. New shared helper; `items/actions.js`, `indexing/actions.js`, and `indexing/ui.js` updated.
- **CTA module restyle** — the injected call-to-action block drops its filled bordered card for a lighter, transparent look: a single accent line down the left edge, and the solid button becomes an accent-coloured arrow link. The link now uses a higher-specificity selector so a theme's own link styles (colour, underline) no longer bleed through the CTA. Presentation only; markup, classes, and stored CTA fields are unchanged (`assets/css/frontend/cta.css`).
- **`Queue_Repository`** — removed an unused `Logger` import from the class file; the composing traits already import `Logger` themselves, so the class-level `use` was dead code.
- **`Article_Schema`** — dropped the dead `key_claims` plumbing from the BlogPosting path: `emit_blog_posting_schema()` accepted a decoded `$key_claims` array (read from `_cherry_tree_key_claims`) that it never used — Speakable carries only CSS selectors, so the claim strings had nowhere to go. Removed the unused parameter, its JSON decode, and the now-pointless meta read. Output is unchanged (`src/Frontend/class-article-schema.php`).
- **Skill build and download now leave audit-log entries** — a successful Build & Lock (and the API-key-rotation auto-rebuild) writes a `SKILL` INFO line recording the built date, the config/context hashes, and which user triggered it (`Skill_Compiler::build()`); a skill download writes a matching `SKILL` INFO line recording which build was pulled and by whom (`Config_Ajax_Handler::handle_download()`). Previously only build *failures* were logged, so a completed rebuild or a download left no trace and could not be confirmed from the logs.
- **`Settings_Page` (Cherry Bee) split for maintainability** — the Settings-API render callbacks (the four section descriptions and the checkbox / text / textarea / number / verify-button field widgets, plus the shared secret show-hide-copy controls) moved out of `Settings_Page` into a new `Settings_Field_View` class, mirroring how `Settings_Sanitizer` was already separated. `Settings_Page` now owns only option/section/field registration and the page/section wrappers, dropping from 608 lines (past the 500 hard ceiling) to 383. Registration callbacks bind to the injected renderer; output and behaviour are unchanged (`src/Indexing/Admin/class-settings-field-view.php`, `class-settings-page.php`, loaded by `class-indexing-module.php`).

### Fixed
- **Admin menu callbacks now use long array syntax (WPCS)** — the seven `add_menu_page()`/`add_submenu_page()` render callbacks in `Admin_Menu` were passed as short-syntax `[ $this, 'render_…' ]` arrays, which the WordPress Coding Standards disallow; normalised to `array( $this, 'render_…' )`. No behaviour change (`src/Admin/class-admin-menu.php`).
- **Freshly built skill no longer reports "settings drift" instantly, and Build & Lock now latches** — on an install carrying legacy data that `get_config()` coerces on read (bare-string `competitors` upgraded to `{ name }` records), the drift gate compared two hashes of *different* bytes: Build & Lock baked `compute_hash( get_config() )` (the normalized config) while the live `GET /info` / admin page read the stored `cherry_tree_config_hash`, frozen by `save_config()` from the raw un-normalized array. They could never match, so every fresh build reported drift and the Build & Lock button never latched to disabled. `get_config_hash()` now derives the live hash from `get_config()` — the exact normalized bytes Build bakes — so both sides are byte-identical; a fresh (empty) install still reports `''`. No re-save or data migration needed (`Config_Repository`).
- **Primary admin buttons no longer go invisible on hover** — a WordPress submit button carries both `button` and `button-primary` classes, so the polish layer's secondary `.button:hover` rule (equal specificity, but later in the source than the primary hover rule) won and repainted the label cherry-on-cherry — the button became a solid magenta block with no readable text on hover, site-wide across the admin. The secondary rule is now scoped `.button:not(.button-primary):hover`, so primary buttons keep white hover text (`admin/polish.css`).
- **Build & Lock now activates on a system/code update, not only a settings change — and says so** — the button's enable logic compared only `config_hash` (the owner's *settings*), so after a plugin update that improved the skill's own instructions with settings unchanged, Build & Lock stayed greyed out and the owner had no way to rebuild — the fixes never reached the skill. It now also compares `context_hash`: when the installed plugin's context layer differs from the last-built skill's, the button activates and a **"System update available — rebuild required"** notice appears at the top of the Content Skill screen (a pre-4.4.0 build has no stored `context_hash`, which correctly reads as "update available"). The last build's `context_hash` is stored in the skill meta (`Skill_Repository`) and surfaced to the view (`Config_Page`). Paired with a code-driven check in the vendored client: `Skill_Assembler` now bakes `baked/build.json` (built date + both hashes) and the client compares it against live `GET /info` on boot, so a stale skill is caught even when the boot prose is skipped.
- **"Create an article" no longer re-researches a topic whose research is already done** — even when correctly routed to write (not ideas), the `-final` stages never told the model that the claimed queue item's `research` object *is* the finished, owner-approved writing brief, so it treated the quality gates (≥5 stats, Tier-1/2 sources) as a fresh to-do list and ran new web searches instead of writing the approved item. The stages now open with a field-by-field map of `item["research"]` and state outright: write from it, do **not** web-search; the only live lookups are cross-links and the item's own `permalink_preview`; if the package is genuinely empty, release and report rather than papering over it. Seeds get the parallel fix worded for their flattened shape (the claimed seed carries the drafted `short_answer` + `sources`, no `research` object). Instruction-layer only.
- **Stage-1 audit no longer lets a topic-label, preamble-first article through — and now ties the body to the citable core (Phase 2 enforcement)** — the article structural audit (`audit_stage_1`) derived its "answer-first" and "question-headings" checks from `<h2 id="…">` only, so a content section that simply *omitted its id* vanished from both — `ANSWER_FIRST` counted zero expected sections and `QUESTION_HEADINGS` returned NA, both passing — letting an article ship with topic-label, non-answer-first headings. Both checks now derive the expected content-H2 count from **every** `<h2>`, so a content section that dodges its id **fails** instead of skipping. A new 12th check, **`FAQ_HEADING_MATCH`**, requires every FAQ question (i.e. every `citable_core` unit) to have a matching content H2 — so the downstream article is provably written *around* the approved core: the FAQ, the answer-first sections, and the FAQPage JSON-LD all trace to the same questions and cannot drift. The `article-final` stage now writes each core unit into its own `<h2 id>` question section (answer-first, sourced), with `FAQ` derived from the units rather than re-authored. Client twins and `invariants.md` updated in lockstep. *(The independent pre-publish verify pass — the last spec item — is not yet built.)*
- **No more duplicate FAQPage schema — Yoast's FAQ is now actually suppressed when we emit ours** — `Article_Schema` registered its Yoast/RankMath FAQ-suppression filters from inside `emit_schema()`, which runs on `wp_head` at priority 4 — but Yoast builds and prints its schema graph on `wp_head` at priority 1, so the filters attached one hook too late and never fired. The result: a post with our FAQ meta emitted **two** FAQPage graphs (ours *and* Yoast's). The suppression filters now register at construction (plugin init, long before any `wp_head`), each self-gated to act only on a single post carrying our own non-empty FAQ meta. Suppression is also made robust to the Yoast **FAQ block**, which injects its FAQ by adding standalone `Question` nodes and dual-typing the WebPage as `FAQPage` rather than via the `FAQ` generator piece the old filter targeted: the new filter works on the assembled `wpseo_schema_graph`, dropping every `Question` node and any pure-`FAQPage` node and un-typing `FAQPage` (with its `mainEntity`) from a dual-typed WebPage, leaving Article/breadcrumbs/org/WebPage intact. The file docblock is corrected to name only Yoast and RankMath (AIOSEO/SEOPress were never handled).
- **Articles now always ship their own AEO schema — the model can no longer silently drop it at publish** — the four AEO fields (`ai_summary`, `key_claims`, `data_points`, `faq_pairs`) that write the `_cherry_tree_*` post meta behind the plugin's FAQPage / BlogPosting / Speakable JSON-LD (`Article_Schema`) were documented as *optional* ("omitting them still publishes"), so the model published with only the required fields and left them out — every article went live with the required content (title, excerpt, SEO meta) but **zero** `_cherry_tree_*` meta, `Article_Schema` never fired, and each post fell back to whatever a third-party SEO plugin emitted (Yoast) — or no schema at all on installs without one. These four fields are now part of `REQUIRED_PUBLISH_FIELDS` in the content-skill client (bundled twin updated in lockstep), so a publish missing any of them raises `PublishError` before the POST instead of quietly stripping the article's schema — and each is additionally checked for a **non-empty** value (`REQUIRED_NONEMPTY_AEO_FIELDS`), since the server writes the meta behind an `! empty()` guard and a present-but-empty value (`[]`/`""`) would otherwise pass a key-only check and still drop the schema. Stage 1 already validates all four exist, so the publish always carries them; the `-final` stage wording is updated to match (they are no longer described as optional). Existing posts published before this fix still need a one-off meta backfill.
- **"Create an article" now writes one instead of generating ideas or asking which to do** — the content-skill router conflated a request to *write* an article with a request for *topics*, so it loaded the ideas stage, ran the pillar-gap analysis, and either posted a new research idea or hedged with a "write vs. generate ideas" A/B menu — ignoring the already-approved queued items. The router (`skill-template/SKILL.md`) is rewritten as a single decisive rule: any "create/write/make/publish an article (or seed)" request routes straight to the `-final` stage and writes a queued item immediately, with an explicit ban on presenting an A/B choice, on running ideas-stage gap analysis, and on loading an `-ideas` stage; only "ideas/topics/suggest/brainstorm/fill the queue" wording routes to `-ideas`. The competing lifecycle prose that primed the model to re-confirm intent is demoted to labelled background, `"what should I write"` is no longer listed as an ideas trigger, and the `-ideas` stages carry a wrong-stage guard that redirects a write request to `-final`. The `-final` stages claim the highest-scored queued item and write it with no menu or confirmation step.
- **Concurrent writers no longer race for the same article ("the one I chose was taken")** — the article-final stage picked a queue item with `list_queue` (a read) and then locked it in a *separate* `claim` call, leaving a read-then-lock window in which another session could claim the same item first. The server-side compare-and-swap always kept the DB consistent (only one writer ever won), but the loser surfaced as "already claimed by another writer" even though it had chosen first. The content-skill client now defaults to the atomic `GET /queue/next` path via new `CherryTreeClient.claim_next()` / `claim_next_session()` (bundled twin updated in lockstep), which selects **and** locks the highest-scored queued item in one operation, so two sessions running at once are each handed a different, already-locked article and the window is gone. The list-then-claim path is retained only for the case where the user names a specific topic. No server change — `/queue/next` already existed; the client simply now uses it.
- **"New sections available" no longer shows raw AI instructions, and gains a one-click button** — the block that surfaces optional sections added after setup was printing each field's schema `description` verbatim, which is written for the AI ("ASK THE OWNER DIRECTLY… Say roughly: …") and read as confusing raw script to a human owner, with no obvious action to take. Each section now shows just its heading and a **"Set this up now"** button: pressing it drops a plain-words request (`Set up "…" for me.`) into the Describe-a-change box, scrolls up to it, and prompts the owner to press **Prepare my changes** and paste the result into their AI — the AI still receives the full schema description (via `/config/schema` and the prompt builder), so the owner never needs to know what a section does. The lede is trimmed to one line to match. The matching dismissible admin banner (`Config_Update_Notice`) is simplified the same way — it no longer lists the raw schema section names in a `<code>` block or tells the owner to "start a change session", just a short heading and the **Open Cherry Tree setup** button.
- **inPIPE Event Name preview now shows on page load** — the Call to Action settings screen rendered each premium CTA's "inPIPE Event Name" label with a blank value until the first edit, because the derived name was painted from the server-rendered value (empty for any CTA stored before Keys existed) and the client-side preview only recomputed on interaction. The screen now recomputes every card's event name once on load, so a stored CTA shows its real name (e.g. `cherry_tree_cta_untitled`) immediately instead of after clicking another card.
- **FAQ block no longer shows "unexpected or invalid content" in the editor** — the HTML→block converter emitted each Yoast FAQ entry with only the legacy `jsonQuestion`/`jsonAnswer` keys, but Yoast 28.x's block `save()` reads `question`/`answer`. With those keys absent, Yoast regenerated an empty `<strong></strong><p></p>` that no longer matched the stored text, so Gutenberg rejected the block. `Html_Block_Converter` now emits each question with the `question`/`answer` keys the current save reads (alongside `jsonQuestion`/`jsonAnswer` for schema, and `images`), and rebuilds the block body in Yoast's exact whitespace-free shape so it byte-matches `save()` regardless of how the source HTML was formatted. The frontend was unaffected (invalid blocks still render their saved HTML) — the error only surfaced when a post was opened in the editor.
- **API-created posts no longer risk corrupting block-comment JSON on insert** — `wp_insert_post()`/`wp_update_post()` re-run KSES on `post_content` for any request without the `unfiltered_html` capability (every API-key request, since it authenticates with no WordPress user), which entity-encodes inline markup (`<strong>`, `<a>`, …) carried inside a block delimiter's JSON — e.g. a FAQ answer with bold text. The content is already sanitised with `wp_kses_post()` before block assembly, so `Posts_Controller` now suspends KSES around the insert and slug fix-up (`write_post_without_kses()`), storing the assembled block markup intact.
- **Queue reconciliation lookup is now actually indexed** — the content queue table gained `post_id_idx`, the index the reconciler's `post_id` lookup assumed. That lookup runs on every `transition_post_status` and `deleted_post` site-wide, so without the key each unrelated post save or delete scanned the whole queue table. The DB schema version bumps to `1.11.0` so `dbDelta()` adds the index on existing installs during the on-load upgrade.
- **Live status-count poll no longer fires on screens that don't use it** — `initStatusCounts()` now binds only to a stat-cards container carrying `data-counts-version` (the Articles dashboard). The seeds and analytics dashboards reuse the `.cherry-tree-stats-cards` class for static cards, and previously triggered a pointless 60-second queue-counts AJAX poll there that updated nothing.
- **Welcome modal keyboard access** — the modal's close control (a `role="button"` span) now activates on Enter/Space, and Escape closes the modal, matching standard dialog behaviour. Previously keyboard users could focus the close control but had no way to activate it.
- **Articles list "Created" date** — the Created column now renders "—" when a row has no timestamp, instead of passing an empty string to `strtotime()` (which trips PHP 8.1+'s null-to-non-nullable deprecation and yields a bogus epoch date). Mirrors the existing guards on the Published and Last Indexed columns.
- **Search-engine API keys are no longer shown in plain view on the Settings page** — the Cherry Bee "API Keys" section rendered the saved IndexNow key, Bing Webmaster key and Google service-account JSON directly into plain, fully-visible fields, exposing them to anyone looking at the screen (including over-the-shoulder or a screen share). Each secret field is now masked by default with a 👁 show/hide toggle and a **Copy** button beside it, matching the treatment of Cherry Tree's own API key. The value still round-trips through the standard settings form on save, so nothing changes about how the keys are stored or submitted (`Indexing\Admin\Settings_Page`, `section-cherry-bee.php`).

- **Stage-2 audit now scores all 11 rubric categories — the Self-Citation category is no longer silently skipped** — `invariants.md` §7 defines 11 article rubric categories (the 11th, **Self-Citation**, enforces on-merit product placement), but the `-final` stage's worked `SCORES` example listed only 10 and said "Score all 10". Because `audit_stage_2_pass` grades exactly the keys it is handed, a model following the example never scored Self-Citation, so the placement rule went un-audited on every article. The stage now scores all 11, includes `Self-Citation` in the example (with the conditional "score 100 when there is no `product_angle`" note), and states outright that an omitted category is silently never audited. Rubric-and-gate change only — no Python change.
- **AEO content system overhaul — question headings, answer-first, dense FAQ, plugin-owned schema** — the content skill's structural contract (`invariants.md`, `article-final.md`) and the Stage-1 audit are rewritten to produce answer-engine content that AI extractors actually cite. **H2s must be questions** people ask AI (end with `?`; new Stage-1 check #11 `QUESTION_HEADINGS` enforces it). **Answer-first** replaces the italic-subtitle pattern — the first `<p>` after each H2 is the direct answer, no preamble (check #5 renamed `ANSWER_FIRST`). **FAQ density** raised from 3–5 to **8–12** pairs — every question the article answers in prose gets a dedicated FAQ entry. **Cross-link framing** changed from "You may be interested in:" to "Related:". **Title / H1 alignment** — seo_title and H1 share the same framing. **Plugin-owned article schema** — new `Article_Schema` class (`src/Frontend/class-article-schema.php`) emits FAQPage, BlogPosting (with author + Speakable), and HowTo JSON-LD from post meta, independent of Yoast/RankMath. `Posts_Controller` now stores `faq_pairs`, `ai_summary`, `key_claims`, and `data_points` as post meta (`_cherry_tree_*`) at publish time. The italic `<p><em>subtitle</em></p>` pattern is retired and explicitly forbidden.

> # 🔒 Security release — please update
>
> **If you are running 4.0.0 or 4.0.1, update to this release.** The findings below were
> present in those versions, and updating is not sufficient on its own — you will also need
> to regenerate your API key and **reissue your content skill to every AI platform you have
> installed it on**. See *What to do* at the end.
>
> Before this release we commissioned a **deep, line-by-line security audit of the entire
> system** — both halves of it. The WordPress plugin: REST API, admin, database layer,
> frontend, indexing engine, updater. *And* the Python content-skill client, which most
> audits would miss, because it does not run in WordPress at all — it runs inside the AI's
> own runtime holding your site's credentials.
>
> **14 findings were raised. Every code finding is fixed in this release** — 12 closed
> outright, one verified as already correct, and one requiring a credential rotation on our
> side rather than any change to the software. Every fix ships with tests: the suites grew
> to **2,565 PHP tests, 169 Python tests and 163 JavaScript tests** — all passing, linter
> clean on every changed file.
>
> ### What was found and fixed
>
> - **Your API key was recoverable from a database backup alone.** The compiled content
>   skill was stored in the database with the key readable inside it, defeating the
>   encryption built specifically to stop a stolen backup being useful. Now encrypted.
> - **The content-skill client would have re-sent your API key to another server** if your
>   site ever redirected off-host — a lapsed domain, a hijacked one, a firewall block page.
>   It now refuses cross-host redirects while still following harmless same-host ones.
> - **Plugin updates installed whatever the update URL served, unverified.** Now
>   checksum-verified, restricted to the official repository, and no longer silently
>   redirectable to another repo by a different plugin on your site.
> - **Visitor IP addresses were stored in full** in logs and in the database. They are now
>   anonymised before anything is written down, with the retention policy documented for
>   your privacy policy.
> - **Failed API-key attempts were unmetered and invisible** to the site owner.
> - **The firewall diagnostic sent your API key over an unverified TLS connection.** It now
>   verifies by default and reports when it cannot.
> - **Compiled Python bytecode was shipping inside the release zip** — binary files nobody
>   reviews, beside the source they claim to mirror.
>
> ### What the audit verified was already correct
>
> Worth stating, because it is most of the system: **no SQL injection** across 102 prepared
> statements; **every one of the 41 REST routes** carries a real permission check, none left
> open; **every AJAX handler** gated on both a nonce and a capability, with no
> unauthenticated surface at all; output escaping consistent throughout; secrets compared in
> constant time; encryption correctly constructed. The things most plugins get wrong were
> already right.
>
> ### Four controls were deliberately made *less* strict
>
> This was the harder judgement, and we want it on the record. A security control that
> strands a site owner — a refused update, a dead API — stops them receiving later fixes,
> which is usually worse than the risk it was guarding against. Where a stricter setting
> risked breaking a working install over something the owner could neither diagnose nor
> change, it now **warns loudly instead of failing shut**. Each of those four decisions is
> documented in the code so it is not quietly reversed later.
>
> ### The API talks to an AI, so its errors are deliberately verbose
>
> One more decision worth putting on the record, because it looks like an information-
> disclosure bug and is not. Cherry Tree's REST API has exactly one intended caller: **an AI
> content skill running in a controlled runtime**, holding your key and acting for you. So
> its `400`/`403`/`500` error messages are written to *teach the AI what to do next*, in
> plain language — a rotated key returns not a bare "key does not match" but a full
> explanation that the key was likely rotated and that the owner must **Build & Lock**,
> download the new skill, and replace the old one in the AI. That lets the AI relay a fix in
> one turn instead of failing opaquely. The unauthenticated, bot-facing statuses
> (`401`/`404`/`429`) stay **stripped**, so the fingerprinting surface is still closed. The
> trade-off — verbose errors are a mild disclosure surface — is accepted here because the
> endpoints are key-gated and the consumer is an AI, not a browser. **Treat the API as
> private to your AI: do not publish, proxy, or embed it where a third party can call it.**
> This is a speed-for-the-AI choice that is correct only while the key stays with the AI it
> was issued to. See `readme/Security/Security.md` → *Verbose responses for AI consumers*.
>
> ### Prevention, not just repair
>
> The audit also found credentials committed in our own internal documentation. Beyond
> removing and rotating them, this release adds a pre-commit hook and a server-side CI scan
> that refuse to let a credential into the repository at all, a build gate that refuses to
> ship one inside a release zip, and automated checks that stop the security-relevant
> pieces silently drifting out of sync.
>
> ### What to do
>
> **1. Update to 4.4.0.**
>
> **2. Regenerate your Cherry Tree API key** — if you ran 4.0.x and any database backup
> from that period still exists anywhere you would not keep a password: a backup service, a
> staging clone, a migration export, a developer's laptop. The key was readable inside
> those backups, and installing 4.4.0 does not change that. Only regenerating does.
> Cherry Tree settings → regenerate the key.
>
> **3. Rebuild your content skill and replace it everywhere you have installed it.** This
> step is easy to overlook and it is the one that matters most:
>
> - Your content skill package contains a **copy of your API key**, baked in — that is how
>   the AI authenticates to your site, and it is by design.
> - So every copy you have uploaded to an AI platform — **Claude Skills, or any other
>   assistant, workspace or automation you gave it to** — holds a copy of the *old* key.
> - After regenerating, those copies **stop working**, and more importantly they are stale
>   copies of a credential you have just retired, sitting on someone else's platform.
>
> So: **Build & Lock** in Cherry Tree to produce a fresh package, upload the new one to
> every platform where the skill lives, and **delete the old one** from each. Do not leave
> a superseded skill in place — it is a copy of a key you have decided you no longer trust.
> If you shared the skill with colleagues or clients, they need the new package too.
>
> **4. Review your privacy policy** — if you operate in the EU or UK. 4.0.x stored complete
> visitor IP addresses; 4.4.0 anonymises them before writing anything down.
> `readme/Logging/Data-Retention.md` has wording you can use.
>
> Cherry Tree is distributed from GitHub rather than the WordPress.org repository, which
> means there is no third party reviewing it on your behalf. This audit is us doing that
> job ourselves, and publishing the result — including the parts that reflect badly on us.

### Added
- **The setup screen now shows when each setting was last changed.** The Content Skill screen records a per-field timestamp every time a setting's value actually changes (stored in a new `cherry_tree_config_updated` option, written on the one config write-path so both the AI's REST edits and any admin path are covered), and renders a small "Last updated …" stamp under each setting in "Your settings so far". Only fields that change from now on are stamped — settings that predate this feature stay unstamped until their next change, so the screen never claims a change happened at a time it did not actually record.
- **The setup screen's sections are now collapsible and remember their state.** Each settings group in "Your settings so far", plus the **Overview** and **Config version** boxes, is a native `<details>` disclosure (the same mechanism as "What this skill enforces") — click the title to collapse or expand it. The open/closed state of every section is remembered per browser via `localStorage`, wrapped so a storage-denied browser simply falls back to the default. Everything defaults to open, so nothing is hidden until the owner chooses to collapse it.
- **A "Rotate API Key" button on the settings screen (Cherry Tree → API Key).** It generates a fresh key, invalidates the old one, and — if a skill has already been built — **automatically rebuilds the stored skill** so the new key is baked into the downloadable zip. This closes a real gap: rotating the key does not change the config hash, so the Content Skill screen's Build button would otherwise stay disabled and the stored skill would keep serving the retired key with no way to refresh it from the UI. Because the plugin can only rebuild the package on the server — it cannot reach into the AI and swap the installed skill — a blocking notice then walks the owner through downloading the new package and replacing the old skill in their AI, then starting a fresh session. If no skill has been built yet, rotation is silent (there is nothing to re-install).
- **A polished coat over the whole admin area.** A single scoped stylesheet (`assets/css/admin/polish.css`) layers refined type, spacing, cards, form controls and a cherry primary-button accent over the plugin's admin screens. It rides entirely on WordPress's native markup (`.wrap`, `.button`, `.form-table`, `.notice`, `.postbox`) and the plugin's own `ct-*` elements — no markup changes, no JavaScript, no framework, no build step. Every rule is scoped under a `cherry-tree-admin` body class (stamped by `Admin_Assets::add_body_class` on plugin pages only), so it cannot leak into WordPress core or another plugin, and the whole palette is tunable from one custom-properties block.
- **Existing installs are now told about new, optional config sections.** The whole client config is collected through the AI setup flow, not an admin form, so an optional field added *after* a site was first set up (e.g. the `products` self-citation catalogue introduced in 4.2.0) was invisible to owners who predated it — nothing prompted them, and an empty optional never blocks a rebuild, so the capability silently stayed off. Four changes close this backward-compat gap for every future optional field, not just `products`:
  - **A visible "New sections available" block on the Content Skill setup screen.** Unconfigured optional sections (added after this site's setup) render as a labelled, blank section — "Not set up yet" plus what it unlocks — telling the owner to set it up with their AI. Only shows once setup is complete/locked, so a fresh install isn't nagged.
  - **A proactive "new sections" surface in the edit prompt.** `Config_State::unconfigured_optional()` reports optional sections that are still absent or empty; the setup/edit prompt gains a `{{NEW_SECTIONS}}` slot, and `edit.md` now instructs the AI to raise these with the owner and offer to fill them — distinct from the existing "outstanding" list, which deliberately tells the AI to leave optional fields alone.
  - **A dismissible admin notice.** A new `Config_Update_Notice` banner tells the owner new sections exist and points them at the Cherry Tree setup screen to ask their AI to add them. Dismissal is **permanent per round** (stamped with a `NOTICE_ID` that a later release bumps when it adds further sections) and the banner **self-clears** once every optional section is filled.
  - **Read-time optional defaults on `GET /config`.** Absent optional keys now come back as their schema default (so a client reading `products` gets `[]` rather than a missing key), and the response carries an `unconfigured_optional` list. The stored option is never rewritten, so this cannot trip the Build-lock reset.

### Changed
- **The REST "key does not match" (403) response now explains a likely rotation.** When a request arrives with a key that does not match, the AI is told the most likely cause is that the owner rotated the key — so the key baked into the skill is now stale — and is walked through the fix (owner runs Build & Lock, downloads the new skill, replaces the old one in the AI, starts a fresh session). An AI hitting a retired key gets an actionable explanation instead of a bare rejection.
- **Python client: stateless HTTP helpers lifted into `_http.py`.** The A1/A3 security work had pushed `transport.py` to 535 lines, over the 500-line hard ceiling. Its five dependency-light, side-effect-free helpers — `_version_tuple`, `_check_requests_version`, `_default_session`, `_host_of`, `_untrusted` — carry no `Transport` state, so they now live in a new `_http.py` (140 lines) and `transport.py` drops to 425, under the ceiling. `transport.py` re-exports the two names the tests import, so nothing downstream changed; behaviour is identical. Applied to both byte-identical client copies and enforced by `test_twin_parity.py`, which now covers ten modules. The harder half of the split (the `self`-bound `_request` / `_parse_json` glue, which would take the file under 300) was deliberately left for a separate pass.
- **HTML→Gutenberg conversion moved out of the posts controller.** `Posts_Controller::create_post()` carried ~370 lines of DOM-based HTML-to-block conversion (paragraphs, headings, lists, blockquotes, groups, and the Yoast FAQ-block extraction) inline, pushing the controller past the 500-line ceiling. That logic is a pure, self-contained transform with no REST, request, or database dependencies, so it now lives in its own `Seresa\CherryTree\API\Html_Block_Converter` class and the controller calls `->convert()`. No behaviour change — the emitted block markup is byte-for-byte identical, verified by the existing controller tests, which exercise conversion end-to-end through `create_post()`. This is a first, low-risk slice of the API-controller size cleanup; the remaining `create_post()` decomposition is deliberately left for a separate pass.
- **Plainer, non-technical drift message when settings change.** When the boot/drift gate finds the skill is out of date (settings changed in WordPress, or a pillar category was deleted/renamed), the AI no longer stops with jargon like "config hash drifted" and raw queue/hash numbers. `SKILL.md` now scripts a plain-English heads-up — *"you've changed your Cherry Tree settings since this skill was last built… anything you changed recently won't be reflected in the articles I create in this session (drift has occurred)"* — plus a numbered **Build & Lock → give the new skill to your AI → reopen a session** path, and a **"continue"** option for when the change was minor. A parallel script covers the deleted-category case.
- **The config hash is now a plain "Config version" callout the AI speaks to.** The setup screen used to tuck the raw hash into the status strip labelled "Config hash". It now sits in its own labelled **Config version** box with a plain note (it changes when you change settings; your AI warns you when its skill is built from an older one). The drift message the AI gives was aligned to match: it calls this the **version** (never "hash"), and names both the version the skill was built from and the live version using the **same first 10 characters** the screen shows — so an owner sees the two don't match and understands they've changed settings without rebuilding.
- **The setup screen leads with an "Overview" box.** New owners were dropped straight into settings with no orientation. A short Overview now explains that the engine is managed through their AI (describe a change → *Prepare my changes* → paste into the AI, which updates everything automatically) and reassures them not to worry if a field looks confusing — they can just ask their AI about it, which the edit prompt is now explicitly told to answer from each field's description.
- **The settings summary is grouped under section headings.** The flat "Your settings so far" list scattered related settings (product fields in particular) down the page; it now groups them by the schema's sections — **Market & Audience**, **Brand & Quality Gate**, **Voice & Product**, **Publishing & Pillars** — with a heading and rule per section, so all the product settings sit together (products and terminology at the tail of *Voice & Product*). The summary markup moved to `templates/admin/partials/config-summary.php` and the grouping to the tested `Config_Summary::grouped()`.
- **The "Watch intro" link is now a stand-out video box** instead of a bare text link, matching the other video call-to-actions.
- **JSON escape artifacts no longer leak into the summary.** A value stored with its slashes escaped (e.g. a research community shown as `r\/analytics`) now renders normally via `Config_Summary::clean_display()`, applied across the list, product, and terminology values.
- **The setup summary shows the default author's name, not just their id.** The "Your settings so far" list rendered `default_author` as its bare WordPress user id (e.g. `2`); it now resolves that id to a real name and keeps the id in square brackets — `David [2]` — so the value is human-readable while the id the AI needs stays visible. It prefers first/last name, then nickname, display name, or login, and skips any candidate that is an email address (a `display_name` set to the account email is never surfaced); if no non-email name exists it falls back to the bare id.
- **The products catalogue reads like a list, not a raw JSON dump.** Each product in the "Your settings so far" summary was rendered as its raw JSON (`{"name":"Transmute Engine","summary":"… — …", …}`) — technically-correct escaping (`—` is an em dash, `\/` a slash) but unreadable and easily mistaken for corruption. Products now render as a clean list: bold **name** — summary, then *Markets:* tags, the product link, and *Why it belongs:* merit. The generic array fallback also now emits unescaped Unicode/slashes, so any remaining JSON reads normally.
- **The default author falls back to a name derived from the email when the account has no real name set.** If a user's only name fields are the account email (which we never surface), the summary showed a bare id; it now derives a readable name from the email local part (`david.anttony@… → David Anttony [2]`) so a name always shows — never the raw address.
- **Content type now reads in plain English, with the raw value in brackets.** The `content_types` row showed the bare enum (`both`); it now renders a human sentence with the stored value kept in square brackets — e.g. *"Both — full articles and short, citable Q&A seeds [both]"* — so a person understands it while the AI still gets the exact value. This is the general summary pattern (English outside, raw setting in `[…]`), matching `default_author` (`David [2]`).
- **Product terminology now reads as plain instructions instead of a cryptic list.** The `product_terminology` map (each entry is a preferred term → the wording to avoid) was rendered by the generic value-only list, which dropped the preferred term entirely and showed slash-separated fragments with no label. It now renders one explicit line per entry — **Use *transmute* — not *our tracking tool*, *our solution*** — under a short caption, so it's clear which word to use and which to avoid.
- **Summary value-formatting extracted to a tested `Config_Summary` class.** The per-field humanising logic above (author name, content-type label, terminology rows, product list) now lives in `Seresa\CherryTree\Admin\Config_Summary` rather than inline in the setup template, keeping that template a thin view (markup + escaping only, back under the 300-line guideline) and covering the whole "English outside, raw value in brackets" pattern with unit tests in one place.
- **Clearer "what to do next" steps after the AI saves a change or finishes setup.** The `edit.md` and `setup.md` prompt templates ended with a one-line "go to Cherry Tree and click Build & Lock" instruction that people missed — the button sits at the very bottom of the Content Skill page and the saved change doesn't appear until the page is reloaded. Both now instruct the AI to hand the owner a plain, explicit numbered checklist: **refresh** the Content Skill page (the change only shows after a reload), **scroll to the very bottom**, then **click Build & Lock** (plus an optional backup note for `edit.md`).

### Fixed
- **Dead pre-4.2.0 product keys no longer haunt the setup flow.** A site set up before 4.2.0 kept removed config keys (`product_name`, `product_mention_section`, `product_max_sentences_per_mention`, `product_forbidden_in_intro`) in its stored config, because the write path merges every save onto the existing config and never shed them. The setup AI saw those dead keys in the config summary, tried to edit one, and hit a confusing 422 (`the request contained no fields this system recognises`) — the sanitizer had silently dropped the removed field. Now: `GET /config`, the edit-prompt config summary, and the setup screen's "Your settings so far" all present **only current schema fields** (read-time filter — the stored option is untouched, so no Build-lock reset), and any section write **sheds dead keys from the stored config** so they self-heal on the next edit. Also fixed a stale `product_name` reference in the seed-ideas stage, now keyed off the `products` catalogue's `market_tags`.

### Security
- **Full system security audit completed (9 Sep 2026).** Both layers were audited end to
  end: the Python content-skill client that runs inside the AI runtime, and the WordPress
  plugin itself. The core request-handling security verified clean — no SQL injection
  (102 prepared call sites, whitelisted `ORDER BY`), all 41 REST routes carry a real
  `permission_callback` with zero `__return_true`, all ~30 AJAX handlers gate on nonce +
  `manage_options` with no `nopriv` surface, output escaping is uniform, `hash_equals` is
  used for every secret comparison, and the AES-256-CBC encrypt-then-MAC key storage is
  correctly constructed. Findings and the remediation plan are tracked in
  `readme/TODO/TODO-audit-seciurity-sept9.md`; each fix lands against this version.
- **The compiled skill is now encrypted at rest.** Building a skill produced a zip containing `baked/credentials.json` — which carries the install's live API key, by design, because the AI receiving the skill has no credential store of its own — and then parked a second, readable copy of that zip in the `cherry_tree_skill_zip` option so the Download button could re-serve it. That copy was plain base64. It meant the key was recoverable from a database dump alone (a leaked backup, SQL injection in some *other* plugin, a shared-host neighbour), which is exactly what `API_Key_Manager`'s AUTH_KEY/AUTH_SALT-derived encryption exists to prevent — the protected copy and the unprotected copy sat in the same table, and the unprotected one set the real security level. The blob is now stored through the same encrypt-then-MAC scheme. **Nothing about the skill or the download changes** — the key still travels inside the package, as it must. A blob written before this release is read, served, and transparently re-stored encrypted on first download, so an install that never rebuilds still stops leaking. Ciphertext that will not decrypt (AUTH_KEY changed under the install) reports "rebuild the skill" rather than serving a corrupt zip.
- **The content-skill client no longer follows redirects.** Every call carries `X-API-Key`. `requests` strips `Authorization` when a redirect crosses to another host but does **not** strip arbitrary custom headers, so `X-API-Key` would have been re-sent verbatim to whatever host `Location` named — and `validate_site_url()` guards the URL the client *asks for*, not the one it would *end up at*. A lapsed domain re-registered by someone else, a WAF block page, or an `https → http` downgrade would have carried the install's live key off-site, and that key authorises `POST /posts`, `PATCH /queue/*` and `PATCH /config`. All seven call sites now route through one private `Transport._request()` that refuses **cross-host** redirects with a typed error naming the `Location`. Same-host redirects are still followed (up to a loop cap): an `http → https` canonicalisation or a trailing-slash rule keeps the key on the server that already has it, and refusing those would break healthy installs for no security gain. Collapsing seven near-identical call sites into one also removes the copy-paste that let this land in all of them at once.
- **Plugin updates are verified before they are installed.** Cherry Tree ships from GitHub, not WordPress.org — there is no repository review and no signed package between a release artifact and code executing on every install, and the updater previously handed WordPress whatever URL the GitHub API returned, with no host check and no integrity check. Four changes:
  - **A published SHA-256 is verified when present.** `release.sh` appends the built zip's digest to every release body, and the installed plugin downloads the package itself, hashes it, and compares with `hash_equals()` before anything is extracted. **A mismatch refuses the update; a *missing* digest does not.** That asymmetry is deliberate: a mismatch is never benign, but a missing digest means our own release process slipped, and blocking there would strand a site on an old build — cutting it off from later security fixes — over a mistake that was ours, with an error most owners cannot diagnose. A site wanting strict behaviour can define `CHERRY_TREE_REQUIRE_UPDATE_CHECKSUM`. Note the honest limit: the digest is published beside the asset, so it proves the download was not altered in transit, not that the release is genuine — that is the repo trust gate's job.
  - **The update repository is trusted, not merely configured.** `CHERRY_TREE_UPDATE_REPO` sits behind an `if ( ! defined() )` guard so an owner can point at a fork — which also meant any plugin loading *earlier* could claim that constant and silently redirect the update channel to a repo it controlled. The official repo is now a class constant (which cannot be pre-empted), and any other repo requires the owner to also define `CHERRY_TREE_ALLOW_UNOFFICIAL_UPDATES`. The documented fork/staging override still works; it just has to be deliberate.
  - **Release assets must be https on a GitHub host** (`github.com`, `objects.githubusercontent.com`, `release-assets.githubusercontent.com`). Anything else is ignored and logged.
  - **The `zipball_url` fallback is gone.** It yields the repository *source tree*, not the built plugin zip that was tested and released — installing it under the released version number silently swapped the shipped file set. A release with no attached `.zip` is a broken release, and now offers no update rather than an untested one.
- **Three live API keys were removed from the repository docs, and a guard added so it cannot recur.** `readme/` never ships (the release zip excludes it), but the keys were committed: two for `staging.seresa.io` and one for a client install, `blog.asalta.com`. All are redacted, each affected file carries a notice, and **all three still need rotating** — a committed secret is not undone by deleting the line. The recurrence guard has three layers: a tracked `.githooks/pre-commit` that refuses to commit a credential (activate with `npm run hooks:install`; exempt a genuine example with an inline `not-a-real-secret` marker), `npm run secrets:scan` for an on-demand sweep, and `.github/workflows/secret-scan.yml` running the same patterns server-side on every push so `--no-verify` cannot get one through. The scanner found two of the three keys that a careful manual review had merely flagged for follow-up.
- **Failed API key attempts are now metered.** A wrong or absent key was rejected *before* the rate limiter recorded anything, so bad-key requests were entirely unmetered — unlimited attempts at full speed, with no counter anywhere for the site owner to see. A failed authentication now consumes the caller's normal read budget, exactly as a successful call would, and increments a separate visible tally. **Deliberately metered, not locked out:** the key is 256 bits of `random_bytes`, so guessing it is not a practical threat, while a lockout keyed on client IP would be actively dangerous — `Client_IP` correctly refuses to trust forwarded headers unless the site declares a trusted proxy, so on a Cloudflare or reverse-proxy install that has not set those constants every visitor resolves to the same address, and a lockout there would take the whole API offline for everyone. Throttling stops a scanner and a misconfigured client from flooding the logs; it cannot strand a legitimate install.
- **The `requests` floor is raised, and checked at runtime — as a warning.** The declared floor was `>=2.25`, which admits releases that leak `Proxy-Authorization` across a cross-host redirect (CVE-2023-32681) and that can drop TLS verification for an entire `Session` after one `verify=False` call (CVE-2024-35195). The specifier is now `>=2.32.4,<3`. That alone binds nothing in production — the shipped skill is a folder of `.py` files importing whatever the AI runtime provides — so the floor is also checked at runtime, but it **warns rather than fails**: the runtime's `requests` version is not something a plugin user controls or can change, and taking a working install offline over it would be a far greater harm than the residual risk. That risk is small in any case, because this client refuses cross-host redirects itself (so the first CVE has no path here) and never passes `verify=False`.
- **The firewall diagnostic no longer sends the API key over an unverified connection.** `WAF_Diagnostic::probe()` fires a loopback request at the site's own REST API carrying the live API key, and it did so with certificate verification switched off — meaning the connection was not authenticated at the transport layer, and anything sitting between PHP and that hostname (a local proxy, a mis-set `WP_HTTP_PROXY`, a hostile DNS answer for the site's own name) could have collected the key. It now **verifies by default**. The original reason for disabling it was real, though: staging and local installs often present a self-signed or mismatched certificate on their own hostname, and refusing to run would break the diagnostic for exactly the people debugging a firewall problem. So a certificate failure — and only a certificate failure, not a plain connection error — triggers one unverified retry, and the result **says so** rather than quietly pretending the connection was sound: *"Reached, but this site's SSL certificate could not be verified."* Secure by default, never a broken diagnostic, and the owner learns something true about their setup.
- **Visitor IP addresses are now masked before anything is written down.** Every log line and every row in the API log table recorded the full client IP — personal data under the GDPR / UK GDPR, for which the site owner is the data controller. The host part is now zeroed before storage (`203.0.113.42` → `203.0.113.0`; IPv6 keeps only its /48 routing prefix), which is the anonymisation Matomo and Google Analytics apply. Nothing operational is lost: the logs still answer "is one network hammering the API?", they just no longer identify a device, and two visitors on the same network become indistinguishable. Masking happens in three places — the log line's IP field, the API log table, and the rate-limit *identifier embedded in the log message body*, which was a second, easily-missed route for a full address to be persisted. That last one is masked inside `Logger::rate_limit()` rather than at each call site, so a future caller cannot reintroduce it by forgetting. The full address is still resolved in memory for rate limiting, where keys are hashed and never stored readably. A site that needs real addresses to block an abusive host can set `CHERRY_TREE_LOG_FULL_IP`, with the data-protection consequences documented and consciously taken on.
- **Log retention is documented for site owners.** `readme/Logging/Data-Retention.md` sets out, in plain English, what is recorded and why, that IPs are masked and how, the 21-day retention and the cron that enforces it, how to change it, why the log directory's protection is a random filename token rather than `.htaccess` (which does nothing on Nginx), that nothing is transmitted anywhere — and ready-made privacy-policy wording. The mechanism was already correct; it simply was not written down anywhere an owner could find, and "read the source" is not an answer for someone being asked what their site retains.
- **The `requests` security floor can no longer drift between the four places that state it.** It is declared in `pyproject.toml`, in `constants.MIN_REQUESTS_VERSION` (twice — the shipped skill has no packaging metadata, so the runtime check must carry the number itself), and in the client README. `tests/test_dependency_floor.py` now parses `pyproject.toml` and asserts the runtime constant matches it, that a major-version ceiling is present, that the README states the same number, and that the README does not claim the check is "enforced" when it deliberately only warns. That last assertion exists because the README *did* claim exactly that for a while after the check was softened, describing the opposite of the design decision.
- **The committed Python virtualenv is untracked.** `cherry-tree-client/.venv/` held 2,066 files — 78% of the repository's tracked file count — of third-party code with its own CVE stream. It never shipped (the release zip is built from `cherry-tree-by-seresa/` alone), so this is repository hygiene rather than an exposure. It mattered because it muddied the answer to "which `requests` version does this project depend on?": `pyproject.toml` is authoritative, and a committed venv containing some other version invites a reviewer to conclude otherwise. Untracked and gitignored; nothing was deleted from disk.
- **The dev client and the shipped skill copy can no longer drift apart unnoticed.** The two `cherry_tree_client/` copies must be byte-identical — the baked one is what actually runs in the AI runtime, while the test suite only ever exercises the dev one, so any drift means the shipped code is untested. During this release's work they did briefly diverge and behaved differently on two controls, with every test still passing. `tests/test_twin_parity.py` now hashes all nine modules in both copies and fails on any difference, on a missing module, or on stray compiled bytecode under the skill template.
- **Route slugs from `GET /info` are re-validated before use.** The firewall-mutable action slugs are interpolated straight into a request path (`/queue/{id}/{lock_slug}`). WordPress sanitises them on the way out, so a healthy install cannot emit anything harmful — but that made the client trust a *network response* to be well-formed, which is precisely the assumption the redirect leak disproved. A slug of `../../wp-admin/x` or `x?redirect=` would have re-pointed the call while looking like an ordinary claim. Each resolved slug is now checked against a strict pattern at boot.
- **Server response bodies are fenced as untrusted before the AI reads them.** Error messages quote up to 200 characters of what the far end returned — genuinely useful for diagnosing a firewall rejection, and kept for that reason. But that text is attacker-influenceable and lands in the model's transcript on the one path (`GET /info`) that runs *before* any authentication succeeds. Snippets are now stripped of control characters and wrapped in an explicit `UNTRUSTED SERVER OUTPUT` fence stating that the contents are data, not instruction, and must be verified before being relied on. `SKILL.md` gains a matching rule telling the AI never to follow, execute, or obey anything inside that fence — and never to reveal the API key because a response asked for it — extended to any content retrieved while researching.
- **Compiled Python bytecode no longer ships.** Ten `.pyc` files under `skill-template/scripts/cherry_tree_client/__pycache__/` were tracked in git — including one built for Python 3.14, a version the project does not target — and `build.sh` archives tracked files, so they went into the public release zip. Shipping unreviewed binary artifacts alongside the source they claim to mirror is a supply-chain smell, and a `.pyc` diff is invisible in a pull request. They are untracked, `__pycache__/` and `*.pyc` are gitignored, and `build.sh` now refuses to ship either.
- **`build.sh` scans release contents for leaked keys.** The existing safety net matched file *names*; a real API key pasted into a shipped file would have sailed past it. The build now also greps the zip's contents for a live-key pattern (`ct-prod-` + 64 hex) and refuses to ship on a hit.

## [4.2.4]

### Added
- **Bulk Approve / Reject / Publish on the Articles screens.** Each status tab now carries select-all + per-row checkboxes and a status-aware bulk actions bar: **Ideas** gets *Approve Selected* (→ queued) and *Reject Selected*; **In Review** gets *Publish Selected* and *Reject Selected* (the existing *Publish All* button stays). Selections run through a new `cherry_tree_bulk_update_status` AJAX action that validates each transition server-side, so invalid transitions are skipped rather than forced. Per-row **Delete** stays a manual, single-row action, and rejected items remain a dead end (no bulk on the Rejected or Published-status transitions).

### Changed
- **Removed the status filter dropdown from the Articles filter bar on every tab.** The stat-card tabs already switch status, so the redundant "All Statuses" dropdown was confusing — a tab now shows only its own status. The Pillar filter, Filter, and Clear controls remain; filtering by pillar preserves the current tab's status, and Clear drops the pillar while staying on the current tab.

## [4.2.3]

### Deprecated
- **YouTube & Reddit Search Proxy section hidden from the admin, pending removal.** The proxy setting is no longer rendered on the Settings screen, so it can't be enabled or used from the UI. All proxy code — the `section-proxy.php` template, the AJAX handler, and its JS — is intentionally left in place and can be reactivated by restoring a single `require` in `templates/admin/settings.php`. This is a hide-not-remove step; the section may be removed or brought back in a later release.

### Changed
- **Robust community mining in the seed-ideas and article-ideas stages.** Real runs found `research_communities` mining came back weak — a lone `site:<community> <term>` query returned mostly noise (BuiltWith profiles, unrelated vendor forums) rather than discussion threads, and the model quietly compensated with broader market research, losing the community grounding the stages intend. Both stages now carry a **Community-mining protocol**: a query matrix of intent-shaped searches (`"not working"`, `"how do I"`, `"anyone else"`, `"vs"`, `error/won't/keeps`, …) run per community × pillar term, a relevance gate that discards directory/profile pages and off-topic-product forums, a two-tier fallback (drop `site:`, then adjacent communities) before abandoning a community, recurrence as the free demand signal in place of upvote counts, and a requirement to **record** when community grounding was thin so the fallback to market research is logged, not silent.

## [4.2.2]

### Fixed
- **Setup screen: "Prepare my changes" now scrolls back to the top and keeps the instructions on screen.** After preparing a prompt, the page and the prompt box scrolled to the very bottom of the (long) copied message, hiding the "paste this whole message" instruction at its top, and the confirmation toast vanished after 3.5s — before users had read what to do. The view now scrolls the box's top back into view (resetting its internal scroll too), and a persistent **Next step:** block tells users exactly what to do — open Claude (Claude Desktop), start a new chat, and paste the whole message in — and stays visible for 25 seconds.

## [4.2.1]

### Added
- **More CTA positions.** The CTA slot picker gains three mid-article positions — **one third down**, **halfway down**, and **two thirds down** — alongside the existing after-intro, before-conclusion, and end. The fractional positions snap to the nearest section heading so the button always lands on a clean block boundary (and fall back to appending when an article has no headings).

### Fixed
- **CTA button focus visibility (accessibility).** The injected CTA action button previously signalled keyboard focus only by swapping its background colour, which can be invisible against some themes. Added an explicit `:focus-visible` outline (themeable via `--ct-cta-focus`) so the button always shows a distinct focus ring (WCAG 2.4.7).

## [4.2.0]

### Added
- **Self-promotion system** — two separate rails that let a client's own products show up in their own content.
  1. **Self-citation.** A product catalogue (`products`: `name`, one-line `summary`, `market_tags`, `url`, `merit`) drives the pipeline to test every topic against each product's market and, when it fits, include that product as a legitimate entry positioned on merit — final third, in context, balanced, never in the first half. The ideas stage records a `product_angle` on the research package, the writer must cite it, and a new Stage-2 audit category (**Self-Citation**) gates it (N/A → 100 when nothing matched). Closes the gap where a competitive-landscape article could silently omit the client's own product.
  2. **CTA modules.** Structural call-to-action buttons managed in **Settings → Call to Action** and injected at render time via `Frontend\CTA_Injector` on `the_content` — never written by the model, and stored in their own `cherry_tree_ctas` option (`Database\CTA_Repository`), separate from the baked skill, so a CTA edit takes effect immediately and never triggers a rebuild. Each has a headline, optional line, button label + URL, a slot (after the intro, one-third / halfway / two-thirds down, before the conclusion, or the end — the fractional slots snap to the nearest section heading), an `enabled` toggle, and per-pillar targeting (none = all articles); one CTA per slot, capped at 0–3 (default 1) per article, with a baseline boxed-button stylesheet.

### Changed
- **Product placement is now baked, not client-configured.** Removed the `product_name`, `product_mention_section`, `product_max_sentences_per_mention`, and `product_forbidden_in_intro` config keys in favour of the `products` catalogue; the placement rules (final third, on merit, never in the first half) are baked invariants. Setup Section 3 now collects the product catalogue. The writer no longer authors a CTA close — the References section is the article's last element and CTAs are injected by the plugin.
- **REST API rate limits raised for sustained multi-session publishing, and made overridable.** The six `CHERRY_TREE_RATE_LIMIT_*` ceilings in the plugin bootstrap were sized for a single setup workflow, so a fleet of parallel publishing sessions — which share one rate-limit bucket, since the limiter keys on identifier (authenticated user, else egress IP) — hit `429 Too Many Requests` under load: writes topped out at 300/hour (~30 articles) and deletes at just 10/hour. New ceilings: read `120/min · 1000/hour` (was 60/300), write `150/min · 1000/hour` (was 50/300), delete `40/min · 150/hour` (was 5/10). These endpoints are all API-key gated, so the key check — not this limiter — is the abuse wall; the limiter's role is server protection, and the ceilings are sized for comfortable headroom accordingly.
- **Rate-limit constants are now overridable from `wp-config.php`.** Each `CHERRY_TREE_RATE_LIMIT_*` `define()` is guarded with `if ( ! defined() )`, so an install can tune any ceiling against real database load without shipping a plugin release. The shipped values are the defaults.

## [4.1.6]

### Changed
- **`Content\Publish_Indexer`** — the publish-time `indexed_at` stamper now derives the planning-queue table name from the canonical `Database\Schema::get_table_name()` helper instead of re-building the `$wpdb->prefix . 'cherry_tree_queue'` literal inline, so there is a single source of truth for the table name. Behaviour is unchanged.

### Removed
- Purged all remaining **multi-tenant** code and artifacts now that the plugin is single-tenant throughout. Deleted the pre-rebuild skill bundles (the `WAP-ready-cherry-tree-skills` bundle and the `seresa-aeo-*` skill set), the archived multi-tenant `publish_helpers.py` copies, the `clients-config.json` registry, and every `client-keys.yaml` client-key file. Nothing in the shipped code paths referenced them; the single-tenant `cherry_tree_client` (baked `credentials.json`) is the sole client.

### Changed
- Content Skill client: the supported **Python floor is now 3.12** (was 3.9). `cherry-tree-client/pyproject.toml` declares `requires-python = ">=3.12"`, and every `cherry_tree_client` module docblock records `Requires: Python 3.12`. The full client suite passes on a real 3.12 interpreter.

### Fixed
- Content Skill client: closed two **SSRF-style bypasses** in the `site_url` guard — a trailing FQDN-root dot (`localhost.`, `127.0.0.1.`) and integer IP literals (`https://2130706433/`, `https://0x7f000001/`, `https://0/`) both defeated the private/loopback check and reached the transport; the host is now normalised (trailing dot stripped) and integer forms are resolved to their address before classification.
- Content Skill client: hardened the network and audit boundaries against malformed input — `GET /info` and `claim()` now raise the typed `InfoError`/`ClaimError` on a non-object JSON body instead of leaking an `AttributeError`, and the article/seed structural audits coerce `None` fields (from malformed LLM frontmatter) so they fail the check cleanly rather than throwing `TypeError`. `CherryTreeClient.slug()` raises a typed `InfoError` on an unknown key instead of a bare `KeyError`.
- Content Skill client: `list_queue()` and `list_published()` now raise the typed `PublishError` on a non-2xx response instead of `requests`' own `HTTPError`, so every transport failure is catchable through the one `CherryTreeError` family like the other endpoints.
- Content Skill client: the Stage-2 score gate (`audit_stage_2_pass`) no longer raises `ZeroDivisionError` on an empty category list (it raises a clear `ValueError`), and it now scores only the required categories so extra keys in the score map can't skew the average or add spurious failures.

## [4.1.5]

### Changed
- Refactored the `Plugin_Core` orchestrator (was 918 lines, past the 500-line ceiling) into a clear modular structure. Extracted its leaf callback logic into dedicated, independently-testable domain classes — `Content\Publish_Indexer` (planning-queue `indexed_at` stamping on publish), `Frontend\Link_Target_Filter` (internal-link `target` stripping), `Frontend\Rewrite_Flusher` (rewrite flush + page-cache purge), and `Cron\Cron_Schedules` (custom cron interval) — and moved the manual `require_once` manifest into `Includes\Dependency_Loader`. `Plugin_Core` is now purely hook wiring; behaviour and hook names/priorities are unchanged.

### Fixed
- Content Skill client: the `site_url` private-address guard now parses the host as an IP (`ipaddress`) instead of matching string prefixes. The old `"172.16."` prefix let the rest of the `172.16.0.0/12` block (`172.17`–`172.31`) through, and IPv6 unique-local/link-local hosts were unguarded; both are now rejected, while genuinely public hosts outside the block (e.g. `172.32.x`) are still allowed.

## [4.1.4]

### Fixed
- Content Skill: the stage docs and baked `credentials.json` no longer disagree about `api_base`. The stages built the client with `CherryTreeClient(site_url, api_key, api_base)` — but `api_base` is baked as a *full URL* while the third argument is a *path*, so construction threw `ConfigError`. Added `CherryTreeClient.from_credentials(cred)` as the single, unambiguous entry point (it derives the REST path from `api_base`) and switched every stage to it; direct-request setup now uses `api_base` as the full base URL instead of `site_url + api_base`.
- Content Skill: `invariants.md` is now shipped at `baked/invariants.md`, where `SKILL.md` and every stage reference it. The assembler previously placed it at the skill root, so the referenced file did not exist. All stage cross-references normalised to `baked/invariants.md`.
- Content Skill build: the skill assembler now normalises its `template_dir` argument with `trailingslashit()`. Without the trailing slash the computed relative paths gained a leading `/`, so the `SKILL.md` match failed and the built skill shipped with its `{{SITE_URL}}`/`{{API_BASE}}` placeholders left unfilled. Hardened against a caller that omits the slash.
- Content Skill client: a failed `GET /info` boot now returns an actionable message (WAF / User-Agent / auth hint plus a response-body snippet) instead of a bare `HTTP 403`.

### Changed
- Content Skill: the article- and seed-ideas stages now include the client's own product in competitive/landscape research when `product_name` competes in the topic's space, so the market research isn't lopsided. Ideas stages also ignore categories whose name is purely numeric (migration/test artefacts), and `SKILL.md` now states the install's `content_types` scope upfront rather than only declining mid-stage.

## [4.1.3]

### Fixed
- The "Title:" label shown under an article's topic on the Articles screen is now translatable (previously a hardcoded English string).
- Fixed double-escaping of the rejected-by label in the Articles screen's row actions — rejecter names containing `&`, `'`, or `<` now display correctly instead of showing literal HTML entities.

## [4.1.2]

### Changed
- The published articles list now floats items still needing indexing (no index date) to the top by default, so they no longer appear at random positions. An explicit column sort is still respected.
- Extracted the Articles screen's view helpers (sortable column URL/class and per-row action buttons) out of the `articles.php` template into a namespaced `Articles_View` class, bringing the template back under the file-size ceiling and making the helpers unit-testable.

## [4.1.1]

### Fixed
- Fixed a TypeError on the seeds page caused by the `branch_slug` parameter.

### Changed
- Minor unit test fixes.

## [4.1.0]

### Changed
- Integrated the Cherry Bee indexer into Cherry Tree — search-engine submission (Google, Bing, IndexNow) now ships as part of Cherry Tree rather than a separate plugin.

## [4.0.1]

### Changed
- Internal code-quality pass: resolved all WordPress Coding Standards (PHPCS/WPCS) findings — docblock corrections and a documented, safe prepared-SQL annotation. No functional or behavioural changes.

## [4.0.0]

### Added
- Release scripts for building and publishing releases.

## [3.9.1]

### Changed
- The welcome video now auto-opens only on the Content Skill tab.

## [3.9.0]

### Changed
- Large security-hardening refactor across the API controllers, admin AJAX handlers, and admin JavaScript.

### Fixed
- Silent database schema failures during activation and upgrade are now surfaced to the error log instead of being swallowed.

## [3.8.3]

### Added
- New modal promoting Cherry Tree Lite.

### Changed
- Updated the proxy admin copy.

## [3.8.2]

### Changed
- Further minor copy changes on the admin screens.

## [3.8.1]

### Changed
- Minor copy changes on the admin screens.

## [3.8.0]

### Added
- YouTube and Reddit proxy options.
- Automatic plugin updates from GitHub.

### Changed
- Updated the copyright company name to include Singapore across all files.

## [3.7.0]

### Removed
- Removed the mesh system so it can be rebuilt later in a better format.

## [3.6.11]

### Fixed
- Fixed the admin copy for downloading the skill.

## [3.6.10]

### Added
- Added info for the Build & Lock button.

## [3.6.9]

### Fixed
- Minor formatting fixes on the confirmation page.

## [3.6.8]

### Fixed
- Updated the info collected for the AI context document.

## [3.6.7]

### Fixed
- Fixed the welcome modal auto pop-up not triggering.

## [3.6.6]

### Changed
- Refined the welcome popup modal settings and styling.

## [3.6.5]

### Changed
- Expanded the install welcome modal with more information.

### Fixed
- Fixed the welcome modal not appearing on reinstall.

## [3.6.4]

### Fixed
- Fixed drift in the admin list API endpoints.

## [3.6.3]

### Added
- More informative feedback on button clicks during setup.

### Fixed
- Fixed a video playback issue.

## [3.6.2]

### Added
- Intro walkthrough shown in a video-player modal.

### Changed
- Increased the API rate limits for better workability.

### Fixed
- Reduced the width of the admin form fields.

## [3.6.1]

### Fixed
- API error paths (e.g. `POST /queue/{id}/lock` and `/unlock` for a missing item)
  returned HTTP 500 instead of a proper 4xx. The `Api_Error` class was never
  loaded via `require_once` and the plugin ships no runtime autoloader, so any
  code path that built an instructive error fatalled with "class not found".
  Added the missing require and a regression test asserting every `src` class is
  explicitly required.

### Changed
- API Health diagnostic now reports a 5xx as a "Server error — check logs"
  (plugin/server fault) rather than mislabelling it as a firewall block.

## [3.6.0]

### Changed
- Updated log management.

### Fixed
- Fixed AI reporting.

## [3.5.1]

### Changed
- Updated unit tests.
- Restricted log file sizes in the logger.

## [3.5.0]

### Changed
- Refactored several large PHP files into smaller components.
- Changed the namespace to `Seresa\CherryTree` to avoid conflicts in future.

## [3.4.0]

### Changed
- Updated the AI instruction doc.
- Split `admin.js` into smaller modules for better test coverage.
- Refactored several large files, breaking them into smaller component modules.

## [3.3.6]

### Fixed
- Disabled the skill download button until the skill build is actually complete — previously the skill could be downloaded before it was finished.

## [3.3.5]

### Removed
- Removed the AI paste field from the setup admin.

## [3.3.4]

Major rebuild: the five per-client Claude skills collapse into **one Content Skill that the plugin
compiles from a single config**. WordPress becomes the product; the skill is its build output. Setup is
AI-driven — the AI writes config to WordPress directly via the REST API — with a thin state-mirroring
admin, no 30-field form. All other subsystems (Mesh, Klaviyo, Email, WAF) are untouched.

### Added
- **AI-driven setup, direct API.** New thin admin screen (`Config_Page`, its own submenu) mirrors
  config state (empty / complete / locked) with one primary button per state — Copy Setup Instructions
  / Prepare my changes / Build & Lock. Copyable prompts (`ai-prompts/setup.md`, `edit.md`) bake in the
  site URL, API key, endpoints, field schema, and live category list; the AI researches, confirms with
  the owner, then writes config to WordPress itself via `PATCH /config` (`X-API-Key`).
  The API key never leaves WordPress.
- **Config store.** New `cherry_tree_config` (JSON, ~30 fields across Market/Brand/Voice/Publishing) and
  `cherry_tree_config_hash` (SHA-256) options, owned solely by `Config_Repository` (written lock-step so
  they cannot drift). Seeded at activation, removed on uninstall.
- **Config domain (`src/Config/`).** `Config_Schema`, `Config_State`, `Config_Sanitizer`,
  `Config_Secret_Scanner`, `Config_Field_Validator`, `Config_Validator`, `Config_Message`,
  `Config_Service` (single write path), `Prompt_Builder`, `Category_Lister`,
  `Config_Factory`. New REST `Client_Config_Controller` — `GET /config`, `GET /config/schema`,
  `PATCH /config`.
- **Talk-back contract (§0b).** Every validation response — especially errors — returns full plain-English
  what-happened / why / how-to-fix / what-next prose that survives the copy-paste round-trip to the AI.
  Every error path is tested.
- **The compiler / Build & Lock (`src/Compiler/`).** `Skill_Assembler`, `Skill_Packager`,
  `Skill_Repository`, `Skill_Compiler` — validate the full config (fails loud on missing thresholds,
  pillars without categories, empty core promise/sources, placeholder text, or a foreign/cross-tenant
  secret) → bake config + credentials → stamp `built-date` + `config_hash` → zip → store → lock. The
  built skill is one current base64 `.zip` in an option (no history/rollback). Nonce-protected download.
  The install's own API key + site URL + API base are baked into the skill deliberately (RQ3).
- **Bundled master skill template (`skill-template/`).** `SKILL.md` (boot + drift gate → 4-stage router),
  `invariants.md` (two-stage audit model, 90/95/100 scale with thresholds injected from config, HTML/class
  contract + Yoast FAQ markup, article & seed Stage-2 rubrics, Cherry Rose persona), `stages/{article-ideas,
  article-final,seed-ideas,seed-final}.md`, and `config.schema.json`.
- **`cherry-tree-client` Python package.** Lifted from `publish_helpers.py`, single-tenant, modular
  (`client`, `transport`, `audits`, `seed_audits`, `gates`, `constants`, `validation`, `errors`). New
  `audit_seed_structural` (7 mechanical seed checks) and `assert_published` (raises unless
  `queue_status=="review"`). Thresholds injected from baked config — no hardcoded fallback. Vendored into
  the skill template so the compiled skill is self-contained (no runtime pip install).
- **Drift gate.** `config_hash` added to `GET /info`; the runtime skill checks at boot that its baked
  hash and pillar category IDs still match WordPress, and halts-with-choice on a mismatch.
- **Tests.** New PHP unit coverage across the config, compiler, and skill-assembly surfaces; new Vitest
  suite for `cherry-config.js`; pytest for the Python package. Totals green: PHP 2546 · JS 15 · Python 79.
- **Install guide** (`readme/cherry-tree-content-skill-guide.md`) covering install → setup → Build & Lock
  → add to Claude Desktop.

### Changed
- **Five skills → one compiled Content Skill**, single-tenant and Claude-only. The four production stages
  (article/seed × ideas/final) become routes in one skill loading baked references on demand; `client-setup`
  becomes the generated setup/edit prompts. De-Seresa'd throughout: research targeting is config-driven
  (`research_communities`) instead of hardcoded subreddits.
- New settings-screen JS/CSS ship as separate `cherry-config.js` / `cherry-config.css`; the admin
  monoliths (`settings.php`, `admin.js`, `admin.css`, `class-sanitizer.php`) were left untouched. Every
  new code file is ≤300 lines.
- Boot-time YAML config fetch replaced by baked `baked/config.json`; runtime endpoint slugs still resolve
  from `GET /info` (never baked), respecting WAF-mutable action slugs.

### Removed
- Multi-tenant machinery in the Python helper (~155 lines): `clients-config.json` resolution, the boot
  YAML fetch/cache, and the legacy `claim_article` PATCH path. The Seresa-only YouTube research source
  and `X-Seresa-Token` are gone.

### Notes
- One remaining human step (§0c): pasting the finalized prompts into a live AI and running a full
  setup / resume / edit end-to-end cannot be done headless — everything up to that boundary is built and
  self-tested.

## [3.3.0]

### Changed
- General clean-up in preparation for the open source release.

## [3.2.0]

### Added
- WAF compatibility layer so server firewalls (ModSecurity / OWASP CRS on shared hosting) can no longer silently block the REST API.
- Admin-configurable action slugs and API User-Agent via a new **Cherry Tree → API Health** page. Slugs are validated (lowercase alphanumeric plus hyphens, unique per track, max 30 characters) and stored in `wp_options`; routes read from there at registration.
- Endpoint diagnostic with one-click auto-fix. The API Health page runs a loopback self-test (200/404/400 = reachable, 406/403 = firewall-blocked) and, for any blocked endpoint, assigns fresh randomized slugs and re-tests. A dismissible admin notice flags blocked endpoints until they pass.
- Bypass documentation banner shown on the API Health page whenever a custom slug or User-Agent is active.
- `GET /info` configuration endpoint returning current route aliases, User-Agent, and encoding flag (API-key protected) so pipeline clients auto-discover the correct paths.
- Optional base64 content encoding for `POST /posts` via the `X-Content-Encoding: base64` header, so a firewall body scanner cannot flag article HTML that resembles SQL or XSS. Off by default.

### Changed
- Renamed action endpoints to WAF-safe slugs with no backward-compatibility aliases:
  - `/queue/{id}/claim` → `/queue/{id}/lock`
  - `/queue/{id}/release` → `/queue/{id}/unlock`
  - `/seeds/{id}/claim` → `/seeds/{id}/lock`
  - `/seeds/claim-next` → `/seeds/lock-next`

  Callbacks, permissions, the `processing` status, and the `claim_token` field are unchanged — only the URLs move.

## [3.1.0]

### Added
- API Key section in Settings with masked display, reveal/hide toggle, and copy-to-clipboard.

### Changed
- API key management moved to admin settings. The key is auto-generated on activation and stored encrypted (AES-256-CBC) in the database; the `API_KEY_CHERRY_TREE` wp-config.php constant is no longer required.
- Centralized API key verification. Removed 10 duplicate `verify_api_key()` methods across API controllers; all endpoints now use the shared `API_Key_Manager::verify_request()`.
- Database version bumped to 1.7.0 — existing installs auto-generate a key on upgrade.

## [2.7.2]

### Removed
- Leftover IndexNow UI block from the settings page, along with the `CHERRY_TREE_INDEXNOW_KEY` constant and the unused `indexnow` tooltip label. Search-engine submission has been Cherry Bee's responsibility since 2.7.0.
- Dead `wp_cherry_tree_index_history` table, dropped on upgrade via `cherry_tree_migrate_drop_index_history()`. Database version bumped to 1.5.0.
- IndexNow and Google Indexing API third-party-service documentation from the readme; those services are documented by Cherry Bee.

## [2.7.1]

### Changed
- Skip admin-hook registration on public requests. `Plugin_Core::__construct()` now only calls `define_admin_hooks()` when `is_admin() || wp_doing_ajax() || wp_doing_cron()`.
- Extracted `define_content_hooks()` so the `transition_post_status` listener keeps firing for posts published via REST API, WP-CLI, or programmatic `wp_insert_post()` calls.
- Added namespace-qualified `\` prefixes to global function calls for correct namespace resolution and to satisfy strict static analyzers.

## [2.7.0]

### Changed
- Cherry Bee is now the single search-engine submitter. Cherry Tree no longer pings IndexNow or the Google Indexing API directly on publish; submission is delegated to Cherry Bee's unified queue.
- Renamed `Plugin_Core::maybe_auto_index_on_publish()` to `mark_planning_queue_indexed_on_publish()` to reflect that it now only updates `indexed_at` on the planning-queue row.
- Added info-level logging on every seed-sitemap cache invalidation.

### Fixed
- Seed-sitemap object cache never invalidated. The invalidator was hooked to a non-existent action name; it is now hooked to `cherry_tree_seed_status_transitioned` plus the create/update/delete actions, so newly published seeds appear in `/seeds-sitemap.xml` immediately.
- `uninstall.php` left orphan tables. It now drops all 8 tables created by the activator (`seeds`, `branches`, and `api_log` were previously missed).

### Removed
- `Seresa\CherryTree\Indexing\Google_Indexing` class and its admin settings UI section. Google credentials are managed from Cherry Bee → Settings → Google Indexing API.

## [1.1.0]

### Changed
- **Plugin renamed from "AEO Publisher Seresa" to "Cherry Tree by Seresa".**
- Namespace updated from `Aeo_Publisher_Seresa` to `Seresa\CherryTree`.
- REST API namespace changed from `/seresa/v1/` to `/cherry-tree-by-seresa/v1/`.
- Database tables renamed from `wp_seresa_*` to `wp_cherry_tree_*`.
- API key constant renamed from `API_KEY_SERESA_AEO_PUBLISHER` to `API_KEY_CHERRY_TREE`.
- All internal constants updated to the `CHERRY_TREE_` prefix.

## [1.0.0]

### Added
- Initial release as Cherry Tree by Seresa.
- REST API endpoints for content queue management.
- Admin dashboard with filtering and sorting.
- Status workflow implementation.
- Rate limiting.
- Comprehensive logging.
- YAML control files for AI skill configurations.
- Mesh internal linking system.
- IndexNow and Google Indexing API integration.
