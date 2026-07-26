# BarberNow — Handoff (2026-07-13, updated 2026-07-15: real ratings)

Project: "תורים זמינים 24/6" (brand: BarberNow) — Uber-style marketplace connecting customers to available barbers in real time, no advance booking.

## Where things live
All web demo files: `BarberApp/`
- `customer.html` — customer app (onboarding, barber list, live search, Google Map, confirmation card, live tracking, chat)
- `barber.html` — barber dashboard (online/offline, accept/reject, arrival/no-show, earnings, live tracking readout, chat)
- `index.html` — old combined file, not maintained
- `landing/index.html` — marketing landing page
- `web-deploy/` — clean folder for Netlify (index.html + customer.html + barber.html) — **stale, needs refresh** before next deploy, doesn't yet have today's features
- `customer-app/`, `barber-app/` — React Native/Expo versions, not yet updated to match the web demo's design or today's features

## Business rules (fixed, don't change without asking)
- Platform commission: 10% of every completed haircut
- No-show fee: full haircut price, charged the same as a completed visit, if customer doesn't arrive within 30 min of barber confirming (changed 2026-07-18 at Harel's request — previously a flat 18 ILS; code in `completeVisit()` in barber.html already paid out no-shows at full price before this doc was updated, so no code change was needed, only this doc)
- Barber has 15 min to confirm an incoming request
- Customer has 30 min to arrive after confirmation

## Backend: Firebase (live)
- Project ID: `barbernow-6fc53`
- Realtime Database URL: `https://barbernow-6fc53-default-rtdb.firebaseio.com`
- Web app registered as "BarberNow Web"; `firebaseConfig` is pasted directly into both `customer.html` and `barber.html` (no env vars / build step — this is a static, no-build project)
- Published security rules:
```json
{
  "rules": {
    "barbers": {
      ".read": true,
      ".write": true,
      "$barberId": {
        ".validate": "newData.hasChildren(['name','price','rating','reviews','distanceKm','address','isOnline','lat','lng','earningsToday'])"
      }
    },
    "requests": {
      ".read": true,
      "$requestId": {
        ".write": true,
        ".validate": "newData.hasChildren(['status','createdAt'])"
      }
    }
  }
}
```
  Note: `.write` had to be granted at the `barbers` parent level (not just `$barberId`), because the app seeds the 3 demo barbers with a single parent-level `.set()` call — child-level write grants don't cascade up to permit that.
- Chat (`requests/$reqId/messages/$msgId`) works with these same rules — no rule change needed, since `.write:true` on `$requestId` cascades down to its children.

## Features built so far
Core demo (already existed before today):
- Real Google Maps embed, search animation, barber list, confirmation flow, Waze nav, reviews
- SOS urgent-request toggle + surcharge (added by a separate design agent working in parallel)
- AI style-match modal with photo upload (design agent)
- "Barber Match" Tinder-style swipe deck (design agent)
- Phone call button, barber `specialties` (design agent)

Added 2026-07-13 (both customer.html and barber.html):
1. **Live tracking** — after a barber accepts, a marker animates across the real map from barber → customer over the arrival window, with a live-updating "X ק"מ · מגיע בעוד Y דק'" pill. Pure client-side interpolation (haversine distance + bearing), no extra Firebase writes. Barber side shows the same km/ETA as text (no map there).
2. **Live chat** — chat modal + unread-message badge on the accepted-request card, both sides. Messages stored under `requests/$reqId/messages`.

