# puppeteer proxy: how to wire one up without 407s, rotate real residential IPs per request, and pick a plan that matches your scraping volume

Your script ran clean for 180 requests. Request 181 came back 403, and request 190 was a CAPTCHA. That moment is when most people start searching for a Puppeteer proxy setup, and it's also when they discover that Puppeteer's proxy support has a couple of sharp edges that aren't obvious from the docs.

The flag itself takes thirty seconds to add. The part that eats an afternoon is authentication, because Chrome silently ignores credentials you pass inline, and Puppeteer never tells you why. The other part is choosing a proxy that survives real traffic instead of dying after ten minutes.

This covers the setup, the authentication trap, how rotating and sticky sessions map onto 9Proxy's two residential models, the full current package ladder, and the failure messages you'll actually see when something is wrong.

## What a proxy does for Puppeteer, and what it doesn't

A proxy changes the IP address the target site sees. That's it. It doesn't change your TLS fingerprint, your user agent, your viewport, or the fact that headless Chrome announces itself in a dozen small ways.

That distinction matters when you're debugging. If a site blocks you *and* you're behind a residential IP, the problem is almost certainly in your browser fingerprint, not your network layer. If a site blocks you with a datacenter IP, the IP reputation is usually the culprit. Those are two different fixes.

Also worth setting expectations: proxies route traffic, they don't slow sites down on purpose. If you notice big latency, it's usually the proxy's gateway, distance to target, or a serialised request pipeline in your own code.

## Three places a proxy can attach in Puppeteer

There's no single "proxy setting." There are three, and they solve different problems.

**1. The launch argument.** Applies to the whole browser instance. Simplest option, works with HTTP, HTTPS and SOCKS5 endpoints.

js
const browser = await puppeteer.launch({
  headless: 'new',
  args: ['--proxy-server=http://gateway.example.com:8000'],
});


Use it when every request from that browser should go through the same exit.

**2. Page-level credentials.** `page.authenticate()` supplies the username and password for a proxy that requires them. It's the standard pairing with the launch flag.

**3. Per-request routing.** Packages like `puppeteer-page-proxy` (or the `puppeteer-proxy` / `puppeteer-extra-plugin-page-proxy` variants) intercept requests and re-issue them from Node instead of from the browser. That's how you get a different proxy per request or per page. The trade-off is real: every request/response now bounces between Chrome and Node, so throughput drops. For a script that rotates an IP on every page load, it's often worth it. For a script making thousands of requests a minute, it usually isn't.

## The credential trap that ruins the afternoon

This is the single most common Puppeteer proxy bug:

js
// This does not work. Chrome ignores the credentials and drops them.
args: ['--proxy-server=http://myuser:mypassword@gateway.example.com:8000']


Chrome doesn't support inline `user:pass@` inside `--proxy-server`. It doesn't throw an error either. Requests simply fail with a 407, which gets misread as "the proxy is down" or "my credentials are wrong."

The working pattern is to pass the endpoint without credentials and authenticate on the page, before you navigate:

js
const browser = await puppeteer.launch({
  args: ['--proxy-server=http://gateway.example.com:8000'],
});
const page = await browser.newPage();

await page.authenticate({
  username: process.env.PROXY_USER,
  password: process.env.PROXY_PASS,
});

await page.goto('https://ipinfo.io/json');


Order matters. `authenticate()` has to be called before `goto()`. Reverse them and the first request goes out unauthenticated, so you'll see one failure before everything starts working, which is a confusing thing to debug.

If you'd rather not touch page objects at all, `proxy-chain` will "anonymise" an authenticated proxy into a local unauthenticated one:

js
const proxyChain = require('proxy-chain');
const newUrl = await proxyChain.anonymizeProxy(
  'http://user:pass@gateway.example.com:8000'
);
// pass newUrl into --proxy-server


Two things to remember there: every `anonymizeProxy()` call opens a local server on a random port, so call `closeAnonymizedProxy()` in a `finally` block or you'll leak ports over a long-running job. And if you install `proxy-chain` v3, note that it dropped CommonJS support, so `require()` fails. Pin v2 if you're on CommonJS.

Last piece of housekeeping: keep credentials in environment variables. Hardcoding them in the script is how they end up in a commit.

## Rotating vs sticky: decide this before you shop

Proxy vendors sell "rotation" as a feature, but rotation and persistence solve opposite problems.

