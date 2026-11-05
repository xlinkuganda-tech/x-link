# X-Link: A Website From One WhatsApp Message

**Send a text, photo, or voice note. Get a live, payable, multilingual storefront back in about a minute.**

**Status: Beta.** The core flow (message to live shop, edits by chat, card/PayPal checkout, in PayPal test mode) works end to end. Mobile Money, SafeBoda rides, and AI photo clean-up are not finished (see "Honest limits").

Built for the Meta Global AI Developer Hackathon · Theme: *Build for Your World* · Track: Indie Developer

- Demo video: https://youtu.be/Dy8ecIZcBLw
- Live demo (message this WhatsApp number): https://wa.me/256767535525
- Workflow exports: see the `workflows` folder in this repository

---

## The problem

A very large number of small vendors already sell on WhatsApp: market stalls, small electronics shops, and other small businesses. But their "shop" is a photo album and a price list buried in a chat. Buyers cannot browse, compare, or pay, and every price change means retyping it to each customer. These vendors typically have only a phone, no developer, and no hosting budget.

## What X-Link does

The merchant messages the business WhatsApp number, for example:
*"We're called MetroTech, we sell JBL speakers and Oraimo earbuds at Pioneer Mall."*

X-Link replies with a link to a live storefront that includes:

- the merchant's own photos, prices, and brand colours
- a cart, offer negotiation, and checkout by card/PayPal or pay on delivery
- a map pin and delivery notes
- WhatsApp order handoff, with merchant confirmation
- a language switcher (17 inline locales), and prices shown in each visitor's own currency
- a **/map discovery page** where buyers browse all shops, search, filter by category, and read merchant pop-ups
- **sales insights in chat:** a merchant can ask "how much did I sell this week?" and get an answer from the orders table
- **two free websites per phone number**, enforced: the third is refused with a clear message, and the second business gets its own slug and its own record instead of merging into the first

Updates also happen through chat, by text or by voice: *"change the iPhone to 6,500,000"* changes that price and nothing else, and *"make the website green"* restyles the whole storefront in its new brand colours. The first draft is not always perfect, so the merchant corrects any mistake the same way they made the shop: by sending another message, and X-Link fixes it. The merchant never opens an editor, installs an app, or writes HTML.

## How it uses Meta's stack

| Component | Role |
|---|---|
| **WhatsApp Cloud API** | The entire front end: inbound webhook, text, voice notes, images, and outbound replies |
| **Meta Model API: Muse Spark** | Multimodal vision on merchant photos, plus an automated quality review of each generated page |
| **Meta Model API: Muse Image** | Experimental: improves a merchant's own photo when it is too poor to use (see limits) |
| **Muse Spark open-weights model (served via Baseten)** | The runtime agent that turns a conversation into a structured shop record |

## How it works

1. The merchant sends a text, photo, or voice note.
2. Voice is transcribed. Photos are downloaded and read by the vision model.
3. The message is parsed into an intent: build a shop, edit something, ask a question, or reset.
4. The agent produces a structured shop record: name, tagline, items, prices, currency, location, brand colours, payment note.
5. Location is resolved to coordinates, and any previously saved shop is attached so edits change the shop rather than rebuild it.
6. The **storefront compiler** generates a complete, self-contained, responsive page (about 110 KB) for that merchant, with nothing to deploy.
7. The page is stored and the link is sent back. Buyers order on WhatsApp or pay online, and orders and status changes are logged.

## What is technically hard

- **Runtime storefront compiler:** a full HTML/CSS/JS storefront is generated per merchant, per update. It has a cart, offers, payment tiles, a map, and accessibility checks, and there is no CMS or template deployment.
- **Per-business isolation:** one phone number can host several businesses, each with its own record. This is implemented and still being hardened: we have seen a second shop inherit details from the first in testing.
- **Photo intelligence:** the vision model decides whether an image is a product shot, a screenshot, or a bad photo, and routes it to the right place on the page.
- **Voice-first and multilingual:** merchants can speak instead of type, and pages render in 17 inline languages.

## Implementation

X-Link is orchestrated in **n8n** (99 nodes) with a large amount of custom JavaScript for the storefront compiler, intent routing, isolation logic, and image handling. Data lives in n8n Data Tables (catalogue, orders and session state); Baserow holds a legacy store table and merchant revenue. Every system it depends on, and its current status, is listed in "Systems and dependencies" below. A separate workflow handles checkout, PayPal order creation and capture, webhook status sync, and merchant confirmation.

## Systems and dependencies

X-Link is a chain of services, and some are on free or test tiers. Status as of submission:

