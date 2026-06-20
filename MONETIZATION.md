# Monetization Guide — goliveapp.net

## Part 1: Google AdSense

### Before You Apply
AdSense has a soft threshold: they want to see real content, not just one post. Do this first:

1. **Publish at least 5–10 posts** — 3 posts minimum, but 5+ gives you a much higher approval rate.
2. **Add an About page** — create `src/pages/about.astro` with who you are and what the blog covers.
3. **Add a Privacy Policy page** — AdSense requires this. Use a free generator at https://www.privacypolicygenerator.info and save it as `src/pages/privacy.astro`.
4. **Add a Contact page** — even a simple email link works.
5. Make sure the site loads fast (Cloudflare Pages + Astro = ✓) and has no broken links.

### Apply for AdSense
1. Go to https://adsense.google.com
2. Sign in with your Google account
3. Enter your site URL: `https://goliveapp.net`
4. Select your country, accept terms
5. Add the AdSense verification code to your site — paste it inside `<head>` in `src/layouts/Base.astro`:
   ```html
   <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-XXXXXXXX" crossorigin="anonymous"></script>
   ```
6. Push and deploy, then click "Done" in AdSense dashboard
7. Wait — review takes 1–14 days. You'll get an email.

### After Approval
- In AdSense dashboard → **Ads** → **By site** → turn on **Auto ads** for the easiest setup.
- Or use manual ad units — the slots are already in `BlogPost.astro`, just uncomment them.
- Replace `ca-pub-XXXXXXXXXXXXXXXX` and `data-ad-slot` with your actual IDs.

### Revenue expectations
With a DevOps/tech audience, CPC (cost per click) is typically $0.30–$2.00. At 1,000 monthly visitors you might earn $10–$30/month from AdSense. At 10,000 visitors it becomes meaningful (~$100–$300/month).

---

## Part 2: Affiliate Programs

These are higher-value than AdSense for a tech blog. One DigitalOcean signup can pay $25–$100.

### DigitalOcean (Recommended)
- **Commission:** $25 for every new customer who spends $25; up to $100 for high spenders
- **Sign up:** https://www.digitalocean.com/referral-program
- **Replace in blog post:** find `YOUR_REF_CODE` in the blog post and replace with your referral code
- **URL format:** `https://www.digitalocean.com/?refcode=YOUR_CODE`

### Vultr
- **Commission:** $35 for each new paying customer
- **Sign up:** https://www.vultr.com/affiliate/
- **URL format:** `https://www.vultr.com/?ref=YOUR_CODE`

### Better Stack (Uptime + Logging)
- **Commission:** 25% recurring monthly commission for 12 months
- **Sign up:** https://betterstack.com/affiliates
- **URL format:** `https://betterstack.com/?ref=YOUR_CODE`
- Great fit for SRE/DevOps audience

### Linode / Akamai Cloud
- **Commission:** $100 per new qualified customer
- **Sign up:** https://www.linode.com/referral-program/

### How to update affiliate links in the blog post
Open `src/content/blog/fixing-502-errors-kubernetes-rolling-updates.md` and replace:
- `YOUR_REF_CODE` with your DigitalOcean referral code
- `YOUR_REF_CODE` in the Vultr link with your Vultr referral code
- `goliveapp` in Better Stack link with your Better Stack referral slug

### Disclosure (required by law)
The blog post already marks links with `*(affiliate link)*`. This satisfies FTC disclosure requirements. Keep this in all posts with affiliate links.

---

## Part 3: Future Revenue Ideas (once you have traffic)

- **Sponsored posts** — DevOps tool companies pay $200–$1000 per post once you have 5k+ monthly readers
- **Gumroad/Lemon Squeezy** — sell a Kubernetes config cheatsheet PDF or a Notion template
- **Newsletter** — use Beehiiv (free up to 2,500 subs) and add a subscribe box to the blog

---

## Summary Table

| Channel | Effort | Timeline | Potential |
|---|---|---|---|
| Google AdSense | Low (once approved) | 2–4 weeks to approve | ₹1k–5k/month at scale |
| DigitalOcean affiliate | Low (add link) | Immediate | ₹2k–8k per referral |
| Better Stack affiliate | Low (add link) | Immediate | 25% recurring |
| Sponsored posts | Medium | 6–12 months | ₹15k–80k per post |
