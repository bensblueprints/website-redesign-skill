# Premium Website Rebuild

Scrape a client's existing website, download all assets, and rebuild it as a premium dark luxury site with booking/lead-gen platform and admin dashboard, then push to GitHub and deploy to Netlify. Works for ANY business type.

## Input: $ARGUMENTS

The argument should be the client's existing website URL. If no URL provided, ask for: business name, owner/contact name, phone, email, address, existing website URL, and logo file location.

## Template Repository
Clone from: `bensblueprints/sml-wicked-striper` (the proven blueprint — adapt structure to business type)
Local template: `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\sml-wicked-striper\`

## Execution Steps

### Phase 1: Research & Gather Info

**1a. Scrape the existing site to auto-detect:**
- **Business type** (e.g., fishing charter, law firm, restaurant, salon, contractor, gym, medspa, auto shop, realtor, photographer, etc.)
- **Business name**
- **Owner/key person name(s)** and role (Captain, Chef, Attorney, Owner, etc.)
- **Phone number**
- **Email address**
- **Physical address** (for Google Maps directions link)
- **Services/pricing** (trip types, menu items, service tiers, packages — whatever the business offers)
- **Existing website URL**
- **Google Maps Place ID** (search via Google)
- **Facebook page URL** (for additional photos)
- **All social media links** (Facebook, Instagram, YouTube, TikTok, Twitter/X, LinkedIn, Yelp, etc.)
- **Industry-specific details** (see Phase 1c)
- **Business hours** (if listed)
- **Certifications/licenses** (if listed)
- **Service area / nearby cities** (for area SEO)

**1b. Detect Site Platform**
Before scraping, detect what platform the site runs on. Try these in order:
1. **WordPress** — Try `{site}/wp-json/wp/v2/media?per_page=1`. If 200 → WordPress. Use WP REST API for everything.
2. **Wix** — Check page source for `wix.com`, `static.wixstatic.com`, `_api/v2`.
3. **Squarespace** — Check for `squarespace.com`, `static1.squarespace.com`.
4. **Shopify** — Check for `cdn.shopify.com`, `myshopify.com`.
5. **GoDaddy/Website Builder** — Check for `godaddy.com`, `img1.wsimg.com`.
6. **CloudFront/WAF Protected** — If 403 from both direct fetch AND Playwright, site has bot protection. Fall back to third-party sources.
7. **Static/Custom** — If none of the above, use Playwright only.

**Platform-specific scraping strategies:**
- **WordPress**: Use REST API (`/wp-json/wp/v2/media`, `/wp-json/wp/v2/pages`, `/wp-json/wp/v2/posts`). Fastest and most reliable.
- **Wix**: Use Playwright. Images are lazy-loaded. Scroll the full page. Append `/v1/fill/w_1920,h_1080` to get full-size.
- **Squarespace**: Use Playwright. Images use `data-src` for lazy loading. Append `?format=2500w` for max resolution.
- **CloudFront-blocked sites**: Skip site. Scrape from Google Maps, Facebook, Yelp, TripAdvisor, industry-specific listing sites.
- **Any platform**: ALWAYS also scrape Google Maps, Facebook, Yelp as backup photo sources.

**1c. Industry-Specific Research**
Based on detected business type, also gather:

| Business Type | Extra Data to Collect |
|---|---|
| **Fishing/Hunting Charter** | Target species, body of water, boat specs, fishing license link, FishingBooker/Guidesly listings |
| **Restaurant/Bar** | Menu items, cuisine type, reservation system, health dept rating, DoorDash/UberEats links |
| **Law Firm** | Practice areas, bar admissions, case results, Avvo/Justia profiles |
| **Salon/Barbershop** | Services menu with prices, stylists, booking system (Vagaro, Square, etc.) |
| **Contractor/Home Services** | License number, service types, service area, HomeAdvisor/Angi profile |
| **Gym/Fitness** | Class schedule, membership tiers, trainers, equipment list |
| **Medspa/Medical** | Treatments, providers with credentials, before/after galleries |
| **Auto Shop/Dealer** | Services, brands serviced, certifications (ASE), inventory if dealer |
| **Real Estate** | Listings, areas served, transaction history, Zillow/Realtor profiles |
| **Photographer/Creative** | Portfolio categories, packages, booking calendar |
| **Any Other** | Services, team members, portfolio/gallery, testimonials, unique selling points |

### Phase 2: Scrape Photos & Identify Key Images

**2a. Download all photos using platform-appropriate strategy:**
1. Try platform-specific API first (WordPress REST, etc.)
2. Fall back to Playwright with aggressive scrolling and lazy-load detection
3. ALWAYS also scrape third-party listing sites for additional/backup photos
4. Download all images >10KB to `images/` folder using Node.js (NOT Python — not installed)
5. Check Downloads folder for any pre-downloaded photos
6. Remove business cards, flyers, tracking pixels, tiny icons, map screenshots
7. Find and copy logo to `images/logo.png`
8. **Hero image MUST be >100KB** — keep scraping until you get at least one high-quality photo

**2b. Browse Site with Playwright to Identify Key Photos**
Use Playwright to browse the actual website and identify:
1. **Owner/team photo** — find the "About", "Our Team", "Meet the Owner" page. Save with descriptive name. Be selective — must be the actual owner/team, not a stock photo or customer.
2. **Hero/banner photo** — the best showcase image (storefront, product, action shot, etc.)
3. **Social media links** — scrape ALL social links from every page.

**Playwright browse script pattern:**
```javascript
import { chromium } from 'playwright';
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto(url, { waitUntil: 'networkidle' });
await page.waitForTimeout(3000);

