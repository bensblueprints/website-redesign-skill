# Website Redesign Skill - Claude Code

A Claude Code slash command (`/website-redesign`) that scrapes any business's existing website and rebuilds it as a premium dark luxury site with booking platform and admin dashboard.

## What It Does

Give it any business website URL and it will:

1. **Research** the business - auto-detect industry, scrape info, photos, reviews, social links
2. **Build** a complete SEO-optimized website (2,500+ lines of HTML/CSS/JS) adapted to the business type
3. **Deploy** to Netlify with custom domain support
4. **Set up** an admin dashboard with booking management, calendar, and payment processing

## Supported Business Types

Works for any business — auto-detects and adapts:

- Fishing/Hunting Charters
- Restaurants & Bars
- Law Firms
- Salons & Barbershops
- Contractors & Home Services
- Gyms & Fitness Studios
- Medspas & Medical Practices
- Auto Shops & Dealers
- Real Estate Agents
- Photographers & Creatives
- And any other local business

## Features

### Public Website
- Full-viewport hero with GSAP scroll animations
- Owner/team bio with real photos (auto-identified from scraped images)
- Industry-adaptive service cards with descriptions
- Seasonal/availability section
- Equipment/facility showcase with specs
- 3-tier pricing cards with live calculator
- Photo gallery with lightbox (auto-deduplicated via perceptual hashing)
- Booking/contact form with Netlify form handling
- Testimonials from real reviews
- Preparation/what-to-expect checklist
- Service area guide with nearby cities and drive times
- 12-item FAQ accordion
- Schema.org structured data (industry-specific type + FAQPage, 15+ questions)
- 30+ long-tail SEO keywords
- Fully responsive with mobile-first design
- Premium dark luxury aesthetic with industry-matched color scheme

### Admin Dashboard
- Password-protected login
- Booking/lead management (confirm, cancel, edit, calendar sync)
- Color-coded calendar with provider/resource filtering
- Stripe deposit payment requests
- PayPal deposit payment links
- Branded email notifications (via Resend API)
- Traffic analytics (sessions, sources, devices)
- Google Calendar OAuth integration

### Netlify Functions
- `create-checkout.js` - Stripe Checkout session creation
- `send-deposit-email.js` - Branded HTML deposit emails via Resend

## Usage

### As a Claude Code Slash Command

1. Copy `SKILL.md` to `~/.claude/commands/website-redesign.md`
2. In Claude Code, run: `/website-redesign https://example-business.com`
3. Wait for the full build, deploy, and verification

### What You Need

- Node.js (for Playwright photo scraping)
- GitHub CLI (`gh`) for repo creation
- Netlify account with API token
- Git configured

## Built By

[Advanced Marketing](https://advancedmarketing.co) - Web design and digital marketing for local businesses.
