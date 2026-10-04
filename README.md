# sticky proxy: how to pin one IP to a whole session, and when to let it rotate

Search for "sticky proxy" and you get two very different kinds of results: provider sales pages promising "stickiness," and forum threads where someone's bot got logged out mid-session and nobody can explain why. Both are about the same thing — whether the exit IP stays put — but only one of them tells you what actually breaks when it doesn't.

So let's start there, then get into how to configure a sticky session properly, what session lengths are realistic, and how the pricing model behind it changes the cost math.

## What "sticky" actually means

A sticky proxy keeps the same exit IP for the life of a session. A rotating proxy hands you a new one — either per request or on a timer.

The reason this matters more than it sounds is that a browser visit is not one request. Loading a page means a document, then dozens of subresources, XHR calls, sometimes a WebSocket, all from the same tab. Three things in that stream assume the requests keep coming from the same place:

- The session cookie the site set on the first response gets presented on every request after it.
- Any auth or CSRF token minted during your visit is bound to the context it was created in.
- The Referer chain carries the previous URL into each navigation, so the requests form a path rather than a pile of unrelated hits.

Rotate the exit in the middle of that and you hand the site a visitor whose IP contradicts its own cookies: the cookie says this browser arrived from address A a minute ago, and the request carrying it leaves from address B, in another city, on another network. Nothing in the browser changed. That contradiction is a cheap signal to read — cheaper than any in-page fingerprint.

That's the honest version of why sticky exists. It isn't a feature you buy because it sounds more advanced. It's the default assumption a browser session already makes.

## Sticky vs rotating: which one your workload actually wants

|  | Sticky | Rotating |
| --- | --- | --- |
| IP behavior | One exit held for the session window | New exit per request or per short window |
| Session state | Cookies, tokens and Referer chain stay consistent | Requests look unrelated to each other |
| Best fit | Logging in, carts, account work, dashboards, ads accounts | Many independent fetches, SERP checks, price scans |
| Weakness | One IP doing heavy volume is easier to rate-limit | State-dependent flows break mid-session |

The decision isn't about which is better. It's about whether your workload is one visit or many independent ones.

If it's many independent visits, rotation belongs *between* launches, not inside one. That's the version people get wrong most often: they buy a rotating endpoint, run a browser through it, and then wonder why logins fail on the third page.

## How long can a sticky session actually hold?

This is where providers diverge, and where marketing language gets vague.

Typical sticky windows across the industry run from around 1 minute to 30–60 minutes, with some providers offering longer holds on dedicated IP products and others capping stickiness at 10 minutes. Anything claiming a fixed "sticky IP for 24 hours" is usually describing a dedicated or ISP-style IP, which is a different product with a different price.

9Proxy splits the difference by product type:

- On its **GB-based residential** plans, stickiness is a parameter you set, expressed in minutes. Third-party reviews list sticky sessions of up to about 30 minutes on the network.
- On its **IP-based** plans, you're buying individual residential IPs rather than sessions. Those IPs behave like real residential connections and stay live anywhere from a few hours up to roughly a day, depending on the IP.

That second point is worth pausing on, because it's the part people miss. With IP-based purchasing, "sticky" isn't a session parameter at all — it's just *how long that residential IP happens to stay online*. You don't configure it. You get what the residential device gives you, and when it drops, you replace it.

## How to build a sticky session on 9Proxy

The GB-based product embeds everything in the proxy username string. The format is:


<sub-user>-country-<cc>-st-<state>-city-<city>-isp-<isp>-sst-<minutes>-ssid-<session_id>


Only some parameters are required. For a sticky session, `sst` is the one that matters — it's the session duration in minutes. `ssid` is optional but you'll want it as soon as you run more than one instance in parallel, because each unique `ssid` gets its own IP even when every other parameter is identical.

A working sticky request looks like this:

bash
# US IP held for 15 minutes
curl -x yourproxyhost:yourport \
     -U "subuser-country-us-sst-15:yourpassword" \
     https://ipinfo.io

# Hanoi IP, sticky for 20 minutes
curl -x yourproxyhost:yourport \
     -U "subuser-country-vn-city-hanoi-sst-20:yourpassword" \
     https://ipinfo.io

# Two parallel sticky IPs from one config
subuser-country-us-sst-15-ssid-bot01
subuser-country-us-sst-15-ssid-bot02


Two practical notes from the docs:

- Sticky sessions don't require the desktop app — they work from the dashboard with username/password auth, or with IP whitelisting.
- In the whitelist flow, generated sticky proxies get a **Switch** button in the proxy list, so you can force a new IP without waiting out the session timer. That's more useful than it sounds when a target starts throwing 403s at a specific IP.

### Steps, in order

