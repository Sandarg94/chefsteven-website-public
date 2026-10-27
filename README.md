# Chef Steven Catering LLC

React/Vite site for Chef Steven Catering LLC. The public experience keeps a refined black, champagne-gold and ivory editorial direction. Firebase-backed forms, authentication, content management and hosting configuration are prepared locally; nothing is deployed by this repository.

## Local development

```bash
npm install
npm run dev
npm run build
```

The build creates `dist/sitemap.xml`. Set `SITE_URL=https://your-real-domain.example npm run build` before production publishing. The default sitemap URL is intentionally `http://localhost:5173` when no real domain has been supplied.

Copy `.env.local` with the existing `VITE_FIREBASE_*` values. The current project ID is `chef-steven-s-website`. The browser configuration contains only public Firebase client identifiers; it contains no service-account credential, API secret, or owner-claim write capability.

## Firebase architecture

Public collections:

- `gallery`: `title`, `description`, `category`, `image`, `path`, `published`, `featured`, `sortOrder`.
- `menus`: `title`, `description`, `category`, `itemsJson`, `image`, `published`, `featured`, `sortOrder`.
- `videos`: YouTube records use the real `youtubeId`, `channelId`, `publishedAt`, `duration`, official thumbnail and `source: youtube`; local records use `source: local` and a Storage `path`. Manual `category`, `published`, `featured`, `sortOrder`, title/description overrides and hide/show choices survive later syncs.
- `siteContent/public`: public contact details, social links and editable home copy.

Private collections:

- `quoteRequests`: validated server submissions with `status`, `archived`, `createdAt`, `updatedAt`, and notification delivery state.
- `consultationRequests`: same lifecycle for Kitchen Consulting submissions.
- `rateLimits`: server-only hourly anti-abuse records.

`functions.submitInquiry` validates and stores both form types using the Admin SDK. Firestore rules do not allow anonymous writes to request collections. Admin reads and content writes require the Firebase custom claim `admin: true`.

## Admin access

`/admin` uses Firebase Email/Password with no public registration. Access requires a Firebase-authenticated account, a verified email, the exact server `OWNER_UID`, the exact server `OWNER_EMAIL`, and claims tying `admin: true`, `ownerUid` and `ownerEmail` together. The claim script is intentionally not deployed or run automatically. It is a dry run unless `--apply` is explicitly supplied:

```bash
cd functions
npm install
OWNER_UID="the-owner-uid" OWNER_EMAIL="the-owner-email" npm run set-admin -- --check the-owner-uid
# Only after explicit authorization to write claims:
OWNER_UID="the-owner-uid" OWNER_EMAIL="the-owner-email" npm run set-admin -- --apply the-owner-uid
```

The command also requires Google Application Default Credentials or the Firebase service-account credential configured in the server environment. The local `functions/.env` contains only the non-secret owner parameters and is ignored by source control. The administrator must sign in again after the claim is assigned.

## YouTube synchronization

The server function resolves `@ChefStevenCaterinLLC4603` with `channels.list`, follows its real uploads playlist through every `playlistItems.list` page at `maxResults=50`, then reads video metadata with `videos.list`. Public uploads and Shorts are deduplicated by the real video ID; private, deleted and unavailable items are skipped. The first successful sync stores the verified `channelId` and `uploadsPlaylistId` in `youtubeSync/status` and returns the found/added/updated/skipped report to Content Studio. A scheduler checks hourly and the owner-configured interval (1–168 hours) controls the effective refresh; the owner can also force a sync from `/admin`.

Configure the API key only as a Firebase secret:

```bash
firebase functions:secrets:set YOUTUBE_API_KEY
```

Cloud Functions scheduling and Secret Manager may require the Blaze plan. The repository does not contain a YouTube key and no real YouTube sync is claimed as tested until that secret is configured.

## Local content inventory and migration