| System | Role | Status |
|---|---|---|
| **n8n** (self-hosted) | The whole product: main workflow (99 nodes), payments, storefront/map server, domain workflow | Working |
| **n8n Data Tables** | Primary database: store catalogue, orders, session state | Working |
| **WhatsApp Cloud API** | Entire front end: webhook, text, photos, voice notes, location pins, replies | Working |
| **Meta Muse Spark** (`muse-spark-1.3`) | Vision on merchant photos; quality review of each generated page | Working |
| **Meta Muse Image** (`muse-image-1.0`) | Improves a merchant's own photo when it is too poor to use | Experimental |
| **Muse Spark open-weights** (`muse-spark-open-1026`) | Reasoning agent that turns a message into a structured shop | Working |
| **Baseten** | Serves the open-weights agent model | Working |
| **Groq** (`whisper-large-v3`) | Voice transcription (WhatsApp voice notes arrive as Ogg/Opus, which Meta's voice model could not take) | Working |
| **PayPal** (sandbox) | Order creation, buyer approval, capture | Test mode only |
| **SerpAPI** | Google Images search for product and shop-front photos | **Usage limit already reached in testing** |
| **imgbb**, **freeimage.host** | Photo hosting (primary, fallback) | Working |
| **open.er-api.com** | Live exchange rates, fetched on every build | Working |
| **ipapi.co** | Detects visitor country to show prices in their currency | Working |
| **Nominatim**, **Photon** | Geocoding a place name to coordinates | Working |
| **OSRM** | Driving directions to the shop | Working |
| **Leaflet** and Google-hosted map tiles | Storefront map and `/map` discovery page | Working |
| **short.io** | Short links | **Not working** (credential misconfigured); falls back to long URLs |
| **Cloudflare** | Domain availability and registration | Prototype, not part of this submission |
| **Baserow** | Legacy store table and merchant revenue only | Has caused a failed run (exec 4787, HTTP 503) |

## If something doesn't work (for judges)

If you test X-Link and something looks wrong, check here first. Most problems come from a free-tier or outside service, not from the core flow.

| What you see | Likely reason | What to do |
|---|---|---|
| No reply for a while | Builds take about 30 to 60 seconds, voice notes and photo edits can take up to about 85 seconds | Wait up to two minutes before resending |
| Still no reply, or an error message | A service we depend on (Baserow, Baseten, Meta or WhatsApp APIs, or our self-hosted n8n) was temporarily down. A Baserow 503 has caused a failed build before | Send the message again after a minute. The demo video shows the full flow working |
| Shop built but some product cards have no picture (initials tile) | Image search (SerpAPI) hit its usage limit or returned nothing, and we have no fallback | The shop itself works. Send your own photo and the vision model will place it |
| You get a long link, not a short one | short.io is misconfigured | The long link works the same |
| Voice note not understood | Voice is transcribed by Groq Whisper, which supports Swahili but not Luganda | Send text (Luganda typed as text is accepted), or a voice note in English or Swahili |
| Checkout shows PayPal sandbox | Payments run in PayPal test mode only | No real money moves. Mobile Money is shown but not wired up |
| Prices appear in an unexpected currency | The visitor's country is detected by ipapi.co, and the card charge may be made in US dollars | The page shows the amount before you pay |

The demo video (above) was recorded with everything working end to end.

## Results so far

Measured n8n build times:

| Case | Time |
|---|---|
| Best run (exec 3961) | 30.6 s |
| Typical text build, 3 products (exec 4782) | 42.7 s |
| Voice note build (exec 4792) | 59.2 s |
| Photo edit on a 4-product shop (exec 4771) | 83.7 s |

One run failed at about 92 s because of a Baserow 503 error (exec 4787), so reliability is still being hardened.

- Sandbox test orders have reached **PAID** status (for example, an order of USh 162,000 paid 17 seconds after creation).

## Honest limits and what's next

- **Image understanding is not perfect yet.** Poor photos can still be misread, and we are improving this.
- **Voice works in the languages Whisper supports, including Swahili, but Luganda is not one of them.** The voice transcription model has no Luganda, so a Luganda voice note may be flagged as unclear. Merchants can type in Luganda instead. English works best today.
- **Payments run in PayPal test mode only.** No real money moves in this submission.

- **Third-party usage limits.** X-Link depends on outside services (for example SerpAPI for image search, plus the photo-hosting, exchange-rate, location and geocoding APIs). We have already reached SerpAPI's usage limit during testing, and search sometimes returns no usable photo, so some product cards may show an initials tile instead of a picture. Because we only generate AI images to improve a merchant's own photo, there is no fallback when search finds nothing; a merchant can fix this by sending their own photo. A production version would need paid plans or fallbacks for these services.
- **Mobile Money payments and SafeBoda ride requests are not wired up yet.** The storefront shows these options as part of the intended design, but they are not functional in this submission. They are the first integrations planned, because Mobile Money is how many vendors and buyers in our market actually pay.
- **AI photo clean-up is not perfect and can make mistakes.** First-build pictures come from image search, and AI image generation only runs to improve a merchant's own photo when it is too poor to use. Results are not always placed or rendered correctly, so the best results come from good original photos, which the vision model routes correctly. Merchants can ask for a different image by sending another message.
- **Currency:** merchants set prices in their own currency (built and tested with Ugandan shillings). Each visitor sees prices in their own currency based on their location, and card/PayPal payments are charged in the visitor's currency when PayPal supports it, otherwise in US dollars (PayPal does not accept shillings). The page tells the buyer the amount before they pay. Exchange rates are fetched live on every build.
- WhatsApp sending and PayPal capture are proven in simulation and partly in live sandbox, not yet in sustained production.
- **Voice notes are transcribed with Groq (Whisper), not a Meta model.** We tried Meta's Muse Voice Transcribe, but it requires mono 16-bit PCM WAV at 16 or 24 kHz, and WhatsApp voice notes arrive as Ogg/Opus, so they would need a conversion step we did not finish in time. Moving voice transcription to Muse Voice Transcribe is planned.
- Short links currently fall back to long URLs (a credential misconfiguration on our side, not a platform limit).
- The page includes an inline translation pack that should be trimmed.
- The custom-domain purchase flow is a prototype and is not part of this submission.

**Planned after the hackathon (not part of this submission):** improving image understanding and local-language support (including a transcription option for Luganda voice notes), and making builds more reliable under load, as well as batched multi-photo handling (ten photos, one rebuild), real short links, and a scheduled sales digest sent to the merchant.

## Why it matters

Small vendors do not need to learn how to build a website. They already know how to send a message. X-Link turns that message into a business.

## Credits

Background music in the demo video was generated with Gemini. It is not part of X-Link itself.
