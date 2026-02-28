# Trip Memories Engine — Design Document

> Date: 2026-02-28
> Status: Draft

---

## Problem

The UK trip page (`invitation.html`) is currently a static invitation. We want to extend it into a living keepsake — a curated journal of the trip's best moments plus a full scrapbook of every photo. Participants should be able to contribute by simply texting photos and messages to a phone number. Everything else is automated.

The infrastructure should be reusable for future trips.

---

## Goals

1. **Zero-friction contribution**: Guests text photos/messages to a number. That's it.
2. **Fully automated curation**: AI organizes, captions, and selects highlights. No human in the loop.
3. **Two views**: A curated day-by-day journal (the main experience) and a filterable scrapbook (everything).
4. **Keepsake**: Exportable to a self-contained static bundle that costs nothing to host forever.
5. **Reusable**: The processing engine works for any trip — UK 2026 is just the first instance.

---

## Architecture

Three cleanly separated layers:

### 1. The Engine (reusable, lives across trips)

| Component | Service | Purpose |
|-----------|---------|---------|
| SMS/MMS ingestion | Twilio (one phone number) | Receives photos and text messages from participants |
| Webhook handler | Cloud Run service | Receives Twilio webhooks, stores raw media, writes to Firestore |
| Batch processor | Cloud Run job + Cloud Scheduler | Nightly AI analysis, curation, and manifest generation |
| AI | Claude API (vision) | Captioning, quality scoring, day summaries, curation |

The Twilio number is reused across trips. Incoming messages are routed to the "active trip" based on a participant registry in Firestore.

### 2. Trip Data (per-trip, stored in GCS)

```
gs://trip-memories/
  uk-2026/
    raw/              # Original photos as received
    processed/
      large/          # 1600px resized
      thumb/          # 400px thumbnails
    manifest.json     # Complete structured output for the frontend
```

Firestore holds live state:
- **Participants**: name, phone number, mapped to trip
- **Incoming queue**: raw message records (sender, timestamp, text, media URLs)
- **Processing status**: last batch run, per-photo processing state

### 3. Frontend (per-trip, templated)

Each trip gets its own page. The invitation/planning content is bespoke. The memories rendering is a shared JavaScript component that reads `manifest.json` and renders the journal + scrapbook views.

---

## Processing Pipeline

### Ingestion (real-time, lightweight)

Triggered by Twilio webhook on each incoming SMS/MMS.

1. Cloud Run endpoint receives the webhook
2. Downloads media from Twilio's URL, saves to `gs://trip/raw/{timestamp}_{sender}.{ext}`
3. Writes a record to Firestore `incoming` collection:
   - `sender`: phone number
   - `senderName`: looked up from participant registry
   - `timestamp`: message timestamp
   - `text`: any accompanying text (or the full message if text-only)
   - `mediaUrls`: list of GCS paths for attached images
   - `processed`: false
4. Sends SMS reply: "Got it!" (or rotating fun responses)

Processing time: under 1 second. No heavy work here.

### Nightly Batch Job

Runs via Cloud Scheduler at 2 AM local trip time. Processes all records where `processed: false`.

**Step 1 — Extract & Enrich**

- Extract EXIF data: GPS coordinates, timestamp, camera/device info
- Reverse-geocode GPS to location name (Google Maps Geocoding API or a simple lookup table of known trip locations)
- Match timestamp to trip day (Day 1 = May 21, Day 2 = May 22, etc.)
- Resize images: generate 1600px (large) and 400px (thumbnail) versions
- Store processed images in `gs://trip/processed/large/` and `gs://trip/processed/thumb/`

**Step 2 — AI Analysis (Claude vision)**

For each photo, send to Claude with trip context:

```
This is a photo from Day {N} of a trip to {destination}.
Today's planned locations: {locations from itinerary}.
The photo was taken at {time} near {reverse-geocoded location}.
It was sent by {sender name}.

Please provide:
1. A natural, warm caption (1-2 sentences)
2. Activity category: sightseeing | food | sports | nature | group | transport | nightlife
3. Quality score 1-10 (composition, sharpness, visual interest)
4. Brief description for accessibility (alt text)
```

For text-only messages, classify as: quote, reaction, or story snippet.

**Step 3 — Curate the Journal**

- Group all items by day
- Within each day, cluster by time proximity and location
- For each cluster, select top 2-3 photos by quality score, prioritizing variety (different subjects, different people)
- Detect near-duplicates (bursts of similar shots) and keep only the best
- Select one hero photo per day (highest quality, most representative)
- Arrange into narrative arc: morning → afternoon → evening
- Place text messages chronologically as pull-quotes
- Generate a 2-3 sentence day summary via Claude, given the selected photos and their captions

**Step 4 — Build the Manifest**

Write `manifest.json` to GCS:

```json
{
  "trip": {
    "id": "uk-2026",
    "title": "England 2026",
    "dates": { "start": "2026-05-21", "end": "2026-05-28" },
    "participants": ["Elizabeth", "Jessica", "Kevin", "Grant"]
  },
  "days": [
    {
      "date": "2026-05-22",
      "dayNumber": 2,
      "label": "Day 2 — Fri, May 22",
      "title": "Theatre Night",
      "summary": "Borough Market for brunch, the Globe in the afternoon, then the main event — Aidan Turner live on stage at the National Theatre.",
      "journal": [
        {
          "type": "photo",
          "url": "processed/large/20260522_143022_elizabeth.jpg",
          "thumb": "processed/thumb/20260522_143022_elizabeth.jpg",
          "caption": "The view from the Globe Theatre's upper gallery — the Thames glinting in the afternoon sun.",
          "alt": "View from Shakespeare's Globe Theatre looking out over the Thames river",
          "by": "Elizabeth",
          "time": "14:30",
          "location": "Shakespeare's Globe",
          "category": "sightseeing",
          "quality": 8
        },
        {
          "type": "quote",
          "text": "This cream tea is incredible",
          "by": "Jessica",
          "time": "16:15"
        }
      ],
      "scrapbook": [
        {
          "url": "processed/large/20260522_101512_kevin.jpg",
          "thumb": "processed/thumb/20260522_101512_kevin.jpg",
          "caption": "Stacked wheels of cheese at Borough Market.",
          "alt": "Display of artisan cheese wheels at Borough Market stall",
          "by": "Kevin",
          "time": "10:15",
          "location": "Borough Market",
          "category": "food",
          "quality": 6
        }
      ]
    }
  ]
}
```

