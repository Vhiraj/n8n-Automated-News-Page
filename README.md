# Automated News Carousel Poster (n8n Workflow)

An n8n workflow that scrapes news articles from a website, renders them into styled
image cards, and automatically publishes them as Instagram carousel posts on a
schedule.

## How it works

```
Schedule Trigger (hourly)
        │
        ▼
Scrape article listing page → extract headlines + links
        │
        ▼
Loop through each article → scrape full article body + image
        │
        ▼
Build an HTML "news card" per article (title, image, summary, branding)
        │
        ▼
Render HTML → PNG screenshot (via Gotenberg)
        │
        ▼
Upload PNG to Cloudinary → get a public image URL
        │
        ▼
Batch images into groups of 10 → create Instagram carousel container
        │
        ▼
Poll container status until ready → publish to Instagram
```

## Prerequisites

You'll need the following set up before importing this workflow:

| Requirement | Where to get it |
|---|---|
| n8n (self-hosted, Docker recommended) | [n8n.io docs](https://docs.n8n.io/hosting/installation/docker/) |
| Gotenberg (HTML → PNG rendering) | [Docker Hub: gotenberg/gotenberg](https://hub.docker.com/r/gotenberg/gotenberg) |
| Cloudinary account (free tier is fine) | [cloudinary.com](https://cloudinary.com) |
| Meta Developer App | [developers.facebook.com/apps](https://developers.facebook.com/apps) |
| Instagram Professional (Business/Creator) account | Instagram app → Settings → Account type |
| Facebook Business Manager | [business.facebook.com](https://business.facebook.com) |

## Setup steps

### 1. Gotenberg
Run Gotenberg in Docker alongside n8n, on the same Docker network:
```
docker run -d --name gotenberg -p 3000:3000 gotenberg/gotenberg:8
```

### 2. Cloudinary
1. Sign up, note your **Cloud Name** from the dashboard
2. Go to **Settings → Upload → Upload presets → Add upload preset**
3. Set **Signing Mode** to **Unsigned**, save, note the preset name

### 3. Meta / Instagram setup
1. Create a Meta Developer App at developers.facebook.com/apps
2. Add the **Instagram Graph API** product
3. Under **Business Settings → System Users**, create a system user and generate
   an access token with these permissions:
   - `instagram_business_content_publish`
   - `instagram_basic`
   - `pages_show_list`
   - `business_management`
4. Find your **Instagram Business Account ID**:
   `GET /me/accounts` → grab the Page ID → `GET /{page-id}?fields=instagram_business_account`
5. Fill in required App Settings → Basic fields (Privacy Policy URL, Category, App
   Icon) and switch the app to **Live** mode — required for full rate limits

### 4. Import the workflow
1. Import `workflow.json` into n8n
2. Reconnect all Facebook Graph API nodes to your own credential (containing the
   access token from step 3)

## Configuration — replace these placeholders

| Placeholder | What to put there | Where to find it |
|---|---|---|
| `*workflow-name*` | Any name you like | — |
| `*website-listing-page-url*` | The article listing page you want to scrape | Your chosen news source |
| `*instagram-business-account-id*` | Your Instagram Business Account ID | See Meta setup step 4 above |
| `*gotenberg-api-url*` | Your Gotenberg container's address | e.g. `http://gotenberg:3000` or `http://host.docker.internal:3000` |
| `*cloudinary-cloud-name*` | Your Cloudinary cloud name | Cloudinary Dashboard |
| `*cloudinary-upload-preset*` | Your unsigned upload preset name | Cloudinary → Settings → Upload |
| `*footer-brand-name*` | Your own branding text for the card footer | — |
| `*caption-footer-text-and-hashtags-go-here*` | Your own caption text / hashtags | — |

Credential references (`*credential-id*`, `*facebook-graph-credential-name*`) don't
need manual editing — just reconnect each Facebook Graph API node to your own saved
credential in n8n after import.

## Known limitations

- **Instagram carousels are capped at 10 images per post** via the Graph API, even
  though the Instagram app itself allows up to 20 — this workflow batches accordingly.
- **Carousel containers aren't always immediately ready to publish.** The workflow
  waits a fixed delay before publishing; for full reliability, replace this with a
  polling loop that checks the container's `status_code` until it returns `FINISHED`.
- **New Meta apps start in "Development" mode**, which has a much smaller rate-limit
  ceiling. Switch to "Live" mode (requires a valid Privacy Policy URL and app Category)
  to avoid hitting limits during normal use.
- **Personal Facebook accounts can get temporarily flagged** by automated
  abuse-detection if API calls are made too rapidly (e.g. heavy testing/debugging).
  Using a Business Manager System User token, as described above, is more resilient
  than a personal-account token.
- **Content ownership**: this workflow republishes content from a third-party news
  source. You are responsible for ensuring you have the right to republish this
  content in your target market.

## License

MIT — see `LICENSE`.
