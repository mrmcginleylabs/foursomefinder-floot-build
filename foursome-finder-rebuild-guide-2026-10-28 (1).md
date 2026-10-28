# Foursome Finder — Complete Rebuild Guide

This document describes every part of the Foursome Finder app in enough detail that another
person or AI builder can recreate it from scratch. It is written as a specification, not a
summary: it names the pages, the rules, the numbers, the data tables, the server functions and
the visual design.

**How to use it:** hand this whole document to the building system, then work through
Section 11 (Rebuild Playbook) in order. Whenever a rule in this document has a number in it
(points, deadlines, distances), treat that number as a requirement, not an example.

---

## 1. The product

Foursome Finder is one app holding two products that share a membership, a login, a logo and a
look, but almost nothing else:

| Half | What it is | Who it is for | Where it lives in the app |
| --- | --- | --- | --- |
| **Schedule Caddie** | Social golf logistics: when your friends are free, finding a round, organising trips, splitting a foursome, and a directory of courses, pros, venues and partners. | Golfers who want to actually get out and play with people they know. | The `/schedule-caddie` tab and everything under `/home`, `/network`, `/tee-times*`, `/trips*`, `/pro-shop*` |
| **4SF Mini Games** | A club: a daily closest-to-the-pin contest, a Sunday 9-hole stroke-play challenge, head-to-head Match Play, a branded sponsor challenge, plus a point economy, lockers, balls, emblems, trophies and a player directory. | The same golfers, playing short mobile games against each other every day. | The `/games` tab and everything under `/games/*`, `/play`, `/play-9hole`, `/match-play*`, `/the-range`, `/practice-9hole`, `/hole-in-one`, `/points-history` |

There is also a third, small marketing surface: a **B2B page** (`/schedule-caddie-for-clubs`)
that pitches clubs, leagues and courses on running their own version, collects leads, and links to
walkthroughs. A separate **DM LABS** business page (`/dm-labs`) exists for the operator's agency
work and is thematically separate.

### Live app and addresses

- Custom domain: **https://foursomefinder.com** (this is the address used in every email and share link).
- Platform host: the app is published from Base44 (its preview/published host, plus the
  `/functions/<functionName>` routes for inbound webhooks such as Stripe).
- Everything is **mobile-first**: it is designed at a 373 px-wide viewport and up, installable as
  a PWA (manifest + service worker + install prompts), and publishable to iOS/Android.

---

## 2. Technology

- **Frontend:** React 18 + Vite, JavaScript (.jsx), Tailwind CSS, shadcn/ui components from
  `@/components/ui`, lucide-react icons only, react-router-dom routing, TanStack Query for
  data, framer-motion for animation, recharts for charts, moment/date-fns for dates.
  Extra libs in use where they earn their place: html2canvas (share cards), jspdf, qrcode,
  canvas-confetti, react-leaflet (maps), three.js (parked 3D game only), posthog-js (analytics),
  @hello-pangea/dnd (admin drag-to-order), react-quill-new (rich text in admin messaging).
- **Backend:** Base44 platform only. Entities (schemas + row-level security), backend functions
  (`base44/functions/<name>/entry.ts`, Deno), shared server modules (`base44/shared/*.ts`),
  workflows (`base44/workflows/*.jsonc`), email templates (`base44/emails/*.html`),
  authentication, file storage, and integrations (LLM, image generation, email, push).
- **Auth:** platform-managed email/password + Google + Apple + Microsoft/Facebook, OTP email
  verification on sign-up, password reset, and an access-request flow for invite-only apps.
- **Secrets in use:** `ONESIGNAL_APP_ID`, `ONESIGNAL_REST_API_KEY`, `ONESIGNAL_PUSH_ENABLED`,
  `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`.
- **Rules of the platform that shape the code:** every read is server-filtered (no filtering
  whole tables in the browser); a list on screen is one page with "load more"; counts come from
  `count`, totals from `aggregate`; nothing may write to the database from the client except
  through an entity's own permissions; anything that must not be faked runs in a backend function
  with the service role.

---

## 3. How it looks (design system)

The look is a **clubhouse**: deep turf green, fairway white, scorecard typography, gold for
anything that is an achievement.

### Colour tokens (defined in `src/index.css`, mapped in `tailwind.config.js`)

| Token | Value (HSL channels) | Used for |
| --- | --- | --- |
| `--background` | `164 74% 12%` | The app's dark green canvas — the default surface on every page |
| `--foreground` | `96 30% 96%` | Body text on that dark green (near-white) |
| `--card` | `0 0% 100%` | White cards sitting on the green |
| `--card-foreground` | `150 38% 12%` | Text inside white cards |
| `--primary` | `164 72% 15%` | Header bands and primary buttons |
| `--primary-foreground` | `0 0% 100%` | Text on headers/buttons |
| `--muted-foreground` | `96 25% 80%` | Secondary text on the dark surface |
| `--gold` | `43 90% 48%` | Badges, chips, awards, CTAs, active nav, dividers |
| `--gold-foreground` | `164 80% 12%` | Text on gold |
| `--gold-soft` | `45 60% 92%` | Soft gold panels |
| `--destructive` | `0 72% 42%` | Destructive actions only |
| `--flag` | `0 74% 48%` | The red flag in the golf loader |
| `--radius` | `0.75rem` | Card/button corner radius |

A `.dark` block re-declares every token for dark mode. A scoped `.labs-theme` class overrides
the palette (dark teal + yellow) for the DM LABS pages only.

### Typography

- Headings: **Oswald** (`--font-heading`, `font-heading`), tight letter-spacing, often uppercase
  for section dividers ("COMING SOON").
- Body: **Inter** (`--font-body`, `font-body`).
- Numbers: tabular figures everywhere (`font-variant-numeric: tabular-nums`) so scorecards and
  tallies never jitter.

### Surface and contrast rules (a hard requirement)

- The app renders on dark green, so text straight on it must be `text-foreground` or
  `text-muted-foreground`; hairlines are `border-white/…`.
- `surface-light` is used **only** on a genuinely light surface (a `bg-white` card); it flips
  `--foreground` to dark green. Never on a dialog or panel that sits on the dark background.
- Never `border-black/…` or `divide-black/…` on the dark surface.
- `text-destructive` is too dark on green: carry meaning with a label or a "−" sign instead.

### Reusable visual pieces

- A **golf ball into the cup loader** (`.golf-loader` in index.css) with a rolling dimpled ball,
  waving red flag and pulsing logo; respects `prefers-reduced-motion`.
- A **split-flap points board** (`.split-flap-*`): each character flips like an old departure
  board when a point total counts up, with a soft clack sound.
- **`.award-glow`**: gold perimeter glow + gold surface used on any ball or emblem that was
  earned rather than sold.
- **Gold chips** on game cards (`CardPointsChip`) telling the player what the game pays.
- Page headers are a `bg-primary` band with a back arrow and an Oswald title; content is
  `max-w-3xl mx-auto` and cards use `rounded-2xl`.

---

## 4. Navigation and shell

- **Bottom navigation** (fixed, 3 tabs, `src/components/BottomNav.jsx`):
  1. **Schedule Caddie** (`/schedule-caddie`) — highlighted on any caddie route
     (`/home`, `/network`, `/tee-times*`, `/host`, `/my-games`).
  2. **The Hub** (`/pro-shop`) — highlighted on any `/pro-shop/*` route.
  3. **Mini Games** (`/games`) — the centre tab, rendered as a raised golf-ball card
     (`MiniGamesTab`).
- **Signed-in shell** (`src/components/AppLayout.jsx`, a lazy-loaded layout route): top header
  with the club logo, notification bell and account entry; body; bottom nav; plus the
  conditional banners — onboarding tour, network tour, activation checklist, welcome-back,
  pending friend requests, install nudge, push permission prompt, SOS banner, system
  announcement banner and a beta status banner.
- **Public routes** sit outside the shell: `/` (marketing landing), `/caddie-landing`,
  `/games-landing`, `/play` (the daily game, playable signed-out), `/play-9hole`,
  `/bdb-par3`, `/par3-challenge/:code`, `/d/:code`, `/dm-labs*`, `/sponsor`, `/support`,
  `/terms`, `/privacy`, and the four auth pages.
- **Guided tours:** a scripted onboarding tour with gold spotlight rings and bouncing arrows
  (CSS in `index.css` driven by `body.onboarding-active[data-onboarding-step]…`), a separate
  Network tour, and per-area walkthroughs for Schedule Caddie, Trips, and clubs.
- **Route-level code splitting:** every page is a `React.lazy` chunk; the signed-in shell and
  the games home are prefetched for returning members; a tiny spinner is the Suspense fallback.

---

## 5. Roles and identity

| Role | Can do |
| --- | --- |
| **Visitor (not signed in)** | Marketing pages, the daily Par 3 game as a guest, the 9-hole challenge as a guest, the Bad Day Balls challenge as a guest, the Par 3 Duel via a share link, support form. |
| **Member (`user`)** | Everything on both halves, with their own data only: their availability, tee times, trips, wallet, scores, matches, messages. |
| **Admin (`admin`)** | Everything a member can, plus the whole `/admin` dashboard, the games config pages, member management, messaging, exports, and the ability to act on any record. |

Sign-up is deliberately frictionless: **name + zip code** (plus optional home course, social
handles). Everything is stored on the User record (built-in fields: id, created_date, full_name,
email, role) or on the member's `MiniGameSettings` record for the games side.

**Guests** on the games side are a first-class concept: a device-local anonymous id
(`anonymous_id` in localStorage) lets someone play without an account, with their score kept on
the leaderboard and their points previewed; signing up migrates the guest score to the new
account and banks the previewed points.

---

## 6. Time rules (get these right or every game day breaks)

- **Display and reminders** use Eastern time (`America/New_York`).
- **The game day is Pacific** (`America/Los_Angeles`): every daily game, leaderboard, winner
  record and reset rolls over at **12:00 AM Pacific = 3:00 AM Eastern**. All of them derive the
  day key from the same helper (`getEtDateString`, returning `YYYY-MM-DD` in Pacific) so a score
  can never land on the wrong day.
- Sunday is the one day the daily Par 3 contest is closed and the 9-Hole Challenge is open; both
  read Sunday off the Pacific calendar.
- Streaks, "today's points" tallies, weekly course rotation and the Bad Day Balls schedule all
  use that same Pacific day key.


---

# 7. Schedule Caddie — the logistics half

## 7.1 What it does

A member tells the club when they can play; the app works out who is free at the same time, finds
or creates a round, fills the foursome, and later handles the trip, the chat, the reminders and
the money. Around that it adds a friends network with private tiers, a partner directory, and a
public open-rounds board.

## 7.2 Onboarding and the member profile

1. **Register** (email/password or Google, then OTP verification) or **access request** if the
   club is invite-only. Sign-up asks only for **real name** and **zip code** — no handicap, no
   preferences, nothing that would make someone stop.
2. After the first login the member lands on the availability dashboard with a **Getting Started**
   tour, an **activation checklist** (set availability → find a friend → create or join a round →
   invite someone) and an install/push nudge. These are dismissible and remembered.
3. **Profile** (`/profile`): name, zip, home course, Instagram / X / TikTok handles, photo
   (uploaded **privately**, shown through a signed URL), search radius, notification preferences
   and push opt-in. **Calendar sync** can publish their rounds to an external calendar.