Mark all processed records in Firestore as `processed: true`.

---

## Frontend Rendering

### Page Lifecycle

| Phase | When | What shows |
|-------|------|------------|
| Pre-trip | Now → May 20 | Invitation: hero, overview, map, highlights, timeline (the plan), packing |
| During trip | May 21-28 | Completed days show memories in the timeline; future days still show the plan |
| Post-trip | May 29+ | Journal becomes the main event; invitation sections become a nostalgic header |

### Journal View (default)

Extends the existing timeline design. Each day's card expands from a plan description to a photo-rich journal entry:

- **Hero photo**: the single best shot of the day, displayed large at the top of the card
- **Supporting photos**: 2-3 additional photos in a row beneath
- **AI day summary**: replaces the planned description with what actually happened
- **Pull-quotes**: text messages styled as italic callout blocks with sender attribution
- Scrolls as one continuous story from Day 1 through Day 8

Uses the existing timeline CSS — same alternating layout, gold dots, cards, hover effects.

### Scrapbook View

Toggled via a button in the nav. Full masonry grid of every photo received.

Filter chips across the top:
- By day: `Day 1` `Day 2` `Day 3` ...
- By person: `Elizabeth` `Jessica` `Kevin` `Grant`
- By location: `Borough Market` `Selhurst Park` `Botallack` ...

Click a photo for a lightbox: full-size image, caption, who sent it, when and where. Arrow keys / swipe to navigate.

### Implementation

All client-side vanilla JavaScript. On page load:
1. Fetch `manifest.json` from GCS (or embedded in the page post-export)
2. Determine page phase (pre-trip, during, post-trip) based on current date
3. Render the appropriate view

No framework. Consistent with the existing codebase (vanilla JS, Leaflet map, CSS animations).

---

## Export & Keepsake

After the trip, when all photos are processed and the journal is complete, run an export:

**Export script** (run locally or as a Cloud Run job):

1. Download all processed images from GCS
2. Download `manifest.json`
3. Rewrite image URLs in the manifest to relative paths (`assets/large/...`, `assets/thumb/...`)
4. Embed the manifest as a `<script>` block in the HTML
5. Output a self-contained directory:

```
uk-2026-keepsake/
  index.html          # Complete page with embedded manifest
  assets/
    large/            # Full-size photos
    thumb/            # Thumbnails
```

**Result**: A folder that works anywhere — GitHub Pages, any static host, a USB drive, or just opened as a local file in a browser.

**Post-export cleanup**:
- Deactivate trip in Firestore (Twilio number freed for next trip)
- Optionally delete raw images from GCS (processed ones are in the export)
- Keep the export bundle on GCS in a cold storage bucket (pennies/year)

---

## Cost Estimates

### During the trip (~1 week active)

| Service | Estimate | Notes |
|---------|----------|-------|
| Twilio SMS/MMS | $5-15 | ~$0.0079/SMS + $0.01/MMS received, maybe 100-200 messages |
| Cloud Run | $1-3 | Webhook handler + nightly job, minimal compute |
| GCS | < $1 | A few GB of photos |
| Claude API | $5-15 | Vision analysis of ~200 photos + day summaries |
| Google Maps Geocoding | < $1 | Reverse geocoding, ~200 calls |
| **Total** | **~$12-35** | For the entire trip |

### After export

| Service | Estimate | Notes |
|---------|----------|-------|
| Static hosting | $0 | GitHub Pages (free) or GCS static site (~$0.01/month) |
| Cold storage backup | < $0.10/year | GCS Coldline for the raw bundle |

---

## Spinning Up a New Trip

1. Create a new trip config in Firestore: name, dates, destination, location lookup table
2. Register participants: name + phone number
3. Create GCS bucket/prefix for the trip
4. Customize the invitation page (bespoke per trip)
5. Set the trip as "active" so the Twilio number routes to it
6. Update Cloud Scheduler with the trip's timezone for nightly batch runs

The engine code doesn't change. Each trip is just configuration + data.

---

## Open Questions

- **Participant authentication**: Should the system only accept messages from registered phone numbers, or allow anyone to text in? (Recommendation: registered only, to avoid spam and enable sender attribution.)
- **Photo moderation**: Is any content filtering needed, or is this a trusted small group? (Recommendation: skip for now, trusted group.)
- **Multiple active trips**: Support concurrent trips with different participant groups? (Recommendation: not now, but the data model supports it — just route by sender phone.)
- **Video**: Should the system handle short video clips too? (Adds complexity to processing and storage. Recommendation: photos + text only for v1.)

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| SMS/MMS | Twilio |
| Webhook handler | Cloud Run (Python or Node.js) |
| Batch processing | Cloud Run Jobs + Cloud Scheduler |
| AI | Claude API (vision + text) |
| Storage | Google Cloud Storage |
| Database | Firestore |
| Geocoding | Google Maps Geocoding API |
| Frontend | Vanilla HTML/CSS/JS (no framework) |
| Hosting | GitHub Pages (or GCS static site) |
