# selenium proxy authentication: How to Stop Chrome's 407 Popup in Scripts, Headless Mode, and CI

You set the proxy, ran the script, and Chrome opened a login box that no one can click. The script sits there until the timeout kills it, or every request comes back 407. You've correctly identified that this is the `selenium proxy authentication` problem, and it isn't a bug in your code. Chrome simply has nowhere to put a proxy password.

Here's what's actually happening, the five ways people work around it, and the one design decision most guides skip: the proxy provider you pick determines whether you need to solve this problem in Selenium at all.

## Why `--proxy-server` can't take a username

The flag accepts a scheme, host, and port. It does not accept credentials.

python
# Credentials get silently dropped here.
# Chrome falls back to the native auth dialog, which Selenium cannot reach.
opts.add_argument("--proxy-server=http://user:pass@gw.example.com:913")


When Chrome receives a `407 Proxy Authentication Required` and has no credentials stored, it hands the challenge to the operating system. In a normal desktop session you'd see a small modal window. In a Selenium run that window is a native UI element, not part of the DOM, so no amount of `find_element` or JavaScript will touch it. In headless mode there is no window at all, so the request just hangs or fails.

`--proxy-server` doesn't help with SOCKS5 either. Chrome's SOCKS5 support has no credential field, so an authenticated SOCKS5 endpoint breaks the same way.

The fix always involves moving authentication out of the browser flag and into something that can answer the challenge: a request interceptor, Chrome DevTools Protocol, an extension, or the operating system's proxy environment. Or you remove the challenge entirely by making the proxy accept your IP without a password.

## Five approaches that actually work

| Approach | Survives headless | Passes user:pass | Main catch |
| --- | --- | --- | --- |
| selenium-wire | Yes | Yes | Wraps the driver; version compatibility needs checking |
| CDP / SeleniumBase CDP Mode | Yes | Yes | Now required on Chrome 142+ |
| Chrome extension with `onAuthRequired` | Yes, on `--headless=new` | Yes | Credentials sit in plaintext in a temp folder |
| `HTTP_PROXY` / `HTTPS_PROXY` env vars | Yes | Yes, in the URL | Applies to the whole process, not just the browser |
| IP whitelisting | Yes | Not needed | Only works from a fixed, known egress IP |

### selenium-wire

This is the route BrowserStack's own guide recommends for Python and the route Oxylabs documents for its residential endpoints. It wraps Selenium, intercepts HTTP/HTTPS traffic, and takes credentials directly instead of relying on the browser.

python
from seleniumwire import webdriver

wire_options = {
    "proxy": {
        "http": "http://USERNAME:PASSWORD@HOST:PORT",
        "https": "http://USERNAME:PASSWORD@HOST:PORT",
        "no_proxy": "localhost,127.0.0.1",
    }
}

driver = webdriver.Chrome(seleniumwire_options=wire_options)
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)
driver.quit()


The trade-off is that it's a third-party layer over the official driver. If your project is pinned to a specific Selenium version, or you're using a patched driver for anti-bot work, that layer can be the thing that breaks the build.

### CDP and SeleniumBase CDP Mode

Chrome DevTools Protocol can answer authentication challenges from inside the browser session. In the .NET bindings that's `NetworkAuthenticationHandler`; in SeleniumBase it's CDP Mode.

This matters more than it used to. Chrome 142 removed the `--load-extension` command-line switch, along with the `--disable-features=DisableLoadExtensionCommandLineSwitch` workaround, which means the extension trick below stopped working. SeleniumBase's answer, per its maintainers, is to set proxy auth through CDP after CDP Mode activates:

python
from seleniumbase import SB

with SB(uc=True, proxy="user:pass@ip:port") as sb:
    sb.activate_cdp_mode("https://api.ipify.org")
    print(sb.get_page_source())


If you were using an extension and it silently stopped authenticating after a Chrome upgrade, this is why.

### The Chrome extension route

You generate a small extension that calls `chrome.proxy.settings.set` and returns `authCredentials` from `chrome.webRequest.onAuthRequired`. It works, and it works in `--headless=new`.