1. Open the Proxy Generator in the dashboard (Residential Proxies by GB → Get Proxy).
2. Create or select a sub-user. Sub-users exist so you can assign bandwidth limits per person, per client, or per project instead of sharing one credential.
3. Pick the location: country, then state, city, and ISP if you need that precision.
4. Choose **Sticky Session** as the session type and set the minutes.
5. Set the quantity of endpoints and pick your output format (`username:password:hostname:port` and similar).
6. Generate, then test with one request before you batch anything.

For whitelist auth, note the limits: up to 50 IPv4 addresses, IPv4 only, and the IPs have to be yours — not routed through a VPN or another proxy.

If you'd rather skip the config work entirely, 👉 注册 9Proxy 账号 (that link carries an invite code, which is worth remembering because the platform's starter balance is small enough that any discount matters).

## Where sticky is the right call, and where it quietly wastes money

**Good uses:**

- Multi-account management and anti-detect browser profiles, where each profile keeps one identity for the length of a working session.
- E-commerce flows with carts or logged-in pricing.
- Ad account and dashboard work where a mid-session IP change reads as suspicious.
- Account warming and social automation at human-ish speeds.

**Bad uses:**

- SERP rank tracking across many queries. You want rotation here, and you want it per request.
- Price scraping thousands of product pages where each fetch is independent.
- Anything where the goal is to *look like different users*. Sticky does the opposite.

The waste usually shows up in one of two shapes. Either you pay for sticky sessions while running a workload that never needed session continuity, or you run a single sticky IP so hard that the target starts rate-limiting it — at which point the consistent identity is doing exactly what a shared identity always does.

There's a subtler one: sticky is not clean. A held IP is still a residential IP with whatever reputation it carried in. Keeping it longer doesn't improve it.

## Four pitfalls that show up in real setups

**Session expiry mid-job.** If your `sst` is 15 minutes and the job runs 40, you get two identities and one confusing failure. Set the window to match the task, not the other way round.

**Reusing one sticky IP across accounts that should be separate.** Linking two profiles through one IP is one of the easier patterns for a platform to spot. Different `ssid` per profile, or different IPs entirely.

**Assuming IP-based plans behave like GB-based ones.** They don't. IP-based means unlimited bandwidth on each IP, a desktop app for local port forwarding, and IP lifespans measured in hours rather than minutes. GB-based means no app, a bandwidth budget, and explicit sticky/rotating control. The docs put these side by side precisely because people mix them up.

**Forgetting that residential IPs die.** It's inherent to a network built on real devices. 9Proxy handles this with a replacement policy for IPs confirmed dead, and a reuse list — proxies you've used in the last 24 hours that come back online can be reused at no additional cost. That matters for sticky work because a recycled IP is, by definition, one you already know behaves on your target.

On the performance side, third-party tests put the network's claimed success rate around 97% with average response times in the 0.6–1.3 second range depending on the test and region. Treat those as provider and reviewer figures rather than guarantees — your target site mix will move them more than the provider does.

## 9Proxy plans and pricing

Everything on 9Proxy is pay-as-you-go, not a monthly subscription. IP-based packages don't expire; GB-based packages carry a 180-day validity unless you're on an enterprise tier.

Note the timing: IP-based and bundle prices went up on June 1, 2026, while GB-based pricing stayed unchanged.

### IP-based residential (per IP, unlimited bandwidth)

| Package | What you get | Price | Expiry | Buy |
| --- | --- | --- | --- | --- |
| 100 IPs | 100 residential IPs, unlimited bandwidth | $0.24/IP — $24 total | IPs never expire | 购买 100 IP 套餐 |
| 500 IPs | 500 residential IPs | $0.144/IP — $72 total | IPs never expire | 购买 500 IP 套餐 |
| 1,000 + 500 IPs | 1,000 IPs plus 500 bonus | $0.084/IP — $126 total | IPs never expire | 购买 1,000+500 IP 套餐 |
| 2,500 IPs | 2,500 residential IPs | $0.084/IP — $210 total | IPs never expire | 购买 2,500 IP 套餐 |
| 5,000 IPs | 5,000 residential IPs | $0.072/IP — $360 total | IPs never expire | 购买 5,000 IP 套餐 |
| 15,000 IPs | 15,000 residential IPs | $0.048/IP — $720 total | IPs never expire | 购买 15,000 IP 套餐 |
| 25,000 IPs | 25,000 residential IPs | $0.035/IP — $863 total | IPs never expire | 购买 25,000 IP 套餐 |
| 50,000 IPs | 50,000 residential IPs | $0.029/IP — $1,438 total | IPs never expire | 购买 50,000 IP 套餐 |

### Business IP packages (high volume)