4. **App settings** (`/settings`): account, alerts, device/push, delete account
   (`/delete-account` → `AccountDeletionRequest`), sign out.

## 7.3 Availability — the core of the product

Members post **availability blocks**: a date, a time window, an optional list of courses or a
radius, and a visibility choice.

- `AvailabilityBlock` = owner, date, start/end window, course preference, notes, visibility.
- `/home` (Availability Dashboard) is the member's own week at a glance: their blocks, who is
  free when they are (`MatchingFreeTime`), suggestions, next round ticker, sponsor banner,
  active chats, pending pings and friend requests.
- **Who is free** (`/network` → Who's Free? / `AvailableFriends`): pick a day and a window and
  the server returns friends whose blocks overlap.
- **Matching** is always computed **server-side** (`match-on-block`, `base44/shared/matching.ts`)
  — overlap maths, radius/zip distance (haversine over a zip→lat/lng lookup, `base44/shared/geo.ts`)
  and visibility filtering never happen in the browser.
- **Friends' calendars** (`FriendAvailabilityCalendar`, `PublicAvailabilityCalendar`,
  `PublicFreeTimeTutorial`) show a friend's free windows in a week grid, with a simulator tag
  (`SimTag`) when the data is demo data.
- Past blocks are cleaned nightly by the **Clean Up Past Availability** workflow.

## 7.4 Friends, tiers and privacy

- `Friendship` links two members after a request/accept.
- `FriendTier` is **private to the owner**: each member sorts their own friends into
  **A-Team / B-Team / Unassigned** and the friend is *never* told which tier they are in.
- Tier is what drives round visibility (see 7.5). Because tier records are locked to their owner
  by row-level security, the visibility decision is made by a **service-role function**
  (`visible-tee-times`) that reads each host's private tier on the viewer's behalf and returns
  only the filtered list — the labels never reach the client.
- `/network` holds the whole social surface: **Find Golfers**, **Manage Friends** (tiers,
  nickname, remove), **Messages** (direct messages, `/messages/:userId`), **Invite Friends**
  (share link, QR, referral points), **Friends of friends** (mutual friends dialog),
  **Activity feed**, **Player suggestions**, and a **Network tour**.
- **Demo golfers** (admin) seed a set of believable members so a new club is never empty; they are
  clearly marked and can be removed in bulk.

## 7.5 Tee times — the round engine

A member creates a tee time (`/tee-times/new`) with: course (or "any"), date, time, number of
open slots, who may see it, whether they want priority friends first, payment details, and notes.

**Visibility is time-based and tier-based** (implemented in `visible-tee-times`):

| Round setting | Who sees it, and when |
| --- | --- |
| Priority friends | While the VIP window is open, **only** the named priority friends who have not declined. When the window ends (or everyone declines) it falls back to the tier rule below. |
| "All connections" | Every connection of the host. |
| Default (staggered) | First **24 hours**: only the host's **A-Team**. Hours 24–48: A-Team **and** B-Team. After **48 hours** (or as soon as slots run out): everyone. A round with no open slots is never shown. |

Everything else about a round:

- **Invites** (`send-tee-invite`, `TeeTimeInvite`, `TeeTimeInviteManager`) go to named friends,
  with `accept-invites`/decline, nudges and "Let's Play" one-taps.
- **Broadcasts** (`tee-time-broadcast`, `TeeTimeBroadcast`, `BroadcastDialog`) push an open
  round out to a wider group; the sender sees who has seen/answered.
- **Open Tee Times** (`/tee-times`) is the club-wide board of rounds this member may join;
  `/tee-times-hub` is the hub with the open board, the member's own rounds, and the round
  checklist; `/host` (Host Dashboard) tracks each round the member is hosting and who is in/out;
  `/my-games` lists rounds the member has joined.
- **Round detail** (`/tee-times/:id`) shows the foursome, course, conditions (weather button),
  chat (`/chat/:teeTimeId`), player suggestions (`suggest-player`, `PlayerSuggestion`),
  the "Let's Play" prompt for friends who are free, drop-out, and a **Share** card
  (`CardShareButton`, `src/sidecar/shareCard.js`) plus a **GolfNow** booking button where a
  course can be booked online.
- `Tee Time Reminders` workflow sends day-before and day-of reminders; `send-tee-reminders`
  also handles the nudge. `Weather Check` + `check-weather` fill the cache entity
  (`WeatherCache`-style record) that the WeatherButton reads so forecasts are not re-fetched.
- **Play requests** (`PlayRequest`, `send/accept/decline-play-request`) are "can I get in?" asks
  on a specific round; **pings** (`Ping`, `send-ping`, `respond-ping`) are lightweight
  "are you free right now?" nudges; **SOS** (`SosAlert`, `send-sos`, `claim-sos`) is "I have a
  slot that needs filling in the next few hours" — visible to the member's network, first to
  claim joins.

## 7.6 Trips

A trip is a multi-day, multi-foursome event with its own group chat and (optionally) payments.

- **Hub** (`/trips`): upcoming trips the member is on, trips they host, invites.
- **Create** (`/trips/new`): name, dates, destination(s) (`TripDestination`), lodging notes,
  group size, and whether it is paid. `create-trip-checkout` opens a Stripe Checkout session;
  `stripe-webhook` records payment; `confirm-trip-payment` reconciles;
  `trip-roster-phones` gives the host the roster with contact numbers.
- **Trip detail** (`/trips/:id`): roster (`TripRosterEntry`), destination board, an
  **availability heatmap** so the group can find the days that work, **course voting**
  (`CourseVote`, `CourseProposal`), packing/notes, and per-person payment state.
  `visible-trips` enforces who may see a trip.
- **Trip chat** (`/trips/:id/chat`, `TripChat`) is a separate realtime-feeling thread from
  round chat.
- A scripted **Trip tutorial** dialog explains it the first time.

## 7.7 The Hub and the partner directory

**The Hub** (`/pro-shop`) is the club's directory and the second bottom-nav tab. It holds
cards into:

- **Instructors** (`/pro-shop/instructors`, profile `/:id`) — `Instructor` records: name, photo,
  bio, specialty, rates, contact, optional social icon, and an **instructor flag** on the member.
- **Sponsors** (`/pro-shop/sponsors`, profile `/:id`) — `Sponsor` records with logo, offer,
  links, and a sponsor banner slot on the dashboard.
- **Partner courses** (`/pro-shop/courses`) — course records with booking links, plus
  **course proposals** (`CourseProposal`, `course-proposals`) so members can suggest a course.
- **Golf partners** (`/pro-shop/golfers`, profile `/:id`) — `GolfPartner` records (a playing
  partner / ambassador with photo, bio, handles) with a flag badge on the member.
- **Venues** (`/pro-shop/venues`, profile `/:id`) — `Venue` records (clubhouse, bar, sim
  studio) with logo, address, hours and map.
- Partner profiles are also reached from badges next to a member's name
  (`InstructorBadge`, `SponsorBadge`, `GolfPartnerBadge`, `PracticeBuddyBadge`).

## 7.8 Schedule Caddie for clubs (B2B)

- `/schedule-caddie` is the **member hub**: availability, rounds, friends, trips and the way into
  the club pitch.
- `/schedule-caddie-for-clubs` is the **operator pitch**: a hero, a 2x2 grid of feature bubbles
  (white surface, dark green text), "what we build" sections, an audience/benefits block, two live
  examples (a matchup calendar, a free-time calendar), a **build request form**
  (`BuildRequestForm` → `ScheduleCaddyLead` records, read only by admins) and a link to
  `/schedule-caddie/how-it-works` — a hub of every walkthrough (`Tutorials`,
  `ScheduleCaddieTours`, `ScheduleCaddieTourSteps`).
- Deliberately **no pricing and no payment handling** anywhere on this page: it is a lead
  generator only. No financial or transaction language appears on it.

## 7.9 Notifications, email and messaging (caddie side)

- The **bell** (`NotificationBell` + `Notification` entity) is the in-app record of everything:
  `join_request`, `invite_accepted`, `chat_message`, `direct_message`, `tee_nudge`,
  `friend_request`, `sos`, `player_suggestion`, `tee_updated`, `lets_play`, `play_request`,
  `ping`, `trip`, `referral`, `report`, `warning`, `broadcast`, `achievement`, `match`.
  Notification reads are locked to the recipient by row-level security.
- `notify` (and the shared `notify.ts`) writes the bell entry, fires **OneSignal** push
  (`send-push`, gated by `ONESIGNAL_PUSH_ENABLED`), and can send an **email** as a fallback. All
  of it is opt-in per member: push pref, alert prefs and match reminders are switches.
- **Email templates** (`base44/emails/`) are designed MJML/HTML files: welcome, invite,
  referral bonus, prize awarded, BDB promo code, contact/feedback acknowledgements. Marketing
  sends carry an unsubscribe link; transactional sends (codes, resets, awards) do not.
- `send-broadcast` + `SystemAnnouncement` give admins a club-wide banner/notification; the
  admin **Game messaging** page composes messages to a filtered audience.
- **Reports** (`Report`, `resolve-report`, `ReportDialog`, admin `ReportReviewDialog`) let any
  member flag content; admins resolve it. A **name filter** (`src/lib/nameFilter.js`) screens
  chosen display names.

## 7.10 Admin (caddie side)

`/admin` is a long, sectioned dashboard (admin-only route, admin-only entity rules):

- Membership: invite users, roles, member list with activity, edit member dialog, member IDs,
  join-request handling (`manage-join-request`, `JoinRequest`), first-off-tee assignment,
  onboarding reset.
- Content: courses (`/admin/courses`), partners (instructors/sponsors/venues/golf partners),
  demo golfers, promo items, collections, achievements/prizes.