- **Rotating** — a new IP per request. Right for scraping catalogues, price checks, SERP monitoring, anything where no single request depends on the previous one.
- **Sticky** — the same IP held for a fixed window. Required for logins, shopping carts, multi-step forms, anything with session state, and most platforms with serious bot detection.

A script doing both at once — some tasks that need continuity, others that need volume — is normal, and it's the case that forces you to look at how a provider structures its plans rather than just picking the cheapest per-gigabyte rate.

## How 9Proxy's two residential models map onto Puppeteer

9Proxy, a residential proxy provider with a pool advertised at 20M+ IPs across 90+ countries, sells two distinct products. They are not interchangeable from a Puppeteer point of view.

**Residential by GB.** Billing is by traffic. You get a fixed hostname and port from your dashboard, then build targeting into the username. Traffic rotates through residential IPs, either per request or held sticky for a session. Authentication is username/password or an IP whitelist. Nothing to install. This is the model that drops straight into `--proxy-server` + `page.authenticate()`, and it's the only one that works if your Puppeteer jobs run in Docker or on a headless Linux box.

**Residential by IPs.** Billing is per IP, with unlimited traffic on each IP while it's active. The catch: it runs through the 9Proxy desktop app, which forwards IPs to local ports on your machine. Puppeteer then points at `127.0.0.1:<port>` instead of a remote gateway. IPs stay live anywhere from a few hours to roughly 24 hours, unused IPs don't expire, and auto-rotation can be configured on selected ports. There's also an app feature called Today List that keeps recently used proxies reusable for 24 hours at no extra cost, which is handy when a script finishes and you want to re-run it with the same IPs.

The practical implication: if you're deploying to CI, a VPS, or a Kubernetes job, the IP-based model is a non-starter because the app is a desktop install. Use it when the Puppeteer script runs on your own workstation or a Windows/macOS machine you control.

👉 Compare the two 9Proxy models on the signup page before you decide

## Wiring 9Proxy into a Puppeteer script

The GB-based endpoint takes a structured username. Every targeting option lives inside it, so you don't need extra API calls:


<sub-user>-country-<code>-st-<state>-city-<city>-isp-<isp>-sst-<minutes>-ssid-<id>


- **sub-user** — your sub-account name from the dashboard. Always present.
- **country** — two-letter country code. `country-us`, `country-de`, `country-vn`.
- **st** — optional state or region filter. `st-ohio`.
- **city** — optional city filter, underscores for spaces. `city-newyork`.
- **isp** — optional filter by ISP name or ASN. `isp-as22773_Cox_Communications_Inc.` Useful when your target fingerprints by network operator.
- **sst** — session duration in minutes. Required for sticky sessions.
- **ssid** — optional session ID. Lets you hold several sticky IPs from the same configuration at once.

Leave out `sst` and `ssid` and you get a fresh IP on every request:

js
const browser = await puppeteer.launch({
  args: ['--proxy-server=http://your_host:your_port'],
});
const page = await browser.newPage();
await page.authenticate({
  username: 'myuser-country-us-city-newyork',
  password: 'your_password',
});
await page.goto('https://ipinfo.io/json');


Add `sst` and `ssid` and the IP holds for the session window, with a distinct IP per `ssid`:

js
username: 'myuser-country-us-sst-30-ssid-worker3'


That `ssid` detail is what makes parallel Puppeteer workers manageable. Four instances of the same job each get their own sticky IP as long as they use different session IDs — run them on the same config and they'd otherwise fight over one address.

Before any of this, test the endpoint outside Puppeteer. A single curl call tells you whether the credentials and host are right, which is much faster than reading Puppeteer's error output:

bash
curl -x your_host:your_port \
     -U "myuser-country-us-sst-15-ssid-bot01:your_password" \
     https://ipinfo.io


If that returns a residential IP, everything above will work. If it returns a 407, the problem is the credentials — not Puppeteer.

One note on SOCKS5. Puppeteer accepts `--proxy-server=socks5://host:port`, and 9Proxy does offer SOCKS5 endpoints, which is useful for antidetect browser workflows. But the same Chrome limitation applies to authenticated SOCKS5, so if your endpoint requires a username and password, prefer the HTTP endpoint when you're driving Chrome from Puppeteer.

👉 Set up a 9Proxy endpoint and confirm it with that curl test first

