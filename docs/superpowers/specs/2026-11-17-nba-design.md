# NBA Q&A + Trivia: design

Status: approved for planning (2026-11-17)

**Scope update (2026-11-17, from the user):**
- Local-only, run on the laptop. No hosting.
- **No incremental or nightly updates.** One full import that can be re-run from scratch.
- The first plan covers **player and team profiles with Wikipedia enrichment**, which is build-order steps 1 and 2. Ask and Quiz get their own plans later.
- The audience is the user, locally, so there are no accounts.

Sections 4 and 8 still describe the nightly and hosted design. They are kept for later and are out of scope now.

## 1. Intent

An NBA companion that does two things well:

- **Ask anything.** Free-form questions answered by an AI. The answers are grounded in official stats for the numbers and in Wikipedia for the stories. Example: "how many threes did JJ Redick make in 2015-16, and why did the Lakers hire him as coach?"
- **Quiz me.** Random trivia rounds. Every answer key is computed from real data, so the quiz is never wrong.

It is also the follow-up learning project to the HN semantic-search reader. The ideas carry over: hybrid search, Elasticsearch, embeddings and evaluation. Two new ones are added: an AI agent that calls tools (text-to-SQL), and an entity model with history.

**The hard problem is people over time.** A person can be a player, then a broadcaster, then a coach (JJ Redick). A person can change name (Lew Alcindor → Kareem Abdul-Jabbar). A team can move (Seattle SuperSonics → Oklahoma City Thunder). The data model is built around this.

## 2. Goals, non-goals, anti-goals

**Goals**
1. Answer factual NBA questions correctly, with the source visible: which stat table, which wiki article.
2. Generate fun, varied trivia whose answers are provably correct.
3. Track people and teams across roles, names and eras.
4. Fast person search: names, nicknames, former names, typos, and descriptions such as "lefty Knicks shooter from the 90s".
5. An admin can correct data and tune the AI without code changes.

**Non-goals (v1)**
- Live scores, betting, predictions, fantasy.
- Play-by-play data. Box scores are enough for v1.
- WNBA, ABA, G League, college.
- Native mobile apps. The app is a responsive web app (PWA), like the HN reader.

**Anti-goals: failure even if it "works"**
- The AI states a number that is not in a tool result. That is a hallucinated stat.
- A quiz marks a correct answer wrong.
- An admin correction is silently overwritten by the next nightly import.
- AI spend creeps up unnoticed.

## 3. Stack

The stack is the same as the HN project. It is familiar, it is proven on this laptop, and it can share one server.

| Layer | Choice | Why |
|---|---|---|
| Backend | Python 3.12, FastAPI, uv | Same as HN; `nba_api` is Python |
| Main store | SQLite (WAL) | Small enough (about 2M box-score rows); one file; trivial backups |
| Search | Elasticsearch 9 (shared instance, `nba-*` indices) | Person search and wiki-passage hybrid search; reuses the HN setup |
| Embeddings and rerank | Voyage (`voyage-4`, `rerank-3-lite`) | Existing key, 200M free tokens, cached as in HN |
| LLM | Anthropic SDK: Q&A agent on `claude-sonnet-5-5`; quiz phrasing and fun facts on `claude-haiku-5-5` | The agent needs good tool use; phrasing is cheap and high-volume. Models are admin-switchable (section 7). |
| Web | React, Vite, Tailwind, TanStack Query, PWA | Same as HN |

**Cost controls (carried over from HN):**
- a per-request token cap;
- a daily spend cap in config;
- every LLM call logged with its tokens;
- quiz text cached and reused, so it is not regenerated per play.

## 4. Data sources