- Ops: feedback inbox (`AppFeedback`, `submit-app-feedback`), reports, engagement report,
  match monitor, broadcast composer, points leaderboard, exclusive emblems, **data backup**
  (JSON export of every table via `export-my-data`) and **image backup** (ZIP of every uploaded
  image plus the app's artwork) and **this rebuild guide** export.
- Previews: signup preview, finish-screen preview, games landing preview — phone-frame renders of
  screens that are otherwise hard to inspect.

## 7.11 Entities used by this half

`User` (built-in, extended with golf profile fields), `Course`, `CourseProposal`, `Venue`,
`Instructor`, `Sponsor`, `GolfPartner`, `AvailabilityBlock`, `Friendship`, `FriendTier`,
`TeeTime`, `TeeTimeInvite`, `TeeTimeBroadcast`, `PlayRequest`, `Ping`, `SosAlert`, `Chat`,
`DirectMessage`, `Notification`, `Trip`, `TripDestination`, `TripRosterEntry`, `TripAvailability`,
`TripChat`, `CourseVote`, `JoinRequest`, `AppInvite`, `Referral`, `Report`, `AppFeedback`,
`SystemAnnouncement`, `ScheduleCaddyLead`, `ConsultationInquiry`, `AccountDeletionRequest`,
`Activity`, `PlayerSuggestion`, `GolfPartner` (partner directory), `InstagramVideo`.

## 7.12 Server functions behind this half (by job)

- Visibility and matching: `visible-tee-times`, `match-on-block`, `public-availability`,
  `search-tee-times`, `visible-trips`.
- Rounds: `send-tee-invite`, `accept-invites`, `decline-priority-invite`,
  `send-priority-invites`, `tee-time-broadcast`, `delete-tee-time`, `drop-out`,
  `assign-first-off-tee`, `send-tee-reminders`.
- Social: `send-play-request`, `accept-play-request`, `decline-play-request`, `send-ping`,
  `respond-ping`, `send-sos`, `claim-sos`, `lets-play`, `suggest-player`,
  `resolve-players`, `get-mutual-friends`, `get-friends-friends`, `network`,
  `log-join`, `archive-expired-chats`.
- Trips: `create-trip-checkout`, `confirm-trip-payment`, `stripe-webhook`,
  `trip-roster-phones`.
- Lifecycle: `onboard-user`, `onboarding-setup`, `onboarding-cleanup`, `reset-onboarding`,
  `complete-foursome-finder-profile` UI, `welcome-new-member`, `send-onboarding-welcome`,
  `send-pioneer-email`, `relocate-user`, `clean-past-availability`,
  `regenerate-practice-availability`, `setup-practice-buddies`, `add-practice-buddies`.
- Weather, maps and third parties: `check-weather`, `nearby-signups`, `fetch-instagram-photo`,
  `sync-calendar`, `get-referral-link`, `send-app-invite`.
- Admin and support: `export-my-data`, `manage-join-request`, `dismiss-join-request`,
  `resolve-report`, `get-member-activity`, `get-game-activity`, `log-round-activity`,
  `submit-app-feedback`, `send-support-request`, `request-account-deletion`,
  `send-broadcast`, `demo-golfers`, `remove-demo-golfers`.

## 7.13 Workflows for this half

`Clean Up Past Availability` (nightly housekeeping), `Tee Time Reminders`, `Weather Check`,
`Regenerate Practice Availability`, `Archive Expired Chats`, `Welcome New Member`.


---

# 8. 4SF Mini Games — the club half

## 8.1 The idea

The games half turns the same membership into a club with a daily habit: a short game every day
that produces a leaderboard everybody can see, a point economy that is earned only by playing,
and a locker of balls and emblems that members wear in the games. It shares the login and the
look with Schedule Caddie but keeps its own profile, its own settings, its own messages and its
own vocabulary (member, club, locker, Pro Shop, 19th Hole, the Range).

**Guests play too.** Every game can be played signed-out with a device-local anonymous id; their
score goes on the day's leaderboard, they see the points they *would* have earned, and they are
invited to sign up to bank them.

## 8.2 The Pro Shop point economy

Points are the club's currency. They are **only ever earned by playing** and are spent on balls
and emblems. All crediting and charging happens **server-side**, and **every award carries a
unique `award_key`** so a retry, a re-read or a replayed event can never pay twice
(`ProShopAward` + `ProShopWallet`, `base44/shared/proShop.ts`).

| Action | Points |
| --- | --- |
| One Par 3 shot: hole-in-one | **20** |
| One Par 3 shot: within 5 ft | **10** |
| One Par 3 shot: within 20 ft | **5** |
| One Par 3 shot: on the green | **2** |
| Finishing the daily Par 3 round (bonus, once a day) | **5** |
| Par 3 daily leaderboard places (paid after the day closes) | 1st **50**, 2nd **40**, 3rd **30**, 4th **20**, 5th **10**, 6th–10th **5** (members only) |
| One Range shot: hole-in-one / ≤5 ft / ≤10 ft / ≤20 ft / on the green | **5 / 4 / 3 / 2 / 1** (first 20 scoring balls per day) |
| A completed 9-hole round (either 9-hole game) | **20** completion, + **20** per hole-in-one, + **3** per birdie, + **2** per par (best category on a hole wins; a bogey pays nothing) |
| Match Play: accept-and-play as challenger | **5** |
| Match Play: each hole won | **2** |
| Match Play: finishing the match | **20** |
| Match Play: winning the match | **20** |
| A friend who joins the daily game through your link and plays | **100** (+1 bonus shot) |
| A friend who signs up and becomes a member | **1000** |
| Starter balance for a new member's wallet | **20** |
| Purchases | negative (a charge, never gated by the launch date) |

- The economy has a **go-live date** (`POINTS_START_DATE = 2026-10-21`): nothing dated before it
  pays, so no back-filling is ever needed.
- A member has exactly **one wallet** (the server folds duplicates into the oldest and keeps the
  fullest balance); the wallet holds balance, total earned, owned ball ids, owned emblem ids and
  the equipped ball.
- **Guests never earn or spend points** — every economy helper is a no-op without a member id, and
  a guest sees a preview of the points instead.
- Members can read **their own** award ledger (`/points-history`), which shows each day's board
  of lines (Shot 3 — Within 5 ft · Round complete …) and the day's total.

## 8.3 The daily Par 3 Contest (`/play`) — the flagship

- **Open Monday to Saturday**, closed on Sunday (the 9-Hole Challenge owns Sunday). The games hub
  shows it greyed out with a MON–SAT chip on Sunday.
- **One session per member per day:** **5 shots** at a single par 3 that rotates daily by ET
  weekday (six hand-designed holes, one per weekday, each with its own theme: default, desert,
  mountain, ocean, links). The hole is defined in `base44/entities/Par3Hole.jsonc` and built in
  the admin **hole builder** (canvas editor: tee position, green X/radius or a custom polygon,
  yardage, theme, wind, and obstacles of type tree / bunker / water / fairway_cut / rock).
- **Daily pin placement and wind** come from `Par3DailyConditions` (pin as a fraction of the green
  radius on both axes, wind speed 0–20 mph, wind angle in degrees), auto-generated for each day and
  editable by an admin on `/admin/par3-conditions`.
- **Play:** drag back and release to set shot direction and power, with wind, obstacles, bounce and
  roll simulated on a 2D canvas; the ball's resting distance to the pin is the score. The best
  (closest) distance of the five shots is what counts; 0 ft is a hole-in-one.
- **Scoring:** the leaderboard for the day ranks by **best distance ASC**, then **more
  hole-in-ones**, then **earliest completion**. Only *completed* sessions (all 5 shots) rank.
- **Replay:** one replay per day, which re-opens the 5 shots and resets the earliest-completion
  tiebreak to the new time.
- **Referral bonus shots:** inviting a friend who actually plays the daily game earns an extra
  shot (one per distinct new player); the session is re-scored from the best five of all attempts,
  and the bonus shot is offered with a referral card + modal.
- **Shot map:** every shot's landing point is stored, so the finish screen and the leaderboard can
  draw all five shots on the hole, colour-coded by lie (hio / green / bunker / water / rough), and
  can be shared as a Wordle-style emoji card or a generated scorecard image.
- **Winner records:** after a round is committed the server works out the day's best score across
  all players and upserts a `MiniGameWinner` for that day (one per game type per day), which is
  what feeds the **19th Hole**, the **Trophy Room** and the **Hall of Fame**.
- **End-of-round surfaces:** result screen with the distance, the points ticker (split-flap
  animation), the shot map, "Best Shot Plaque" share card, keep-playing suggestions, the invite
  card (point economy explainer), and — when the sponsor challenge is live — the Bad Day Balls
  promo banner.
- **Admin tools for this game:** hole builder, daily conditions, hole preview, test play modal,
  shot statistics (`/admin/par3-shot-stats`, `get-par3-shot-stats`: bands, lies, distances),
  mini-game history, and a finish-screen preview.

## 8.4 The Sunday 9-Hole Challenge (`/play-9hole`)

- **Open on Sunday only**, from 12:00 AM Pacific. On Sunday the hub promotes its card to the top
  of the grid; the rest of the week it is greyed out with a SUNDAYS chip.
- The club's 9-hole courses live in `NineHoleCourse` (nine holes, each with par, yardage, shape,
  dogleg, green size/position, wind and obstacles — built in the same admin canvas editor at
  `/admin/hole-builder` and `/admin/courses`). A **weekly rotation** activates the next course
  (the **Sunday Course Rotation** workflow + `activate-next-course` + `get-released-nine-hole-courses`),
  so each Sunday has a new course and past courses stay replayable in practice.
- **Play:** nine holes of stroke play with the same shot mechanic (drag to shoot, wind, obstacles,
  bounce/roll, putting on the green). Hole by hole, with the scorecard updating live.
- **Ranking** (`compareScores` in `base44/shared/nineHoleGame.ts`): total strokes ASC, then
  **more aces**, then **more holes under par**, then **fewer bogeys**, then **earliest
  completion**. The board is never trimmed — everyone who finished that Sunday is on it.
- The board shows each round as **score to par** (e.g. −8) which is what the 19th Hole rows and the
  champion plaques use, plus per-hole scores on click.
- **Guests** may play the Sunday round too; their score sits on the same board and their points are
  previewed (Round complete +20, ace +20, birdie +3, par +2).
- Finishing posts a `MiniGameScore`/`NineHoleScore` record, pays the round points, updates the
  winner record for the day, and unlocks titles (Sunday Challenger, Bogey-Free, Going Low …).

## 8.5 9-Hole Practice (`/practice-9hole`) and Practice Buddies

- Replays **any released** Sunday course, chosen by the member, with no leaderboard effect. It pays
  exactly the same 9-hole point rates as the Sunday round, **once per member per day**.
- **Practice buddies** are simulated club members who post practice rounds
  (`Base44 shared/practiceBuddies.ts`, `setup-practice-buddies`, `regenerate-practice-availability`
  + its workflow) so practice never feels empty; they also appear as availability on the caddie
  side. The member can add or remove them.
- A practice history view (`NineHolePracticeHistory`) and scorecard modal let a member compare
  rounds; the title **Range Rat** (50 practice rounds) and **Greens Seeker** (par or better on all
  nine) come from here.

## 8.6 The Range (`/the-range`)

- A members-only practice mode: any hole with **random wind and pin**, and **scores never count**
  toward any leaderboard.
- Rings are drawn around the pin with their point values (5 / 4 / 3 / 2 / 1) — the same numbers the
  server pays (`RANGE_RINGS` in `src/lib/rangePoints.js` mirrors
  `rangeShotPoints` in `base44/shared/proShop.ts`).
- The first **20 scoring balls** each ET day pay; further balls are free practice
  (`RANGE_DAILY_SCORING_BALLS`, enforced in `award-range-points`).
- Guests get an access prompt (`RangeAccessPrompt`) rather than entry.

## 8.7 Hole-in-One Contest (`/hole-in-one`)

- A "keep going until you sink one" mode on the daily hole: the player takes shot after shot and
  the session ends when a hole-in-one is made.
- The leaderboard ranks by **fewest shots to the ace** (ties share a rank), and only completed
  sessions rank. A member's own in-progress session comes back so the round can be resumed.
- Currently surfaced in the hub's **Coming Soon** section (behind an access code) while it is
  being tuned, exactly like the Par 3 Duel.

## 8.8 Par 3 Duel — async 1-on-1 (`/par3-challenge`, `/par3-challenge/:code`, `/d/:code`)

- **Share-link 1-on-1 duel.** The challenger creates a match (optionally password-gated,
  `Par3PasswordGate`), gets a **6-character share code** (and a friendly `/d/:code` short link),
  and the opponent opens it from a text message — no account needed.