## What 9Proxy costs

9Proxy doesn't sell monthly subscriptions. You top up a balance and buy packages, which means the numbers below are one-off purchases rather than recurring charges. That matters for budgeting: unused IPs never expire, and GB packages have a validity window instead of a renewal date.

One thing to know before reading the table: 9Proxy announced its first price change in three years, effective 1 June 2026, affecting IP-based and bundle packages. GB-based pricing was left untouched. The figures below reflect post-update pricing as reported in 2026 third-party breakdowns; the mid-tier IP rungs are the ones most worth confirming in your own dashboard before a large top-up.

| Package | Model | What you get | Price (USD) | Validity | Buy |
| --- | --- | --- | --- | --- | --- |
| 5 GB | GB-based | 5 GB rotating residential traffic, $3.00/GB | $15 | 180 days | Get the 5 GB pack |
| 50 GB + 5 GB | GB-based | 55 GB total, $2.10/GB | $105 | 180 days | Get 55 GB |
| 100 GB | GB-based | 100 GB, $1.50/GB | $150 | 180 days | Get 100 GB |
| 200 GB | GB-based | 200 GB, $1.00/GB | $200 | 180 days | Get 200 GB |
| 1,000 GB | GB-based | 1,000 GB, $0.80/GB | $800 | 180 days | Get 1,000 GB |
| 2,000 GB | GB-based | 2,000 GB, $0.75/GB | $1,500 | 180 days | Get 2,000 GB |
| 3,000 GB | Enterprise GB | 3,000 GB, $0.72/GB | $2,160 | No expiry | Get 3,000 GB |
| 6,000 GB | Enterprise GB | 6,000 GB, $0.70/GB | $4,200 | No expiry | Get 6,000 GB |
| 10,000 GB | Enterprise GB | 10,000 GB, $0.68/GB | $6,800 | No expiry | Get 10,000 GB |
| 100 IPs | IP-based | 100 residential IPs, unlimited data per IP | $24 | Unused IPs don't expire | Get 100 IPs |
| 500 IPs | IP-based | 500 IPs | $72 | Unused IPs don't expire | Get 500 IPs |
| 1,000 + 500 IPs | IP-based | 1,500 IPs (500 bonus) | $126 | Unused IPs don't expire | Get 1,500 IPs |
| 2,500 IPs | IP-based | 2,500 IPs | $210 | Unused IPs don't expire | Get 2,500 IPs |
| 5,000 IPs | IP-based | 5,000 IPs | $360 | Unused IPs don't expire | Get 5,000 IPs |
| 15,000 IPs | IP-based | 15,000 IPs | $720 | Unused IPs don't expire | Get 15,000 IPs |
| 25,000 IPs | IP-based | 25,000 IPs | $863 | Unused IPs don't expire | Get 25,000 IPs |
| 50,000 IPs | IP-based | 50,000 IPs | $1,438 | Unused IPs don't expire | Get 50,000 IPs |
| 100,000 IPs | Business IP | 100,000 IPs | $2,300 | Unused IPs don't expire | Get 100,000 IPs |
| 200,000 IPs | Business IP | 200,000 IPs | $4,140 | Unused IPs don't expire | Get 200,000 IPs |
| 500,000 IPs | Business IP | 500,000 IPs | $8,625 | Unused IPs don't expire | Get 500,000 IPs |
| Starter bundle | Bundle | 100 IPs + 5 GB | $30 | 180-day traffic validity | Get the Starter bundle |
| Popular bundle | Bundle | 1,500 IPs + 50 GB | $180 | 180-day traffic validity | Get the Popular bundle |
| Pro bundle | Bundle | 5,000 IPs + 500 GB | $720 | 180-day traffic validity | Get the Pro bundle |

Two comparison notes that aren't obvious from the ladder. The bundle tiers are priced below the cost of buying their two halves separately, which is the main reason to consider them if your scripts genuinely need both stable IPs and raw volume. And the enterprise GB tiers are the only packages where traffic never expires, which changes the maths for teams whose monthly usage is spiky rather than steady.

9Proxy lists the pool at 20M+ residential IPs across 90+ countries, with targeting down to country, state, city, postcode and ISP, and supports both HTTP/HTTPS and SOCKS5. Payment options include cards, crypto, bank cards, Alipay, Apple Pay and Google Pay. Support runs through email (`support@9proxy.com`), live chat and a Telegram channel.