const images = await page.evaluate(() =>
  [...document.querySelectorAll('img')].map(img => ({
    src: img.src, alt: img.alt, classes: img.className,
    width: img.naturalWidth, height: img.naturalHeight
  }))
);

const socialLinks = await page.evaluate(() =>
  [...document.querySelectorAll('a[href]')].map(a => a.href)
    .filter(h => /facebook|instagram|youtube|tiktok|twitter|x\.com|linkedin|yelp/i.test(h))
);

const bgImages = await page.evaluate(() => {
  const els = document.querySelectorAll('*');
  const bgs = [];
  els.forEach(el => {
    const bg = getComputedStyle(el).backgroundImage;
    if (bg && bg !== 'none' && bg.includes('url(')) {
      bgs.push(bg.match(/url\("?(.+?)"?\)/)?.[1]);
    }
  });
  return bgs.filter(Boolean);
});
```

Look for team/about pages: `/about`, `/about-us`, `/our-team`, `/meet-the-team`, `/staff`, `/the-crew`, `/our-story`

**2c. Visual Dedup with Playwright Perceptual Hashing**
WordPress sites often have duplicate images. Run visual dedup after all downloads:

1. Install Playwright locally: `npm install playwright`
2. Load each image as base64 data URI into a Playwright page
3. Draw each to 16x16 canvas, extract grayscale pixel data
4. Compute average hash (pixel > average = 1, else 0)
5. Compare all pairs using hamming distance
6. 90%+ similarity = duplicate → keep highest resolution, delete others

**Visual dedup script** (save as `visual-dedup.mjs` and run with `node visual-dedup.mjs`):
```javascript
import { chromium } from 'playwright';
import fs from 'fs';
import path from 'path';

const DIR = './images';
const THUMB_SIZE = 16;
const SIMILARITY_THRESHOLD = 0.90;

function getBase64DataUri(filePath) {
  const ext = path.extname(filePath).toLowerCase();
  const mime = ext === '.png' ? 'image/png' : ext === '.webp' ? 'image/webp' : 'image/jpeg';
  return 'data:' + mime + ';base64,' + fs.readFileSync(filePath).toString('base64');
}

