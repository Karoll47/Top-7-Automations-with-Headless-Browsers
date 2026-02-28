# Top 7 Automations with Headless Browsers

*Seven patterns that actually come up, and what breaks when you build them.*

If you've ever tried to scrape a React app with `requests`, you know the pain. You get back an empty div and a bundle of JavaScript that does nothing outside a real browser. Headless browsers solve this — they run a full browser engine, execute JavaScript, handle cookies, and let you interact with pages like a real user would. Just without the screen.

The problem is running them at scale. Spinning up Chrome in a Docker container works fine for a one-off script. But once you need proxy rotation, CAPTCHA handling, parallel sessions, and reliable uptime, the infrastructure becomes the job. You end up spending more time debugging zombie Chrome processes and rotating proxies than actually building the thing you set out to build.

That's where [Steel.dev](https://steel.dev) comes in. It's a managed cloud platform for headless browser sessions — you connect via WebSocket, use whatever automation framework you already know (Puppeteer, Playwright, Selenium), and Steel handles the infrastructure underneath. No Docker, no proxy management, no CAPTCHA service accounts.

Seven patterns below, with the bits that actually trip people up.

---

## 1. Price Monitoring & Product Tracking

Price monitoring sounds simple until you actually try to do it at scale. Product pages are JavaScript-rendered, prices update dynamically, and e-commerce sites are some of the most aggressive when it comes to detecting and blocking scrapers. Amazon, for example, will silently serve you stale cached data or block your session entirely if it decides your traffic looks automated.

The flow itself is boring — open a session, wait for the price element, extract, store, repeat. Keeping it reliable over weeks without getting flagged is the actual problem.

```js
const session = await steel.createSession({ proxy: true, captcha: true })

await page.goto('https://amazon.com/dp/B09X')
await page.waitForSelector('.a-price-whole')

const price = await page.evaluate(() => {
  // extract live-rendered price from DOM
  return document.querySelector('.a-price-whole').innerText
})

if (price < TARGET) await sendAlert(price)
```

- **Set up a scheduled job** — cron or a queue worker that spins up a fresh Steel session for each check. Don't reuse sessions across runs.
- **Wait for JS render** — use `networkidle2` or a specific selector wait, never a fixed `setTimeout`. Pages load at different speeds.
- **Diff the results** — store previous values and alert only on meaningful changes. Price fluctuations of a few cents are noise.
- **Handle "price unavailable"** — products go out of stock, get delisted, or show regional prices. Your extractor needs to handle null gracefully.

**Common gotcha:** Sites like Amazon personalise prices based on your account, location, and browsing history. If you're not using proxies that match your target region, you'll scrape the wrong price every time.

> → A common setup: nightly price checks across 5 competitors, stored in Postgres, with a Slack alert when any SKU drops more than 10% from its 7-day average.

---

## 2. Form Autofill and Submission Workflows

Some forms simply cannot be automated with raw HTTP requests. Multi-step wizards carry state between pages in ways that aren't obvious from the outside, CSRF tokens rotate on every page load, and dropdowns often trigger XHR requests on change that load the next set of options. A headless browser handles all of this naturally because it's running the actual page — the same way a user would.

Steel sessions persist cookies and `localStorage` across the full form flow, so you stay authenticated from step 1 through to the confirmation page. Worth knowing: a lot of multi-step forms will silently kick you back to the start if the session drops between steps — no error, just back to step 1.

```js
const session = await steel.createSession()

await page.goto('https://ats.example.com/apply')
await page.fill('#email', candidate.email)
await page.fill('#password', candidate.password)
await page.click('#sign-in')

await page.waitForSelector('.step-2')
await page.fill('#first-name', candidate.firstName)
// ... continue through each step

await page.click('#submit')
const confirmation = await page.waitForSelector('.confirmation-number')
```

- **Keep one session per submission** — don't create a new session mid-flow or you'll lose auth state and probably land back at step 1.
- **Use `page.waitForSelector()`** between steps — confirm the next page has actually loaded before interacting with it. Clicking into a form that hasn't rendered yet causes hard-to-debug failures.
- **Capture the confirmation** — screenshot or extract the confirmation number before closing the session. You can't go back and get it.
- **Add retry logic at the step level** — not at the whole form level. Retrying from the beginning wastes time and can trigger duplicate submission detection.

**Common gotcha:** File upload fields. Most automation frameworks handle text inputs fine but struggle with `<input type="file">`. You need to use `page.setInputFiles()` rather than trying to click the upload button and interact with the OS file picker.

> → Job application bots that fill candidate profiles into ATS portals are the obvious use case. What takes 20 minutes per application manually takes about 4 seconds automated.

---

## 3. Dynamic Web Scraping for SPA Sites

This is the core use case headless browsers were basically invented for. If a site builds its content in React, Vue, or Angular, the server response is usually just a shell — the actual data gets fetched and rendered by JavaScript after the initial page load. A simple HTTP scraper only gets the empty shell.