Two things to know before you build one. First, it only loads in `--headless=new`; the old `--headless` mode doesn't load extensions at all. Second, the credentials you bake into that extension are plaintext. If you generate it into a temp directory, delete it when the driver quits, and never commit it.

There are packaged versions of this on PyPI that build the extension for you, which is worth considering if you'd rather not maintain manifest boilerplate.

### Environment variables

python
import os
os.environ["HTTP_PROXY"] = "http://username:password@proxy.example.com:8080"
os.environ["HTTPS_PROXY"] = "http://username:password@proxy.example.com:8080"


TestingBot documents this as the way to get a Selenium session out through a corporate proxy. Simple, and honest about its scope: it's process-wide. Anything else in that process making HTTP calls will also go through the proxy, which can confuse a remote grid connection.

### IP whitelisting

If your authentication challenge can be removed rather than answered, remove it. Whitelisting an egress IP means the proxy authenticates you by address, and Chrome never receives a 407, so Chrome never opens a dialog. This is the cleanest option for CI runners with a static outbound IP, and it's the model that makes one of the two proxy architectures below so much easier to script.

## Your proxy's auth model decides which method you need

Most tutorials assume every proxy is a remote `host:port` with a username and password. That's only one architecture. 9Proxy sells two residential products with genuinely different authentication mechanics, and the difference changes what your Selenium code looks like.

**Residential Proxy by IPs** works through a local port-forwarding system inside the 9Proxy app, available for Windows, macOS, and Linux. You filter IPs by country, state, city, ZIP code, or ISP, then forward one to a port on your own machine. The docs list the standard connection pattern as:


socks5://127.0.0.1:60000


Selenium points at `localhost`. There is no remote proxy to authenticate against, so there is no 407 and no dialog. Port forwarding can also be scripted on headless Linux, which is useful when the browser runs on the same box as the agent:

bash
9proxy proxy -c US -p 60000
# Now serving a US residential exit on 127.0.0.1:60000


If you do want credentials on the local port — for example, so another process on a shared machine can't use your proxy — Linux has a command for it:

bash
9proxy setting --basic_auth --proxy_password [password] --proxy_username [username]


But note that Selenium doesn't need to know. The authentication happens between your local forwarder and 9Proxy's network, not between Chrome and a remote server.

**Residential Proxy by GB** is the remote-endpoint model. You get a fixed hostname and port from the dashboard, and the username is a structured string that carries your targeting:


subaccount-country-us-sst-15-ssid-device1


Every part is optional except the sub-account name, so the same credentials can produce geo-targeted rotating IPs, sticky sessions, or ISP-filtered exits. Authentication options for this product are username/password or IP whitelisting, and because the endpoint is remote, this is the case where you genuinely need selenium-wire or CDP.

That's the decision in one line: **if your browser can reach a local forwarded port, you can skip proxy auth in Selenium entirely; if it's talking to a remote endpoint, you need one of the interception methods.**

If you're setting this up from scratch, 👉 [start a 9Proxy account here](https://bit.ly/9-Proxy) and pick the product matching where your driver runs.

## Working configurations

### Remote endpoint (GB-based) with selenium-wire

Use the `<sub-user>` value from your dashboard as the username and your sub-user password as the password. The `sst` parameter controls how long the exit IP is held, in minutes.

python
from seleniumwire import webdriver
from selenium.webdriver.chrome.options import Options

# Sticky US exit held for 15 minutes, one session ID per browser instance
USER = "subaccount-country-us-sst-15-ssid-bot01"
PWD  = "your_sub_user_password"
HOST = "proxy_host_from_dashboard"
PORT = "proxy_port_from_dashboard"

opts = Options()
opts.add_argument("--headless=new")

driver = webdriver.Chrome(
    options=opts,
    seleniumwire_options={
        "proxy": {"http": f"http://{USER}:{PWD}@{HOST}:{PORT}",
                  "https": f"http://{USER}:{PWD}@{HOST}:{PORT}"}
    },
)
driver.get("https://ipinfo.io/json")
print(driver.page_source)
driver.quit()


