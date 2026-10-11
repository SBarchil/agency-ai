# Legacy System Audit — Contracts & Billing

Oct 10, 2026

The current Agence 2024 system (VB.NET desktop + SQL Server `agence_sti_2026`) keeps contracts for reference only, because several versions of the same contract were circulating; prices are not calculated from them. Making pricing automatic means re-architecting supplier and sales contracts around one approved version per contract, and rebuilding billing (no real VAT, no invoice lines, unsafe numbering) on a new data model.

## The old system

It is a multi-operator Windows desktop app where almost all business logic lives in stored procedures and in form click handlers.

| Item | What was found |
| --- | --- |
| Application | VB.NET WinForms "Agence 2024", .NET Framework 4.8, about 140 forms; 154 more `.vb` files are no longer compiled |
| Reports | Crystal Reports 13, 93 `.rpt` files loaded from disk at runtime |
| Database | SQL Server 2017, `agence_sti_2026`: 127 tables, 780 procedures, 105 views, 36 functions |
| Data access | ADO.NET calling about 940 distinct procedures; about 40 of them are called by the code but do not exist in the database |
| Years | One database per season (`agence_sti_<year>`), picked at login |
| Web folder `agence/` | No web code: only a second solution file and NuGet packages; a separate "factmarge" API/web project is referenced but was not included |

| Module | Main forms | Forms |
| --- | --- | --- |
| Login, users, settings | `login`, `mere`, `liste_operateurs`, `parametrage` | 14 |
| Reference data | clients, suppliers, services, products, room types, guides, cities | 32 |
| Contracts (buy and sell) | `le_contrat_achat`, `le_contrat_vente`, `ALLOTTEMENT_achat`, `offres` | 10 |
| Files and bookings | `le_dossier_lp_glo`, `sejours`, `CIRCUITS`, `excursions`, `le_transferts_ok`, `rooming_liste` | 25 |
| Vouchers | `voucher_sejours_ok`, `annulation_voucher` | 4 |
| Client billing and receivables | `facture_clients`, `avoir_clients`, `reglements_clients`, `la_balance_agee`, `facturation_expedia` | 26 |
| Supplier payables | `facture_frs`, `reglements_frs`, `VOUCHER_LETTRAGE`, `relevee_frs` | 23 |
| Cash desk | `encaissement`, `versements`, `depenses_ok`, `folio_recette` | 16 |
| Reporting | `etat_liste_clients` (report hub), `synthese_sejours`, `margebyoperateur` | 12 |
| Accounting export and closing | `exportation` (Sage), `importation_dossiers`, `cloture_mois` | 7 |

## How contracts work today

There is one purchase contract (`contrat_achat`) and one sales contract (`Contrat_vente`) per hotel/service, per client, per season; prices sit in fixed season columns.

**Contracts are a preview, not a price source.** Several versions of the same contract were in circulation, so the contract screens were limited to consultation: operators see the contract rates, but the booking amount is entered and checked by hand. The customer now wants the price calculated automatically from contracts, which only works if there is exactly one valid version at any time.

- **Header**: service, client (tour operator), season year, currency (MAD, USD, EUR, GBP) and an average exchange rate.
- **Seasons** (`saisons_achat` / `saisons_vente`): labels "SAISON I" to "SAISON V" with date ranges.
- **Rates** (`tarifs_achat` / `tarifs_vente`): one row per product (room type + board) and pax range, with prices pivoted into `saison_1_t` … `saison_6_t`. Supplements, reductions and taxes are extra rows typed `SUPPLEMENT`, `REDUCTION`, `TAXE`, as an amount or a percentage, with an optional age range for children.
- **Early booking** (`early_booking_*`): deadline, stay window, % or amount, and a "cumulable" flag.
- **Special offers**: five "stay N pay M" pairs and five date ranges stored as columns on the contract header, with dates kept as text.
- **Allotments** (`allottements_*`): per season, a fixed count, an elastic count and a release delay. The operator ticks fixed / elastic / outside allotment on each booking line by hand.

How the preview price is looked up (shown to the operator, not used as the booking amount) (`tarif_sejours_ok`, `tarif_achat_hotel_ok`):

1. The season year is derived from the arrival date with a hard-coded November–October cycle.
2. The proc finds the contract for client + service + season, and one early-booking rate.
3. It returns the five season prices for the product and pax range, multiplied by (1 − EB rate).
4. The VB form picks one of the five columns from the arrival date. Season VI is ignored, and a stay that spans two seasons is priced entirely at the arrival season.

What the model lacks before it could drive prices:

- More than 5 usable seasons, or prices that vary by day of week or length of stay.
- Stacking rules: only one early-booking rate is applied; offers, supplements and child reductions are not combined automatically.
- Stop sales: there is no stop-sale concept anywhere.
- Per-night pricing across season boundaries.
- Contract versions, approval status or an audit trail of rate changes.
- Release enforcement: release delays are stored but nothing blocks a booking inside release.
- Version control: nothing stops two contracts for the same hotel and season, or tells which one is current.

## How billing works today

An invoice is not a document with lines: it is one row in a wide client ledger table (`pieces_clients`, 108 columns), and its booking lines are just tagged with the invoice number.

**Client invoice** (button `facturer_Click` in `le_dossier_lp_glo.vb`, proc `rechercher_facture_Dossier`):

1. Checks: the file is completed, not in a closed month, and no unbilled files remain from last month.
2. For each client + payment mode on the file, the app reads the next number from a counter column in the `agence` table (`numero_facture`). Format: `F` + `A`/`M` (Agadir/Marrakech) + 5 digits + `/` + season.
3. The proc first tries to reuse a number from cancelled invoices (`facture_a_recuperer`), otherwise takes the counter and increments it.
4. Amount = Σ pax × nights × unit price × file exchange rate × (1 − commission %), over two price columns per line. This is called "TTC" (`montant_ttc`), but no tax is added.
5. If the file was already invoiced, the **existing invoice row is overwritten** with the new amount and the number gets an `A` suffix.
6. The amount in words always ends in "Dhs", even for EUR/USD invoices.

**Credit notes**: same flow with `numero_avoir` / `rechercher_avoir_dossier`, prefix `A`.

**Payments and receivables**: payments, advances, deductions and carry-forwards are more columns on the same ledger, matched to invoices by "lettrage". Aged balance and statements are procs over that ledger.

**Supplier side**: vouchers are created from bookings, supplier invoices are matched to vouchers, then paid; same wide-ledger pattern in `pieces_fournisseurs` (101 columns).

**VAT**: not computed by the invoicing procs, and the ledger has no VAT column (the Crystal invoice layouts could not be inspected, so confirm what the printed invoice shows). It appears only in the accounting export, hard-coded as 20% of the margin (`marge/1.2*0.2`). Tourist tax is only handled for Expedia/Orbitz clients through flags.

**Currencies**: rate stored per contract and per file; `LE_taux_moyen` returns an integer, so rates are truncated. Some rates are hard-coded (11 for Expedia, 10.7 in `sejours.vb`).