The tempting alternative is to reverse-engineer the underlying API calls. Sometimes this works great and you end up with something much faster. But sites change their internals, add auth to endpoints, or rotate request signatures — and suddenly your scraper breaks on a Tuesday morning and you have no idea why. Scraping the rendered DOM is slower but it stays working.

```js
await page.goto('https://listings.io/search?city=split')
await page.waitForSelector('.listing-card')  // wait for React to render

const listings = await page.evaluate(() => {
  return [...document.querySelectorAll('.listing-card')].map(el => ({
    title: el.querySelector('.title').innerText,
    price: el.querySelector('.price').innerText,
    rating: el.querySelector('.rating').innerText,
  }))
})
```

- **Wait for the right signal** — `networkidle2` works for most sites, but for data-heavy SPAs you often need to wait for a specific element or a custom data attribute that only appears after the XHR completes.
- **Paginate carefully** — SPAs commonly use infinite scroll or client-side routing. Trigger scroll events programmatically or click "load more" rather than changing the URL directly, which can break the app state.
- **Let Steel handle the proxy layer** — large crawls across many pages will get your IP blocked quickly. Rotating manually is a full-time job.
- **Don't over-extract** — pull only what you need. Grabbing the entire DOM for every page eats memory and slows everything down.

**Common gotcha:** Infinite scroll that loads content lazily will give you incomplete data if you extract too early. Scroll to the bottom, wait for the new items to load, then extract. Repeat until you hit the end of the list.

> → Real estate aggregators pulling listings from SPA-based portals are the textbook example. A pipeline that runs nightly, extracts listings across 10 portals, and deduplicates by address is maybe 200 lines of code.

---

## 4. AI Web Agents

A year ago this section wouldn't have existed. Now it's one of the more interesting things you can do with a headless browser. Instead of writing explicit automation logic — click this button, fill this field, extract that selector — you give an LLM a task and let it figure out the steps using a live browser session. The agent reads the page, decides what to do, acts, observes, and repeats.