| Package | What you get | Price | Expiry | Buy |
| --- | --- | --- | --- | --- |
| 100,000 IPs | High-volume residential pool | $0.023/IP — $2,300 total | IPs never expire | 购买 100,000 IP 套餐 |
| 200,000 IPs | High-volume residential pool | $0.021/IP — $4,140 total | IPs never expire | 购买 200,000 IP 套餐 |
| 500,000 IPs | High-volume residential pool | $0.018/IP — $8,625 total | IPs never expire | 购买 500,000 IP 套餐 |

### GB-based residential (sticky or rotating, per endpoint parameters)

| Package | What you get | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 5 GB | 5 GB of traffic, sticky/rotating by parameter | $3.00/GB — $15 total | 180 days | 购买 5 GB 套餐 |
| 50 + 5 GB | 50 GB plus 5 GB bonus | $2.10/GB — $105 total | 180 days | 购买 50+5 GB 套餐 |
| 100 GB | 100 GB of traffic | $1.50/GB — $150 total | 180 days | 购买 100 GB 套餐 |
| 200 GB | 200 GB of traffic | $1.00/GB — $200 total | 180 days | 购买 200 GB 套餐 |
| 1,000 GB | 1,000 GB of traffic | $0.80/GB — $800 total | 180 days | 购买 1,000 GB 套餐 |
| 2,000 GB | 2,000 GB of traffic | $0.75/GB — $1,500 total | 180 days | 购买 2,000 GB 套餐 |

### Enterprise GB packages (no expiry)

| Package | What you get | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 3,000 GB | Enterprise bandwidth, sticky/rotating | $0.72/GB — $2,160 total | No expiry | 购买 3,000 GB 套餐 |
| 6,000 GB | Enterprise bandwidth, sticky/rotating | $0.70/GB — $4,200 total | No expiry | 购买 6,000 GB 套餐 |
| 10,000 GB | Enterprise bandwidth, sticky/rotating | $0.68/GB — $6,800 total | No expiry | 购买 10,000 GB 套餐 |

### Bundle packages (IPs + bandwidth)

| Package | What you get | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| Starter Bundle | 100 IPs + 5 GB | $30 | Traffic valid 180 days | 购买 Starter Bundle |
| Popular Bundle | 1,500 IPs + 50 GB | $180 | Traffic valid 180 days | 购买 Popular Bundle |
| Pro Bundle | 5,000 IPs + 500 GB | $720 | Traffic valid 180 days | 购买 Pro Bundle |

Prices above reflect the post-adjustment list published by 9Proxy; the platform runs periodic promos, so checking the current figure before checkout is worth the thirty seconds.

## Which plan fits which sticky workflow

If your sticky sessions are short and targeted — 10–30 minutes per account, a handful of profiles, moderate bandwidth — the **GB-based** plans are the cleaner fit. You control session length with `sst`, you don't need the desktop app, and 5 GB at $15 is a cheap way to find out whether the network behaves on your specific targets.

If your stickiness is really "I need a set of stable identity-bearing IPs that stay up for hours and don't meter my bandwidth," buy **IPs**. Not because it's stickier in the parameter sense, but because that's what stable residential IPs with unlimited transfer costs. At 100 IPs for $24, it's roughly the price of entry for account-based work.

Bundles exist for the case where you have both: some accounts that need a persistent IP and some scraping that just needs traffic. The Starter Bundle at $30 for 100 IPs + 5 GB is the smallest version of that, and the 180-day traffic validity means uneven project schedules don't burn anything.

Worth knowing before you commit: there's no self-serve free trial on the website. Trial access is handed out through the community channels and support, which adds a step compared to providers that just flip a switch.

## FAQ

**What's the difference between a sticky proxy and a static proxy?**

A sticky proxy holds one IP for a defined session window, then it can change. A static (ISP or dedicated) IP stays on your account for the billing period. If your workflow needs the *same* IP tomorrow as today, sticky isn't the product — a static IP is.

**Can I keep the same sticky IP for 24 hours?**

On a GB-based residential network, no — residential devices churn. Long holds come from dedicated IP products. On 9Proxy's IP-based plans you can realistically get individual IPs staying live for hours up to about a day, but that's the IP's natural lifespan, not something you set.

**Do sticky sessions cost more?**

Not as a line item. On GB-based plans you pay for traffic, and sticky or rotating costs the same per GB. Where you feel it is volume: sticky sessions send more requests through fewer IPs, so if the target rate-limits, you'll spend bandwidth on retries.

**How many sticky IPs can I run at once?**

On the GB-based product, as many as you create endpoints for — use a different `ssid` per instance. Each unique `ssid` gets a distinct IP even if country, city and session length are identical.

**Will a sticky IP keep me from getting blocked?**

No. Stickiness keeps your session internally consistent; it doesn't improve the reputation of the IP you're holding. Rate limits, account-level flags and bad neighbours in the pool are separate problems.
