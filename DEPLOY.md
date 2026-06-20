# Deploy goliveapp.net to Cloudflare Pages

## Prerequisites
- Node.js 18+ installed
- GitHub account
- Cloudflare account (domain already added ✓)

---

## Step 1: Install dependencies locally & test

```bash
cd goliveapp-blog
npm install
npm run dev        # opens http://localhost:4321
npm run build      # builds to dist/
```

Make sure the site looks right at localhost:4321 before deploying.

---

## Step 2: Push to GitHub

```bash
git init
git add .
git commit -m "Initial blog setup"

# Create a new repo at github.com/new — name it goliveapp-blog (make it private or public, either works)
git remote add origin https://github.com/YOUR_USERNAME/goliveapp-blog.git
git branch -M main
git push -u origin main
```

---

## Step 3: Connect to Cloudflare Pages

1. Go to [dash.cloudflare.com](https://dash.cloudflare.com)
2. In the left sidebar, click **Workers & Pages**
3. Click the **Create** button (top right)
4. You'll see two tabs — **Workers** and **Pages**. Click the **Pages** tab
5. Click **Connect to Git**
6. Authorize GitHub and select your `goliveapp-blog` repo
7. Set build settings:

| Setting | Value |
|---|---|
| Framework preset | Astro |
| Build command | `npm run build` |
| Build output directory | `dist    ` |
| Node.js version | `18` (set in Environment Variables: `NODE_VERSION = 18`) |

8. Click **Save and Deploy** — first deploy takes ~1 minute.

---

## Step 4: Connect your custom domain

1. In Cloudflare Pages → your project → **Custom domains**
2. Click **Set up a custom domain**
3. Enter `goliveapp.net` and click **Continue**
4. Since your domain is already on Cloudflare, it will auto-add the DNS record. Click **Activate domain**.
5. Also add `www.goliveapp.net` if you want the www version.

Done — your blog is live at goliveapp.net.

---

## Step 5: Future deployments

Every `git push` to `main` automatically triggers a new build and deploy. Zero extra steps.

```bash
# Write a new post → push → it's live
git add src/content/blog/new-post.md
git commit -m "New post: <title>"
git push
```

---

## Enabling AdSense (after approval)

Once Google approves your AdSense account:

1. Open `src/layouts/Base.astro`
2. Uncomment the AdSense `<script>` tag and replace `ca-pub-XXXXXXXXXXXXXXXX` with your publisher ID
3. Open `src/layouts/BlogPost.astro`
4. Uncomment the `<ins class="adsbygoogle">` blocks and set your `data-ad-client` and `data-ad-slot` IDs
5. Push to main — ads go live automatically after the next deploy
