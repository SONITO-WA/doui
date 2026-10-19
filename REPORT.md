# doui — reverse analysis of www.betterhelp.com (front-end, logged-out)

Method: skill `reverse-engineer-anything` (morluto/rea), applied by hand.
The `npx rea-agents` CLI was blocked by the session's security policy, so the
skill's "JavaScript application" workflow was followed manually: static reading
of the first-party scripts and the homepage HTML. No login, no authenticated
calls, no runtime capture. Date: 2026-10-19.

Raw HTML/JS captured from the site are NOT stored in this repo (proprietary).
Only findings are recorded.

## Observations (from public HTTP response + HTML)

- Edge: CloudFront (`Via`, `X-Cache`), behind Istio/Envoy (`server: istio-envoy`).
- Back end sets `PHPSESSID` -> PHP server-rendered pages; a `page_info` JS
  object is injected into each page (feature flags, user type, csrf token,
  socket config, locale, A/B experiments).
- Responses are `no-store`; CSP is only `frame-ancestors 'self'`; `X-Frame-Options: SAMEORIGIN`; HSTS preload.
- Cookies: `PHPSESSID`, `BHTOK` (httponly, samesite=strict), `pse` (experiment
  assignment), `nte`, `ncy`, `cookies_migrated_v2`, Snowplow `_sp_id/_sp_ses`.
- Front-end stack: jQuery + Bootstrap 3.3.6 + jQuery Mobile CSS, select2, slick,
  AOS, odometer, tippy/popper, lozad, jstz, ua-parser.
- Observability/analytics: Coralogix browser SDK, Snowplow, Cloudflare Zaraz
  (`z.betterhelp.com`), GTM references, custom `bhLogger.js`.
- First-party bundles (12): main, modals, bhLogger, events, everywhere,
  international_utils, loggedout, language_selector, socket, social_proof_counter,
  gift, gpc_toast. Served from `static.betterhelp.com/dist/js/` with a build
  hash query (`?v=<hash>`).

## API surface referenced by client JS (endpoints only, not called)

| Endpoint | Referenced in | Purpose (inferred) |
|---|---|---|
| `/api/confirm_cookie_choices/` | everywhere.js | save cookie consent |
| `/api/back-to-portal-login-verification` | everywhere.js | login check |
| `/api/set_counselor_activity/` | everywhere.js | counselor presence |
| `/api/get_billing_history_url/` | everywhere.js | billing link |
| `/api/presignup_experiments` | everywhere.js | A/B experiment assignment |
| `/api/sharing-settings` | everywhere.js | sharing prefs |
| `/api/locale/selector` | language_selector.js | language switch |
| `/api/main_counters/` | social_proof_counter.js | homepage member/session counters |
| `/api/multi_cookie/` | loggedout.js | cross-domain id sync across sister sites (`sites_fqdn`) |
| `/api/set_performance_cookie` | loggedout.js | store `bh_mc_id` |
| `/api/pusher-trigger`, `/api/pusher-user-authentication`, `/api/pusher-user-authorization` | socket.js | realtime messaging auth/trigger |

`/api/` is disallowed in robots.txt for crawlers; nothing here was probed.

## Realtime (socket.js) — inference, static only

Pusher client. Channel naming: `private-encrypted-to_counselor_p-<id>`,
`private-encrypted-to_client_p-<id>`, `private-encrypted-to_client_g-<counselor id>`,
`private-encrypted-to_manage_user_p-<first 10 hex of SHA-256(manage_user)>`.
Events include `client_connected`, `client_disconnected`,
`send_your_information_to_counselor`. A `-<subdomain>` suffix is appended on
non-production clouds. All triggers go through the server (`/api/pusher-trigger`
with `csrf_token`), not directly from the browser.

## Product feature flags seen in `page_info` (names only)

smart_match, funnel/friendly_funnel/private_funnel, videosessions, phone_calls,
livechat, scheduling, group_sessions, psychiatry, teen, couples, insurance,
referral_program, vouchers/offer_promo_codes, gift, journal, mood_tracker,
worksheets, EAP co-branding, parent_child_account, pause_membership,
push/text notifications, locale/language matching.

## Unknowns / not covered

- Signup funnel, matching algorithm, billing and video: server-side, not visible.
- No runtime capture, so event order and payload shapes are unverified.
- `events.js` (43 KB) and `everywhere.js` (49 KB) were only grepped, not fully read.

## Next steps (if wanted)

Clean-room re-implementation of the logged-out UI structure from this spec
(own copy, own branding), or a deeper pass with browser scenario capture.
