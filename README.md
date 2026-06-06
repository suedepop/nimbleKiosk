# nimbleKiosk — Project Plan

**nimbleKiosk** is a cloud-based digital signage platform: users subscribe online, build playlists of images and videos, assign them to virtual displays, and a Raspberry Pi at each screen plays the assigned content.

---

## 1. Decisions locked in

| Decision | Choice | Why |
|---|---|---|
| Builder | Solo developer | Plan minimizes custom work; buy/borrow everything non-core |
| First-version goal | Polished, sellable product | Sequenced so there's a sellable core before marketing polish |
| Hardware baseline | Raspberry Pi 4 (2GB) recommended; Pi 5 also supported | Pi 4 = best value for 1080p, runs cooler on a simpler PSU, in production through ≥2030. Pi 5 only needed for 4K/heavy dashboards/multi-display |
| Device OS / image | One 64-bit Raspberry Pi OS (Bookworm or later) image for both boards | Single image boots Pi 4 and Pi 5; agent detects the model at runtime. No second build to maintain |
| Device onboarding | Generic image + Raspberry Pi Imager for WiFi + pairing code | No per-user image generation, no storing WiFi passwords, trusted official tool |
| Display target | 1920×1080, horizontal or vertical | Per the original spec |
| Pricing model | Per-display with volume discounts (monthly + annual) | Industry norm, maps to cost, simplest Stripe build, scales with customer size |
| Trial | 14-day no-card trial, pauses on expiry | Maximizes signups for a new product; convert via reminders + Customer Portal |
| Billing | Stripe (existing account) | Already set up and in active use |

**Your two differentiators worth real custom effort:** the content/playlist editor, and a rock-solid Pi player + onboarding. Everything else is a managed service.

---

## 2. Architecture

Three pieces:

1. **Cloud app** — marketing site + logged-in dashboard (signup, trial, content upload, playlist builder, display management, device pairing).
2. **Backend + storage** — API, Postgres database, object storage + CDN for media, a sync channel the Pis use to fetch their content.
3. **Pi player** — generic OS image with a small agent that connects to WiFi, shows a pairing code, downloads its assigned playlist, caches it, and plays the loop.

### Data model (core entities)

```
Organization (tenant)
  └─ User(s)
  └─ Subscription (Stripe customer + status + trial end)
  └─ MediaAsset(s)        # uploaded image/video, stored in R2, + transcoded variant
  └─ Display(s)           # name, orientation (H/V), assigned Playlist
  └─ Playlist(s)
       └─ PlaylistItem(s) # ordered: media ref, duration (video default = full length), position
  └─ Device(s)            # the Pi: pairing code, bound Display, last-seen heartbeat, agent version
```

A `Display` is the logical screen the user configures. A `Device` is a physical Pi bound to a Display. Keeping them separate means a user can swap the Pi behind a screen without rebuilding the content.

---

## 3. Tech stack

