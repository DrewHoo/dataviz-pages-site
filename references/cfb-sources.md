# College-football data sources (and their specific lies)

Read this when a project needs CFB games, polls, coaches, or logos. Everything
here was learned the hard way on `hostile-territory` (road records vs the AP
top 10); the receipts for each claim live in that repo's `data/research/`.

## Bulk sources that work with plain curl

| need | source | caveats |
| --- | --- | --- |
| Games 2001+ | `raw.githubusercontent.com/sportsdataverse/cfbfastR-data/main/schedules/csv/cfb_schedules_<yr>.csv` | See the lies below. Updates in-season. No pre-2001 files. |
| Games 1990–2000 | jhowell.net per-team files | `@` rows are exactly the true road games and mirror-dedupe for free. Files exist only for I-A teams, so FCS visitors self-exclude. |
| Weekly AP polls 1990+ | collegepollarchive.com | Row layout: bold cell = current rank, then movement arrow, then previous rank — the cell adjacent to the team anchor is the PREVIOUS rank. |
| Team logos | `a.espncdn.com/i/teamlogos/ncaa/500/<espn_id>.png` | The `home_id`/`away_id` columns in cfbfastR ARE ESPN ids. |

## cfbfastR's specific lies

- **`start_date` is UTC**, so Saturday night kickoffs stamp Sunday and match the
  wrong AP poll. A -8h shift fixed 99.86% of dates when validated against
  jhowell's independent dates. But 2001 rows with `T04:00:00.000Z` are
  midnight-ET placeholders that the shift moves a day EARLY.
- **`neutral_site` is wrong for 2001–2007 rivalry and title games**: the
  Florida–Georgia Jacksonville series, every Red River game, SEC/Big 12/ACC
  championship games in domes and Arrowhead all read as home games. Every case
  found fell in that window. Post-2007 flags were clean.
- The 2001 file also carries a few garbage scores (UNC–Oklahoma "10-0" for a
  41-27 game).
- Team-name variants: raw "Miami" is Miami (FL); "Hawai'i", "San José State".
- Division columns go `NA` on a few hundred rows; derive each season's FBS set
  from the labeled rows and hold division-less visitors to membership in it.

## No-key ESPN APIs (the adjudicators)

- **AP poll for any week**: `sports.core.api.espn.com/v2/sports/football/leagues/college-football/seasons/<yr>/types/2/weeks/<wk>/rankings/1`.
  Settled five rank disputes where Sports-Reference's schedule column
  disagreed with College Poll Archive — ESPN sided with CPA all five times.
- **Game detail / overtime**: `site.api.espn.com/.../summary?event=<game_id>`
  (cfbfastR's game_id). `status.type.shortDetail` reads `Final/2OT`. ~300ms
  spacing is tolerated; a handful of 2001 event ids are dead.

## Sports-Reference through the Wayback Machine

SR 403s curl AND WebFetch, but `web.archive.org/web/<season+1>/https://www.sports-reference.com/cfb/schools/<slug>/<season>-schedule.html`
serves the real page. Schedule rows carry BOTH teams' ranks at game time, the
site marker (`''` home / `@` road / `N` neutral), and the result — the perfect
per-game receipt. Gotchas, all hit in production:

- The year picker can land on a PRESEASON capture with blank result cells —
  check for a result before trusting, and pin a post-season capture via the
  CDX API when needed. Current-season pages often have no post-game capture
  at all; re-run months later.
- Column layouts vary by era (no Time column pre-2000s, ties column in the
  90s); anchor parsing on the date cell and header labels, never fixed offsets.
- SR's own rank column is sometimes wrong (five documented cases) and its
  stale snapshots can disagree with its current page — adjudicate with the
  ESPN rankings API before "fixing" anything toward SR.
- Wayback throttles parallel fleets hard (curl code 000, spurious 404s).
  9–12s gaps per agent, and treat two-strike failures as retryable, not
  missing.
- Slugs: mississippi (Ole Miss), southern-california (USC), texas-christian,
  brigham-young, north-carolina-state, miami-fl / miami-oh, connecticut.
  BYU's page title says "Brigham Young Cougars" — don't title-check by
  common name.

**Fetch-count inversion:** verifying visitor games via the HOME team's season
page cut 1,495 pages to 600 — ranked hosts concentrate.

## Logos as one-color marks

ESPN's 500px PNGs flatten interior counters (Georgia's G, FSU's spear) to
opaque white, so a CSS `brightness(0)` cutout fills them into blobs. The
pipeline that works: at FULL resolution, bake ink-density alpha =
`max((1-luminance)*1.2, saturation*0.9)` onto pure black, then trim
transparent padding (normalizes optical size across schools), then resize.
One asset serves both treatments: wins = plain `opacity`, losses =
`filter: invert(.66)`. Never threshold-key after resizing — anti-aliased
boundary pixels half-key into speckle.

## Judgment conventions worth pre-deciding

- Acting-coach games (suspension with game-day ban, medical leave) are
  credited per the NCAA/school record — Harbaugh keeps his 2023 November
  games, Kill keeps his 2013 leave games. Encode per-game in a versioned
  overrides file, because a narrowest-dated-window join tiebreak picks the
  acting coach.
- A championship game on a campus site (2011 Pac-12 CG at Autzen, G5 title
  games at the higher seed) IS a true road game; the neutral flag decides,
  not the game type. UTSA's home stadium is the Alamodome.
- Wikipedia coach-list pages omit interims often enough that season articles
  must be cross-checked; `{{Sortname}}` templates mean full names don't exist
  as contiguous strings in wikitext, so name-grep silently misses.