async function run() {
  const browser = await chromium.launch();
  const page = await (await browser.newContext()).newPage();
  await page.setContent('<html><body></body></html>');

  const files = fs.readdirSync(DIR)
    .filter(f => /\.(jpg|jpeg|png|webp)$/i.test(f) && f !== 'logo.png').sort();

  const hashes = {};
  for (const file of files) {
    const dataUri = getBase64DataUri(path.join(DIR, file));
    const pixelData = await page.evaluate(async ({ dataUri, size }) => {
      return new Promise((resolve) => {
        const img = new Image();
        img.onload = () => {
          const canvas = document.createElement('canvas');
          canvas.width = size; canvas.height = size;
          const ctx = canvas.getContext('2d');
          ctx.drawImage(img, 0, 0, size, size);
          const data = ctx.getImageData(0, 0, size, size).data;
          const gray = [];
          for (let i = 0; i < data.length; i += 4)
            gray.push(Math.round(data[i]*0.299 + data[i+1]*0.587 + data[i+2]*0.114));
          resolve({ gray, width: img.naturalWidth, height: img.naturalHeight });
        };
        img.onerror = () => resolve(null);
        img.src = dataUri;
      });
    }, { dataUri, size: THUMB_SIZE });
    if (!pixelData) continue;
    const avg = pixelData.gray.reduce((a,b)=>a+b,0) / pixelData.gray.length;
    hashes[file] = {
      hash: pixelData.gray.map(g => g > avg ? '1' : '0').join(''),
      width: pixelData.width, height: pixelData.height,
      size: fs.statSync(path.join(DIR, file)).size
    };
  }

  const fileList = Object.keys(hashes);
  const inGroup = new Set();
  for (let i = 0; i < fileList.length; i++) {
    if (inGroup.has(fileList[i])) continue;
    const group = [fileList[i]];
    for (let j = i+1; j < fileList.length; j++) {
      if (inGroup.has(fileList[j])) continue;
      let matches = 0;
      for (let k = 0; k < hashes[fileList[i]].hash.length; k++)
        if (hashes[fileList[i]].hash[k] === hashes[fileList[j]].hash[k]) matches++;
      if (matches / hashes[fileList[i]].hash.length >= SIMILARITY_THRESHOLD) {
        group.push(fileList[j]); inGroup.add(fileList[j]);
      }
    }
    if (group.length > 1) {
      inGroup.add(fileList[i]);
      const sorted = group.sort((a,b) =>
        (hashes[b].width*hashes[b].height) - (hashes[a].width*hashes[a].height) || hashes[b].size - hashes[a].size
      );
      console.log('KEEP: ' + sorted[0]);
      for (const r of sorted.slice(1)) {
        fs.unlinkSync(path.join(DIR, r));
        console.log('  REMOVED: ' + r);
      }
    }
  }
  await browser.close();
  console.log('Remaining:', fs.readdirSync(DIR).filter(f => /\.(jpg|jpeg|png|webp)$/i.test(f)).length);
}
run();
```

**Run order:**
1. SHA-256 exact dedup (fast, catches identical files)
2. Playwright visual dedup (catches renamed/re-encoded copies)

### Phase 3: Build The Site

**3a. Create Project**
1. Create project directory in `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\{slug}\`
2. Add `.gitignore` with: `node_modules/`, `package-lock.json`, `package.json`, `*.mjs`, `*.png` screenshot patterns
3. Build `index.html` from the template, adapting ALL sections to the business type
4. Copy all photos into `images/` folder
5. Copy logo to `images/logo.png`

**3b. Adaptive Section Architecture**
The site sections adapt based on business type. Here's the universal structure with industry-specific variations:

**Universal sections (EVERY site gets these):**

| Section | Purpose | Adapts How |
|---|---|---|
| **Hero** | Full-viewport hero with best photo + zoom animation | Photo/headline changes per industry |
| **About** | Owner/team bio, 3+ paragraphs, feature grid | "Meet the Captain" → "Meet the Chef" → "Meet the Attorney" etc. |
| **Services/Offerings** | Card grid of what they offer | Species → Menu Items → Practice Areas → Treatments → Services |
| **Seasonal/Availability** | When to use their services | Fishing seasons → Seasonal menu → Court schedule → Class schedule |
| **Equipment/Facility** | Showcase their tools/space | Boat specs → Kitchen/dining → Office → Studio → Equipment |
| **Pricing** | 3-tier pricing cards with calculator | Trip packages → Service packages → Membership tiers |
| **Gallery** | Photo gallery with lightbox, deduplicated | Universal — works for any business |
| **Booking/Contact Form** | Lead capture with Netlify form handling | Trip booking → Reservation → Consultation → Appointment |
| **Testimonials** | 3 cards, 5 stars from real reviews | Universal |
| **Preparation/What to Expect** | Help customers prepare | "What to Bring" → "What to Expect" → "How to Prepare" |
| **Service Area** | City cards with drive times for local SEO | Universal — always research nearby cities |
| **FAQ** | 10-12 expandable accordion items | Questions adapted to industry |
| **CTA Banner** | Strong call-to-action with action photo | Universal |
| **Contact** | Phone, email, social, directions cards | Universal |
| **Footer** | Admin link, section links, social icons | Universal |

**Industry-specific section mapping:**

| Fishing Charter | Restaurant | Law Firm | Salon/Spa | Contractor | Generic |
|---|---|---|---|---|---|
| Species Guide | Menu Highlights | Practice Areas | Treatment Menu | Services | Services |
| Seasonal Calendar | Seasonal Specials | Case Results | Before/After | Project Gallery | Portfolio |
| The Boat | The Kitchen/Space | Our Attorneys | Our Stylists | Our Work | Our Team |
| What to Bring | Dining Info | Free Consultation | Aftercare Tips | Project Process | What to Expect |
| Fishing License | Reservation Policy | Legal Resources | Booking Policy | Licensing/Insurance | Resources |

**3c. SEO Content Sections (MANDATORY for every site)**

**1. Services/Offerings Section** (`#services`)
- Grid of cards (4-column) for each service/offering
- Each card: icon/emoji, service name, brief description, price or "from $X" if applicable
- Below the grid: 2-column text block (3-4 paragraphs) explaining why this business excels, their approach, experience, and unique value
- Target keywords: "[service] [location]", "[service] near me", "[business type] [city] [state]"