**Month closing**: `cloturer_mois` flags invoices as closed, but `annuler_cloture_mois` can reopen a month.

## How bookings work today

The file ("dossier LP") is the centre of the system, but it is a list of sales lines: there is no day-by-day operational itinerary, so vouchers, transfers and guides are entered by hand.

**Files**: numbered `A`/`M` + 5 digits + `/` + season (read-then-increment, not atomic). Statuses are manual combos: En cours / Réalisé / Annulé, plus Vérifié and Contrôlé flags; invoiced, credited and closed flags are set by billing. Totals (sales, purchases, margin, pax) are cached on the file and recomputed by `calcul_dossier` after every save.

**Bookings and lines**: a booking header (`reservations`, keyed by the client's booking reference) and sales lines (`ligne_vente`, 85 columns covering product, stay, pax and ages, two price/commission pairs, billing status, excursion pick-up and both transfer legs). Each service screen exists twice: one inside a file and a "\_resa" twin for bookings captured before a file exists (about 2,000–2,700 duplicated lines each).

| Service | How it is entered | Price |
| --- | --- | --- |
| Hotel stays | Booking header + room lines | Proposed from the sales contract, editable |
| Circuits | Same screen as stays (a clone); one circuit product per pax line | Proposed from the sales contract |
| Transfers | One line per booking: arrival and departure flight, date, time, meeting point and vehicle as free text | Per vehicle or per person, from product tariffs |
| Excursions | Adult/child lines with seller, ticket number, pick-up hotel and time | Product tariffs |
| Other (assistance, cruises, packages) | Generic "Divers" screen | Product tariffs |
| Guide expenses (débours) | Cash voucher per guide and file | Expense rates |

**Circuits and itinerary**: a circuit template (`eti_circuit`) is a list of days with day number, nights, city, one service (hotel) and one product (room and board), plus a note. `importer_itineraire` copies it onto a file (`eti_dossier`) with real dates, but the screens for this are no longer compiled. The copy is only used to send booking faxes to each hotel of the circuit. Times, flights, pick-up points, transfers between cities, excursions, meals, guides and vehicles per day are not modelled.

**Vouchers**: one supplier voucher per supplier per file. "Generate voucher" prices the hotel lines of one booking from the purchase contract; other vouchers are typed by hand. Invoicing the file marks its vouchers as invoiced, and supplier invoices are matched to vouchers.

**Operational documents**: hotel booking and cancellation faxes (sent by email), circuit fax per hotel, transfer fax, voucher, hotel rooming list, excursion pick-up list, arrivals and departures list, guide expense voucher, itinerary print of a circuit template.

Issues specific to this module:

- Auto-voucher prices every product with the current line's adults and ignores children; a second booking at the same hotel in a file cannot get an automatic voucher.
- Three different season definitions (Nov–Oct for prices, Jan–Dec in one check, the fiscal season for vouchers).
- File deletion code is still live behind a hidden button and leaves orphan lines.
- The arrivals report ignores its arrival/departure filter and only returns excursions.
- Excel import between agencies calls a procedure that does not exist (`inserer_reservation_importation`).

## How cash works today

Cash is handled in two separate "caisses" with no bank accounts, no reconciliation, no closing and no export to accounting; several procedures also corrupt data.

| Area | How it works | Main gaps |
| --- | --- | --- |
| Ticket books (carnets) | Paper ticket ranges assigned to representatives who sell excursions in hotels; the ticket number is typed on the sale line | First and last ticket of each book are not recognised; no check for missing tickets; "verified" flag displays wrongly |
| Collections (encaissements) | Cashier records cash, cheque or card collected from a rep and matches it to tickets or cash invoices | Excess collected is shown but never stored; no cheque bank or due date; editing a collection makes it count twice in cash reports |
| Deposits (versements) | Meant to match cheques deposited at the bank | Entry screen not compiled; numbering points to a column that does not exist |
| Expense cash (folios, bons de caisse) | Cash vouchers, mostly guide expenses on a file, grouped into folios with opening balance, top-ups and closing balance | Opening balance typed by hand; folio closing and transfers never built; "verified" amount is copied, not checked |
| Client payments | Payment amount matched to invoices, credit notes, advances | No payment mode recorded |
| Supplier payments | Split into cash, cheque and bill of exchange with bank as free text | No bank account table, no payment run |
| Banks and currencies | Not modelled; sales converted with a per-file rate | No reconciliation, no exchange gain/loss |
| Closing | Month closing covers invoices only and any user can undo it | Cash records stay editable forever; no audit trail |

Data-corrupting defects (verified in the procedures):

- Saving or deleting any cash voucher line recalculates the amount of **every** voucher in the database; vouchers without lines lose their amount (`UPDATE bon_caisse` with no `WHERE` in `inserer_ligne_bon` and related procedures).
- Folio closing balance subtracts unverified vouchers with the wrong sign.
- A duplicate guide voucher number shows a warning but is saved anyway.
- Sage export contains no cash, collection or voucher data, so cash never reaches accounting.

## Defects and risks

None of these block daily use today, but the first five are reasons not to carry the old billing logic forward.

| # | Issue | Where | Impact |
| --- | --- | --- | --- |
| 1 | Issued invoices are overwritten when a file is re-invoiced, and cancelled numbers are reused | `rechercher_facture_Dossier`, `facture_a_recuperer` | Breaks sequential numbering and audit trail |
| 2 | Invoice numbers read then incremented in separate steps, no transaction | `numero_facture`, `incrementer_facture` | Two operators can get the same number |
| 3 | Money stored as `float` everywhere | All ledger and rate tables | Rounding errors on totals and balances |
| 4 | No VAT on invoices; 20% margin VAT hard-coded in exports | `facturation_marge.vb`, `exportation_ventes.vb` | Tax rules can't change without code edits |
| 5 | Closed months can be reopened | `annuler_cloture_mois` | Closed periods are not locked |
| 6 | 112 of 127 tables have no primary key; no foreign keys at all | Whole database | Duplicates and orphan rows are possible |
| 7 | Availability procs write into shared tables (`cle_saisons`, `Societe`) | `allo_disponible` and similar | Two users running it at once overwrite each other |
| 8 | Early-booking rate applied as `price × (1 − rate)` with no /100 | `tarif_sejours_ok` | Correct only if users type 0.10, not 10; to confirm |
| 9 | Season straddle check compares the arrival date with itself | `sejours.vb:1250` | Stays across two seasons are never flagged |
| 10 | About 40 procedures called by the code are missing from the database | e.g. `cloturer_saison`, `attribuer_facture_voucher`, `pourquoi_ca_donne_pas` | Those screens fail at runtime |
| 11 | Plain-text SQL and Gmail passwords in `app.config`; AES key hard-coded | `app.config`, `Module1.vb:506-541` | Anyone with the install folder has DB and mail access |
| 12 | One database per year | `Module1.vb:543-558` | Cross-year reports and contracts spanning Nov–Oct need workarounds |
| 13 | Hard-coded client codes, operators, service codes and rates | `E181`, `E372`, `004/005/007`, `taux = 11` | Business changes require a new build |
| 14 | Heavy duplication: buy/sell contract forms are clones; invoice block copied 6 times | `le_contrat_achat.vb`, `le_dossier_lp_glo.vb` | Fixes must be made in several places |

## The new system at a glance

The new system has three front ends on one shared backend: guests, B2B partners and staff see different screens, but prices, availability and bookings come from the same rules and the same database.

*(Diagram in the shared doc: system overview · public front end, backend, back office)*

| Side | Who uses it | Modules |
| --- | --- | --- |
| Public front end | Final travellers, potential customers | Catalogue pages and search; availability and indicative prices; quote and booking requests; AI assistant on web and WhatsApp; guest account with their bookings |
| Back office | Agency staff | Products and catalogue setup (what the public site shows); supplier and sales contracts with AI intake; reservations and files from every source; suppliers and clients; billing (invoices, credit notes); treasury (cash desk, bank, receipts, supplier payments, reconciliation); reports, users and settings |
| Backend API | Both front ends, the AI assistant, the worker | Pricing and availability engine, booking rules, billing and ledger, permissions and audit log |
| B2B channel | Tour operators, travel agencies, OTAs | Partner portal (search, book, manage bookings, invoices and statements); partner API with webhooks for their systems; booking emails read by AI into draft bookings |

Reservations reach the back office from several sources: the public site, the AI assistant's leads, operators entering them directly, tour operators by email, and OTAs such as Expedia. All of them become the same kind of booking file.

## New contract engine

Version control comes first: one contract per supplier, service and validity period, changed only through numbered, approved versions. On top of that, use one model for buying and selling, store rates as rows per date range instead of season columns, and price every stay night by night.

| Entity | Key fields | Replaces |
| --- | --- | --- |
| Contract | direction (buy/sell), supplier or client, service, currency, valid from/to, status (draft, active, expired), version | `contrat_achat`, `Contrat_vente` |
| Season | contract, name, date ranges (any number) | `saisons_*` (max 5–6) |
| Rate | contract, season, room type, board, occupancy / pax range, price per night or per person, min stay, days of week | `tarifs_*` season columns |
| Supplement / reduction | contract, type (child, 3rd adult, single, board upgrade, gala dinner), age range, amount or %, per night/stay | rows typed `SUPPLEMENT` / `REDUCTION` |
| Promotion | contract, kind (early booking, stay N pay M, long stay, honeymoon), booking window, stay window, value, combinable with, priority | `early_booking_*`, offer columns on contract |
| Allotment | contract, room type, date, fixed / elastic count, release days | `allottements_*` (per season) |
| Stop sale | contract, room type, date range, reason | did not exist |
| Tax rule | country/city, type (VAT, tourist tax), basis (per person-night, % of price, % of margin), rate, valid dates | hard-coded values |

### One approved version per contract

The system, not the operators, decides which version is valid: a contract can have many versions, but only one is active for any given date.

*(Diagram in the shared doc: contract version lifecycle · 5 states)*

- **Contract identity**: supplier + service + market or client (or "all clients") + validity period. The database refuses a second contract with the same identity; a change becomes a new version instead.
- **Version record**: number (v1, v2…), status, effective-from date, the supplier's file attached, change note, entered by, approved by, dates.
- **Approval by a second person**: whoever enters a version cannot approve it. Only active versions are used for pricing; drafts can be previewed but never priced.
- **Amendments** (avenants, new promotions, rate changes mid-season): copy the active version into a new draft with an effective date. Approving it supersedes the previous version from that date.
- **Bookings keep their version**: each booking stores the contract version and price breakdown it was priced with. A new version never changes confirmed bookings unless an operator re-prices one on purpose.
- **Compare screen**: show what changed between two versions (rates, promotions, allotment) before approval.
- **AI assistant**: reads supplier and client contract files into draft versions (see AI contract intake); a person always reviews and approves.
- **Starting contracts**: on day one, current contracts are entered through AI intake as drafts, each approved against the supplier's latest signed document.

### Sales contracts linked to supplier contracts

Every sales price should trace back to a supplier contract version, so a change on the supplier side shows up on the sales side instead of being missed.

- **Two pricing methods per sales contract**: a markup on the supplier contract (% or fixed amount per night or per person, by client, market or service family), or negotiated net rates entered for a tour operator.
- **Traceability**: each sales contract version records which supplier contract version it was built on.
- **Supplier change → review flag**: when a new supplier version is approved, every sales contract built on the old one is flagged "needs review". Markup-based contracts recalculate on approval; negotiated ones wait for a person.
- **Promotions**: a setting per sales contract says whether supplier promotions (early booking, stay N pay M) are passed on to the client, kept as margin, or replaced by the client's own promotions.
- **Currency**: supplier contracts in MAD, sales contracts in the client's currency, with the exchange rate rule (fixed contract rate or daily rate) stored on the sales contract.
- **Margin check**: at approval and at booking, compare sale and cost per night; a negative margin or one below a set minimum shows a warning or blocks, depending on the user's rights.
- **Same lifecycle**: sales contracts follow the same draft → review → active → superseded versions as supplier contracts.

Pricing rules for the engine (one service, used by bookings, quotes and the future web booking engine):

1. Find the active contract for the client (or the default "all clients" contract) on the booking date.
2. Split the stay into nights; for each night pick the season and rate for that date.
3. Check stop sales and allotment (respecting release days) per night.
4. Add supplements and apply reductions per guest (children by age).
5. Apply promotions in priority order, respecting combinability rules.
6. Add taxes from tax rules.
7. Return a price breakdown (per night, per guest, per rule), store it on the booking, and keep it even if the contract changes later.

The breakdown in step 7 is what billing uses, so invoices never recompute prices from contracts.

## New bookings and itineraries

Keep circuits as pre-configured packages, but give each one a detailed day-by-day template. When a circuit is booked, the template becomes the file's own itinerary with real dates, and the itinerary generates the service lines, vouchers, transfers and lists. Operators no longer build the itinerary; they only adjust it when a guest needs something different.

| Entity | Key fields | Replaces |
| --- | --- | --- |
| Circuit template (versioned, like contracts) | code, name, days, nights, start city, valid dates, hotel category options (e.g. 4★ / 5★) | `eti_circuit` |
| Template day | day number, overnight city, description for the guest document | `eti_circuit` rows |
| Template item | day, type (hotel night, transfer, excursion, guide, meal, entrance fee, flight), product, default supplier, time of day, from/to place, pricing basis (per person, per room, per vehicle by group size) | did not exist |
| Booking file | number (atomic), client or partner, channel, lead guest, status, currency | `dossier_lp` |
| Traveller | name, age or date of birth, room assignment, passport details only if required | ages on lines |
| Flight | direction (arrival/departure), flight number, date, time, airport, travellers on it | free-text fields on lines |
| Itinerary item | the file's copy of a template item with real date and time, supplier, status, deviation flag | `eti_dossier` |
| Service line | one bookable service: supplier, product, dates, pax, sale price and cost (with contract versions), supplier confirmation number, status (requested, confirmed, cancelled) | `ligne_vente` + `reservations` |
| Voucher | supplier, service lines covered, version, status (draft, sent, confirmed, amended, cancelled) | `pieces_fournisseurs` VO + `lignes_vouchers_doc` |
| Transfer order | date, time, from, to, flight, travellers, vehicle type, transport supplier or own driver and vehicle | free-text transfer fields |

File life cycle, set by actions instead of manual combos:

1. **Quote**: priced itinerary, nothing booked with suppliers.
2. **Option**: allotment held until an expiry date; expires automatically.
3. **Confirmed**: service lines requested from suppliers, vouchers sent.
4. **Travelling**: between first and last service date.
5. **Completed**: after the last service; ready to invoice.
6. **Invoiced** and **Closed** follow billing. **Cancelled** at any point applies the contract's cancellation fees.

One booking screen replaces each pair of in-file and "\_resa" screens: a booking can exist with or without a file, and attaching it is an action.

*(Diagram in the shared doc: itinerary flow · template to generated documents)*

### What the itinerary generates

Everything below is calculated from the itinerary and the flights; operators review and send, they do not retype.

**Vouchers**

- One voucher per supplier per continuous run of services (a hotel used on nights 1 and 4 gets two vouchers), replacing the old "one voucher per supplier per file" rule.
- Lines and costs come from the service lines, priced with the active purchase contract version, children included.
- Sent by email (later a supplier portal); the supplier's confirmation number is recorded on the voucher.
- A change creates a new voucher version marked "amended"; a cancellation sends a "cancelled" voucher. Earlier versions are kept.
- Sending is automatic or after operator review, set per supplier.

**Transfers**

- Generated from flights and hotel changes: airport → first hotel, hotel → hotel when the circuit changes city without its own vehicle, last hotel → airport.
- Pick-up time: flight arrival time for arrivals; flight departure minus a margin set per airport for departures.
- Vehicle type from pax count, using a rules table the agency maintains (e.g. car, minivan, minibus, coach by group size).
- A daily dispatch board groups transfers by date, airport and time window, so guests on the same flight to nearby hotels can share a vehicle.
- Each transfer is assigned to a transport supplier (with a voucher) or to the agency's own driver and vehicle, and the driver gets a job sheet (PDF or WhatsApp message).

**Lists**: rooming list per hotel and arrival date, arrivals and departures per day, excursion pick-up lists, guide and driver schedules. All are queries on itinerary items, so they are always in sync with the file.

**Changes**: shifting the start date moves every item; changing pax re-prices and flags vouchers and transfers; swapping one hotel changes only that day. Before saving, the screen shows the impact: which vouchers will be amended and which transfers updated.

**Checks**: a confirmed file without an arrival flight, a night without a hotel, or a transfer without a vehicle shows a warning on the file and on the dispatch board.

Worked example (illustrative): a 4-day circuit, 2 adults, arriving on flight XX123 at 14:30.

| Day | Template items | Generated |
| --- | --- | --- |
| 1 Marrakech | Arrival transfer; hotel night (4★) | Transfer order airport → hotel at 14:30, car; hotel voucher (night 1) |
| 2 Ouarzazate → Merzouga | Circuit vehicle with driver; lunch; desert camp night | Transport voucher (days 2–4); restaurant voucher; camp voucher (night 2) |
| 3 Merzouga → Ouarzazate | Camel ride; hotel night | Excursion voucher; hotel voucher (night 3) |
| 4 Marrakech | Drive back; departure transfer | Transfer order hotel → airport, time from departure flight |

Result: 6 supplier vouchers, 2 transfer orders, rooming lists for 2 hotels and 1 camp, and lines on the arrivals and departures lists, with no manual typing.

**Setting up circuits**: staff build each circuit template once, day by day, with its hotels, transfers, excursions, meals and times, then approve it. AI can draft a template from the circuit's programme or brochure to save typing.

## AI contract intake

Operators upload the contract file; AI turns it into a draft version, and nothing it produces is used for pricing until a person approves it. The same pipeline serves supplier contracts (hotels, transport, excursions) and sales contracts (tour operators, agencies).

1. **Upload**: drag-and-drop or a dedicated mailbox; PDF, scanned PDF, Excel or Word. The file is stored with a hash, so the same file sent twice is recognised.
2. **Identify**: AI reads the header and proposes the party (supplier or client), service, market and validity period, then matches it to an existing contract. A match becomes an **amendment draft** (v+1) of that contract instead of a new contract; this is what stops duplicate contracts.
3. **Extract**: AI fills the contract schema (seasons, rates per room type and board, supplements, child reductions, promotions, allotment and release, stop sales, cancellation terms) as structured JSON validated against the schema. Every value carries its source: page and quoted text.
4. **Check**: rules run on the draft without AI: overlapping or missing dates, rates outside expected ranges, a price change above X% versus the active version, missing board or room types, currency mismatch. For sales drafts, margins against the linked supplier version are checked too.
5. **Review**: a side-by-side screen shows the file on one side and the fields on the other. Clicking a field highlights its source. Flags and low-confidence fields come first, plus a diff against the active version. The reviewer corrects fields, then approves or rejects.
6. **Approve**: only a person with approval rights can make a version active, and the approval is logged with their name.

Guardrails:

- The AI service account can only create and edit drafts; the approve action requires a logged-in user, enforced in the database permissions and the API, not only in the screen.
- AI never fills a value it cannot find in the file; it leaves it empty and flags it.
- The audit log keeps the file hash, the model and prompt version, the AI's original values and every human correction.
- Contracts are confidential commercial data: use an AI provider contract that excludes training on your data, and keep files in the customer's storage.

*(Diagram in the shared doc: AI contract intake · people and system lanes)*

The same flow runs for supplier and sales contracts; a rejected draft stays a draft and is never priced.

### Measuring accuracy

Treat extraction quality as a number the customer can see, not a promise.

- **Test set**: 20–30 real past contracts (both sides, several hotel chains and tour operators, at least a few scans and Excel rate sheets), entered by hand once and checked by the contracting team. This is the reference answer.
- **Field accuracy**: share of fields the AI gets exactly right on the test set, reported per field type (dates, rates, child ages, promotions). Run it after every prompt or model change.
- **Review effort**: in production, track corrections per draft and minutes from upload to approval; the target is that review takes a fraction of manual entry time.
- **Rejection reasons**: tag why reviewers reject or correct, and use the most common reasons to improve the prompt or the schema.
- **Rollout**: start with supplier hotel contracts for one branch, compare against manual entry for a few weeks, then add sales contracts and the other branch.

## AI guest assistant

One assistant answers potential customers on the website chat and on WhatsApp, using the agency's own published information and live prices, and hands the conversation to a person when it should. It is a separate AI use from contract intake, with far fewer permissions.

**What it knows** (managed by staff in a "Knowledge" screen of the back office):

- The catalogue: tours, circuits, excursions, transfers, hotels, with descriptions, photos, inclusions and itineraries.
- Policies: booking, payment, cancellation, children, visas and travel documents, health and safety, FAQ.
- Searchable with PostgreSQL's `pgvector` extension, so no extra database is needed. Every answer is grounded in these texts; staff update them, not the AI.

**What it can do** (tools that call the same API as the back office):

- Check availability and the public price for given dates and guests, from active sales contracts only.
- Create a quote request or lead in the back office with the guest's details and the conversation attached.
- Give the status of an existing booking, only after the guest proves who they are (booking reference + a code sent to their email or phone).
- Hand off to a person: the conversation appears in a back-office inbox, where staff reply on the same channel and the AI can suggest a reply.

**What it never does**: confirm a booking or take payment by itself, reveal supplier costs, tour-operator net rates, margins or contract drafts, share other guests' data, or ask for passport or card numbers in chat.

**Languages**: answers in the guest's language (French, English, Spanish, Arabic and others), from knowledge written in one or two languages.

*(Diagram in the shared doc: guest assistant · channels, data and handoff)*

Both channels share one assistant and one conversation history, so once a guest leaves a phone number on the website, the conversation can continue on WhatsApp with the same context.

### Guardrails, WhatsApp and quality

**Guardrails**

- The assistant's API account can read only published catalogue, policies and public prices, and can create leads; it has no access to contracts, costs or billing. This is enforced in the API, not only in the prompt.
- Prices are shown as "from" or "indicative" with the date checked; final price and confirmation come from an operator or the booking flow.
- If the answer is not in the knowledge or the tools, it says so and offers a person, instead of guessing.
- Conversations are stored with a retention period and a privacy notice at the start of the chat; guests can ask for deletion.

**WhatsApp specifics**

- Use the WhatsApp Business Platform (Meta's Cloud API directly, or through a provider such as Twilio). It needs a verified business account and a dedicated phone number.
- Messages arrive by webhook, so the assistant needs a public HTTPS endpoint (the cluster's Ingress).
- Free-form replies are allowed within 24 hours of the guest's last message; after that, only pre-approved template messages can be sent. Plan templates for quote follow-ups and booking reminders.
- Meta charges for some message types; check current pricing before estimating running costs.

**Measuring quality**

- A test set of 50–100 real guest questions with the agency's correct answers, run after every change to prompts, knowledge or model.
- In production: share of conversations solved without a person, handoff rate, leads created, quotes that become bookings.
- Staff review a sample of conversations each week and flag wrong answers; each flag becomes a knowledge fix or a new test question.

## B2B channel

Partners (tour operators, travel agencies, OTAs) get three ways in, and all three create the same booking file and use the partner's own active sales contract. Offer both API and email: large partners connect their systems by API, smaller ones use the portal, and those who still send emails are handled by AI instead of retyping.

| Way in | For whom | How it works |
| --- | --- | --- |
| Partner portal | Agencies and smaller tour operators | Web screens on the same API: search, net prices, book, cancel, vouchers, invoices, statement of account |
| Partner API | Tour operators and OTAs with their own systems | REST/JSON API with OpenAPI documentation, a sandbox for testing, and webhooks for status changes |
| Booking emails | Partners who keep emailing | AI reads the email into a draft booking (partner, guests, dates, services); an operator checks and confirms it, the same human-approval pattern as contract intake |

Rules that apply to all three:

- **Their prices, not the public ones**: each partner sees net rates from their own active sales contract version, never another partner's rates or supplier costs.
- **Instant or on request**: inside allotment and outside stop sales, a booking is confirmed instantly; otherwise it is created "on request" and an operator confirms with the supplier.
- **Credit limit and payment terms per partner**: a booking that would exceed the partner's credit limit is held for approval, which ties the B2B channel to billing and treasury.
- **Cancellation policy from the contract**: cancellation fees are calculated from the sales contract and invoiced automatically.
- **Industry formats later**: if a large partner requires an XML standard (such as OpenTravel), add it as an adapter on the same API rather than a second system.

### Partner API

A small, versioned set of operations covers the whole booking life cycle.

| Operation | Method and path | Notes |
| --- | --- | --- |
| Search availability | `POST /v1/availability` | Destination or hotel, dates, rooms and guests; returns options with net price per the partner's contract |
| Price quote | `POST /v1/quotes` | Locks the price and availability for a short time; returns a quote ID and cancellation policy |
| Create booking | `POST /v1/bookings` | From a quote ID; requires an `Idempotency-Key` header so a retried request never creates two bookings |
| Get booking | `GET /v1/bookings/{id}` | Status (confirmed, on request, cancelled), services, voucher link |
| Modify or cancel | `PATCH` / `POST /v1/bookings/{id}/cancel` | Returns the fee from the contract before the partner confirms |
| Documents | `GET /v1/bookings/{id}/voucher`, `GET /v1/invoices` | Vouchers, invoices and statement of account |
| Webhooks | Partner registers a URL | Notified when a booking on request is confirmed, changed or cancelled by the agency |

Around the operations:

- **Authentication**: one set of credentials per partner (OAuth2 client credentials or API keys), rotated and revocable from the back office.
- **Rate limits** per partner, so one partner's traffic cannot slow down the others or the back office.
- **Sandbox**: a test environment with test contracts, so partners integrate without touching real allotment.
- **Logging**: every request and response kept for a period, to settle disputes about what was quoted or booked.
- **On Kubernetes**: the partner API runs as its own deployment of the same image, on its own host name, so partner traffic is scaled and limited separately.

## Manager assistant (MCP)

Managers ask for figures in plain language and get an answer, a PDF report, or a spoken reply. An MCP (Model Context Protocol) server exposes the agency's reports as tools any MCP-capable AI client can call. The AI chooses the tool and explains the result; the numbers are always computed by the backend.

**Where it sits**: a new host, `Agency.McpServer`, built with the official MCP C# SDK and calling the same Application use cases as the back office. It runs as its own Kubernetes deployment, reached over HTTPS (MCP's Streamable HTTP transport).

**Tools** (read-only to start):

| Tool | Answers questions like |
| --- | --- |
| `get_sales_summary` | "What did Agadir sell last month compared with last year?" |
| `get_margin_breakdown` | "Which hotels, clients or operators gave the best margin this season?" |
| `get_arrivals` | "How many arrivals next week, and which files are still on request?" |
| `get_allotment_usage` | "Where are we close to running out of allotment?" |
| `get_receivables_aging` | "Which clients owe us more than 60 days?" |
| `get_supplier_payables` | "What do we have to pay suppliers this month?" |
| `get_cash_position` | "What is our cash and bank position today?" |
| `get_pending_approvals` | "Which contract versions and held bookings are waiting for approval?" |
| `generate_report_pdf` | "Send me the monthly management report as a PDF" |

**PDF reports**: `generate_report_pdf` uses the same report templates as the back office, stores the file in object storage and returns a short-lived download link. The AI never draws the figures itself.

**Voice**: use an MCP client that supports voice conversation, or add a small voice layer (speech-to-text in, text-to-speech out) in front of the same assistant. Spoken answers give the headline numbers; details go into the PDF.

**Security**:

- Sign-in with OAuth (as the MCP specification requires for remote servers), using the same users and roles as the back office; each tool checks permissions, e.g. a branch manager sees only their branch.
- No free-form SQL tool: the AI can only call defined tools, so it cannot read data it should not or run heavy queries. Ad-hoc analysis can come later through read-only reporting views with row limits.
- Write actions (approve a contract, release a held booking) are left out at first; if added, they require an explicit confirmation step and are logged.
- Every tool call is logged with the user, the parameters and the result size.

**Quality**: a set of 30–50 typical manager questions with the expected figures taken from the back office reports, re-run after every change.

*(Diagram in the shared doc: manager assistant · question to answer or PDF)*

## New billing engine

Treat every invoice, credit note and payment as an immutable document with lines, and derive balances from a proper double-entry style ledger instead of wide tables.

| Entity | Key fields | Rule |
| --- | --- | --- |
| Document | type (proforma, invoice, credit note, supplier invoice), number, date, client/supplier, currency, exchange rate, status (draft, issued, cancelled), source file | Once issued, never edited; corrections go through a credit note |
| Document line | description, service, quantity, nights, unit price, discount, tax rule, net, tax, gross | Copied from the booking's price breakdown |
| Number series | type, branch (Agadir / Marrakech), year, prefix, next number | Taken inside the same transaction that issues the document; no reuse |
| Payment | date, method, amount, currency, rate, bank reference | Allocated to one or more documents (replaces "lettrage") |
| Ledger entry | account, debit, credit, document, date, period | Balances, aged balance and statements are queries on this |
| Period | month, status (open, closed) | Closed periods reject new entries; reopening needs admin rights and is logged |

Other requirements:

- **Money**: `decimal(18,2)` for amounts, `decimal(18,6)` for exchange rates; round per line, then total.
- **Currencies**: invoice in the contract currency, store the rate used, and post the MAD equivalent to the ledger; amount in words in the invoice currency.
- **VAT and tourist tax**: computed from tax rules, including a margin-based scheme if the accountant confirms it applies, and shown on the invoice.
- **Proformas and deposits**: proforma → deposit invoice → final invoice, with the deposit deducted.
- **OTA billing** (Expedia, Orbitz, bulk invoices in `FacturesBulk`): a client setting, not hard-coded client codes.
- **Exports**: keep the Sage export as a mapping table (accounts per service family and branch) instead of fixed columns.
- **Audit log** on every create, issue, cancel and reopen.

## New treasury

Every place money sits is an account, every movement is a transaction between accounts posted to the ledger, and every cash register is counted and closed each day. Treasury shares the billing ledger, so balances, statements and the Sage export include cash and bank automatically.

| Entity | Key fields | Replaces |
| --- | --- | --- |
| Treasury account | type (cash register, petty cash, bank, card terminal, online payment), branch, currency, bank details (RIB), responsible person | folios, free-text bank names |
| Person balance | rep, guide or driver; running balance of cash they hold or are owed | did not exist |
| Cash session | register, date, opening balance (carried from last close), expected, counted, difference, closed by | folio opening balance typed by hand |
| Receipt | payer, mode (cash, cheque, transfer, card, online), amount, currency, rate, cheque number/bank/due date and status | `ENC` and `REG` rows in `pieces_clients` |
| Payment | payee, mode, amount, currency, approval, bank reference | supplier `REG` rows |
| Transfer | from account, to account (cash → bank deposit, bank → petty cash, between banks) | versements (never finished) |
| Expense | category, amount, receipt photo, file and service it belongs to, person | `BON_CAISSE` + `lignes_bon` |
| Allocation | receipt or payment matched to invoices, credit notes, vouchers, tickets | `lettrage_*` tables |

**Representatives and ticket books**

- Ticket books are assigned by range; every ticket has a status: unused, sold, cancelled, void.
- A rep's balance = cash tickets sold − amounts handed in. The collection screen shows what the rep owes and lists missing or skipped ticket numbers.
- Excess handed in becomes a credit on the rep's balance instead of disappearing.
- Later option: reps sell from a phone app with electronic tickets, which removes paper books and typing.

**Guides and expenses**

- Cash advance to a guide for a file → expense report with receipt photos (AI can read receipts into draft lines) → settlement: the guide returns the remainder or is reimbursed.
- Each expense is linked to a file and a service, so file margins include real costs.
- Expenses above a set amount, or without a receipt, need approval.

**Receiving and paying**

- Cheques have a life cycle: received → deposited → cleared or bounced; a bounced cheque reopens the invoice automatically.
- Overpayments become an advance on the client's account automatically.
- Supplier payment runs: select due vouchers and supplier invoices, approve (with limits per user), export a bank transfer file in the bank's format, mark paid when the bank confirms.
- Online card payments from the public site and the partner portal post to an online payment account, settled to the bank by the provider.

**Banks, currencies and closing**

- Bank statements are imported (file from the bank) and matched to receipts, payments and transfers by amount, date and reference; unmatched lines are handled by hand.
- Accounts in MAD, EUR and USD. When a foreign-currency invoice is paid at a different rate, the exchange gain or loss is posted automatically.
- Daily cash session close with counted vs expected; differences are recorded with a reason. The monthly period close includes treasury, and reopening needs admin rights and is logged.
- Cash forecast: expected receipts (invoice due dates) and payments (supplier due dates) by week, also available to managers through the MCP tool `get_cash_position`.

*(Diagram in the shared doc: treasury · money flow between accounts)*

Bank statements are matched against these movements, so every arrow into or out of a bank account is confirmed by the bank before it counts as cleared.

## At a glance

*(Diagram in the shared doc: contract-to-invoice flow · 7 parts)*

The booking keeps its own price breakdown, so a later contract change never alters an issued invoice; corrections only happen through credit notes.

## Target stack

C# on .NET, delivered as a web application, with PostgreSQL. Business rules live in C# services, not in stored procedures, so the 780 old procedures are replaced, not ported.

| Layer | Choice | Why |
| --- | --- | --- |
| Runtime | .NET LTS release, ASP.NET Core | Long support window; runs on Linux or Windows |
| Back-office UI | React + TypeScript (Vite) for back office and partner portal; Next.js for the public site | Large ecosystem for data-heavy screens; static files that run easily on Kubernetes; server rendering for search engines on the public site |
| Public booking site | Separate site calling the same API | Search engines and guests need a fast public site; operators need rich screens |
| API | ASP.NET Core Web API, versioned | Same pricing service for back office, website and AI assistant |
| Database | PostgreSQL + EF Core (Npgsql provider) | No license cost; strong date-range and JSON support |
| Money | `numeric(18,2)` amounts, `numeric(18,6)` rates; `decimal` in C# | No float rounding errors |
| Validity rules | `daterange` columns + exclusion constraints | The database itself refuses overlapping seasons and two active versions for the same contract and dates |
| AI extraction output | `jsonb` column on the draft version | Keeps the AI's original answer next to the human-corrected fields |
| Users and rights | ASP.NET Core Identity, roles and policies | "Approve contract", "issue invoice", "reopen period" as explicit permissions |
| Background jobs | A job queue with PostgreSQL storage (e.g. Hangfire) | AI extraction, emails, nightly checks run outside web requests |
| Documents | PDF library for invoices and vouchers (e.g. QuestPDF, check its license tier); ClosedXML for Excel | Replaces Crystal Reports |
| Files | Object storage or a server folder, path + hash in the database | Contract files, signed PDFs |
| Hosting | Kubernetes (see Running on Kubernetes) | Same setup for test and production |

What to watch with PostgreSQL (coming from SQL Server habits):

- **Text comparison**: PostgreSQL is case-sensitive by default; use a case-insensitive, accent-insensitive collation or `citext` for names and codes (French accents in hotel and client names).
- **Dates**: use `timestamptz` for moments and `date` for stay dates; Npgsql expects UTC for `timestamptz`.
- **Identifiers**: use lowercase snake\_case table and column names to avoid quoting.
- **Sage export**: keep it as a file export from the new system, with account mappings in a table.

## Code architecture

Use Clean Architecture for the layers and a modular monolith for the business areas: four layers, each organised by module (Catalogue, Contracts, Pricing, Bookings, Partners, Billing, Treasury, Guest assistant, Identity). Dependencies always point inward, toward the domain.

| Layer | Contains | Examples | Depends on |
| --- | --- | --- | --- |
| Domain | Entities, value objects, business rules, domain events; no framework or database code | `Contract`, `ContractVersion`, `Rate`, `Booking`, `Invoice`; `Money`, `DateRange`, `Occupancy`; "an issued invoice cannot change"; `ContractVersionApproved` | Nothing |
| Application | Use cases (commands and queries), interfaces for everything external, validation, authorization checks, transactions | `PriceStay`, `ApproveContractVersion`, `CreateBooking`, `IssueInvoice`; `IContractExtractor`, `IFileStorage`, `IWhatsAppSender`, `IClock` | Domain |
| Infrastructure | Implementations of those interfaces | EF Core + Npgsql, AI provider client, WhatsApp provider, object storage, PDF and email, job queue | Application, Domain |
| Hosts (presentation) | Thin endpoints and screens; each host wires everything together at startup | Back office API + UI, Partner API, Public site, Worker, MCP server, Migrator | Application, Infrastructure (for wiring only) |

Solution structure:

```
src/
  Agency.Domain/            Contracts/ Pricing/ Bookings/ Billing/ Treasury/ ...
  Agency.Application/       same module folders: use cases + interfaces
  Agency.Infrastructure/    Persistence/ Ai/ WhatsApp/ Storage/ Documents/ Jobs/
  Agency.BackOffice/        back office API + UI host
  Agency.PartnerApi/        B2B API host
  Agency.PublicSite/        public website + guest assistant host
  Agency.Worker/            background jobs host
  Agency.McpServer/         manager assistant (MCP) host
  Agency.Migrator/          database schema updates host (Kubernetes Job)
tests/
  Agency.Domain.Tests/  Agency.Application.Tests/  Agency.IntegrationTests/  Agency.ArchitectureTests/
```

Each host project becomes its own container image and Kubernetes deployment, all built from one solution and one pipeline. Keep modules as folders inside the shared layers to start; split a module into its own projects only if it grows large or gets its own team.

*(Diagram in the shared doc: Clean Architecture · hosts, layers and dependency direction)*

Infrastructure implements the interfaces the Application layer declares, so swapping the AI provider, the WhatsApp provider or the storage changes one Infrastructure class, not the business code.

### Testing and keeping the rules

The main payoff is testability: pricing, contract versions and invoicing are domain code that can be tested in milliseconds without a database or an AI call.

- **Domain tests**: the pricing engine against real contract examples (seasons, child reductions, early booking, stays across two seasons), invoice and version rules. These are the most valuable tests in the project.
- **Application tests**: use cases with fake implementations of the interfaces (fake AI extractor, fake clock, fake storage).
- **Integration tests**: EF Core mappings, exclusion constraints and transactions against a real PostgreSQL started in a container (Testcontainers).
- **Architecture tests**: a test that fails the build if Domain references EF Core or ASP.NET, if Application references Infrastructure, or if one module reaches into another module's internals (NetArchTest or ArchUnitNET).
- **AI behind interfaces**: `IContractExtractor` and the guest assistant's model client can be swapped for another provider or model without touching use cases, and the accuracy test sets run against the real implementation.

Pitfalls to avoid:

- A generic repository wrapped around EF Core adds code without value; use repositories per aggregate (`IContractRepository`) or let use cases use the DbContext through a narrow interface.
- Anemic entities with all rules in services; rules like "only one active version" or "issued invoices are immutable" belong in the entities.
- Over-layering small screens: simple lookups (a list of cities) can be thin queries, not five classes.
- Libraries: MediatR and AutoMapper are common in Clean Architecture templates but moved to commercial licenses; check their current terms, or use plain handler classes and explicit mapping.

## Running on Kubernetes

Build one modular application (a "modular monolith": contracts, bookings, billing as separate modules in one solution) and run each host project (see Code architecture) as its own deployment. Kubernetes does not require microservices, and a small team should not split the database or the code into services.

| Workload | Kubernetes object | Replicas | Notes |
| --- | --- | --- | --- |
| Back office + API | Deployment + Service | 2+ | Stateless; scales with users |
| Public booking site | Deployment + Service | 2+ | Scales separately from the back office |
| Worker (AI extraction, emails, PDF generation) | Deployment | 1+ | Worker host; AI calls never block a web request |
| Database schema updates | Job, run before each release | — | EF Core migrations applied once, not by every pod at startup |
| Scheduled tasks (nightly checks, reminders) | CronJob or the job queue's recurring jobs | — | Must run once per schedule, not once per pod |
| Routing and HTTPS | Ingress controller + cert-manager | — | One domain for the back office, one for the public site |
| Guest assistant (web chat + WhatsApp webhook) | Deployment + Service | 2+ | Public endpoint; reads only published data through the API, never contract drafts or costs |
| Partner API and portal | Deployment + Service | 2+ | Own host name and rate limits per partner; Partner API host, same Application and Domain code as the back office |
| MCP server (manager assistant) | Deployment + Service | 1+ | Own host name, OAuth sign-in, read-only tools |

What changes because of the cluster:

- **Pods are disposable**: no files on the pod's disk. Contract files and generated PDFs go to object storage (cloud bucket, or MinIO on-premise).
- **Shared login keys**: ASP.NET Core Data Protection keys must be stored centrally (database or Redis); otherwise users are logged out when a request hits another pod.
- **Concurrency is real**: with 2+ pods, invoice numbering, contract approval and allotment must be safe inside database transactions. The design above already requires this; the cluster makes it non-negotiable.
- **Secrets**: database password and AI API key in Kubernetes Secrets (or an external secret manager), never in config files. This fixes the plain-text passwords in today's `app.config`.
- **Health checks**: liveness and readiness endpoints, so Kubernetes restarts stuck pods and only sends traffic to ready ones.
- **Observability**: logs to stdout, metrics and traces with OpenTelemetry, a dashboard (Prometheus + Grafana or the cloud provider's).
- **Environments**: dev, staging and production as separate namespaces (or clusters), deployed by a CI/CD pipeline from the container registry with Helm or Kustomize.

**PostgreSQL on or off the cluster**: billing data must not be lost, so the database needs backups and point-in-time recovery. A managed PostgreSQL service is the safest choice. If the cluster is on-premise, use a PostgreSQL operator (e.g. CloudNativePG) with automated backups to object storage, and test a restore before go-live.

**Where the cluster runs — recommendation**: a managed Kubernetes service in a European region close to Morocco (e.g. Paris or Madrid), with the provider's managed PostgreSQL and object storage. First choice: Azure (AKS, Azure Database for PostgreSQL Flexible Server, Blob Storage, Key Vault), because it fits a .NET team and is common with enterprise customers. Lower-cost alternative: a smaller European provider offering managed Kubernetes and managed PostgreSQL (e.g. OVHcloud, Scaleway). Self-managed Kubernetes on the customer's servers only if the law or the customer requires data on-site, because someone must then patch, back up and monitor the cluster and the database. Before choosing, confirm with the customer how Moroccan data-protection rules (Law 09-08, CNDP) treat storing guest data outside Morocco. For development, run the same Helm charts on a local cluster (kind or k3d).

*(Diagram in the shared doc: deployment view · cluster workloads and external services)*

All workloads use the same database and storage; only the worker calls the AI API. Pods hold no data, so any of them can be restarted or scaled at any time.

## Caching

Cache what is read often and changes rarely, but never let a cached value decide a booking, a price that is charged, or an invoice: those are always re-checked in the database inside a transaction.

| What | Cache? | How long | Cleared when |
| --- | --- | --- | --- |
| Reference data (cities, room types, boards, currencies, tax rules) | Yes | Hours | Changed in the back office |
| Catalogue pages, photos, circuit descriptions | Yes, plus CDN for images | Hours | Product or template published |
| Active contract versions and rate tables used by the pricing engine | Yes | Until changed | `ContractVersionApproved` event |
| Availability search results (public site, partner API) | Yes, short | 1–5 minutes | Booking or stop sale on that hotel and date |
| Exchange rates | Yes | Until the next rate update | New rate entered |
| Manager report figures (MCP) | Yes, short | Minutes | Not needed; labelled with the time computed |
| Quote, booking, allotment, invoice number, payment, cash balance | **No** | — | Always read and written in the database |

How to build it:

- **Two levels**: in-memory cache in each pod plus a shared Redis cache, so all pods see the same values and a cleared entry is cleared everywhere. .NET's `HybridCache` provides both levels behind one interface and prevents many requests rebuilding the same entry at once.
- **Redis on Kubernetes**: use the cloud provider's managed Redis rather than running it in the cluster; the same Redis also stores ASP.NET Core Data Protection keys and can carry rate-limit counters for the partner API.
- **Clear by events, not only by time**: the Application layer raises events (contract approved, product published, booking confirmed, stop sale added) and a handler removes the related cache keys. A time limit remains as a safety net.
- **Behind an interface**: use cases depend on a cache abstraction in the Application layer; Redis lives in Infrastructure, in line with Clean Architecture. Tests run with an in-memory cache.
- **Public site**: ASP.NET Core output caching for catalogue pages, and a CDN for images and static files.
- **AI**: the guest assistant's fixed instructions and knowledge can use the AI provider's prompt caching to lower cost and latency; embeddings for the knowledge base are computed once per text change and stored in PostgreSQL, not recomputed per question.
- **Measure first**: add caching where monitoring shows slow or repeated queries, starting with availability search and the pricing engine.

## Delivery plan

The new system starts empty: no data is taken from the old system, which only served as a reference for what to keep and what to do better. Everything is set up fresh, much of it with AI help, and the agency switches to the new system on a chosen date.

1. **Foundation**: cluster, CI/CD, sign-in, users and roles; reference data entered fresh: branches, currencies, cities, room types, boards, service families, tax rules, accounting account mappings.
2. **Partners and catalogue**: suppliers, clients and partners, products, and circuit templates built from the agency's circuit descriptions (AI can draft a template from a brochure or programme document).
3. **Contracts and pricing**: current supplier and sales contracts entered through AI intake and approved; the pricing engine checked with the contracting team on real contract examples.
4. **Bookings and itineraries**: files, itineraries, vouchers, transfers, rooming and arrival lists.
5. **Billing and treasury**: invoices, credit notes, payments, cash registers and bank accounts. On go-live day, cash is counted and current bank balances are entered as starting figures.
6. **Channels**: public site and guest assistant, partner portal and API, manager assistant (MCP).

**Go-live**: from the switch date, every new file is created in the new system. Start with one branch for a few weeks, then the second. Files already running in the old system are finished there; after that, the old system is kept read-only or retired, as the customer decides.

Moving from manual to automatic prices: for the first weeks the system proposes the price and the operator can override it with a reason. Overrides are logged and reviewed each week; once they are rare, only supervisors can override.

## Open questions for the customer

- [ ] Which VAT scheme applies to their invoices (on margin or on full price), and must VAT and tourist tax appear on client invoices?
- [ ] Do they need more than 5 seasons, weekday prices, or minimum stays in contracts?
- [ ] Should stop sales and release dates block bookings automatically?
- [ ] Which reports do staff and managers need every day, week and month?
- [ ] Can you share the "factmarge" API/web project referenced in the solution?
- [ ] Are Agadir and Marrakech still the only branches, with separate number series?
- [ ] Who may approve contract versions: one person per branch, or a central contracting team?
- [ ] Are sales prices mostly a markup on supplier rates, or negotiated net rates per tour operator?
- [ ] Should supplier promotions be passed on to clients by default?
- [ ] What minimum margin should trigger a warning, and when should it block a booking?
- [ ] Which partners would connect by API first, and do any require a specific format (XML, OpenTravel)?
- [ ] Do partners have credit limits and payment terms today, and who approves bookings above them?
- [ ] Does the agency run its own vehicles and drivers, use transport suppliers, or both?
- [ ] How many circuit templates are sold today, and who can describe each day's transfers, excursions and meals?
- [ ] Should vouchers be sent to suppliers automatically once a file is confirmed, or always after an operator check?
- [ ] Do reps still sell excursions with paper ticket books, and would they use a phone app instead?
- [ ] Which banks does the agency use, and can they export statements and import transfer files?
- [ ] Which payment amounts need a manager's approval?
- [ ] What is the switch date, which branch goes first, and are files already running in the old system finished there?
