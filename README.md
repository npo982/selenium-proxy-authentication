# selenium proxy: how to wire up authenticated proxies in Chrome, rotate IPs, and keep per-GB costs sane

Chrome's `--proxy-server` flag has nowhere to put a username and password. That one design decision is behind most broken Selenium proxy setups: the browser throws up its own native auth dialog, Selenium can't reach it, and the script either hangs or comes back with a 407 on every request.

So "selenium proxy" is really two problems wearing one name. First, getting credentials into a browser that won't accept them the obvious way. Second, choosing whose IPs you're renting and whether the per-gigabyte price survives contact with your actual volume.

Below is the setup that works today, the tools that stopped being worth recommending, and a concrete look at one provider whose prices make the cost math easy to do in your head.

## Why proxies are messier in Selenium than in curl

If you're calling `requests` or `httpx`, authentication is one line — you embed `user:pass` in the URL and you're done. Selenium can't do that, because the credential has to reach the *browser's* network stack, not Python's.

Three consequences follow from that:

- `--proxy-server=http://user:pass@host:port` doesn't work. Chrome ignores the credentials and pops a modal dialog instead.
- Chromium does not support authenticated SOCKS5. `socks5://` only works if you're authenticating by IP whitelist, not by username.
- Anything that intercepts traffic at the Python layer instead of the browser layer is a separate dependency with its own maintenance risk.

That last point is why so many tutorials still point at `selenium-wire`. It was a genuinely good library — it let you pass a proxy dict straight into the driver and inspect requests afterwards. It's also been archived upstream and now crashes on import with newer versions of `blinker`. Copying a 2023 Selenium Wire snippet into a 2026 project is one of the more reliable ways to spend an afternoon debugging a ModuleNotFoundError instead of scraping anything.

## The three setups that actually work

### IP whitelist — one line, no credentials in code

If your provider supports it, whitelist your machine's or CI runner's outbound IP in the dashboard, then just point Chrome at the endpoint:

python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument('--proxy-server=http://gw.dataimpulse.com:823')

driver = webdriver.Chrome(options=options)
driver.get('https://httpbin.org/ip')   # confirm the egress IP changed
print(driver.find_element('tag name', 'body').text)
driver.quit()


The catch is that whitelisting ties you to a fixed machine or a fixed CI egress IP. On ephemeral runners with rotating addresses, it becomes an infrastructure chore rather than a one-time config.

### SeleniumBase — the cleanest replacement for selenium-wire

SeleniumBase handles authenticated proxies internally, so you pass the whole thing as a single argument:

python
from seleniumbase import Driver

# __cr.us targets the US; swap in any country code
driver = Driver(proxy="YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823")
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)
driver.quit()


This is the pattern worth defaulting to if you're building a new pipeline. It keeps the browser in charge of the connection, which is what you want — the request should look like it came from a normal Chrome instance, because it did.

### Credential extension — when you're stuck on vanilla Selenium

If you can't add SeleniumBase to the stack, the standard approach is a tiny Chrome extension that sets proxy settings and answers the auth challenge. You can generate one at runtime rather than checking it in:

python
from selenium import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")          # old headless mode doesn't load extensions
options.add_argument("--disable-dev-shm-usage")
options.add_extension("proxy_auth_plugin.zip")  # built by your extension generator

driver = webdriver.Chrome(options=options)


# you have a better option than extensions altogether — the DevTools `NetworkAuthenticationHandler` lets you inject credentials against a URI matcher, which beats shipping a zip file around.

## Which proxy type should sit behind Selenium?

Not every automation task needs the same IP. Testing a login flow doesn't have the same requirements as pulling 200,000 product pages a night.

| Proxy type | Detection risk | Typical price | Where it fits in a Selenium pipeline |
| --- | --- | --- | --- |
| Residential | Very low | ~$1–8/GB | Protected targets: e-commerce, SERPs, social platforms |
| Datacenter | High | ~$0.50–3/GB or a few $/IP/month | Unprotected targets, high volume, speed matters more than trust |
| Mobile (4G/5G) | Very low | ~$2–15/GB | Mobile-first targets, app testing, the hardest anti-bot stacks |
| Premium residential | Very low | $5+/GB | High-stakes work where standard residential exits get flagged |

If you're getting blocked *after* authentication, the credentials were never the problem — the exit IP was. Datacenter ranges get scored as bots on ASN and TLS fingerprint long before the page renders, which is why a working proxy config can still produce a wall of 403s.

## One concrete price anchor: DataImpulse's per-GB model

DataImpulse runs a pool of 90M+ ethically sourced IPs across 195 countries, supports HTTP/HTTPS and SOCKS5, and bills by traffic with no subscription and no expiry on purchased gigabytes. That last part matters more than it sounds: if you buy 50 GB and only push 12 GB this month, the remaining 38 GB is still sitting there in six weeks. On a subscription model, that's budget you donated.

Pricing below reflects what's currently published across their residential, datacenter, mobile, and premium residential lines. Country-level targeting is included; city, ZIP, and ASN targeting is a paid add-on.

| Proxy type | Tier | Traffic | Price | Rate | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5 | $1.00/GB | Start with the 5 GB residential pack |
| Residential | Pay-as-you-go | Any amount | $1/GB | $1.00/GB | Buy residential traffic by the gigabyte |
| Residential | Advanced | 1 TB+ | $800 | $0.80/GB | Check residential volume pricing |
| Datacenter | Intro | 10 GB | $5 | $0.50/GB | Compare the datacenter entry plan |
| Datacenter | Standard | 100 GB | $50 | $0.50/GB | Price out 100 GB of datacenter traffic |
| Datacenter | Advanced | 1 TB | $450 | $0.45/GB | Check datacenter bulk rates |
| Mobile | Intro | 2.5 GB | $5 | $2.00/GB | Look at mobile proxy pricing |
| Mobile | Standard | 25 GB | $50 | $2.00/GB | Compare 25 GB mobile packs |
| Mobile | Advanced | 1 TB | $1,600 | $1.60/GB | Check mobile volume pricing |
| Premium residential | Intro | 1 GB | $5 | $5.00/GB | See the premium residential entry plan |
| Premium residential | Standard | 10 GB | $50 | $5.00/GB | Compare premium residential packs |
| Premium residential | Custom | 5 TB+ | From $20,000 | Custom | Request custom premium pricing |