**2. Seasonal/Availability Section** (`#seasons` or `#availability`)
- 4 cards in a grid (seasons, quarters, or time-based categories relevant to the business)
- Each card: colored heading, description, service highlights for that period
- Below: 1-2 paragraphs about year-round availability + booking CTA
- Navy background section

**3. Equipment/Facility/Team Section** (`#facility` or `#team`)
- 2-column layout: photo (left) + text/specs (right)
- Name/title as heading, 2 paragraphs about capabilities
- Spec/detail grid relevant to industry
- Feature list with check icons

**4. Preparation Section** (`#prepare`)
- 3-column grid of checklist cards
- Items relevant to the specific business/service
- Navy background section

**5. Service Area / Area Guide** (`#area`)
- Grid of city cards (4-column) for every nearby city within reasonable distance
- Each card: city name, drive time, 3-4 sentences about why customers from that city should choose this business
- Below: 2-column text about the region
- Target keywords: "[business type] near [city]", "[service] [city] [state]"

**6. FAQ Accordion** (`#faq`)
- 10-12 expandable items with toggle animation
- Must cover: pricing, what's included, scheduling/hours, cancellation policy, service area, experience/qualifications, what to expect, preparation, payment methods, guarantees, group/corporate rates, accessibility
- Each answer: 3-5 sentences with internal links to other sections
- Duplicate key FAQs in Schema.org FAQ structured data (15+ questions)