- **Rules** (`base44/shared/par3FriendChallenge.ts`): **3 shots per turn**, **9 shots each** in
  total, alternate turns; each turn has a **24-hour deadline**, a reminder 1 hour after the turn
  starts and a warning 1 hour before the deadline (workflow **Par 3 Friend Challenge Reminders**).
  A missed deadline forfeits the match to the other player.
- **Match conditions** are generated per match: wind **5–20 mph** (never calm) and a pin never
  within 25° of straight up/down, so both players face the identical hole.
- **Winner:** both players' nine distances are sorted closest-first and compared position by
  position — the first differing position decides; if the best shot decides it, it is
  "best distance", otherwise it is a "tiebreak" win; all nine equal is a tie.
- **Claiming:** the opponent spot is open until claimed, so the second player can enter their name
  before their turn, whether they are a guest or a member.
- Each player can also open the match with a **shared screen** mode (pass the phone) using
  `Par3SharedScreen`.

## 8.9 Match Play (`/match-play`, `/match-play/:id`)

An asynchronous 9-hole head-to-head match between two members, with a belt on the line.

- **Create** (`create-match-play`): challenge any member (respecting their challenge visibility
  setting). The invite is live for **24 hours**; missing it expires the match.
- **Turn clock:** once both players have played hole 1, each shot has a **3-hour deadline**
  (the very first shot of the match has no clock). A missed deadline **forfeits** the match to the
  other player. A warning notification fires **2 hours in** (1 hour before the deadline), driven by
  the **Match Play Turn Warnings** workflow and `check-match-warnings`; a lazy check on read
  (`applyLazyTimeout`) means a forgotten match still ends correctly without a cron tick.
- **Breaks:** either player may pause for **12 hours** (`pause-match-play`); while paused the clock
  is frozen — nothing times out and nobody forfeits. Pausing again resets the 12 hours. The other
  player can request a resume (`request-match-resume`, at most one nudge per 30 minutes); the
  **Match Play Pause Resume** workflow resumes matches automatically when the break ends
  (`resume-paused-match-play`), and playing your shot ends the break.
- **Play:** alternating hole by hole; on even holes the challenger leads off, on odd holes the
  opponent does (a double-shot rhythm: second player on one hole leads the next). Each hole is
  won by fewer strokes.
- **Ending early:** a player ahead by more than the holes remaining **clinches** it
  (`checkClinch`); all nine holes can be played out; a tie after nine goes to **sudden death by
  closest approach on hole 9** (`resolveSuddenDeath`, revealed with an animation). Players may
  also **concede** (`concede-match-play`) or **cancel** (`cancel-match-play`) while pending.
- **Course snapshot:** the course id, name, theme and the nine hole configs are copied onto the
  match when it is created, so an in-progress match is unaffected by a course rotation.
- **Points and honours:** accepting-and-playing pays the challenger 5, each hole won pays 2,
  finishing pays 20 and winning pays another 20; winning also hands over the club's
  `match_play_win` achievement prizes and unlocks the **Match Play Champion** title.
- **Rivalry and the belt:** every completed match upserts a `Rivalry` row for the pair (wins each
  way, ties, total matches, last date, and who holds the belt — the belt moves to the winner of the
  most recent match). `/match-play` shows your active matches, invitations, past results, rivals
  and the belt holder (`ActiveMatchIndicator`, `MemberChallengeRow`, `MatchPlayMonitor` for admins).
- **Post-match:** a result banner (holes won, the scorecard modal, the sudden-death reveal) and the
  **MatchPlayPointsPanel** which shows the points the match just paid.

## 8.10 The Bad Day Balls Challenge (`/bdb-par3`) — sponsor game

A branded, sponsor-run daily challenge that lives beside the club's own games (it never replaces
the daily Par 3 contest).

- **Who may play is a three-state control** on `/admin/bdb-config`: **on for everyone**, **on for
  guests only**, or **off**. On "guests only" a member sees the tile greyed out at the bottom of the
  games hub with no time-left banner, and the server refuses the challenge on load and on every
  shot, so a stale page or a direct link cannot get around it. Off leaves the tile and the guest
  links visible as unavailable. `is_live` is written in step with the control so readers that only
  ask "is it live" (the promo banner, the guest invitation) still behave, and a record written
  before the control existed falls back to "all" when `is_live` is true, "off" when it is false.
- **5 shots a day**, one completed session per player per game day. Yardage is fixed by the admin
  config; **wind and pin randomise daily but deterministically from the day key**, so every player
  faces identical conditions that day.
- **Prize tiers** (from the best shot of the five): **hole-in-one** → a free sleeve of balls (or
  15% off instead), **≤10 ft** → 10% off, **≤20 ft** → 5% off, worse → no prize. The ace is
  deliberately far harder here than in the club's own game (`BDB_HIO_DIFFICULTY` shrinks the
  capture radius and the velocity ceiling).
- **Delivery:** members who finish get their code by email from the `BdbPromoCode` template with a
  pre-applied redeem link; guests see the prize without the code and are prompted to sign up.
  Codes are 24-hour codes held in the admin config (`code_5`, `code_10`, `code_15` and their
  matching `link_*`).
- **Scheduling:** an admin can schedule which days it opens (a list of Pacific game days); an empty
  list keeps the default "every day except Sunday". Players are never shown the schedule.
- **Branding switches:** the admin config carries the BDB logo and both ball marks — they can also
  be applied to the club's own Par 3 canvas (corner logo + ball logos) and a BDB promo banner can
  appear on the daily contest's finish screen.
- **Admin page** `/admin/bdb-config` holds intro copy, hole geometry, branding, codes, links, the
  schedule, QR code for the challenge link, and a **test play** modal; a test-email function sends
  the promo template to the admin.

## 8.11 Pro Shop, Locker Room and the clubhouse

- **The Clubhouse** (`/clubhouse`, reached from the games hub) is the front door: a card into the
  **Pro Shop** and a card into the **19th Hole**, plus the **Player's Lounge** and the
  **Locker Room**.
- **The Pro Shop** (`/games/pro-shop`) is where points are spent, all server-priced:
  - **Ball skins** — the launch catalog with prices: Bad Day Balls (0), Practice (0), Pink Pearl,
    Sunset Orange, Electric Blue, Crimson Red (50 each), Woodland Camo, Galaxy Swirl, 4SF Striped
    (100 each), Lava Rock, Stars & Stripes (200 each), Gold Sparkle (250), and the **Halloween
    Collection** (Pumpkin 20, Jack-o'-Lantern, Zombie, Mummy, Skull 50 each) also sold as a
    **bundle for 100** — cheaper than buying the five separately. Buying a bundle owns every ball
    in it in one charge.
  - **Game icon emblems** — the free letter badge plus bought symbols: Golf Hole, Golfer, Target,
    Clover (20 each), Golf Cart, Cheers, Jolly Roger, USA Flag, Dollar Sign (50 each). The icon's
    colour is free and chosen by the member.
  - **Exclusive emblems** — Founding Member, Beta Tester, Early Adopter: never sold, never listed,
    granted by hand, visible only in the locker of the member who holds one.
  - **Collections** — named groups of items shown as their own shop section; a collection flagged
    as a **Glass Case** displays its items with no price and no buy button, with a message
    explaining how they will be won.
  - **Promo-code items** — balls and emblems claimed with a code (`redeem-promo-code`,
    `ProShopPromoItem`); codes are matched server-side and never sent to the app; the claim also
    works for giveaways shown in the shop.
  - **Achievement prizes** — the club sets a prize per mark (`AchievementReward`): winning the
    daily Par 3 contest, a top-3 finish, winning the Sunday 9-hole, an ace, winning a Match Play
    match, or a 6-day streak. When the mark is reached the prize lands as a **claim** in the
    member's Pro Shop (gold-ringed, waiting to be claimed rather than straight in the locker) and
    the member is told through the bell and a dedicated email.
- **Buying**: `buy-ball-skin` and `buy-game-icon-emblem` validate the price server-side, refuse a
  second purchase of the same item, charge the wallet and write a negative `purchase` award keyed
  to the item — so a double-tap can never double-charge. `equip-ball-skin` sets the worn ball.
- **The Player's Locker** (`/games/player-locker`) shows everything the member owns — balls on
  racks, emblems, titles, trophies, earned prizes — with quick jumps to the shop and the games.
- **Your ball flies in the games:** the equipped ball's artwork is drawn on the canvas ball
  (with the club or sponsor marks where configured), and each score snapshots the ball it was
  played with so past leaderboards keep the right ball.
- **The Locker Room / your game profile** (`GameProfileDialog`) is the card other members see:
  games name, icon, equipped ball, titles, records, trophies and the Frenemies control.

## 8.12 Titles, badges, trophies and the club boards

- **Titles** (`src/lib/playerTitles.js`, `set-player-title`, `MiniGameProgress.unlocked_titles`)
  are unlocked by playing, in four tiers (bronze / silver / gold / diamond): Rookie, Regular (7
  rounds), Veteran (25 rounds), Sharpshooter (inside 3 ft), Sniper (inside 1 ft), Ace (first ace),
  Triple Ace, Ace Master (10 aces), Champion (win a daily), Dynasty (5 daily wins), Daily Devotee
  (6-day streak), Ironman (30-day streak), Sunday Challenger, Bogey-Free, Going Low (birdie every
  hole), Greens Seeker, Range Rat (50 practice rounds), Tour Pro (every course), Match Play
  Champion. A member wears one at a time; the Trophy Room shows the cabinet.
- **Badges** (`MemberBadge`, `GenrtBadge`) mark staff and special members next to their name.
- **The 19th Hole** (`/games/19th-hole`, day view `/games/19th-hole/:date`) is the clubhouse
  board: each day's champion with the date above their emblem, the winning score, and the way into
  that day's full standings. Past days are always readable (`get-nineteenth-hole`,
  `get-past-leaderboard`).
- **The Hall of Fame / Trophy Room** (`/games/trophy-room`) unifies both daily games: win totals,
  aces, the all-time board, plaques, the member's own trophy cabinet and their places in past
  rounds (`get-trophy-room`, `get-hall-of-fame`, `get-hole-in-one-club`,
  `get-title-rarity` for how rare a title is).
- **The Player's Lounge** (`/games/players-lounge`) is the club directory: everyone playing the
  games, their icon, title, recent activity and who is around now, with private member-to-member
  messages (`/games/lounge/messages`, `LoungeMessages`) that stay apart from the Schedule Caddie
  messages. **Frenemies** are an opt-in list each member keeps for friendly rivalry — add or remove
  from a card in the lounge or a profile (the list is private and de-duplicated server-side).
- **My Games** (`/my-games`) collects everything a member has going: their active matches, friend
  challenges, rounds, and results.

## 8.13 4SF Player Settings (`/games/player-settings`)

One page (plus the Locker shortcut) controls how a member appears and is reached in the club:

- **Games name** (used on every board instead of their account name) and **real name** with a
  "share my real name with members" switch (the club always sees it; members only if on).
- **Sound and music** on/off, **match reminders** on/off (switching it off stops reminders only —
  the match and its deadlines still run).
- **Challenge visibility:** everyone / favourites (currently behaves like everyone) / do-not-disturb
  (no new challenges).
- **Activity status:** whether the lounge shows how often and how recently they play.
- **Icon designer:** pick the letter badge or an owned emblem, choose the colour, preview it live.
- **Privacy note:** this page is deliberately separate from the Schedule Caddie profile.