### Local forwarded port (IP-based) with plain Selenium

No credentials anywhere in the browser config. Start the port forward first, then let Chrome use it.

python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

opts = Options()
opts.add_argument("--headless=new")
opts.add_argument("--proxy-server=socks5://127.0.0.1:60000")

driver = webdriver.Chrome(options=opts)
driver.get("https://ipinfo.io/json")
print(driver.page_source)
driver.quit()


Because the proxy lives on loopback, there's no challenge to answer. The failure modes you'll hit instead are local: the forwarder isn't running, the port was reassigned, or the IP expired.

## Sticky versus rotating, for Selenium specifically

A login flow that spans ten requests needs the same IP for all ten. A search-results scraper that wants coverage across regions wants a new IP per request. These pull in opposite directions, and the two 9Proxy products map onto them neatly.

With GB-based proxies, rotating is the default when you omit `sst` and `ssid`, and sticky is what you get when you set them. One detail worth internalising: `ssid` gives you a distinct IP per value even when everything else in the username is identical. So if you're running eight parallel browser instances, give each one its own `ssid` rather than duplicating `sst-15` and confusing yourself later when two sessions share an exit node.

With IP-based proxies there's no natural rotation, and that's by design. A forwarded IP stays online for a few hours up to roughly a day, varying because these are real residential connections. When one drops, Auto Refresh replaces it, Auto Rotation swaps on a schedule you define, and the Today List lets you reconnect to an IP you already used in the last 24 hours without consuming a new one from your balance. For Selenium suites that run on a schedule, that last feature is the difference between burning 50 IPs a week and reusing the same 50.

## Full plan lineup

9Proxy repriced its IP-based and bundle packages on June 1, 2026. GB-based prices were unaffected. Everything below is the current structure.

### Residential Proxy by IPs — per-IP pricing, unlimited bandwidth

