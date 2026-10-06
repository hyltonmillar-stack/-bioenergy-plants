# Cloud research task: bioenergy plant ownership, financing and operating problems

You are researching individual power and gas plants in the UK and Germany for a business-development database. For each plant in the input CSV, find the facts below by web search and page fetching, then write one JSON record per plant in the exact format of `pilot_example.json` (a real, reviewed example of 20 plants).

## Input and output

- Input: `test20.csv` (columns: plant_id, country, category, plant_name, operator_name, parent_owner, capacity_mwe, biomethane_mw_gas, status, cod_date, region, locality, postcode, primary_source, primary_source_id). `parent_owner` is usually empty.
- Output: `results/test20_results.json`, a JSON array with one object per plant_id. Create the `results` folder if needed.
- Work one plant at a time. Use a separate subagent per plant (or per small group of plants) so context does not carry over between plants. Run subagents in parallel where you can.
- Write results to the output file as you go, so nothing is lost if the session stops.

## Fields to find for each plant

1. `parent_owner`: the ultimate owner or controlling shareholder, named as it would appear in a deal announcement. For a municipal plant, name the municipality. If the operator is only a contractor, say so in the note and look for the real owner.
2. `owner_since`: date the current owner acquired it (YYYY-MM-DD, or YYYY-01-01 if only the year is known). Leave null if not found.
3. `owners`: ownership history, one entry per owner with `owner`, `stake` (percent, if known), `from`, `to`, `note`, `url`. Every entry needs a source and a confidence level.
4. `financing`: debt facilities specific to this plant or its owning group. Fields: `borrower`, `lender`, `instrument`, `amount`, `ccy`, `created`, `refi` (estimated refinancing or maturity date), `refi_basis` (the rule you used, for example "7-year tranche from Mar 2021"), `url`. Include group-level refinancings that cover the plant. Leave the list empty if nothing is found. Do not guess.
5. `issues`: operating problems, outages, fires, enforcement actions, permit breaches, insolvency, odour or noise complaints, failed technology, contract disputes, and plants that the registers show as operating but are not. Fields: `date`, `type`, `summary` (one or two sentences), `severity` (high, medium or low), `url`.
6. `summary`: two sentences at most. State the main finding and, if relevant, what is still unknown.
7. Source and confidence fields (see "Sources and confidence" below). Every entry in `owners`, `financing` and `issues` carries `source_type`, `source_ref` and `confidence`. The record also carries `parent_owner_confidence` and `owner_since_confidence`.
8. Optional where found: `permit_tonnes_pa` (EfW and AD permitted throughput), `heat_offtaker`, `grid_injection` (1 if the plant injects biomethane), `locality`, `postcode`.

## Rules

- Every fact needs a source and a confidence level, as set out in "Sources and confidence" below. Never invent a source.
- If a fact is not found, leave it null or empty and say so in `summary`. A blank is better than a guess. Do not invent dates, lenders or owners. A lead you actually saw in a search result may be recorded with confidence `unverified`; something you did not see anywhere may not.
- If sources conflict, record both in `issues` or `owners` notes and say which you think is more reliable and why.
- Use plain English. No em dashes. Write "c." instead of "~" for approximate figures.
- Check the plant is real and matches the row: the same name can belong to different sites. Use region, locality, operator and capacity to confirm.
- For German plants, search in German as well as English (Betreiber, Eigentümer, Gesellschafter, Störung, Brand, Insolvenz, Genehmigung). The German commercial register (Handelsregister) and company annual reports (Bundesanzeiger) are useful for shareholders if accessible.
- For UK plants, useful sources include company annual reports, Companies House filings, wikiwaste.org.uk, Environment Agency and SEPA public registers, RNS announcements and trade press (letsrecycle, Endswasteandbioenergy, Current News).
- Do not modify any file other than `results/`.

## Sources and confidence

A source does not have to be a web page. Record the source in three fields on each entry:

- `source_type`: one of `url_page` (a web page you opened), `companies_house` (a filing or company record), `register_entry` (a public register such as REPD, MaStR, Ofgem, the Environment Agency or SEPA permit register), `document_ref` (a named report, accounts, planning document or press release, cited by title, publisher, date and page), `snippet` (seen only in a search result and not opened), or `inference` (deduced from other facts, with the reasoning in the note).
- `source_ref`: the identifier for the source. For `url_page` use the URL. For `companies_house` use the company number and filing date or description. For `register_entry` use the register name and record ID. For `document_ref` use title, publisher, date and page. For `snippet` use the search result URL. For `inference` use a short statement of the reasoning.
- `url`: keep this field where a URL exists, so existing loaders still work. It may be null.

Set `confidence` on each entry:

- `verified`: you opened the source (page, filing, register record or document) and it states the fact directly.
- `likely`: the fact is supported by an opened source that implies it but does not state it directly, or by two or more independent snippets that agree.
- `unverified`: the fact comes from a single snippet, an unopened page, or an inference. Say in the note what would confirm it.

`parent_owner_confidence` and `owner_since_confidence` follow the same scale and refer to the top-level `parent_owner` and `owner_since` values. If the value is null, set the confidence to null.

Rules for confidence:

- Do not mark a fact `verified` unless you opened the source. A search-result snippet is `snippet` or `unverified`, never `verified`.
- If sources conflict, record each as its own entry with its own confidence and say which you rate more reliable and why.
- A fact with confidence `unverified` must still come from something you actually saw. Do not guess to fill a field.
- A missing `confidence` in older files (including `pilot_example.json`) means the entry was reviewed and should be treated as `verified`.

## When finished

Report in the session: how many plants completed, how many fields were found per field type, split by confidence level (verified, likely, unverified), and the total tokens used and cost if the session shows them.