## 8.14 Guests, invites and referrals on the games side

- A guest is identified by an `anonymous_id` kept in the browser; all game writes accept it and the
  boards carry their rows.
- A guest's finish screen previews the points they would earn and offers signup; `migrate-guest-score`
  moves a guest score onto a new account so nothing is lost.
- Invitations create `Referral` / `ReferralBonus` records: a friend who plays the daily game via
  the member's link pays 100 points + a bonus shot; a friend who signs up pays 1000
  (`base44/shared/referral.ts`, `get-referral-link`, `ReferralBonusEarned` email).
- Members can invite by link, QR code and share card; every share surface exists
  (`ShareScorecard`, plaque cards, Wordle-style emoji text).

## 8.15 Admin tooling for the games

`/admin/hole-builder` and `/admin/courses` (canvas editors for 9-hole courses with drag-to-order
holes and obstacle shapes), `/admin/par3-holes`, `/admin/par3-conditions`,
`/admin/hole-preview`, `/admin/play-test`, `/admin/bdb-config` (including the three-state
audience control), `/admin/bdb-conditions` (a pin and wind override for any single Bad Day Balls
game day), `/admin/par3-finish-promo` (the messages that greet a member after a daily Par 3 round
— each one either opens by itself over the finish screen or sits as a one-line prompt, in the club's
own wording and order), `/admin/points-tally`
(per-member point audits), `/admin/game-activity`, `/admin/par3-shot-stats`,
`/admin/game-messaging` (compose to a filtered audience), `/admin/finish-preview`,
`/admin/signup-preview`, `/admin/mini-games-landing`, `/admin/mini-game-history`,
`/admin/9-hole-history`, the game-tile arranger (`GamesTabOrder`), plus the shop manager
(ball/emblem catalog with drag-to-order, labels and tags, collections/glass cases), achievement
manager, club awards and the admin "unlock everything" tester switch.

## 8.16 The Frenemies Points Duel (`/frenemies-duel`, `/frenemies-duel/:duelId`)

A one-to-one wager game: two members put Pro Shop points on a blind five-shot shootout.

- **The challenge:** pick a Frenemy and a wager of **5, 10, 25 or 50** points. They may accept,
  decline or **counter** at a different wager, which the challenger then answers. The invite lives
  **48 hours**; once accepted both five-shot rounds must be in within **7 days**.
- **Nothing is ever lost to a duel that is not played:** a decline, a cancellation, an expiry or a
  tie hands every staked point straight back (each refund written once, under its own award key).
- **The hole** is generated per duel — wind **5–20 mph** and a pin never within 25° of straight
  up/down — and snapshotted onto the duel, so both players play exactly the same shot.
- **Blind rounds:** neither player sees the other's distances, landings or shot map until both
  rounds are in; the reveal then shows the two rounds side by side.
- **The money:** a stake leaves the wallet the moment it is committed (`duel:<id>:stake:<side>`),
  so nobody can wager points they do not hold. The **pot** is both stakes, paid to the winner under
  a keyed award. **Resolution** sorts both players' five distances closest-first and compares them
  position by position — the first difference decides; if the best ball decides it the win reason is
  `best_distance`, otherwise `tiebreak`; all five level is `tie`.
- **Test duels:** an admin can throw one down and play both sides to walk the whole flow through.
  `is_test` duels stake nothing, notify nobody and are hidden from members.
- **Live switch:** `FrenemyDuelSettings.is_live`, owned by the Go Live switch on the admin
  dashboard. While it is off the duel is hidden from members and only an admin sees it.
- **Surfaces:** `DuelHub` (open duels, invitations, results), `DuelChallengeDialog`,
  `DuelResponsePanel`, `DuelRoom`/`DuelPlayScreen`, `DuelReveal` (the reveal),
  `DuelResultScreen`, `RematchWagerPicker`, `DuelNudgeButton`, `DuelReportButton`, plus the
  coin-catch and pot-ball animations.
- **Server:** `create-frenemy-duel`, `get-frenemy-duel`, `respond-frenemy-duel`,
  `commit-frenemy-duel-shot`, `nudge-frenemy-duel`. Every rule lives in
  `base44/shared/frenemyDuel.ts` (`WAGERS`, `SHOTS_PER_DUEL`, `INVITE_HOURS`, `ROUND_HOURS`,
  `resolveDuel`, `stakePoints`, `refundPoints`); `src/lib/frenemyDuel.js` only mirrors it for
  the screen.

## 8.17 The Member Guest Challenge (`/mg`, `/mg/:code`) — the acquisition game

The club's growth loop: a member plays, then hands the same hole to a friend who has never signed up.

- **The member plays first.** They take their **five shots**, and only then does the challenge open
  a link for the friend with a **7-day** window.
- **The link is the access key.** `/mg/<code>` carries a ten-character lower-case code (about 49
  bits, so links can be neither guessed nor walked). The guest's browser token is pinned to the
  challenge on their first shot, so a link passed further on cannot be played twice, and it is what
  links their anonymous play to the account they later create.
- **The guest** types a name, plays **five shots** on the identical snapshotted hole and the result
  is revealed the moment the fifth shot lands. Winner is decided the same way as the duel: both
  sorted lists compared position by position.
- **Points:** the member is paid **100** when the guest's round is finished
  (`MEMBER_COMPLETION_POINTS`, once), the guest who signs up is paid **200** starting points and
  the member a further **1000** referral points (`GUEST_SIGNUP_POINTS`, `MEMBER_REFERRAL_POINTS`).
  An expired challenge pays nobody.
- **The share picture** is drawn from the challenge's own shot data and uploaded
  (`set-member-guest-share-image`), so the text message the member sends carries a real preview of
  their round.
- **Live switch:** `MemberGuestSettings.is_live`, owned by the Go Live switch on the admin
  dashboard. While it is off the challenge sits greyed out in the Coming Soon row of the games hub
  and only an admin can start one.
- **Surfaces:** `MemberGuestStart`, `MemberGuestGame`, `MemberGuestScoreboard`,
  `MemberGuestResult`, `MemberGuestShare`, `MemberGuestChallengeList`, `MemberGuestGuestOffer`.
- **Server:** `create-member-guest-challenge`, `get-member-guest-challenge(s)`,
  `commit-member-guest-shot`, `complete-member-guest-signup`, `member-guest-invite`,
  `set-member-guest-share-image`, `get-member-guest-config`; rules in
  `base44/shared/memberGuest.ts`.

## 8.18 The 4SF Tutorial (`/games/tutorial`)

Seven optional steps — welcome, your player and points and the Pro Shop, around the games hub,
learning to hit at The Range, playing the Par 3 Contest, playing against people (Frenemies and
Member Guest), and coming back anytime. Every step can be skipped on its own, and the tour never
starts by itself: it is only opened from the games hub's tutorial button, so a member is never
dropped into it mid-game. Step copy lives in `src/lib/gamesTutorial.js`, one card each in
`TutorialStepCard`.


---

# 9. Every page, route by route

Route access: **Public** = no account needed. **Member** = signed in and inside the app shell.
**Admin** = admin role only in practice (the route is member-gated, the data and tools are not).

## 9.1 Public pages