The [browser-use](https://github.com/browser-use/browser-use) library is the cleanest implementation of this right now. You hand it a Steel session and a task description, and the agent handles the navigation loop. It's not magic — it breaks on unusual UIs, struggles with CAPTCHAs, and needs guardrails to avoid going off-script — but for research and data gathering tasks it can compress weeks of scraper development into hours.

```py
from browser_use import Agent
from steel import Steel

session = Steel().create_session()

agent = Agent(
    task="Find the pricing page and extract all plan names and monthly prices",
    browser_session=session,
)

result = await agent.run()
# { "Starter": "$29/mo", "Pro": "$79/mo", "Enterprise": "custom" }
```

- **Give the agent a constrained task** — *"find the pricing page and extract plan names and prices"* works. *"Research this company"* does not. Ambiguous tasks produce unpredictable navigation paths.
- **Use Steel sessions for isolation** — each agent run should get its own session so parallel tasks don't share cookies or browser state.
- **Always validate output** — LLM agents hallucinate. Cross-check extracted data against what's actually in the DOM before storing or acting on it.
- **Set a step limit** — without a hard cap on navigation steps, an agent can loop indefinitely on a confusing UI. Set `max_steps=20` or similar and handle the timeout gracefully.

**Common gotcha:** Agents struggle with sites that require login. Either pre-authenticate the session before handing it to the agent, or give the agent the credentials explicitly — but be careful about what you log.

> → Competitive intelligence is the obvious use case: agent checks 10 competitor sites weekly, extracts pricing and feature lists, and outputs a structured comparison without anyone writing site-specific scrapers.

---

## 5. Screenshot & PDF Generation

`page.screenshot()` and `page.pdf()` are two of the most underrated methods in Puppeteer. A surprising amount of reporting and archival work that currently lives in custom PDF libraries, wkhtmltopdf, or locally-running Puppeteer scripts can be moved to Steel sessions with almost no code changes.

If you've run headless Chrome on a server before, you know: memory leaks, zombie processes, pages that hang and never resolve. Every clean render requires babysitting the process. Steel gives you a fresh isolated session per request — that problem just goes away.

```js
await page.goto('https://myapp.com/invoice/1042')
await page.waitForNetworkIdle()

// full-page screenshot → S3
const screenshot = await page.screenshot({ fullPage: true, type: 'png' })
await s3.upload({ Body: screenshot, Key: `invoices/1042.png` })

// or export as PDF
const pdf = await page.pdf({ format: 'A4', printBackground: true })
await s3.upload({ Body: pdf, Key: `invoices/1042.pdf` })
```

- **Wait for fonts and images** — screenshots taken before assets load look broken. Use `networkidle0` for pixel-perfect renders where every asset must load before capture.
- **Set viewport explicitly** — don't assume a default. Define width and height upfront so screenshots are consistent across runs and don't depend on whatever the default happens to be.
- **Pipe to storage immediately** — stream the buffer directly to S3. Don't write to disk on the session machine; it creates cleanup problems and slows things down.
- **Handle auth** — if you're screenshotting pages that require login, pass the auth token as a cookie or header before navigating. Steel sessions support this natively.

**Common gotcha:** `printBackground: true` is off by default in `page.pdf()`. If your page uses background colours or images for layout, your PDFs will look completely different from the actual page without this flag.

> → Visual regression testing and daily landing page snapshots are the easy wins. Invoice generation, contract PDFs, and automated report exports are where this pattern really earns its keep.

---

## 6. Mobile-Mode Automation

Spoofing a user agent string and calling it "mobile mode" doesn't work. Sites detect fake mobile sessions by checking touch support, screen dimensions, device memory, and hardware concurrency — not just the `User-Agent` header. Steel's mobile sessions emulate all of these at the browser level, so the site sees a genuine mobile device.

Some sites ship a completely different codebase on mobile — different React components, different API endpoints, sometimes different auth flows. If you're automating a checkout that's only streamlined on mobile, a desktop browser with a spoofed header won't cut it.

```js
const session = await steel.createSession({ mobile: true })
// viewport: 390×844, touch: true, userAgent: iPhone 14 iOS 17
// deviceMemory: 4, hardwareConcurrency: 6

await page.tap('#checkout-btn')  // touch event, not click
await page.waitForSelector('.apple-pay-btn')
```

- **Use touch events, not mouse events** — mobile sites expect `tap` and `touchstart` for some interactions, particularly on custom UI components. `click` works on standard elements but fails on some touch-only controls.
- **Check for mobile-specific endpoints** — open the network tab on a real phone and compare the API calls to what you see on desktop. You might be hitting completely different services.
- **Test both paths separately** — desktop and mobile codepaths diverge over time. A change to the desktop checkout won't necessarily affect mobile and vice versa.
- **Be aware of viewport-triggered lazy loading** — mobile sites often defer more aggressively. Scroll to trigger loads before extracting.

**Common gotcha:** Some sites use progressive web app features like service workers that behave differently in a headless context. If you're seeing unexpected caching behaviour, disable service workers with `page.evaluateOnNewDocument(() => { delete navigator.serviceWorker })`.

Retail checkout flows are the main use case. Fewer fields, one-tap payment options — simpler to automate and less likely to break than the desktop equivalent.

---

## 7. Uptime Monitoring & Incident Alerting

A ping-based monitor tells you when your server stops responding. It won't tell you when the checkout button disappeared, the price element is missing, or your login form is returning 200s that are actually error pages. To a ping monitor, a broken React app looks exactly like a working one.

Headless browser monitors actually load the page, render it, and assert on specific DOM elements and content. You find out about broken user flows before your users do — or before your on-call engineer gets woken up at 2am because cart abandonment suddenly spiked.

```js
// runs every 10 minutes via cron
const session = await steel.createSession()
await page.goto('https://mystore.com')

const priceVisible = await page.$('.product-price') !== null
const ctaVisible   = await page.$('.add-to-cart') !== null

if (!priceVisible || !ctaVisible) {
  const snap = await page.screenshot({ fullPage: true })
  await s3.upload({ Body: snap, Key: `incidents/${Date.now()}.png` })
  await slack.alert('#incidents', 'Homepage check failed — missing elements')
}
```

- **Assert on content, not just status codes** — a 200 with a broken React app looks identical to a healthy page from the outside. Check that the elements you care about are actually present and contain expected content.
- **Check the full critical path** — homepage → product page → add to cart → checkout entry is dramatically more valuable than just pinging the homepage. Most outages affect specific flows, not the whole site.
- **Fire on two consecutive failures** — transient issues (a slow CDN, a momentary database blip) will cause false positives if you alert on the first failure. Two consecutive failures before alerting cuts noise without meaningfully delaying real incident detection.
- **Capture a screenshot on failure** — the first thing anyone asks during an incident is "what does it look like?". Automating that screenshot saves the first five minutes of every incident response.

**Common gotcha:** Don't run your synthetic monitor from the same infrastructure as the service it monitors. If your server goes down, your monitor goes down too. Steel's cloud sessions run independently of your infrastructure, so you'll catch outages even if your own servers are completely unreachable.

> → Every 10 minutes, load the checkout page, assert the product price and CTA button are present, and fire a Slack alert with a screenshot if either is missing.

---

## Why Steel.dev?

None of these automations are new. People have been building price monitors and form bots with self-hosted Puppeteer for years. The difference with Steel is how much you don't have to think about.

Memory leaks, zombie processes, session isolation, proxy rotation, CAPTCHA solvers, pages that hang forever — all of that still exists, Steel just deals with it so you don't. Your Puppeteer or Playwright scripts connect via WebSocket and mostly just work. The things that used to mean stitching together a browser runner, a proxy provider, and a CAPTCHA service are one `createSession()` call.

| Feature | What it means in practice |
|---|---|
| **Cloud-managed sessions** | No Docker, no Chrome install, no process management |
| **Anti-bot evasion** | Proxy rotation + CAPTCHA solving without separate service accounts |
| **Session persistence** | Cookies and `localStorage` survive across requests in the same session |
| **Framework agnostic** | Puppeteer, Playwright, Selenium — connect via WebSocket, no code changes |

The infrastructure for headless automation is a solved problem. Steel just means you don't have to solve it yourself.