The current local inventory is kept in `scripts/content-migration-manifest.mjs`: 27 gallery records (19 official files plus 8 existing illustrative records), 4 menu records, 1 local video, and the public site-content record. `npm run migrate:content` is a dry-run and performs no writes. It prints stable IDs, source paths, and migration keys. Applying requires `--apply`; the script creates a JSON backup first, skips existing IDs or migration keys, never copies the existing `public/assets` files to Storage, and refuses production unless the operator explicitly supplies `--allow-production`.

The public pages keep their existing static source of truth. Content Studio inventories those static records alongside Firestore records so the owner can recognize and manage historical content without replacing the public fallback or introducing duplicates. Saving a historical record creates an idempotent Firebase record with its stable migration ID; it does not change the public pages until a separate, explicitly approved frontend migration is made.

## Functions and email

Install `functions/package.json` dependencies before using the Functions emulator or deploying Functions. Optional email notifications use Resend and require server-only configuration:

- Firebase Secret Manager secret: `RESEND_API_KEY`.
- Runtime environment values: `INQUIRY_NOTIFICATION_EMAIL` and `INQUIRY_FROM_EMAIL`.
- `INQUIRY_FROM_EMAIL` must be a verified sender/domain with the provider.

If these values are absent, requests are still stored and the server logs that email notifications are not configured. No fictitious key or sender is included.

For local Functions work, use the ignored `functions/.env.local` file. Production should use Secret Manager instead:

```bash
firebase functions:secrets:set RESEND_API_KEY
```

Each accepted inquiry is stored before email work starts and receives a server-generated document ID, `status: new`, and `notificationStatus: pending` or `not_configured`. The visitor success response is returned only after that Firestore write is confirmed. The notification to the professional mailbox and the client confirmation are independent, so an email provider or network error does not erase a valid request or turn the submission into a client-facing error. Delivery attempts are tracked per recipient, use deterministic Resend idempotency keys, and can be retried by the owner callable or the 15-minute scheduled retry. Repeated identical submissions are deduplicated for ten minutes, while the existing server-side hourly rate limit remains active. Submitted values are escaped in the black, ivory and gold email templates.

Quote requests submitted from a menu include a server-built `selectedMenu` object containing `menuId`, `menuName`, `menuSlug`, and a limited `menuSnapshot`. The server accepts only a currently published menu document; browser-supplied names and prices are never trusted.

## Deployment preparation only

Review locally, then deploy only with explicit authorization:

```bash
npm run build
firebase deploy --only hosting,firestore:rules,storage,functions
```

The inquiry workflow requires the Functions deployment to include `submitInquiry`, `retryInquiryNotifications`, and `retryInquiryNotificationsScheduled`. Hosting must be redeployed for the updated success messaging and admin request display. Firestore rules remain private and should be deployed if the target project does not already have the rules in this repository; no public Firestore write is required.

Storage uploads are restricted to administrators, images are limited to JPEG/PNG/WebP/AVIF and 10 MB, and video uploads are limited to 100 MB. Firebase Storage may require the Blaze plan depending on the project state. Cloud Functions and outbound transactional email may also incur billing.

## Current limitations requiring owner input

- No official logo, favicon or social-share image was present in the project resources. The current wordmark is a text treatment and is not claimed to be the official logo. Supply the approved logo assets before launch.
- No production domain was supplied; use the actual domain for `SITE_URL`, canonical URLs and final sitemap output.
- The exact owner UID and email are configured locally in `functions/.env`, mirrored in `functions/.env.example`, Functions authorization, and Firestore/Storage rules. The verified owner account must still be checked and claims must be assigned only with explicit authorization. No account is created by this project.
- The supplied Instagram URL is retained as a post URL because the official profile could not be confirmed; replace it in Content Studio when the profile URL is known.
- Firebase Functions dependencies have been declared but not installed or deployed in this workspace. The client build and local validation are verified; emulator and cloud-function integration still require installing the Functions dependencies and using the Firebase Emulator Suite.
- The public site falls back to the existing illustrative Unsplash records until published Firebase content exists. Replace them with licensed Chef Steven photography before launch.