| Route | Page | What it is and what it looks like |
| --- | --- | --- |
| `/` | `Landing` (via `RootRoute`) | The marketing front door for **Schedule Caddie**. Full-bleed turf-green hero with the club logo, a phone-sized demo video, the audience blocks, a promo viewer, an install prompt and a cross-link to the games landing. Signed-in members are routed onward instead of seeing marketing. |
| `/caddie-landing` | `CaddieLanding` | The original Schedule Caddie landing page at a permanent public address, so cross-links and ads always have somewhere to land. |
| `/games-landing` | `GamesLanding` | The **4SF mini-games** marketing page: dark green with gold, game tiles, a hole-design carousel, the locker-room showcase, a points explainer, a swipe carousel of static images and a games CTA. |
| `/play` | `Play` | The daily Par 3 contest. Public: guests play without an account. |
| `/play-9hole` | `NineHolePlay` | The Sunday 9-Hole Challenge (guest-playable; `?test=1` is the admin's throwaway round). |
| `/bdb-par3` | `BdbPar3Challenge` | The Bad Day Balls challenge, open to everyone. |
| `/par3-challenge`, `/par3-challenge/:code`, `/d/:code` | `Par3FriendChallenge` | The async Par 3 Duel, reached from a share link (the short `/d/:code` is what goes in a text message). |
| `/dm-labs`, `/business-labs`, `/dm-labs-golf` | `BusinessLabs` / `DMLabs` | The operator's agency pages, using the scoped `.labs-theme` palette (dark teal + yellow). |
| `/sponsor` | `Sponsor` | Sponsor pitch/landing page. |
| `/support` | `Support` | Support and bug-report page: the `SupportDialog`/support form (`send-support-request`), FAQ, account help. |
| `/terms`, `/privacy` | `TermsOfService`, `PrivacyPolicy` | Legal pages, each with its own meta (see `src/lib/pageMeta.js`). |
| `/login`, `/register`, `/forgot-password`, `/reset-password` | `Login`, `Register`, `ForgotPassword`, `ResetPassword` | The platform's auth pages: email/password, Google/Apple/Microsoft/Facebook, OTP verification step on register, password reset, and the access-request state for invite-only clubs. |
| `*` | `PageNotFound` | 404 page that appends a `prerender-status-code: 404` meta tag so crawlers see a real 404. |

## 9.2 The Schedule Caddie half

| Route | Page | What it does |
| --- | --- | --- |
| `/schedule-caddie` | `ScheduleCaddie` | The caddie hub: availability at a glance, who's free, your rounds, your trips, your network, the club pitch, and the "How Schedule Caddie works" button. |
| `/home` | `AvailabilityDashboard` | The member's home: this week's availability, matching free time, next round ticker, active chats, pings, friend requests, sponsor banner, quick actions. |
| `/network` | `Network` | Find Golfers, Manage Friends (private A/B tiers), Messages, Invite Friends (link/QR/referral), Friends of friends, Activity feed, Who's Free. |
| `/demo-golfers` | `DemoGolfers` | The seeded demo members (so a new club is never empty), with a clear demo label. |
| `/tee-times-hub` | `TeeTimesHub` | Round hub: open rounds, your rounds, create a round, round checklist, host dashboard link. |
| `/tee-times` | `OpenTeeTimes` | The club-wide board of open rounds this member may see, filtered by the tier/48-hour rule. |
| `/tee-times/new` | `CreateTeeTime` | Create a round: course, date/time, slots, visibility, priority friends, payment details, notes. |
| `/tee-times/:id` | `TeeTimeDetail` | One round: the foursome, course, weather, chat, suggestions, share card, golf-now booking, drop-out. |
| `/chat/:teeTimeId` | `ChatThread` | The group chat for that round. |
| `/my-games` | `MyGames` | Everything the member has joined or hosted. |
| `/host` | `HostDashboard` | Host view: each hosted round, who is in, who declined, who has not answered. |
| `/trips` | `TripsHub` | Trips the member is on, hosting, or invited to. |
| `/trips/new` | `CreateTrip` | Start a trip: dates, destinations, group size, paid or not (Stripe Checkout). |
| `/trips/:id` | `TripDetail` | Trip home: roster, destination board, availability heatmap, course voting, payment state. |
| `/trips/:id/chat` | `TripChatThread` | The trip's own chat thread. |
| `/messages/:userId` | `DMThread` | Direct message thread with one member. |
| `/profile` | `EditProfile` | Golf profile: name, zip, home course, handles, photo (stored privately), radius, preferences, calendar sync. |
| `/settings` | `AppSettings` | Account, alerts, device/push, delete account, sign out. |
| `/delete-account` | `DeleteAccount` | Requests account deletion (`AccountDeletionRequest`). |
| `/push-settings` | `PushNotifications` | Push notification preferences and device state. |
| `/pro-shop` | `ProShop` | **The Hub**: cards into Instructors, Sponsors, Partner Courses, Golf Partners, Venues (and the games Clubhouse). |
| `/pro-shop/partners` | `PartnersHub` | All partner categories in one place. |
| `/pro-shop/instructors`, `/:id` | `InstructorsList`, `InstructorProfile` | Teaching pros with rates, bio, contact and booking intent. |
| `/pro-shop/sponsors`, `/:id` | `SponsorsList`, `SponsorProfile` | Sponsor directory and sponsor profiles. |
| `/pro-shop/courses` | `PartnerCourses` | Partner courses with booking links and course proposals. |
| `/pro-shop/golfers`, `/:id` | `PartnerGolfers`, `GolfPartnerProfile` | Playing partners / ambassadors. |
| `/pro-shop/venues`, `/:id` | `VenuesList`, `VenueProfile` | Clubhouses, bars, sim studios: logo, address, hours, map. |
| `/schedule-caddie-for-clubs` | `ScheduleCaddieForClubs` | The B2B pitch to clubs and leagues (feature bubbles, what-we-build, audience benefits, lead form, no pricing). |
| `/schedule-caddie-for-clubs/calendar-example` | `ScheduleCaddieCalendarExample` | A live example of the matchup calendar so operators can see matching. |
| `/schedule-caddie/how-it-works` | `ScheduleCaddieHowItWorks` | Every walkthrough in one place. |
| `/tutorials` | `Tutorials` | All in-app tutorials and how-to videos (with the Instagram reel cards). |
| `/getting-started`, `/about` | `GettingStarted`, `About` | Onboarding checklist and the about page. |

## 9.3 The 4SF mini-games half

| Route | Page | What it does |
| --- | --- | --- |
| `/games` | `GamesHub` | The games home: a two-column grid of artwork tiles in the club's own order (`GamesTabOrder`, arranged by an admin) — Par 3 Contest, Frenemies Points Duel, 9-Hole Challenge (promoted to the top on Sundays), The Clubhouse, Member Guest, The Range, 9-Hole Practice, Match Play, Bad Day Balls (on, guests-only or greyed out at the bottom), the 4SF Player Settings button, a **Coming Soon** row (Hole-in-One Contest, code-gated), the tutorial button, an admin test-round button and the member idea/feedback card. |
| `/clubhouse` | `Clubhouse` | Front door to the Pro Shop, the 19th Hole, the Player's Lounge and the Locker Room. |
| `/games/pro-shop` | `GameProShop` | Spend points: balls, bundles, emblems, collections/glass cases, promo codes, claim banner for prizes, today's tally board, balance card, admin manager when an admin is present. |
| `/games/player-locker` | `PlayersLocker` | Your locker: owned balls on racks, emblems, titles, trophies and earned prizes, with quick jumps to games and the shop. |
| `/games/player-settings` | `PlayerSettings` | Games name, real name + share switch, sound, match reminders, challenge visibility, activity status, icon designer. |
| `/games/19th-hole` | `NineteenthHole` | The board of each day's champion; past days are readable. |
| `/games/19th-hole/:date` | `NineteenthHoleDay` | One day's standings, scorecard rows and the winner's emblem. |
| `/games/trophy-room` | `TrophyRoom` | Hall of Fame: win totals, aces, all-time board, plaques, your cabinet, your places in past rounds. |
| `/games/players-lounge` | `PlayersLounge` | The club directory: who plays, their icon/title/activity, Frenemies, and the way into lounge messages. |
| `/games/lounge/messages` | `LoungeMessages` | Private member-to-member messages inside the club. |
| `/points-history` | `PointsHistory` | The member's own points ledger, day by day, every line that paid. |
| `/the-range` | `TheRange` | Members-only practice range with ring scoring (20 scoring balls a day). |
| `/play` | `Play` | The daily Par 3 contest (5 shots, daily hole, conditions, leaderboard, shot map, points ticker). |
| `/play-9hole` | `NineHolePlay` | The Sunday 9-Hole Challenge (9 holes, live scorecard, Sunday leaderboard). |
| `/practice-9hole` | `NineHolePractice` | Replay any released course, with practice buddies and history. |
| `/hole-in-one` | `HoleInOneContest` | Keep playing until you sink one; fewest shots wins the day. |
| `/match-play` | `MatchPlayHub` | Active matches, invitations, results, rivals and the belt holder; start a match. |
| `/match-play/:id` | `MatchPlayGame` | The match itself: hole-by-hole play, scorecard modal, pause bar, opponent shot reveal, sudden-death reveal, result banner and the points panel. |
| `/par3-challenge*` | `Par3FriendChallenge` | The shared-link duel (also public). |
| `/frenemies-duel`, `/frenemies-duel/:duelId` | `FrenemyDuel` | The Frenemies Points Duel: throw down a wager, answer or counter one, play the blind five-shot round, then the reveal and the result screen. |
| `/mg`, `/mg/:code` | `MemberGuestChallenge` | The Member Guest Challenge: the member plays five shots and shares the link, the guest plays the same hole and signs up (the code link is public). |
| `/games/tutorial` | `GamesTutorial` | The optional seven-step 4SF tour, each step skippable, opened only from the games hub. |
| `/bdb-par3` | `BdbPar3Challenge` | The Bad Day Balls challenge (also public). |

## 9.4 Admin pages

| Route | Page | What it does |
| --- | --- | --- |
| `/admin` | `AdminDashboard` | The sectioned admin console: members and roles, invites, join requests, onboarding reset, courses, partners, demo golfers, shop catalog, collections, achievements, awards, exclusive emblems, promo items, feedback, reports, engagement report, match monitor, points leaderboard, game messaging, system announcements, **Backup & Export** (data JSON, all images ZIP, this rebuild guide). |
| `/admin/mini-game-history` | `AdminMiniGameHistory` | Every Par 3 round, with filters. |
| `/admin/9-hole-history` | `AdminNineHoleHistory` | Every 9-hole round. |
| `/admin/par3-holes` | `AdminPar3HoleBuilder` | The daily hole library (one per weekday). |
| `/admin/par3-conditions` | `AdminPar3Conditions` | Daily pin and wind, with a diagram. |
| `/admin/hole-builder` | `AdminHoleBuilder` | Canvas builder for 9-hole courses. |
| `/admin/courses` | `AdminCourses` | The course rotation: order, release, activate next. |
| `/admin/hole-preview` | `HolePreview` | Preview any hole as a player sees it. |
| `/admin/play-test` | `PlayTest` | Test-play a hole without recording anything. |
| `/admin/bdb-config` | `AdminBdbConfig` | The whole sponsor challenge: copy, hole, branding, codes, links, schedule, the three-state audience control (everyone / guests only / off), QR, test send. |
| `/admin/bdb-conditions` | `AdminBdbConditions` | Pin and wind for one Bad Day Balls game day, with the same diagram as the Par 3 conditions page. |
| `/admin/par3-finish-promo` | `AdminPar3FinishPromo` | The messages that greet a member after a daily Par 3 round (`Par3FinishPromo`): wording, artwork, button and path, automatic popup or one-line prompt, order, on/off. |
| `/admin/points-tally` | `AdminPointsTally` | Points audit per member and per day. |
| `/admin/game-activity` | `AdminGameActivity` | Who is playing what, when. |
| `/admin/par3-shot-stats` | `AdminPar3ShotStats` | Aggregated Par 3 shot bands, lies and distances. |
| `/admin/game-messaging` | `AdminGameMessaging` | Compose a message to a filtered audience. |
| `/admin/signup-preview` | `AdminSignupPreview` | The signup screens in a phone frame. |
| `/admin/finish-preview` | `AdminFinishPreview` | The finish screen in a phone frame. |
| `/admin/mini-games-landing` | `AdminGamesLandingPreview` | Preview of the games landing page. |

## 9.5 Shared components worth rebuilding early

BottomNav (3 tabs), AppLayout (shell + banners), NotificationBell, ProtectedRoute, AuthContext,
RouteMeta/usePageMeta (per-page titles and social tags), SplashScreen + AppReadySignal,
ScrollToTop, GolfLoader (the ball-into-cup loader), Logo/ClubLogo/FoursomeFinder logo components,
the `Image` wrapper used for every content image, the mini-game canvas
(`MiniGameCanvas` for Par 3/Range/HIO, `NineHoleCanvas` for 9-hole, plus
`src/lib/holeCanvasDraw.js` and `src/lib/holeGeometry.js` for drawing and physics helpers),
the scorecard components (`NineHoleScorecard`, `NineHoleScorecardTable`, `NineHoleScorecardModal`),
leaderboard rows and shot-map modal, share-card generators under `src/sidecar/`
(`shareCard.js`, `nineHoleShareCard.js`, `winnerPlaque.js`), and the split-flap points components.


---

# 10. Data model, server and conventions

## 10.1 Entities

Every record also carries the platform's built-in fields (`id`, `created_date`,
`updated_date`, `created_by_id`) and can carry an optional `description` (free notes).
Row-level security is written per entity; the patterns used are listed in 10.4.

### Membership and profile

| Entity | Key fields | Notes |
| --- | --- | --- |
| `User` (built-in) | full_name, email, role | Extended with the golf profile: zip, home course, handles, photo, search radius, alert/push preferences, member id, flags (instructor / partner / practice buddy), display name. |
| `MiniGameSettings` | user_id, display_name, real_name, real_name_shared, sound_enabled, match_reminders, challenge_visibility, activity_status | One per member; the whole 4SF Player Settings page. Owner-only RLS on all four operations. |
| `MiniGameProgress` | user_id, rounds, aces, wins, streak, unlocked_titles, equipped_title, badges, records | Everything a member has achieved in the games; titles are derived from it. |
| `ProShopWallet` | user_id, balance, total_earned, owned_skins, owned_emblems, equipped_skin | One per member; the server folds duplicates into the oldest. |
| `ProShopAward` | user_id, award_key, kind, points, label, day_key, shot_index | The ledger. `award_key` is the idempotency key. kinds: shot, range, daily, practice, place, match, referral, purchase. |
| `MiniGameWinner` | win_date, game_type, user_id/anonymous_id, player_name, distance, total_strokes, ace_count, skin_id, is_member | One record per day per game type; feeds the 19th Hole, Hall of Fame and plaques. Public read. |
| `Referral`, `ReferralBonus` | referrer/invitee ids, day key, credited flags | Track a link share, the friend playing, the points and the bonus shot. |
| `AchievementReward` | trigger, kind, item_key, is_active | The club's prize per mark. |
| `ProShopClaim` | user_id, kind, item_key, name, note, source | A prize waiting to be claimed; claiming is what puts an item in the locker. |
| `Notification` | user_id, type, title, body, link, related_id, read | The bell; recipient-only. |
| `Rivalry` | player_a_id, player_b_id, wins each way, ties, total_matches, belt_holder_id, last_match_date | Match Play head-to-head and the belt. |

