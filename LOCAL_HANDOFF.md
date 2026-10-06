# Handoff prompt for the local Claude Code session

Paste everything below the line into the local session, started in the cloned folder.

---

You are taking over a data-loading task for Hylton Millar, an economist in London. Read this whole brief before you run any command. Do not change the database until you have reported what you found and Hylton has agreed.

## 1. Purpose

Hylton maintains a SQL Server Express database called `bioenergy_plants` (workspace folder `Workspace\bioenergy_plants`). It holds c.4,300 power and gas plants in Great Britain and Germany: anaerobic digestion and biogas, biomethane to grid, energy from waste (EfW), and biomass. It is used for business development. The purpose of the research is to record, for each plant, who owns it, since when, how it is financed, and what operating problems it has had, with a source and a confidence level for each fact.

## 2. What has been done (in a cloud session)

A cloud Claude Code session ran one research agent per plant. The agents searched the web and read pages, then wrote one JSON record per plant. The repository is `https://github.com/hyltonmillar-stack/-bioenergy-plants` (branch `main`, public). The folder name starts with a hyphen, so clone it into a named folder: `git clone https://github.com/hyltonmillar-stack/-bioenergy-plants.git bioenergy-plants`.

Files:

- `INSTRUCTIONS.md`: the research brief that the agents followed. It defines the schema and the source and confidence rules.
- `pilot_example.json`: 20 plants that were reviewed by Hylton earlier. It is the format template. It has no confidence fields. A missing confidence means `verified`.
- `test20.csv` and `remaining.csv`: the input rows (20 and 425 plants) taken from the database. Columns: plant_id, country, category, plant_name, operator_name, parent_owner, capacity_mwe, biomethane_mw_gas, status, cod_date, region, locality, postcode, primary_source, primary_source_id.
- `results/test20_results.json`: 20 plants, researched by Sonnet. A JSON list of records.
- `results/remaining_results.json`: 425 plants, researched by Sonnet. A JSON list of records.
- `results/haiku_test20_results.json`: the same 20 test plants researched by Haiku, for a model comparison only. Do not load it into the database.

The plant IDs in the three input sets (pilot 20, test 20, remaining 425) do not overlap. Each results file covers its input exactly, with no missing, extra or duplicate plant IDs.

## 3. Record format

Each record has these keys:

- `plant_id` (integer).
- `parent_owner` (text or null) and `parent_owner_confidence`.
- `owner_since` (date text or null) and `owner_since_confidence`.
- `owners`: a list of entries with `owner`, `stake` (optional), `from`, `to`, `note`, `url`, `source_type`, `source_ref`, `confidence`.
- `financing`: a list of entries with `borrower`, `lender`, `instrument`, `amount`, `ccy`, `created`, `refi`, `refi_basis`, `url`, `source_type`, `source_ref`, `confidence`.
- `issues`: a list of entries with `date`, `type`, `summary`, `severity` (high, medium or low), `url`, `source_type`, `source_ref`, `confidence`.
- `summary`: what was found and what is still unknown. The brief says two sentences at most.
- Optional: `permit_tonnes_pa`, `locality`, `postcode`. The pilot file also has `heat_offtaker` and `grid_injection`, and a few Haiku records have them too.

Source and confidence scheme:

- `source_type` is one of `url_page`, `companies_house`, `register_entry`, `document_ref`, `snippet`, `inference`.
- `confidence` is `verified` (the agent read the source and it states the fact), `likely` (the source implies the fact or is indirect), or `unverified` (a lead from a search snippet or a guess that was seen but not confirmed).
- No `verified` entry rests on a `snippet` or `inference` source. This was checked.
- Most entries have a URL, but the scheme allows other source types.

## 4. Counts to expect for `remaining_results.json` (425 plants)

- Owners: 979 entries (635 verified, 274 likely, 70 unverified).
- Parent owner present for 419 plants (172 verified, 227 likely, 20 unverified). 6 plants have none.
- `owner_since` present for 255 plants (80 verified, 146 likely, 29 unverified).
- Financing: 209 entries across 157 plants (141 verified, 33 likely, 35 unverified). 108 entries have no amount.
- Issues: 492 entries across 281 plants (90 high, 152 medium, 250 low severity). 45 entries have no date. Fires (76) and outages (30) are the most common types.
- 23 issues are "register mismatch" flags: plants that a register shows as operating but that are closed, demolished, never built, a unit row, or operated only by a contractor. These are among the most useful results for business development.

Use these counts after loading to check that nothing was lost. Small differences of one or two may come from how a count is defined, so report any difference rather than assuming either figure is wrong.

## 5. Known data problems (do not fix silently)