**7. Schema & Meta:**
- Keywords meta: 30+ long-tail keywords covering services, locations, occasion types
- FAQ Schema: 15+ questions
- LocalBusiness schema (use most specific subtype: Restaurant, LegalService, HealthAndBeautyBusiness, HomeAndConstructionBusiness, etc.) with full address, geo, rating, offers, area served
- `sameAs` array with all social media URLs

**Section order in HTML:**
Nav → Hero → About → Services/Offerings → Seasonal/Availability → Equipment/Facility/Team → Pricing → Gallery → Booking/Contact Form → Testimonials → Preparation → Service Area → FAQ → Resource/License Banner (if applicable) → CTA Banner → Contact → Footer

### Phase 3d: Design & CSS

**Color scheme selection by industry:**
| Business Type | Primary | Accent | Vibe |
|---|---|---|---|
| Fishing/Outdoor | Navy #0a1628 | Gold #c8a96e | Nautical luxury |
| Restaurant/Bar | Dark charcoal #1a1a2e | Warm gold #d4a853 | Upscale dining |
| Law Firm | Dark navy #0d1b2a | Silver/gold #b8860b | Authority & trust |
| Salon/Medspa | Deep plum #1a0a2e | Rose gold #c9a87c | Elegant & modern |
| Contractor | Slate #1c2833 | Safety orange #e67e22 | Industrial premium |
| Gym/Fitness | Black #0a0a0a | Electric green #00ff88 | Energy & power |
| Medical | Dark blue #0a192f | Teal #0fbcf9 | Clinical & clean |
| Real Estate | Navy #0f2027 | Gold #c9b037 | Luxury property |
| Default | Navy #0a1628 | Gold #c8a96e | Premium dark luxury |

- If client has existing brand colors, use those instead (update CSS `:root` variables)
- Logo displays in nav (white bg on scroll) with text fallback
- Hero uses best showcase photo (MUST be >100KB)
- About section uses owner/team photo
- CTA banner uses action/portfolio photo

**CRITICAL NAV SCROLL FIX** — The scrolled nav turns white background. MUST add:
```css
.nav.scrolled .nav-logo { color: var(--navy); text-shadow: none; }
.nav.scrolled .nav-logo span, .nav.scrolled .nav-logo em { color: var(--gold); }
.nav.scrolled .mobile-toggle span { background: var(--navy); }
```

**CRITICAL MOBILE HERO STATS FIX** — Hero stats look messy on mobile. MUST add inside `@media (max-width: 768px)`:
```css
.hero h1 { margin-bottom: 1rem; }
.hero-stats { gap: 1.5rem; flex-wrap: wrap; justify-content: center; margin-top: 2rem; padding-top: 1.5rem; }
.hero-stat-num { font-size: 1.5rem; }
.hero-stat-label { font-size: 0.65rem; letter-spacing: 0.5px; }
```

**CRITICAL IMAGE EXTENSION CHECK** — After downloading photos, verify file extensions in the gallery array and all `src=` / `url()` references MATCH the actual file extensions on disk.

**MANDATORY DESIGN STANDARDS:**
- Use GSAP ScrollTrigger or Framer Motion for scroll animations — NO basic CSS-only transitions
- Smooth parallax effects on hero and CTA sections
- Staggered reveal animations on card grids
- Hover micro-interactions on all interactive elements
- Glass-morphism or gradient overlays where appropriate
- The site must feel premium and modern — if it looks like a basic template, it's not done

### Phase 4: Admin Dashboard

Copy `admin.html` from template (`C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\kjs-outdoor-adventures\admin.html`) and customize:

**Universal admin features (every site):**
- Password-protected login
- **Bookings/Leads tab**: stats, search/filter, expandable rows, confirm/cancel/edit
- **Calendar tab**: monthly view with color-coded entries. Click day → add appointment/booking or block day.
- **Traffic tab**: sessions, daily chart, top pages/sources/devices
- **Settings tab**: email notifications, Google Calendar OAuth, Stripe payments, PayPal payments, deposit email settings via Resend API