A few practical notes on that table. The minimum purchase is $5 across every line, which is also the de facto trial — there's no free tier. Sticky sessions cap at 30 minutes, which is shorter than some competitors offer, so if your flow depends on holding one IP for hours, test that early. And third-party review coverage mentions a 7-day money-back window on Intro plans for card payments, conditional on not having consumed most of the traffic; crypto purchases are reported as non-refundable. Confirm that at checkout rather than taking it as settled.

## Wiring DataImpulse into a Selenium run

The gateway is `gw.dataimpulse.com`, with port `823` for HTTP/HTTPS and `824` for SOCKS5. Country targeting rides in the username:

python
from seleniumbase import Driver

# __cr.de routes through Germany
driver = Driver(proxy="YOUR_LOGIN__cr.de:YOUR_PASSWORD@gw.dataimpulse.com:823")


For a sticky session, a session token goes into the credential string so the same exit IP is reused for the duration you specify. Rotating mode hands out a new IP per connection, which is what you want for high-volume crawling and what you don't want for a login flow — changing IP mid-session is a fast way to get a session invalidated.

Two caveats worth internalising:

- Keep authenticated residential traffic on HTTP/HTTPS. Chromium can't do authenticated SOCKS5, so `socks5://gw.dataimpulse.com:824` only works with IP whitelisting.
- Verify the egress IP before you trust the setup. A single `driver.get('https://httpbin.org/ip')` at the top of your run tells you whether traffic is actually routing or silently falling back to your own connection. It's a two-line check that saves you from discovering the problem 40,000 requests later.

👉 Create an account and put $5 of residential traffic behind your scraper

## Errors you'll hit, and what's actually causing them

**407 on every request.** Authentication isn't reaching the browser. The credentials are either in the wrong place (the proxy URL) or the extension/handler isn't loading. Check that the extension loaded at all — old headless mode silently skips extensions.

**Requests hang instead of failing.** Same root cause, different symptom. Chrome opened an auth dialog that nothing can dismiss, and the request is sitting behind it.

**403 or CAPTCHA after a successful connection.** The proxy is working; the IP is the problem. Swap to residential or mobile exits.

**Proxy config appears to be ignored.** Settings applied after the driver launched won't take effect. Proxy arguments have to be on `ChromeOptions` before `webdriver.Chrome()` is called.

**Works locally, fails in CI.** Usually the whitelisted IP. CI runners rarely have a stable egress address, so whitelist-based auth tends to break the moment you move the script into a pipeline. Either pin a static runner IP or switch to username authentication.

**Certificate warnings on HTTPS.** Some setups need the proxy's CA trusted. Bypassing verification for the sake of a green build is a bad habit worth avoiding.

## Does the per-GB math actually hold up?

Here's the part most proxy guides skip. Your cost per page depends on page size, and page weight varies wildly.

A lightweight HTML page might be 100–200 KB. A heavy e-commerce product page with images, fonts, and tracking scripts can easily run 1.5–2.5 MB. A search results page typically lands in the 50–150 KB range.

That difference is the whole argument for controlling what loads. These flags alone can cut your bandwidth bill by an order of magnitude on image-heavy targets:

python
options.add_argument("--blink-settings=imagesEnabled=false")


Block images, fonts, and analytics where you don't need them, and a gigabyte stretches much further. At $1/GB residential, 50 GB of traffic is $50 whether you spend it this week or spread it over three months — which is the practical difference between pay-as-you-go and a subscription that resets whether you used it or not.

Where the per-GB model stops making sense: if your pipeline needs a handful of long-lived static IPs for account work, per-gigabyte billing is the wrong instrument. That's a per-IP rental problem. DataImpulse doesn't sell static ISP proxies, so for that specific use case you'd be looking at a different kind of provider.

## Questions that come up a lot

**Can I just use free proxies with Selenium?** You can, and you'll spend your time managing failure. Free pools are shared with everyone else running the same script, and their exit IPs are already burned on any target worth scraping.

**Do I need residential if I'm only testing my own site?** No. If you're testing a staging environment or an internal app, datacenter proxies at $0.50/GB are cheaper and faster, and detection is irrelevant.

**How many IPs should I rotate through?** It depends on request rate, not script length. A hundred requests spread over an hour from one IP is usually fine. A hundred requests in ten seconds is not, no matter how clean the IP.

**Is headless mode a problem?** Modern `--headless=new` renders like a real browser and loads extensions, so it's worth using over the legacy flag. It still won't hide a datacenter ASN, though. Proxy quality and headless mode solve different problems.

**What's the cheapest way to find out if a provider works for my targets?** Buy the smallest pack, build your integration against it, and measure your cost per successful page rather than per request. A provider that costs twice as much per gigabyte but halves your retry rate is cheaper in practice. DataImpulse's $5 entry point exists precisely for that experiment.

👉 Run the 5 GB test pack against your own targets before scaling up

The short version: fix the authentication layer first, because that's the part that fails silently. Then pick proxy types by target difficulty, not by price alone. And keep an eye on the variable nobody budgets for — retries. A request that fails and gets retried twice costs you triple.