| Source | Used for | How |
|---|---|---|
| **Kaggle "basketball" dataset** (wyattowalsh; SQLite; built from stats.nba.com via `nba_api` by the nbadb project) | Bootstrap: all games since 1946, box scores, players, teams, draft, common player info | Downloaded with the Kaggle API (needs the user's Kaggle token). The maintainer says it refreshes daily or monthly; **check the "last updated" date before relying on it**. |
| **`nba_api`** (stats.nba.com) | Nightly deltas: yesterday's games, rosters, coaching staffs per season, awards | Throttled (1 request/second or slower). **stats.nba.com often ignores cloud-server IP ranges**, so this job runs from the laptop or home, or goes through `NBA_API_PROXY`. If the Kaggle dump stays fresh, the server can depend on Kaggle alone. |
| **Wikidata** (SPARQL) | Identity glue and role history: positions held (head coach, GM, owner) with start and end dates; names and aliases; birth data; links to Wikipedia; the NBA.com and Basketball-Reference IDs on many people | Weekly sync |
| **Wikipedia** (REST API) | Narrative: biographies, team histories, season articles | Weekly sync; split into sections, embedded, indexed in ES for retrieval |
| Basketball-Reference | **Not ingested.** Its terms forbid bulk scraping. It can be used by hand to check facts. | — |

The Wikimedia APIs are called with a generic User-Agent naming the app, with no personal email.

## 5. Data model

Principle: **one record per real-world thing, plus dated facts about it.** Every row carries `source`, `source_id` and `fetched_at`.

### Identity
- `person`: `id`, `nba_id`, `wikidata_qid`, `bbref_id`, `birth_date`, `birth_place`, `height_cm`, `weight_kg`, `hand` (when known), `wiki_title`, `prominence` (computed; see search).
- `person_name`: `person_id`, `name`, `kind` (`display` | `legal` | `former` | `nickname`), `valid_from`, `valid_to`. Examples: "Lew Alcindor" `former` until 1971; "The Mailman" `nickname`.
- `franchise`: the continuous business entity, for example Sonics/Thunder.
- `team_era`: `franchise_id`, `city`, `name`, `abbrev`, `from_season`, `to_season`.
  - Franchise history rights follow the league's official convention. The Charlotte Hornets' 1988–2002 history belongs to Charlotte, not New Orleans.
  - How Seattle-era records are treated is a per-quiz setting: "franchise records" versus "this city/name only".

### Roles over time: the core
- `role`: `person_id`, `role_type`, `franchise_id` (nullable), `from_date`, `to_date` (null means current), `source`, `confidence`.
  - `role_type` values: `player`, `head_coach`, `assistant_coach`, `executive` (GM or president), `owner`, `broadcaster`.
  - The list is extensible from the admin.
- `player_stint`: player × team × season, with games and dates. A mid-season trade gives two stints. Derived from season stats.

This answers questions like "current head coaches who played in the NBA", "everyone who both played and coached for the Celtics", and "what was JJ Redick doing in 2022?"

### Stats (from the NBA, kept close to source shape)
- `season` (`2024-25`, with `season_type`: regular, playoffs, play-in).
- `game` and `game_team_box`.
- `game_player_box`.
- `player_season` (per team, plus a combined `TOT` row).
- `team_season` (record, seed, result).
- `draft_pick`.
- `award` (MVP, ROY, DPOY, All-NBA, All-Star, Finals MVP, championship): `person_id`, `award_type`, `season`, `franchise_id`.

**For the AI**, read-only SQL views with friendly names and documented columns: `v_player_career`, `v_player_season`, `v_coach_record`, `v_awards`, `v_team_history`. The agent queries views, never raw tables, so the schema can change without breaking prompts.

### Narrative
- `wiki_passage`: `id`, `entity_type`, `entity_id`, `article`, `section`, `text`.
  - Indexed in ES with BM25 and vector search, the same hybrid approach as HN.
  - Linked to people, franchises and seasons, so retrieval can filter by entity.

### Linking NBA ↔ Wikidata
- Exact match on the NBA ID stored in Wikidata where it exists.
- Otherwise match on name, birth date and teams. Each match gets a `confidence`.
- Low-confidence matches go to the admin review queue (section 7). They are never silently linked.

## 6. Features

### 6.1 Ask (AI Q&A)
A Claude agent with three tools:
1. `sql(query)`: runs read-only on the views, with a row cap and a timeout.
2. `search_wiki(query, entity?)`: hybrid ES retrieval with rerank, returning passages.
3. `person(name_or_id)`: identity plus role timeline.

Rules:
- Every number in the answer must come from a tool result.
- The answer shows its sources: the SQL used and the wiki passages, behind a "how I know" toggle.
- If the data has no answer, the agent says so.

Other features:
- **"Surprise me":** the agent picks an interesting real fact from the data, such as a record or an odd career path, and presents it as a question and answer.
- **Follow-ups** within a conversation.

### 6.2 Quiz (trivia)
- **Question generators are templates over SQL.** Each template has: a parameter picker (random person, team or season, weighted by prominence for the difficulty level), an answer query, a distractor strategy, and a phrasing prompt.
- **Initial template families:**
  - season leaders;
  - award winners;
  - draft (who went #1 in 2003?);
  - career path (which coach played for 7 teams?);
  - franchise history (the Sonics became…?);
  - "who am I" (clues revealed one by one);
  - higher or lower (career points A vs B).
- **Distractors** are plausible: same era and position, or the same team. Haiku phrases the question and adds a one-line fun fact from wiki retrieval, shown after the person answers.
- **Modes:** a quick 10-question round, an endless streak, and a "daily 5" that is the same for everyone that day.
- **Difficulty** comes from the subject's prominence plus how obscure the stat is (points is easy; defensive rating is hard).
- **Quality loop:** each question has a "this is wrong" flag. A flag disables that question; repeated flags disable the template for review.

### 6.3 Person search
- **ES index `nba-people`.** Holds every name variant (display, former, nicknames), edge-ngram autocomplete, fuzzy matching, and the teams, roles and eras as filters.
- **Ranking:** boosted by `prominence`, a mix of games, All-Star selections, awards and wiki page length. "Jordan" therefore returns Michael before DeAndre.
- **Descriptive queries** ("lefty Knicks shooter 90s") use the hybrid path over an embedded short profile per person, built from facts and the wiki lead.
- **Person page:**
  - a role timeline (player → broadcaster → coach);
  - stints;
  - season table;
  - awards;
  - wiki summary;
  - "quiz me on this person".

## 7. Admin: tuning the data and the AI

### Corrections that survive re-imports
- An `override` table holds `entity`, `entity_id`, `field` (or `add_row` / `delete_row`), `value`, `reason`, `author` and `created_at`.
- Overrides are applied after every import, so a nightly sync never undoes a fix.
- The admin UI can:
  - edit a field;
  - add or end a role (for example "JJ Redick, head_coach, LAL, from 2024-06");
  - add a nickname;
  - merge or split people;
  - fix an NBA↔Wikidata link.
- **Review queue:** low-confidence identity links, conflicts between sources (NBA versus Wikidata dates), and flagged quiz questions.

### Extending the model without migrations
- `role_type` and `award_type` are lookup tables edited in admin.
- `person.attrs` and `franchise.attrs` are JSON columns for ad-hoc facts, such as college or "nickname origin".

### AI settings (versioned)
- The model per feature (Q&A, quiz phrasing).
- System prompts and template phrasing prompts. These are edited in admin and kept with history, so any version can be rolled back.
- Quiz difficulty weights, template on/off switches, and the daily spend cap.

### Evaluation
A small question set (about 50 Q&A pairs with known answers) is re-run after any prompt or model change, showing accuracy and cost. This is the same pattern as the HN benchmark.

## 8. Hosting

**Development:** the laptop, with `make web-lan` for demos, as in HN.

**Production:** the same small Hetzner server as the HN reader (8 GB, about €8–9 a month). It shares one Elasticsearch, with each project's indices under its own prefix, behind one Caddy proxy with a subdomain per app.
- SQLite and Elasticsearch fit comfortably.
- The data refresh pulls the Kaggle dump. Because stats.nba.com blocks cloud servers, `nba_api` deltas run from the laptop or through a proxy.

**Variable cost is the LLM.**
- Quiz text is generated once and cached.
- Q&A is per question, under a daily cap.
- Voyage stays within the free tier.

## 9. Build order

1. **Data:** bootstrap from Kaggle; identity and roles model; Wikidata linking with a review queue; wiki passages indexed.
2. **Person search and the person page,** with the role timeline. This proves the model.
3. **Ask:** the agent with three tools, the "how I know" toggle, and the eval set.
4. **Quiz:** template engine, three template families first, the flag loop, quick round mode.
5. **Admin:** overrides, review queue, AI settings.
6. **Nightly sync and hosting.**

## 10. Decisions made by Claude (not yet confirmed)

- The stack mirrors the HN project.
- SQLite rather than Postgres.
- v1 covers all history since 1946 for stats; wiki narrative for people with articles.
- No user accounts in v1. Quiz scores stay on the device, and a shared "daily 5" needs no login.
- Franchise records follow the official lineage, with a per-quiz toggle.
- Play-by-play is out of v1.
- Sonnet for the agent and Haiku for phrasing. Both can be switched in admin.

## 11. Left to the builder

- Exact template SQL, the distractor heuristics and the prominence formula, tuned against the eval set.
- The ES analyzers for names. Start from the HN `en` pipeline, add `asciifolding` (Dončić, Jokić) and an edge-ngram subfield.
- UI layout. Follow the HN reader's keyboard-first, palette-style conventions.
