# xproxy

Several local xray proxies on one Mac, each on its own port with its own VLESS subscription
and server. One Python file (standard library only) on top of the Homebrew xray service.

```
$ xproxy ls
NAME     PORTS        STATUS   TIME  EXIT SERVER
main     10808/10809  up       0.6s  DE   🇩🇪 Frankfurt (203.0.113.10:443)
ru       10818/10819  up       0.7s  RU   🇷🇺 Moscow (203.0.113.20:443)
```

Each proxy has SOCKS on `<port>` and HTTP on `<port>+1`, routed to its own outbound. A browser
extension such as [ZeroOmega](https://github.com/zero-peak/ZeroOmega) then picks the proxy per site.

## Requirements

- macOS with Homebrew `xray` (`brew install xray`), run as `brew services`
- `python3`
- Subscriptions with VLESS + REALITY over TCP entries. Other VLESS entries are listed but can't be
  used (`probe` shows `unsupported`, `pick` refuses them); non-VLESS links (`vmess://`, `trojan://`,
  `ss://`, ...) are skipped.

Optional, to run it as `xproxy` from anywhere:

```bash
ln -s "$PWD/xproxy" ~/.local/bin/xproxy   # from the clone; any directory on PATH
```

## Commands

```bash
xproxy add <name> <port> '<subscription-url>'   # new proxy: fetch the list, pick a server
xproxy pick <name> [<n>] [--refresh]            # switch server (number, or choose from the list)
xproxy probe <name> [--country] [--refresh]     # test every server of the proxy's list
xproxy down <name> / xproxy up <name>           # disable / enable, settings are kept
xproxy rm <name>                                # delete the proxy and its cached list
xproxy ls                                       # live status: time to Google, exit country
xproxy url <name> '<subscription-url>'          # set or change the subscription URL, fetch it
xproxy apply                                    # regenerate the xray config from the list
xproxy import [--force]                         # rebuild a lost list from the current xray config
```

Plain `xproxy` prints this list; `xproxy <command> -h` shows a command's options.
Keep URLs in single quotes; zsh treats `#` and `?` specially.

Typical flow when a server stops working:

```bash
xproxy probe main --refresh --country   # fresh list, which servers work and where they exit
xproxy pick main 12                     # switch to a working one
```

## How it works

- The proxy list is `~/.config/xproxy/proxies.json` (directory 700, file 600): port, subscription
  URL, chosen server and on/off per proxy. Subscription URLs contain credentials: keep this out of git.
- Each subscription is cached as `~/.config/xproxy/subs/<name>.txt`; `pick` and `probe` use the cache
  unless `--refresh` is given.
- **Fetching** tries the direct connection first, then each enabled proxy (some providers' hosts don't
  answer directly), then falls back to the cached list with a warning.
- **Every change** (`add`, `pick`, `up`, `down`, `rm`, `apply`, `import`) regenerates the whole
  `$(brew --prefix)/etc/xray/config.json`, validates it with `xray run -test`, backs up the old one
  (`config.json.bak-<timestamp>`, newest 5 kept) and restarts xray. The changed proxy is then checked
  (Google + exit country code via `ipinfo.io`); if it gets no traffic, the previous config is restored
  and the list is not changed. `apply` and `import` keep the new config instead and only report the
  dead proxies.
- With every proxy down, xray is stopped.
- `import` recreates `proxies.json` from `config.json` (e.g. if the list was lost). Server names
  become their addresses and subscription URLs are unknown: set them again with `xproxy url`.
- `probe` runs each server in a throwaway xray instance on a free local port, so the live proxies keep
  working. Server labels don't always match the exit: use `--country` when the location matters.
  A provider may spread connections over several exit IPs, so the country is that of one connection.

Generated config, per enabled proxy `<name>`:

```json
"inbounds":  [{"tag": "<name>-socks", "port": <port>, ...}, {"tag": "<name>-http", "port": <port+1>, ...}],
"outbounds": [{"tag": "<name>", "protocol": "vless", ...}],
"routing":   {"rules": [{"type": "field", "inboundTag": ["<name>-socks", "<name>-http"], "outboundTag": "<name>"}]}
```

## ZeroOmega (Chrome)

One Proxy Profile per proxy (SOCKS5 `127.0.0.1:<port>`), plus a Switch Profile with
*Host wildcard* rules, e.g. `*.ru` and `*.xn--p1ai` (`.рф`) → `ru`, default → `main`.
