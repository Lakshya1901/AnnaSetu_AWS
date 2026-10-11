# AnnaSetu: submission writeup

WeMakeDevs x AWS Environmental Hacks, track: Waste and Energy.

Every day, a city's mandis turn good food into garbage. AnnaSetu stops the glut before the truck leaves, rescues what's left before it's dumped, and turns the rest into energy, not landfill.

## Problem

India's fresh produce is lost mostly at or near the farm gate, and mostly because its market value collapses before the food itself spoils. Fruit loss is 6.02-15.05% and vegetable loss 4.87-11.61%, about 7.36 Mt and 11.97 Mt a year (Rajya Sabha, Feb 2025). Tomato loss is 8.37% on the farm against 3.25% at the market. In the Kolar 2025 glut, harvest and transport cost about Rs 70 per 15 kg box while the sale price fell as low as Rs 30, and crops were left unharvested. At the same time, Udumalaipettai tomatoes were dumped while the same box sold for about 5x more across the Kerala border.

The dispatch decision is made with yesterday's local price and a trader's word, and a whole belt ships to the nearest mandi on the same day. Public mandi data is never turned into a per-load decision, and stock left unsold at a city mandi has no channel except the dump. For scale: 1% of the 19.3 Mt of fruit and vegetables lost each year is about 193,000 tonnes of food. We target perishables only and claim no percentage reduction until one is measured.

## What we built

One surplus router with three entry points:

- **Prevent.** An FPO dispatch manager says, by voice or form, what load they have. AnnaSetu ranks every reachable outlet by net value per kg after freight, fees and trip spoilage, and spreads the day's loads so it never pushes a market into a glut itself.
- **Rescue.** A trader at a city mandi (headline: Azadpur, Delhi) logs unsold stock. The engine proposes an edible/spoiled split, the trader confirms or edits it, and the edible part goes to another mandi that still pays, a processor or a food bank.
- **Recover.** The inedible part goes to animal feed, biogas or compost, never a dump.

The Impact Ledger shows kg kept out of landfill (Prevented + Rescued + Recovered) as a range. Redirected kg is always a separate line and never added in.

## How it works

**Data.** Market-level AGMARKNET prices and arrivals for every commodity, 2021 to May 2026, from the India Data Portal bulk files; today's Agmarknet prices in Live mode (arrivals need a captcha, so Live is prices only). Weather from NOAA GHCN, NASA POWER and the Open-Meteo forecast. Road distance and drive time from Amazon Location.

**Model.** Plain Python, no machine learning. Each market is rated safe, watch or glut from arrivals against its seasonal usual and the 3-day price change. Each load's value is the expected price on arrival (including what AnnaSetu already sent there) minus spoilage estimated from temperature and hours on the road, freight, fees, commission and handling. Loads are allocated largest first; a market that would tip into glut is skipped. When no fresh market pays, the food recovery hierarchy takes over. The backtest showed arrivals do not lead prices, so the mode is same-day: AnnaSetu avoids sending a load into a glut today and makes no early-warning claim.

**App.** React Native (Expo) for Android and iOS. Hold-to-speak input; a recommendation card with earnings first, then waste avoided, redirected and extra distance; a "Why not X?" comparison; Glut Radar; Today's loads; Impact Ledger. English plus India's 20 most spoken languages (most machine-drafted, pending native review). Risk is always colour, word and icon. Every recommendation can be overridden.

## AWS usage

Everything runs in ap-south-1, defined in `infra/template.yaml` and built, run locally (`sam local start-api`) and deployed with the **AWS SAM CLI**, the AWS open-source tool we used.