| Package | Per IP | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | Unused IPs never expire | [Get 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | $0.144 | $72 | Unused IPs never expire | [Get 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | $0.084 | $126 | Unused IPs never expire | [Get 1,500 IPs](https://bit.ly/9-Proxy) |
| 2,500 IPs | $0.084 | $210 | Unused IPs never expire | [Get 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | $0.072 | $360 | Unused IPs never expire | [Get 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | $0.048 | $720 | Unused IPs never expire | [Get 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | $0.035 | $863 | Unused IPs never expire | [Get 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | $0.029 | $1,438 | Unused IPs never expire | [Get 50,000 IPs](https://bit.ly/9-Proxy) |

### Business IP packages — high volume

| Package | Per IP | Price | Buy |
| --- | --- | --- | --- |
| 100,000 IPs | $0.023 | $2,300 | [Get 100,000 IPs](https://bit.ly/9-Proxy) |
| 200,000 IPs | $0.021 | $4,140 | [Get 200,000 IPs](https://bit.ly/9-Proxy) |
| 500,000 IPs | $0.018 | $8,625 | [Get 500,000 IPs](https://bit.ly/9-Proxy) |

### Residential Proxy by GB — bandwidth pricing, remote endpoints

| Package | Per GB | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [Get 5 GB](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [Get 55 GB](https://bit.ly/9-Proxy) |
| 100 GB | $1.50 | $150 | 180 days | [Get 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | $1.00 | $200 | 180 days | [Get 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80 | $800 | 180 days | [Get 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [Get 2,000 GB](https://bit.ly/9-Proxy) |

### Enterprise GB packages — no expiry

| Package | Per GB | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 3,000 GB | $0.72 | $2,160 | Does not expire | [Get 3,000 GB](https://bit.ly/9-Proxy) |
| 6,000 GB | $0.70 | $4,200 | Does not expire | [Get 6,000 GB](https://bit.ly/9-Proxy) |
| 10,000 GB | $0.68 | $6,800 | Does not expire | [Get 10,000 GB](https://bit.ly/9-Proxy) |

### Bundle packages — IPs plus bandwidth

| Bundle | Price | Buy |
| --- | --- | --- |
| Starter: 100 IPs + 5 GB | $30 | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular: 1,500 IPs + 50 GB | $180 | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro: 5,000 IPs + 500 GB | $720 | [Get the Pro bundle](https://bit.ly/9-Proxy) |

Across all packages you get country, state, city, ZIP code, and ISP targeting, HTTP/HTTPS and SOCKS5 support, sub-accounts and share codes, and a replacement policy for IPs that go offline. 9Proxy's network is advertised at 20M+ residential IPs across 90+ locations. Payment options include credit card, crypto, Google Pay, Alipay, local methods, and the 9Proxy Wallet.

## Which plan fits a Selenium workload

This is where the pricing model does real work for you.

If your driver runs on a machine where you can install and run the 9Proxy agent — a dev box, a dedicated scraper VM, a self-hosted Linux runner — the IP-based model is the better fit, and not just on price. Unlimited bandwidth means a page-heavy Selenium run can pull as much data as it needs without you watching a GB counter. The 1,000 IP package at $126 is the first tier where the per-IP cost drops meaningfully, and the bonus 500 IPs make it the natural starting point if you plan to run more than a handful of sessions.

If your driver runs in Docker or a hosted CI runner, the desktop-app dependency is a real obstacle, and the GB-based model is easier: it works straight from the dashboard, no local agent, and username/password auth that selenium-wire handles in four lines. Start with the 5 GB package at $15 to prove the setup, then move up. For a headless scraper pulling mostly text and JSON, GB-based billing is often the cheaper shape anyway, since a Selenium request for an HTML page costs kilobytes, not megabytes.

Either way, ask whether you can whitelist your egress IP instead. If your CI runner has a static outbound address, that removes the 407 challenge at the source and you never touch an auth popup.

Once you've decided, 👉 [pick a plan and register with the invite code applied](https://bit.ly/9-Proxy).

## Errors you'll actually hit

**`407 Proxy Authentication Required` with no dialog.** Your credentials never reached the browser. Stop debugging Chrome and check the interceptor: is selenium-wire actually the driver you imported, and is the `seleniumwire_options` key spelled exactly right?

**The script hangs at `driver.get()` with no output.** Classic headless auth failure. The challenge fired, nothing could answer it, and you're waiting on a timeout. Same causes as above.

**`ERR_PROXY_CONNECTION_FAILED`.** Different problem. The host or port is wrong, the forwarder isn't running, or the IP expired. Verify the endpoint with `curl -x` before blaming Selenium.

**The extension loaded fine yesterday and stopped today.** Check the Chrome version. Chrome 142 removed `--load-extension`, which breaks extension-based auth without any error message.

**Two parallel sessions share an exit IP.** You reused the same `ssid`, or omitted it. Each concurrent browser instance needs its own value.

**You set `user_data_dir` and now the proxy won't change.** That's documented behaviour. Once a profile directory is bound to a proxy, it expects the same one. Keep profile dirs and proxy endpoints paired.

**Requests work but the target site still blocks you.** Authentication is not the same problem as detection. A 407 means your credentials didn't arrive; a block means your exit IP or browser fingerprint got scored. Check the exit ASN and TLS fingerprint before adding more retries.

## Quick answers

**Does Selenium support authenticated proxies natively?** Not through browser options. There's no credential field in `--proxy-server`, so you need selenium-wire, CDP, an extension, or IP whitelisting.

**Can I put the password in the proxy URL?** Not in `--proxy-server`; Chrome discards it. In selenium-wire's options or in `HTTP_PROXY` environment variables, the `user:pass@host:port` form works.

**Does this work headless?** Every method on the list does, with one caveat: extensions only load under `--headless=new`, not the legacy headless mode.

**What about SOCKS5?** Chrome's SOCKS5 support has no credential field, so authenticated SOCKS5 needs an interceptor. A locally forwarded port such as `socks5://127.0.0.1:60000` has nothing to authenticate against, which sidesteps it.

**Is there a way to avoid credentials entirely?** Yes. Whitelist your egress IP on the provider side, or use a local port-forwarding model where your machine is the only client.