**Industry-specific admin additions:**
| Business Type | Extra Admin Features |
|---|---|
| Fishing Charter | Fleet management (multi-boat), per-boat calendar |
| Restaurant | Table management, reservation slots |
| Salon/Spa | Stylist scheduling, service duration tracking |
| Contractor | Project status tracking, estimate generation |
| Any with multiple providers | Provider management with color-coded calendars |

- Generate new admin password, update SHA-256 hash
- Update SITE_ID and NETLIFY_TOKEN after Netlify site creation
- Customize: title, branding text, business name, phone number

**CRITICAL `</title>` BUG**: When editing admin.html title, ALWAYS ensure closing `</title>` tag is present. Missing it breaks the entire page.

### Phase 5: Netlify Functions (REQUIRED)

Create `netlify/functions/` with:
1. **create-checkout.js** — Creates Stripe Checkout sessions for deposit/payment
2. **send-deposit-email.js** — Sends branded payment request emails via Resend API (update business name and phone)

Copy from: `C:\Users\admin\Desktop\Github Repo Backup 2-18-2026\kjs-outdoor-adventures\netlify\functions\`
Create **netlify.toml** with `[build] functions = "netlify/functions"`

### Phase 6: Deploy

1. `git init` + `git config` (user.email=ben@advancedmarketing.co, user.name=Benjamin Boyce)
2. `gh repo create bensblueprints/{slug} --public --source=. --remote=origin --push`
3. Create Netlify site via API:
   ```
   POST https://api.netlify.com/api/v1/sites
   Authorization: Bearer {NETLIFY_TOKEN}
   Body: { "name": "{slug}", "repo": { "provider": "github", "repo": "bensblueprints/{slug}", "branch": "master" } }
   ```
4. **CRITICAL**: Patch site to enable form processing:
   ```
   PUT https://api.netlify.com/api/v1/sites/{site_id}
   Body: { "processing_settings": { "html": { "pretty_urls": true }, "ignore_html_forms": false } }
   ```
5. Update `admin.html` with new SITE_ID
6. Deploy via file upload API (handles files with spaces via URL encoding):
   - Scan all files, compute SHA-1 digests
   - POST deploy with file digests
   - Upload required files (URL-encode paths with spaces/parens)
7. Commit + push to GitHub
8. Verify site is live

### Phase 7: Verify & QA (MANDATORY — ~20 minutes)

After EVERY deploy, run comprehensive Playwright QA on both desktop (1920x1080) and mobile (375x812).

**Create `qa.mjs` that tests:**

**Desktop (1920x1080):**
1. Hero section — screenshot, verify background image loads, headline visible, CTA buttons present
2. Nav on scroll — scroll 500px, verify logo/links visible on white background
3. All sections exist — scroll to each anchor and verify content present
4. Gallery — verify 8+ images load, lightbox works
5. Pricing cards — verify cards visible with amounts
6. Booking/contact form — fill test data, verify calculator/form works, do NOT submit
7. Footer — verify admin link, social icons, copyright
8. Social media links — verify hrefs exist
9. Directions link — verify Google Maps URL
10. FAQ accordion — click 3 items, verify expand/collapse
11. Service cards — verify correct count, each has content

**Mobile (375x812):**
12. Hero — verify text doesn't overflow, stats wrap cleanly
13. Hamburger menu — verify visible, opens fullscreen, readable links
14. Hamburger on scroll — verify visible on white background
15. Close menu — verify closes on click
16. Pricing cards — verify single-column stack
17. Gallery — verify 2-column grid
18. Form — verify fields don't overflow
19. Contact cards — verify single-column stack

**Admin Dashboard:**
20. Login page loads — verify password input visible
21. Login works — enter password, verify dashboard appears
22. All tabs functional — bookings, calendar, settings
23. Settings — verify Stripe/PayPal sections exist

```javascript
import { chromium } from 'playwright';
const browser = await chromium.launch();