Added 2026-07-15 (customer.html only):
3. **Real post-haircut ratings** — the old fixed/fake `REVIEWS` constant is gone. Instead: when a barber marks a visit `completed` in barber.html (customer arrived), the customer is shown a star-rating modal (`renderRatingModal()`) next time customer.html re-renders — 1-5 stars + optional comment, or "דלג" (skip). Submitting pushes a new entry to `barbers/$barberId/reviewsList` and updates that barber's `rating`/`reviews` via a Firebase transaction (`db.ref('barbers/'+barberId).transaction(...)`) so concurrent ratings can't race each other. Either way (submit or skip) the request gets `rated:true` so it's never asked twice. The reviews panel on an accepted-request card now reads live from `b.reviewsList` instead of the old hardcoded object. `seedIfEmpty()` seeds each demo barber's original flavor-text reviews into `reviewsList` (both on first-ever seed and, via a migration check, for barbers that already existed without it) so the reviews panel still has content from day one.
   - No-shows never trigger a rating (only `status==='completed'` does — a no-show means the customer didn't show up, so there was no haircut to rate).
   - No security-rule changes needed for this — the live rules are still the original fully-open ones (see below), and this only adds new fields/children the same way every other write in the app already works.

Verified: no leftover references to the old `REVIEWS` constant; customer.html loads clean.

## Known issues / fixed bugs (context for future debugging)
- **Fixed:** barber.html used to flash then go white on load — was a Firebase permission-denied error from the parent/child write-rule mismatch described above. If a similar flash-then-blank bug reappears, check the browser console for `PERMISSION_DENIED` first.
- **Not fixed yet:** the Google Maps JS API key embedded in `customer.html` is **unrestricted** (no HTTP-referrer restriction) — anyone who reads the page source can use it on Google's dime. Fix in Google Cloud Console → Credentials → restrict key to your domain(s). Low effort, should be done before wider sharing.
- A separate "design agent" edits these same two files independently outside of this thread — if you or another Claude session edits them, re-read the file fresh immediately before writing, don't assume the last-known content is current.
- **2026-07-13/14: real phone/SMS login was built and then fully reverted at Harel's request** (he decided not to go ahead with it for now). If picking this up again later: the design was Firebase Phone Authentication for both apps, `targetBarberIds`/`declinedBy` as maps instead of arrays (needed so Realtime Database rules can check membership), a `barberUidIndex` node, and ownership-checked security rules. None of that is live now — rules are back to the fully-open version above, both files are back to the "simulate as..." / hardcoded-phone state. Ask Harel before re-attempting rather than assuming this is still wanted.

## Roadmap / what's next (priority order)
0. ~~Firebase setup~~ — done
1. **Real auth + tighter security rules** — currently anyone can read/write with valid shape (edit their own earnings, fake an arrival, impersonate any barber — the "simulate as..." menu literally does this for testing). Before real money: add phone/SMS auth (Firebase supports this natively) and rules that check `auth.uid` against the record being written. (Attempted and reverted 2026-07-13/14 — see "Known issues" above. Confirm with Harel before restarting.)
2. **Server-side money logic** — commission (10%) and no-show fee (18 ILS) calculations must move into Firebase Cloud Functions, not run client-side, or anyone can tamper with amounts.
3. **Real payment processor** — Tranzila or Cardcom (Israel-focused) or Stripe, for card tokenization + delayed/auto charge. Requires opening a merchant account (Harel's side), Claude can build the integration code.
4. **Real barber signup flow** — currently 3 hardcoded demo barbers; needs a registration screen (name, location, photo, price) writing into Firebase instead.
5. Restrict the Google Maps API key (see above) — cheap, do it soon.
6. Refresh `web-deploy/` to match current customer.html/barber.html, then re-upload to Netlify.
7. (Lower priority) Bring `customer-app/`/`barber-app/` (React Native) up to parity with the web demo.

## Working notes for whoever picks this up
- No build step for the web demo — it's plain HTML/JS/CSS, open the files directly or serve statically.
- User (Harel) is non-technical — can't code or use a terminal. If continuing in VS Code, expect Harel himself won't be editing code directly; if this handoff is being read by an AI assistant, it should keep doing the implementation work rather than asking Harel to write code.
- Full original tech roadmap with more detail: `TECH_ROADMAP.md` / `TECH_ROADMAP.html` in this same folder.