1. Some summaries break the two-sentence limit (about 70 of 425 in the remaining file; 5 of 20 in the Sonnet test file; 16 of 20 in the Haiku file).
2. 9 entries have `source_type` `url_page` but a reference that is not a URL.
3. 1 plant has a parent owner but no `parent_owner_confidence`. Treat a missing confidence as `verified` only if Hylton agrees. Report the plant ID.
4. The `~` characters in the Sonnet test file are inside a real URL for plant 6595. Leave them.
5. No source has been independently checked. The `verified` labels are the research agents' own judgement. On the 20 test plants Haiku labelled 32 of 35 owner entries `verified` and Sonnet labelled 20 of 47, so the labels may not be comparable between models. This is an observation, not a conclusion.
6. Financing is mostly a lead list, not refinancing data, because half the entries have no amount.
7. The registers contain stale rows. Check the register-mismatch issues before any outreach.

## 6. Tasks, in order

Stop and report to Hylton at each "report" step.

1. **Inspect (read-only).** Connect to the SQL Server Express instance (default `.\SQLEXPRESS`) using `sqlcmd` or Python `pyodbc`. Report: the databases present; the `bioenergy_plants` tables and columns; row counts; the plants table's key column and whether `plant_id` matches the CSV and JSON `plant_id`; whether there are existing owner, financing or issue tables; the existing loader script `load_research.py` (find it and read it); and what "Tiers 2-3 pending" means in the workspace documentation. Do not change anything. Report this before going on.
2. **Back up.** Take a full backup of `bioenergy_plants` and tell Hylton where the file is. Do not continue if the backup fails.
3. **Propose the schema change.** Show the exact `ALTER TABLE` or `CREATE TABLE` statements for: `source_type`, `source_ref`, `confidence` on every owner, financing and issue row; `parent_owner_confidence` and `owner_since_confidence` on the plant row; and a decision on the optional fields (`permit_tonnes_pa`, `heat_offtaker`, `grid_injection`, `locality`, `postcode`). Report and wait for approval.
4. **Update the loader.** Change `load_research.py` for the new fields. Treat a missing confidence as `verified` (the pilot rule). Make the load repeatable: loading the same file twice must not create duplicates. Test it on `test20_results.json` in a copy of the database or inside a transaction that you roll back. Report the test result.
5. **Load.** After approval, load `test20_results.json` and then `remaining_results.json`. Also load `pilot_example.json` if it is not already in the database. Never load `haiku_test20_results.json`.
6. **Verify the load.** Compare row counts and the figures in section 4. Report any difference and list the plant IDs affected.
7. **Review lists.** Export CSV files for Hylton: all `likely` and `unverified` entries; all register-mismatch issues; high-severity issues; plants with no parent owner; and plants with an owner but no `owner_since`.

## 7. Work that is decided or not decided

Not decided, and not to be started without Hylton's go:

- Shortening the over-long summaries. The cloud session can do this, and no new research is needed.
- A spot-check of a sample of `verified` entries by fetching the cited sources, to test whether the labels are accurate. This needs research agents and costs tokens.
- A comparison of Sonnet's output with the 20 reviewed pilot plants, entry by entry (c.1.5 million subagent tokens in the cloud).
- Researching the other c.3,850 plants in the database. The cost of the 425-plant run was an estimate of c.33 million subagent tokens, which was not measured. Hylton's usage page shows the real figure.

Decided:

- The cheaper model (Haiku) was tested on 20 plants. It found fewer issues (11 plants against 19) and used about the same tokens, with more tool calls per plant. It is not a like-for-like replacement for Sonnet. Do not plan work around it.

## 8. Rules for working with Hylton

- Do not run any command that deletes or overwrites data without his agreement. Take a backup first.
- Report what you observe. Label any inference as an inference. Do not speculate unless you say it is speculation.
- Use plain English with complete sentences and UK spelling. Do not use em dashes, idioms or metaphors. Write "c." for approximate figures, not a tilde.
- For action requests, put the actions first and the reasoning after. For decisions, give the reasoning in full.
- Present options neutrally. Do not rule options out on inference. Give a recommendation when asked.
- Answer completely in one response. Check your answer against the whole conversation, not only the last message.
- Be a critical colleague, not a flatterer. If something looks wrong in the data or the plan, say so.
- If Hylton seems tired or is writing late at night, keep replies short and give no analysis unless he asks.

## 9. Definition of done

- The database has a backup taken before any change.
- The schema holds the new fields and the loader handles them.
- All 445 plants (20 and 425) are loaded with source and confidence on every entry, with counts matching section 4 or with every difference explained.
- Hylton has the review CSVs from step 7.
- Everything you changed is described to him in plain English.