On trials: 9Proxy doesn't publish a permanent free tier. The vendor states that a limited trial is available for new users subject to availability, and third-party listings describe a one-off paid trial starting around $0.02. If you just want to validate that your Puppeteer setup works end to end, the 5 GB package at $15 is the cheaper way to be sure you're not fighting a trial quota while debugging.

## Matching a package to a Puppeteer workload

The decision tree is short.

**Prototyping or a one-off scrape.** 5 GB at $15. Rotating username, no desktop app, works inside a container. If you burn through it in an afternoon of testing, that's information about your request volume, not about the provider.

**Recurring jobs at steady volume.** 100 GB ($150) or 200 GB ($200). The per-gigabyte rate roughly halves between 5 GB and 200 GB, so this is where the pricing model starts paying for itself.

**Sustained sessions with heavy transfer.** IP-based packages. Unlimited data per IP during the active window is the specific advantage here: a job that downloads large pages or streams data through one session costs nothing extra in bandwidth, which is not true on a GB-metered plan. Start at 100 IPs for $24 and scale from there.

**Continuous pipelines.** Either the enterprise GB tiers, if you're bandwidth-bound and want traffic that never expires, or the 100,000+ IP tiers, if you need tens of thousands of distinct addresses in rotation.

The rule of thumb: count the bytes per task, not the number of tasks. Small requests that each want a fresh IP are cheaper on GB. Large transfers that need a stable address are cheaper on IP.

👉 Pick a 9Proxy package sized to your actual request volume

## Errors you'll actually see, and what they mean

**407 Proxy Authentication Required.** Credentials are missing, malformed, or being ignored. In Puppeteer that usually means inline `user:pass@` in the launch flag, or `authenticate()` called after `goto()`.

**ERR_NO_SUPPORTED_PROXIES.** Chrome couldn't parse the proxy string. Nine times out of ten the scheme is missing — `host:port` instead of `http://host:port`. A trailing comma or a space-separated list instead of a comma-separated one causes it too.

**ERR_TUNNEL_CONNECTION_FAILED.** The proxy is reachable but refused or dropped the CONNECT for that specific host. Often an expired IP, an exhausted traffic balance, or a target the exit won't route to.

**Cert warnings or SSL errors after the proxy works.** Some proxy paths break certificate validation. Adding `ignoreHTTPSErrors: true` will get you moving, but it's a diagnostic step, not a fix — understand why the chain broke before shipping it.

**Works locally, fails in Docker.** This is almost always the IP-based model. The desktop app forwards to `127.0.0.1`, and inside a container, localhost is the container. Switch to GB-based, which needs no local software.

**Random 403s despite a residential IP.** Forcing `headless: 'new'` rather than the legacy mode helps, as does setting a realistic user agent and viewport instead of leaving defaults. Add randomised delays between navigations, and stop reusing one IP at high frequency — even a clean residential address gets flagged if you hit a target fifty times a second.

## FAQ

**Can I use a different proxy for each page in Puppeteer?**
Not through the launch argument, which applies to the whole browser. Use request interception with `puppeteer-page-proxy` or manage it per request via `page.setRequestInterception()`, accepting the added latency.

**Does Puppeteer support SOCKS5 proxies?**
Yes, via `--proxy-server=socks5://host:port`. Authenticated SOCKS5 hits the same Chrome limitation as authenticated HTTP, so for credentialed access from Puppeteer, use the HTTP endpoint.

**Do I need to install anything for 9Proxy to work with Puppeteer?**
Only for the IP-based model, which relies on the desktop app for port forwarding. The GB-based model is dashboard-only and needs no local software.

**How do I keep parallel Puppeteer workers from sharing one IP?**
Use sticky mode with a distinct `ssid` per worker, for example `ssid-worker1` through `ssid-worker8`. Same config, different session IDs, different IPs.

**Is a proxy enough to avoid detection on its own?**
No. A proxy fixes your network identity. Fingerprinting, behaviour timing and request patterns are separate problems.

**Which plan should a single developer start with?**
The 5 GB package. It's $15, it needs no app, it runs in any environment Puppeteer runs in, and it answers the only question that matters at that stage — whether your script's problem was ever the IP.

👉 Create a 9Proxy account and test it against your own script