### Schedule Caddie data

`AvailabilityBlock` (user, date, window, course/radius, visibility) · `Friendship` ·
`FriendTier` (owner-only A-Team/B-Team/Unassigned) · `TeeTime` (host, course, date/time, slots,
invite_strategy, priority friends + window + declines, payment details, status) · `TeeTimeInvite` ·
`TeeTimeBroadcast` · `PlayRequest` · `Ping` · `SosAlert` · `Chat` · `DirectMessage` ·
`Course` · `CourseProposal` · `CourseVote` · `Trip` · `TripDestination` · `TripRosterEntry` ·
`TripAvailability` · `TripChat` · `Venue` · `Instructor` · `Sponsor` · `GolfPartner` ·
`JoinRequest` · `AppInvite` · `PlayerSuggestion` · `Activity` · `AppFeedback` · `Report` ·
`SystemAnnouncement` · `ScheduleCaddyLead` · `ConsultationInquiry` · `AccountDeletionRequest` ·
`InstagramVideo`.

### Games data

`Par3Hole` (one hole per ET weekday: tee x, green x/radius or polygon, yardage, theme, obstacles) ·
`Par3DailyConditions` (date_key, pin_norm_x/y, wind_speed, wind_angle_deg) ·
`NineHoleCourse` (name, holes[9] with par/yardage/shape/dogleg/green/obstacles, theme,
is_active/released, rotation order) · `NineHoleScore` (member or guest, per-hole scores, totals,
tiebreak counts, ball snapshot, profile snapshot, play_date, course) ·
`NineHolePracticeRound` (member practice rounds) · `MiniGameScore` (the daily Par 3 session:
best distance, shot history, per-shot landings and lies, all_attempts, bonus shots, aces,
session_complete, replay flag) · `MiniGameWinner` · `MiniGameScore3D` (**parked**) ·
`HoleInOneContestScore` · `Par3FriendMatch` (duel: code, both sides' ids/names, conditions,
shot history per player, turn, deadlines, status, winner) · `MatchPlay` (both players, course
snapshot with the nine holes, current hole, turn, hole results with strokes and approach
distances, holes won, tie counts, deadlines, pause fields, win reason) ·
`BdbGameConfig` (one active config: the three-state audience control with `is_live` kept in step,
scheduled days, hole geometry, branding logos, intro copy, prize codes and links, promo
banner/branding switches) · `BdbDailyConditions` (one pin/wind override per sponsor game day) ·
`BdbWeeklyAttempt` · `FrenemyDuel` (challenger and opponent ids/names, status, wager, both
stakes, both players' shot arrays with landings and lies, winner, win reason, pot, test flag, the
hole snapshot, invite and round deadlines) · `FrenemyDuelSettings` (the duel's live switch) ·
`MemberGuestChallenge` (share code, member and guest, both shot sets, status, seven-day window,
the two points flags, hole snapshot, share picture) · `MemberGuestSettings` (live switch) ·
`Par3FinishPromo` (the after-round messages: wording, artwork, button, mode, order, on/off) ·
`GamesTabOrder` (the club's own order for the games hub tiles) ·
`BallSkinItem` · `GameEmblemItem` · `ProShopPromoItem` · `ProShopCollection` ·
`ProShopClaim` · `AchievementReward` · `ProShopWallet` · `ProShopAward` ·
`MiniGameConfig` · `DailyGameSponsor` · `MiniGameMessage` · `Frenemy` · `Referral`/`ReferralBonus` ·
`MiniGameSettings` · `MiniGameProgress`.

## 10.2 Server functions

Around 150 functions exist; they fall into these jobs (recreate them by job, not by name):

- **Game commits (never trust the client):** `commit-mini-game-shot`, `submit-game-score`,
  `commit-hio-shot`, `commit-bdb-shot`, `commit-par3-friend-shot`, `commit-par3-friend-turn`,
  `submit-nine-hole-score`, `submit-3d-score` (parked), `play-match-hole`, `migrate-guest-score`.
  Each validates the shot/round, writes the score, pays the points via keyed awards, updates streak
  and progress, hands over achievements, and returns the fresh leaderboard slice.
- **Game reads:** `get-daily-game-config`, `get-daily-leaderboard`, `get-past-leaderboard`,
  `get-nine-hole-course`, `get-released-nine-hole-courses`, `get-nine-hole-leaderboard`,
  `get-hio-game-config`, `get-hio-leaderboard`, `get-hole-in-one-club`, `get-hall-of-fame`,
  `get-nineteenth-hole`, `get-trophy-room`, `get-title-rarity`, `get-player-progress`,
  `get-player-profile`, `get-player-lounge`, `get-points-history`, `get-points-leaderboard`,
  `get-bdb-game-config`, `get-par3-promo`, `get-par3-shot-stats`, `get-game-activity`,
  `get-game-audience`, `get-title-rarity`.
- **Match Play:** `create-match-play`, `accept-match-play`, `cancel-match-play`,
  `concede-match-play`, `play-match-hole`, `get-match-play`, `get-active-matches`,
  `get-all-match-play`, `check-match-warnings`, `pause-match-play`, `unpause-match-play`,
  `request-match-resume`, `resume-paused-match-play`, `get-rivalries`.
- **Frenemies Points Duel:** `create-frenemy-duel`, `get-frenemy-duel`,
  `respond-frenemy-duel`, `commit-frenemy-duel-shot`, `nudge-frenemy-duel` — every rule,
  including staking and refunding the points, lives in `base44/shared/frenemyDuel.ts`.
- **Member Guest Challenge:** `create-member-guest-challenge`, `get-member-guest-challenge`,
  `get-member-guest-challenges`, `commit-member-guest-shot`, `complete-member-guest-signup`,
  `member-guest-invite`, `set-member-guest-share-image`, `get-member-guest-config`.
- **Par 3 Duel:** `create-par3-friend-match`, `get-par3-friend-match`,
  `get-par3-friend-matches`, `commit-par3-friend-shot`, `commit-par3-friend-turn`,
  `check-par3-friend-warnings`, `par3-challenge-preview`.
- **Shop and points:** `get-pro-shop`, `buy-ball-skin`, `buy-game-icon-emblem`,
  `equip-ball-skin`, `save-shop-item`, `save-shop-collection`, `claim-pro-shop-item`,
  `redeem-promo-code`, `get-promo-items`, `save-achievement`, `admin-grant-award`,
  `admin-grant-emblem`, `admin-unlock-all-items`, `award-practice-points`,
  `award-range-points`, `award-placement-points`, `set-player-title`, `save-game-icon`.
- **Notifications, messaging, push:** `notify`, `send-push`, `send-game-message`,
  `get-onesignal-config`, `send-broadcast`, `lounge-messages`, `send-bdb-promo-code`,
  `send-bdb-test-email`, `send-support-request`.
- **Courses, holes and admin content:** `activate-next-course`, `recreate-hole-from-images`,
  `get-nine-hole-course`, `export-my-data`.
- **Schedule Caddie (see 7.12 for the full list):** visibility, matching, rounds, social, trips,
  lifecycle, weather, admin.
- **Support and account:** `submit-app-feedback`, `request-account-deletion`,
  `send-support-request`, `get-member-activity`, `log-round-activity`.

**Function rules to copy:** a function validates its own inputs; anything that must not be forged
(points, scores, purchases, tier visibility, opponent identity) runs with the service role; reads
that feed a board are filtered and limited at the database, not in JS; failures that should not
break a player's action (points, prizes, emails, push) are caught and logged, while genuine errors
bubble up.

## 10.3 Workflows

| Workflow | Trigger | Job |
| --- | --- | --- |
| `Par 3 Placement Points` | nightly, after the game day closes | Pays 1st–10th places for the day's Par 3 board and hands over the related achievement prizes. |
| `Sunday Course Rotation` | weekly (Sunday) | Activates the next released 9-hole course. |
| `Match Play Turn Warnings` | recurring | Warns a player whose 3-hour turn is nearly up and forfeits when it expires. |
| `Match Play Pause Resume` | recurring | Resumes matches whose 12-hour break has ended. |
| `Par3 Friend Challenge Reminders` | recurring | Turn-start reminder, deadline warning and forfeit for duels. |
| `Tee Time Reminders` | scheduled | Day-before and day-of round reminders. |
| `Weather Check` | scheduled | Refreshes the cached forecasts the app reads. |
| `Clean Up Past Availability` | nightly | Deletes expired availability blocks. |
| `Regenerate Practice Availability` | scheduled | Keeps the practice buddies' availability fresh. |
| `Archive Expired Chats` | scheduled | Archives old threads. |
| `Welcome New Member` | on signup | Sends the welcome email and sets up the new member. |
| `Match Notification` | triggered by match events | Routes match notifications through the bell + push pipeline. |

## 10.4 Row-level security patterns used

- **Owner-only** (`data.user_id` / `created_by_id` equals the caller's id) for personal records:
  availability, wallets, awards, progress, notifications, settings, claims, private tiers,
  frenemies, DMs. Reads *and* writes are restricted, not just reads.
- **Participant pairs** with `$or` for two-sided records: Match Play, duels, friendships,
  direct messages.
- **Public read, server-only write** for boards and catalogs: scores, winners, promo items,
  courses, partner records — everyone can read, only admins (or the service role) can write.
  This is what makes a score impossible to forge from the client.
- **Admin-only** for every configuration and content-management entity: game config, catalogs,
  achievements, leads, reports, announcements.
- Never leave a write open on an entity that carries points, scores, ownership or money.

## 10.5 Cross-cutting conventions

- **Idempotency everywhere it matters:** every point credit, prize, referral and purchase carries a
  unique key (`award_key`, `referral:<invitee>`, `<scoreId>:shot:<n>`, `<dayKey>:place`) and is
  written once; re-running a job is always a no-op.
- **One day key:** the Pacific `YYYY-MM-DD` from a single helper, used by the games, the boards,
  the streaks, the points ledger and the sponsor schedule. Display times are Eastern.
- **Guests are first-class:** an `anonymous_id` in the browser identifies a signed-out player
  everywhere a user id would go, and the server treats them as a real participant except that they
  cannot earn points.
- **Notification pipeline:** write the `Notification` row first (the bell is the record), then push
  if the member allows it, then email only when it matters (first contact of a match, prize awards,
  promo codes) — never an email for every turn.
- **Images:** every content image goes through the `Image` wrapper (`@/components/ui/image`) which
  serves it resized for the container; end-user uploads are stored privately and read through signed
  URLs; the club's and the shop's artwork is public so canvases and boards can draw it.
- **Performance:** every page is a lazy chunk; heavy libraries (3D engine, maps, PDF, image export)
  are imported on demand; lists are paged; counts and totals come from the server; the shell and the
  games home are prefetched for returning members.
- **Accessibility and polish:** reduced-motion support for every animation, high-contrast text on
  both surfaces (see section 3), 44 px touch targets, tabular numerals for scores.
- **Analytics:** posthog-js (`src/lib/posthog.js`) plus the platform's own analytics events.
- **SEO:** per-page titles/descriptions (`RouteMeta` + `src/lib/pageMeta.js`), `public/sitemap.xml`,
  `public/robots.txt`, and a 404 that reports a real 404 status to crawlers.

## 10.6 Configuration a rebuilt club needs

1. Secrets: OneSignal (app id, REST key, enabled switch), Stripe (secret, publishable, webhook
   secret) if trips are paid.
2. One active `BdbGameConfig` (branding, codes, links, schedule) if the sponsor game is used.
3. Six `Par3Hole` records (one per weekday) and their daily conditions; one or more released
   `NineHoleCourse` records with a rotation.
4. The shop catalog: any `BallSkinItem` / `GameEmblemItem` the club adds beyond the launch set,
   plus collections, promo items and achievement prizes.
5. Email templates and the notification switches; the club's logos and artwork.


---

# 11. Rebuild playbook

Work through these phases in order. Each phase ends with something that can be seen and used, and
nothing in a later phase should require re-doing an earlier one.

### Phase 0 — Foundations

1. Create the project shell: React + Vite, Tailwind, shadcn/ui, react-router, TanStack Query.
2. Paste in the design tokens from section 3 (colours, fonts, radius, the loader and split-flap
   CSS), and the `surface-light` / contrast rules.
3. Create the auth pages (login, register with OTP, forgot, reset) and the route guard.
4. Build the signed-in shell: header, bottom navigation with three tabs, banners, notification bell.

### Phase 1 — Membership and the golf profile

`User` extensions, `MiniGameSettings`, the profile and settings pages, invite/access-request flow,
onboarding tour + activation checklist.

### Phase 2 — Schedule Caddie core

Availability blocks → the free-time calendars → the server-side matching function → friends,
friendship requests and the **private tier** model → the stagger rule (24 h A-Team, 48 h B-Team,
then everyone) implemented in a **service-role visibility function**.

### Phase 3 — Rounds and trips

Create round (with priority friends and visibility), invites, broadcasts, the open-rounds board,
join/drop, round chat, suggestions, pings, play requests, SOS, reminders, weather cache. Then trips:
destinations, availability heatmap, voting, roster, trip chat, optional Stripe payment.

### Phase 4 — The games engine

1. Hole geometry: the shared drawing/physics helpers and the canvas component used by every game.
2. The admin hole builder (canvas editor with obstacles, green polygon, themes, drag-to-order).
3. The daily Par 3 contest end to end, with its conditions, scoring, completion-only leaderboard,
   winner record, and the referral bonus shots.
4. The 9-hole engine, its scorecard, the weekly rotation and the Sunday challenge.
5. Streaks, titles, progress and the record book.

### Phase 5 — Points, shop and clubhouse

Wallet + keyed award ledger first (never points without an idempotency key), then the catalogs,
buying/equipping, collections and glass cases, promo codes, achievement prizes and claims, the
locker, the 19th Hole, the Hall of Fame, the Player's Lounge and lounge messages.

### Phase 6 — The head-to-head games

Match Play (turn clocks, pause, forfeit, clinch, sudden death, rivalry belt, points) and the async
Par 3 Duel (share codes, 3 shots per turn, sequential tiebreak, claiming the opponent spot).
Both need reminders, so build the workflows at the same time. Then the Frenemies Points Duel
(wager, accept/decline/counter, staking and refunding points under keyed awards, blind rounds,
reveal, payout) and the Member Guest Challenge (the member plays first, a 10-character share link,
the guest's anonymous round pinned to their browser, then signup, points and the share picture).

### Phase 7 — The sponsor challenge

The Bad Day Balls game: live switch, daily deterministic conditions, 5-shot session, prize tiers,
codes, emails, scheduling, branding switches and its admin page.

### Phase 8 — Admin, exports and communications

Membership tools, content managers, game configuration pages, points audits, messaging, the
Backup & Export cards (data, images, this guide), report/feedback handling.

### Phase 9 — Polish and launch

Share cards (scorecard, plaque, Wordle-style text), reduced-motion passes, contrast audit, empty
states, install/PWA prompts, SEO meta, sitemap/robots, the marketing landings, and the club pitch
page with its lead form.

### Traps already hit in this build (do not repeat)

- Do **not** switch tabs programmatically during a guided tour: the tour unmounts and loses context.
- Do **not** try to create `User` records directly: members join through invites and OTP
  verification only.
- Do **not** filter, count or sort a whole table in the browser: the server does it.
- Do **not** grant points, scores, purchases or prizes from the client, and never without a unique
  award key.
- Do **not** let one global high-specificity CSS selector (e.g. forcing every descendant's text
  colour) try to fix contrast: use the surface tokens.
- Do **not** bundle heavy engines (3D, maps, PDF, image export) into the initial load: import them
  on demand.
- Do **not** re-fetch weather or scrape third-party profiles server-side on a schedule: cache
  forecasts, and treat social scraping as blocked from hosting IPs.
- Do **not** invent a second wallet per member, and never let ownership live outside the wallet:
  duplicate wallets are what make an owned item look unbought.
- Do **not** use a dynamic Tailwind class name (`bg-${colour}-500`): it is purged and silently
  disappears.
- Do **not** point a share link at a preview-only host, and never fetch Open Graph tags
  client-side: share crawlers need static HTML.

### Acceptance checklist

- A new member can register, set availability, see who is free, create a round, invite a friend,
  chat, join a trip and get a reminder — without touching a second app.
- A visitor can play the daily Par 3 contest without an account, see the day's board, and sign up
  to bank the points.
- Every game respects the Pacific game day and shows a fresh leaderboard after each reset.
- Points can be earned by every listed action and spent in the shop; a re-run of any job, a retry of
  any shot and a double tap on a purchase never pay or charge twice.
- A match can be paused, resumed, forfeited and won; a duel can be started by a text message link
  and finished by two people who never open the app at the same time.
- Admins can configure holes, courses, daily conditions, the sponsor challenge, the shop catalog,
  prizes and messaging entirely in-app, and can export the club's data, every image, and this guide.


---

# 12. Appendix — the club as it is configured right now

Generated 10/28/2026, 12:32:30 PM (Eastern). Current game day (12:00 AM Pacific reset): **2026-10-28**, ET weekday **3**.
This appendix is read from the live app at the moment of download, so it always describes the club's real configuration rather than an example.

## Ball skins (Pro Shop)

| Name | Catalog key | Price | Label | State |
| --- | --- | --- | --- | --- |
| Raise U | raise-u | 500 | Premium | listed |
| Huckleberry | full-deck | 1000 | Legendary | listed |
| Ocean 1 of 1 | ocean-of-1 | award | Achieved | listed |
| Skull | skull | 50 | Seasonal | listed |
| Mummy | mummy | 50 | Seasonal | listed |
| Zombie | zombie | 50 | Seasonal | listed |
| Jack-o'-Lantern | jackolantern | 50 | Seasonal | listed |
| Pumpkin | pumpkin | 20 | Seasonal | listed |
| Gold Sparkle | golddust | 300 | Legendary | listed |
| Stars & Stripes | stars | 200 | Rare | listed |
| Lava Rock | lava | 200 | Exotic | listed |
| 4SF Striped | striped | 100 | Relic | listed |
| Galaxy Swirl | galaxy | 100 | Exotic | hidden |
| Woodland Camo | camo | 100 | Common | listed |
| Crimson Red | crimson | 50 | Common | hidden |
| Electric Blue | electric | 50 | Common | listed |
| Sunset Orange | sunset | 50 | Common | listed |
| Practice | practice | 0 | Stock | listed |
| Bad Day Balls | bdb | 0 | Collab | listed |
| Pink Ball | pink | 50 | Common | listed |

## Game icon emblems

| Name | Catalog key | Price | Label | State |
| --- | --- | --- | --- | --- |
| Roll 'em | roll-em | 350 | Premium | listed |
| Lucky 7s | lucky-7s | 500 | Legendary | listed |
| BDB2 | bdb2 | award | Limited | listed |
| BDB5 | bdb5 | award | Limited | listed |
| BDB4 | bdb4 | award | Limited | listed |
| BDB3 | bdb3 | award | Limited | listed |
| BDB1 | bdb1 | award | Limited | listed |
| Beta Tester | beta | 0 |  | listed |
| Founding Member | founding | 0 |  | listed |
| Early Adopter | early | 0 |  | listed |
| F2 | f2 | 20 | Common | listed |
| F1 | f1 | 20 | Common | listed |
| Ocean Top 5 | ocean-top-5 | award |  | listed |
| USA Flag | usa | 100 | Rare | listed |
| Jolly Roger | jolly | 100 | Rare | hidden |
| Cheers | cheers | 50 | Rare | listed |
| Golf Cart | cart | 50 | Rare | listed |
| Clover | clover | 20 | Common | listed |
| Target | target | 20 | Common | listed |
| Golfer | golfer | 20 | Common | listed |
| Golf Hole | hole | 20 | Common | hidden |
| Dollar Sign | money | 100 | Rare | listed |

## Promo-code giveaways

| Name | Kind | Claim | Display | State |
| --- | --- | --- | --- | --- |
| Ocean Course Champion | emblem | club award, no code | not shown | claimable |
| BDBBaddie | emblem | code set | not shown | claimable |
| BDBBandit | emblem | code set | not shown | claimable |
| BDB5 | emblem | code set | not shown | claimable |
| BDB4 | emblem | code set | not shown | claimable |
| BDB3 | emblem | code set | not shown | claimable |
| BDB1 | emblem | code set | not shown | claimable |
| BDB2 | emblem | code set | not shown | claimable |
| IG 2 | skin | code set | not shown | claimable |
| Mummy | skin | code set | not shown | claimable |
| Verified | emblem | club award, no code | not shown | claimable |
| IG Partner | emblem | club award, no code | not shown | claimable |
| RRG | skin | code set | not shown | claimable |
| IG Exclusive | skin | code set | not shown | claimable |
| Haddon Twp | skin | code set | not shown | claimable |
| Arnie | skin | code set | not shown | claimable |

## Collections

| Name | Kind | Items | Mode | State |
| --- | --- | --- | --- | --- |
| Bad Day Balls | emblem | 5 | glass case | shown |

## Achievement prizes

_none set up right now_

## Bad Day Balls challenge

- Audience: **on for everyone**
- Scheduled days: none — default (every day except Sunday)
- Yardage: 195 · Theme: default · Intro title: How the Bad Day Balls Challenge Works
- Prize codes set: 5% yes · 10% yes · hole-in-one yes
- Promo banner on the daily contest: off · BDB branding on the canvas: off

## Courses and holes

### 9-hole courses

| Name | Holes | Theme | State |
| --- | --- | --- | --- |
| The Links Course #5 | 9 | links | retired |
| Ocean Course #4 | 9 | ocean | in rotation |
| Mountain Course #3 | 9 | ocean | retired |
| Desert Open Course #2 | 9 | desert | retired |
| Par 3 Nine hole Course #1 | 9 | default | retired |

### Daily Par 3 holes (one per weekday)

| Weekday | Yardage | Theme |
| --- | --- | --- |
| Saturday | 170 | desert |
| Friday | 110 | default |
| Thursday | 155 | links |
| Wednesday | 175 | mountain |
| Tuesday | 135 | desert |
| Monday | 145 | default |

## Club size

- Member accounts: **87**