| Layer | Choice (Azure-first) | Notes |
|---|---|---|
| Web app | Next.js (React) | Marketing pages (SSR for SEO) + dashboard in one codebase |
| Hosting | Azure Container Apps (Consumption) | Scale-to-zero, generous monthly free grant; cheapest start. Static Web Apps free tier is an alternative for the frontend |
| Database | Azure Database for PostgreSQL Flexible Server (Burstable B1ms) | Cheapest managed Postgres tier; free for 12 months on a new Azure account |
| Object storage | Azure Blob Storage (Hot, LRS) + Azure CDN/Front Door | Watch egress (~$0.09/GB); local caching on the Pi keeps it low. See cost notes |
| Auth / multi-tenant | Microsoft Entra External ID | Free for first 50,000 monthly active users; keeps auth all-Azure. (Clerk is the easier-DX alternative if you'd rather not wire up Entra) |
| Billing + trial | Stripe (Billing) | **Already set up** — reuse your existing account. 14-day trial via `trial_period_days` |
| Transcoding | ffmpeg on Azure Container Apps **jobs** | Azure's managed media/transcoding service was retired, so run ffmpeg yourself (event-driven, pay-per-run) |
| Device sync / fleet | HTTPS polling (v1) → Azure IoT Hub (v2) | Polling against your own app is free to start; IoT Hub adds push + device management when you scale |
| Transactional email | Azure Communication Services Email | Azure-native; Resend/Postmark are fine alternatives |
| Error monitoring | Azure Monitor / App Insights | Bundled with Azure; Sentry if you prefer its DX |
| Pi OS | Raspberry Pi OS Lite, 64-bit, **Bookworm or later** | Single image for Pi 4 + Pi 5 (Pi 5 won't boot pre-Bookworm). Both default to Wayland (labwc/Wayfire), so a consistent kiosk stack |
| Pi playback | Chromium in kiosk mode | mpv fallback if browser video proves unreliable. Transcode to **H.264 MP4** — plays on both boards (hardware decode on Pi 4, fast software decode on Pi 5) |
| Pi agent | Python or Node service | Pairing, sync, cache, health heartbeat |
| Device sync | HTTPS polling (v1) → MQTT (v2) | Polling ships fast; MQTT adds instant "change it now" |

---

## 4. Device onboarding flow

This is the flow that saves the most engineering:

1. User installs **Raspberry Pi Imager** (official tool) and selects your hosted generic image.
2. In Imager's settings, the user enters their **WiFi name/password** and writes the SD card. (You never see or store the password.)
3. Pi boots, connects to WiFi, and displays a **6-character pairing code** on screen.
4. User enters that code in your dashboard and picks which **Display** to bind it to.
5. Pi agent pulls the assigned playlist, caches media locally, and starts the loop.

The Pi keeps playing from local cache if the network drops, and re-syncs when it returns.

**v2 upgrade (later):** captive-portal onboarding (the Pi broadcasts its own setup WiFi so users with no keyboard can enter network details from a phone), and/or selling pre-flashed SD cards or devices as a paid add-on.

### Device hardware & image notes

- **One image, both boards.** Build on 64-bit Raspberry Pi OS Bookworm (or later). The same image boots Pi 4 and Pi 5; the Pi 5 just requires Bookworm as a minimum. The agent detects the board model at boot and selects the video-decode path.
- **Video decode differs, but H.264 is safe everywhere.** Pi 4 has hardware H.264 + H.265 decode. Pi 5 dropped hardware H.264 (keeps hardware H.265) but its CPU software-decodes 1080p H.264 easily — faster than the Pi 4's old hardware block. So the H.264 transcoding pipeline covers both. (Reach for hardware HEVC on the Pi 5 only if you later add 4K.)
- **Pi 5 has a real per-unit cost tail.** It needs the official Active Cooler (~$5–10, effectively mandatory — it throttles hard without it) and a 5V/5A 27W USB-C supply. Budget ~$25–40 of accessories on top of the board. The Pi 4 runs on a simpler 15W supply with passive cooling, which is why it's the recommended baseline.
- **Pi 5 config gotcha:** it defaults to 16K memory pages; if a chosen filesystem needs 4K pages, set `kernel=kernel8.img` in `config.txt`.

---

## 5. Build phases

> Estimates assume focused, full-time-equivalent solo work. Roughly double for part-time.

### Phase 1 — Foundation (2–3 weeks)
- Repo, environments, CI, Postgres schema (the data model above) on Azure Database for PostgreSQL
- Entra External ID auth + organization/multi-tenant setup
- Media upload to Azure Blob (presigned/SAS uploads), thumbnail generation
- Basic transcoding worker (ffmpeg → H.264 MP4) so anything plays on the Pi

### Phase 2 — The hard path first: Pi player + sync (3–5 weeks)
- Generic Pi OS image + first-boot agent
- Pairing code generation, display + device on screen, dashboard binding
- Content sync API (polling), local media cache, offline resilience
- Chromium-kiosk player honoring item order, per-item duration, and **video = full length by default**
- Orientation handling (H/V)
- **Goal: cloud → device → screen works end to end before anything else**

### Phase 3 — The editor (3–4 weeks)
- Display CRUD (name, 1080p, orientation, assigned playlist)
- Playlist builder: drag-to-reorder, set durations, default video length
- Live preview in the browser (a web version of the player)
- This is your showcase surface — make it feel good

### Phase 4 — Billing + trial (1–2 weeks)
- Stripe per-display subscription (quantity = displays) with tiered/volume pricing, monthly + annual
- 14-day no-card trial; pause subscription on expiry until a card is added
- Webhook handler + access gating (the load-bearing pieces)
- Customer Portal for self-serve billing; trial-reminder emails
- See **§7 Pricing & billing** for the full design

### Phase 5 — Marketing + onboarding polish (2–3 weeks)
- Landing page, pricing page, signup funnel
- The "flash with Imager → enter pairing code" walkthrough with screenshots
- Empty states, first-run guidance

### Phase 6 — Hardening for sale (3–4 weeks)
- Device heartbeat + online/offline status in dashboard
- Remote reboot / "force re-sync now"
- Swap polling for MQTT (instant content changes)
- OTA agent updates (fix devices in the field without re-flashing)
- Monitoring, alerting, backups

**Total: ~14–21 weeks full-time (≈3.5–5 months), or roughly double part-time.**
After Phase 3 you have a demoable product; after Phase 4 you can take money.

---

## 6. Monthly cost estimate (Azure, scale-as-you-grow)

All figures approximate, US regions, as of mid-2026. The strategy is to start near-zero and only turn on paid tiers as load demands.

### Stage 1 — Cheapest possible start (pre-revenue → first handful of customers)

| Service | Cost | Notes |
|---|---|---|
| Compute — Container Apps (Consumption) | ~$0–10/mo | Monthly free grant (≈180k vCPU-sec + 360k GiB-sec + 2M requests) covers light traffic; scales to zero when idle |
| Database — PostgreSQL Flexible Server B1ms (1 vCore, 2 GiB) | ~$0–13/mo | **Free for 12 months** on a new Azure account (B1ms + 32 GB); ~$12–13/mo after |
| Blob Storage (~100 GB, Hot/LRS) | ~$2/mo | ~$0.018–0.02 per GB-month |
| Egress | ~$0–5/mo | First 5 GB/mo free, then ~$0.09/GB. Low because the Pi caches and only re-downloads on change |
| Auth — Entra External ID | $0 | Free to 50,000 monthly active users |
| Device sync | $0 | HTTPS polling against your own app (no IoT Hub yet) |
| Transcoding — Container Apps jobs | ~$0 | Low volume fits inside the free grant |
| Email — Comm. Services | ~$0–5/mo | Pay-per-email; tiny at low volume |
| Domain | ~$1/mo | ~$12/yr |
| **Stage 1 total** | **≈ $5–40/mo** | Can be near $0 in the first 12 months |
| Stripe | already set up | 2.9% + $0.30 per charge, + ~0.5–0.7% Billing fee — a % of revenue, not fixed |

### Stage 2 — Scaling up (dozens–hundreds of customers, more devices & content)

| Service | Cost | Notes |
|---|---|---|
| Compute — Container Apps (min 1 replica) | ~$15–40/mo | Always-on API for snappy responses; jobs billed per run |
| Database — B2ms or General Purpose 2-vCore | ~$25–120/mo | Move up when the burstable credits run thin |
| Blob Storage (~1 TB) | ~$18/mo | Grows with the media library |
| Egress + CDN | ~$20–80/mo | **The line to watch.** Scales with devices × content changes; CDN caching keeps origin egress down |
| Auth — Entra External ID | $0 | Subscribers are well under 50k MAU (screen *viewers* aren't users) |
| Device sync — IoT Hub | ~$10–25/mo | Basic (~$10) or Standard S1 (~$25) adds push + fleet management |
| Email | ~$10–20/mo | |
| **Stage 2 total** | **≈ $80–300/mo** | Dominated by database, egress, and IoT Hub |

### Where Azure fits — and where to be careful

- **Great fit:** Container Apps (scale-to-zero compute), Blob Storage, Postgres Flexible Server, Entra External ID (all-Azure SSO, free to 50k MAU), and especially **IoT Hub** for device fleet management once you scale — that's purpose-built for exactly this kind of project.
- **Watch egress.** Unlike some object stores, Azure charges ~$0.09/GB to send data out. For signage this stays modest because the Pi caches locally and only re-downloads when content changes — but if egress ever dominates the bill, the escape hatch is putting *media only* on a zero-egress store (e.g. Cloudflare R2) while keeping everything else in Azure.
- **No managed transcoding.** Azure's media-services product was retired, so plan to run ffmpeg yourself on Container Apps jobs (already in the plan).

---

## 7. Pricing & billing

### Model
**Per-display, with volume discounts**, billed monthly or annually. The unit is one display; a customer pays for as many displays as they run. Annual is priced as ~10 months ("2 months free") to improve cash flow and retention.

Volume discounts are built into the price itself (Stripe tiered pricing), so larger customers get a lower per-display rate automatically — no separate named plans to maintain. Illustrative starting points (validate against real costs and competitors; not a fixed recommendation): ~$12/display/mo, ~$120/display/yr, with breaks above 10 and 50 displays.

### Trial
**14-day no-card trial.** Maximizes signups for a new product. Stripe runs the 14 days natively; on expiry the subscription **pauses** (screens show an "inactive" message) until a card is added — no failed charges, clean recovery. Nudge conversion with day-11 and day-13 reminder emails and the Customer Portal.

### Stripe object setup
- **Product:** one ("nimbleKiosk Subscription").
- **Prices:** monthly + annual, both `billing_scheme = tiered` (volume or graduated), unit = one display.
- **Customer:** the Organization (store `stripe_customer_id`).
- **Subscription:** quantity = number of displays; Stripe prorates automatically on quantity change.
- **Checkout:** hosted Stripe Checkout for signup (no custom card form).
- **Customer Portal:** hosted "Manage billing" — card, plan switch, cancel, invoices.
- **Trial config:** `trial_period_days = 14`, `trial_settings.end_behavior.missing_payment_method = pause`.
- **Webhooks (source of truth):** `checkout.session.completed`, `customer.subscription.created/updated/deleted`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.trial_will_end`.

### Access gating (subscription status → access)
- `trialing` / `active` → full access
- `past_due` → short grace, then restrict
- `paused` / `canceled` / `unpaid` → dashboard read-only **and device sync returns an "inactive" screen** so unpaid Pis stop showing content

### Display quantity ↔ billing
When a user adds/removes a display, update the subscription item quantity; show Stripe's proration preview before they confirm. Block creating displays beyond the paid quantity unless they accept the upgrade.

### Build notes
The **webhook handler** and **access gating** are the load-bearing pieces — everything else is UI that Stripe largely hosts. Use **Stripe test clocks** to simulate trial-end, renewal, and failed-payment flows without waiting real days. Add **Stripe Tax** if selling across states/countries; lean on **Smart Retries** for dunning.

---

## 8. Key risks & gotchas

- **SD card wear** — 24/7 signage kills cheap cards. Recommend quality cards or USB-SSD boot, and run the OS read-only where possible.
- **Browser video on the Pi** — if Chromium-kiosk video playback is unreliable, the mpv fallback is your escape hatch. Validate this early in Phase 2.
- **Transcoding** — always transcode uploads to H.264 MP4; users will upload formats the Pi can't hardware-decode.
- **Offline resilience** — a screen that goes blank when WiFi hiccups looks broken. Cache-and-keep-playing is a must, not a nice-to-have.
- **Support burden** — narrowing to Pi 4/5 keeps this sane. Resist supporting older Pis for v1.
- **Pi 5 cost & heat tail** — beyond the higher board price, the Pi 5 needs an active cooler and a 27W PSU and throttles badly without them. For 24/7 signage at 1080p the Pi 4 is the cheaper, cooler, recommended choice; treat Pi 5 as opt-in for 4K/multi-display.

---

## 9. Open decisions still to make

1. **Sell pre-flashed Pis / SD cards?** — a UX win and a revenue line, or stay bring-your-own.
2. **Self-host transcoding (ffmpeg on Container Apps jobs) vs. a third-party service** — build effort vs. usage cost.
3. **Exact price points and volume break thresholds** — set once you've sized infra cost per display and checked competitors.