- **API Gateway (HTTP API)** with a Lambda authorizer: the only entry point; the app holds no AWS credentials.
- **Lambda x3:** `advisor` (radar, recommend, plan, impact), `voice` (upload, transcribe, parse, speak), `ingest` (recomputes market risk per crop).
- **DynamoDB:** MarketDay, MarketRisk, Outlets, Plans (every recommendation and override is logged).
- **S3:** private data bucket (per-crop history, backtest and simulation results) and audio bucket deleted after 1 day.
- **EventBridge Scheduler:** daily ingest at 21:00 IST.
- **SQS + dead-letter queue:** fans out one message per preloaded crop and serves on-request crop loads, with retries.
- **Amazon Transcribe:** batch speech-to-text in 12 Indian languages.
- **Amazon Polly:** spoken replies in Hindi and Indian English.
- **Amazon Location Service:** geocoding markets and any typed place in India; truck routes and drive times.
- **SSM Parameter Store:** API keys as SecureString.
- **CloudWatch:** logs and alarms on ingest failure and the dead-letter queue.
- **Amazon Bedrock:** wired through the Converse API to parse spoken loads and explain recommendations, with a number guard. It is disabled in our account: model access was refused (D23). Per-language templates and a rule parser run instead; no code change is needed to enable it.
- **NOAA GHCN-Daily on the AWS Registry of Open Data:** station temperatures for the spoilage estimate, read from the public S3 bucket.

## Results

We replayed the deployed advisor code on every day of 1 March - 31 May 2026: ten Kolar tomato loads, five 5 t loads from Azadpur and one 2 t unsold tomato lot at Azadpur each day. 1,380 loads, 0 errors, deterministic.

- 94.6% of loads were sent away from the nearest mandi.
- Money saved: Rs 2.20 crore (1.73-2.96), estimate.
- Waste avoided: net +11 t (range -65 to +238 t), estimate. Gains came from glut days (Delhi +18 t, mostly banana on 39 Azadpur arrival-glut days); Kolar was -6 t, because longer trips add spoilage when there is no glut.
- Rescue: 169 t of 184 t unsold stock went to mandis within 100 km.
- Out-of-sample check (next real price at both markets within 7 days, freight-adjusted): the advice held on 1,248 of 1,279 diverted loads. Weak spot: Delhi cauliflower, 14 of 35.

Case study: one simulated 8-acre farm near Narela, Delhi (tomato, bottle gourd, okra, cauliflower). 112 loads (127 t), 0 errors, 85.7% sent elsewhere, money saved Rs 5.2 lakh (3.7-7.7, estimate), waste avoided +844 kg net (estimate), 3,576 kg of 3,900 kg unsold stock rescued, advice held on 79 of 92 diverted loads (cauliflower 4 of 10).

Reports: `docs/stat-report-2026.html`, `docs/case-study-narela-2026.html`.

## What is real and what is simulated

- Real: mandi prices and arrivals, documented glut events, weather, routes.
- Real organisations, not contacted: processors, food banks, biogas and compost units ("Not yet partnered").
- Simulated: all loads, the unsold lots and the Narela farm with its crops and quantities.
- Estimated: spoilage, waste avoided, money saved, biogas energy.
- Desk-research placeholders until FPO interviews: freight, fees, commission, handling, diesel and the dump-share table.

## Limits

- Same-day only; no early warning.
- Live mode has prices but no arrivals, so it cannot split loads across markets; the full model runs in Demo.
- Placeholder costs and dump share drive the waste figure; every such number is a range.
- Spoilage is estimated from temperature and travel time, not measured.
- Delhi has no seeded biogas or compost unit; Recover there shows "No biogas or compost unit near you yet".
- Spoken replies in Hindi and English only; most strings await native review.
- Bedrock is not active in our account.

## Built with

Built with Claude Code (Anthropic) as the AI coding tool, plus Python, React Native with Expo, and the AWS SAM CLI.

## Links

- Repository: https://github.com/Lakshya1901/aws
- Android APK: https://expo.dev/artifacts/eas/zimvjrUv9exYufPVm1UZrpx6QHH-wNO9-IqEBOfqDn8.apk
- Demo video: TBD