const desktop = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
const dp = await desktop.newPage();
await dp.goto(SITE_URL, { waitUntil: 'networkidle' });
// ... desktop tests + screenshots ...

const mobile = await browser.newContext({ viewport: { width: 375, height: 812 }, isMobile: true });
const mp = await mobile.newPage();
await mp.goto(SITE_URL, { waitUntil: 'networkidle' });
// ... mobile tests + screenshots ...

const admin = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
const ap = await admin.newPage();
await ap.goto(SITE_URL + '/admin.html', { waitUntil: 'networkidle' });
// ... admin tests ...

await browser.close();
```

**On ANY failure:** Fix immediately, redeploy, re-run failing test. Do not skip failures.

### Phase 8: Post-Deploy

**Post-deploy checklist (ALL must pass):**
1. index.html loads with hero image
2. admin.html shows login screen
3. Gallery images not broken
4. Mobile nav works on scroll
5. GSAP/scroll animations fire correctly

## Auto-Add to Client Portal (MANDATORY)

After every new site build, add the client to the Advanced Marketing portal at `clients.advancedmarketing.co`.

**API Details:**
- Portal API: `https://portal-api.advancedmarketing.co`
- Login as admin: `POST /api/auth/login` with `{ email: "ben@advancedmarketing.co", password: "JEsus777$$!" }`
- Create client: `POST /api/auth/signup` with admin token in Authorization header

**Signup payload:**
```json
{
  "name": "Owner Name",
  "email": "client@email.com",
  "password": "same-as-admin-dashboard-password",
  "business_name": "Business Name",
  "phone": "(XXX) XXX-XXXX",
  "service_type": "web_design"
}
```

## Admin Password
Generate a random password for each new client:
```javascript
crypto.subtle.digest('SHA-256', new TextEncoder().encode('thepassword'))
```

## CSS for Adaptive Sections
Add these CSS blocks (reference `captain-hoggs-charters/index.html` for proven implementation):

**Required CSS blocks:**
- `.services` / `.services-grid` / `.service-card` — 4-column card grid with icon, title, description
- `.seasons` / `.season-grid` / `.season-card` — 4-column navy-bg cards with colored headings and tag pills
- `.facility` / `.facility-grid` / `.facility-specs` / `.facility-spec` — 2-column layout with spec cards
- `.area-guide` / `.area-cities` / `.area-city` — 4-column city cards with drive times
- `.faq` / `.faq-list` / `.faq-item` / `.faq-q` / `.faq-a` — accordion with max-height toggle
- `.prepare` / `.prepare-grid` / `.prepare-card` — 3-column navy-bg checklist cards
- All sections need `@media (max-width: 768px)` responsive overrides to single column

## Key Gotchas
- Netlify API site creation sets `ignore_html_forms: true` — MUST patch to false
- Files with spaces in names need URL-encoding when uploading via API
- Git branch is `master` not `main`
- Deploy via API file upload (not git-linked) since GitHub permissions are unreliable
- Admin statuses/calendar stored in localStorage (per-browser)
- Always push to GitHub after every change
- Always deploy after code changes
- **ALWAYS deduplicate photos** after downloading
- WordPress REST API is the fastest way to get all photos on WordPress sites
- Use Node.js for all scripts (Python is NOT installed)
- `tail`/`head` commands not available in Windows bash
- **CRITICAL `</title>` BUG**: Always verify closing `</title>` tag after customizing admin.html
- **ALWAYS DEPLOY admin.html** — verify it loads at `{site}/admin.html` after every deploy
- **ALWAYS DEPLOY Netlify Functions** — include in deploy with netlify.toml
- **NEVER build a basic-looking site** — must use GSAP/Framer Motion, premium animations, dark luxury aesthetic. If it looks like a template, it's not done.
