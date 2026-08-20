# Warframe.market Price Tracker

Watches items you specify, checks their current sell orders on
warframe.market, and pings a Discord channel when:

1. The market's lowest price changes at all.
2. The lowest available price drops to or below a target you set per item.
3. A **new** lowest price appears from a seller who is **currently online**.

It also keeps a small local history so it can tell you the rolling
average price and the all-time low it's seen for each item.

## 1. Install

```bash
pip install -r requirements.txt
```

(Python 3.9+ recommended.)

## 2. Set up the Discord webhook

1. In Discord, go to the server/channel you want alerts in.
2. Channel settings (gear icon) → **Integrations** → **Webhooks** → **New Webhook**.
3. Name it something like "WFM Price Tracker", click **Copy Webhook URL**.
4. Keep that URL handy for the next step. Treat it like a password — anyone
   with it can post messages to that channel.

## 3. Configure your watchlist

```bash
cp config.example.json config.json
```

Edit `config.json`:

- `discord_webhook_url` — paste the URL from step 2.
- `items` — one entry per item you want to track:
  - `slug`: the item's URL name on warframe.market. If the item's page is
    `https://warframe.market/items/secura_dual_cestra_blueprint`, the slug
    is `secura_dual_cestra_blueprint`.
  - `rank`: for mods, the specific rank you care about (`0` for unranked,
    up to the mod's max rank), or `null` to ignore rank and match any.
  - `notify_below`: your target price in platinum, or `null` to skip that
    check for this item.
- `rest_poll_interval_seconds`: how often to check via REST (default 45s).
  The public API is rate-limited to ~3 requests/second, so don't go below
  a few seconds per item if you're tracking a lot of items.
- `notify.cooldown_seconds`: minimum time between repeat alerts for the
  same item/trigger, so you don't get spammed if a price flickers.

### Finding the right slug

Easiest way: go to the item's page on warframe.market and copy the last
part of the URL. If you're not sure, run:

```bash
python main.py --dump some_guess_at_the_slug
```

This prints the raw API response so you can confirm you've got the right
item and see the real field names (useful if the API has changed shape
since this was built — see the note in `order_utils.py`).

## 4. Run it

One-time check (prints to terminal, doesn't message Discord's startup ping):

```bash
python main.py --once
```

Continuous background tracking:

```bash
python main.py
```

Leave this running in a terminal window while you play. Ctrl+C to stop.

## About the optional WebSocket mode

`config.json` has `"use_websocket": false` by default. REST polling alone
needs no login and is reliable. WebSocket would give near-instant updates
instead of waiting for the next poll, **but** the official docs state that
WebSocket connections require authentication, and that authenticated
features currently still rely on the old, deprecated v1 login system —
there's no clean public login flow yet.

If you want to try it anyway:

1. Log into warframe.market in your browser.
2. Open DevTools → Application/Storage → Cookies → warframe.market.
3. Copy the value of the `JWT` cookie.
4. Paste it into `websocket_jwt_token` in `config.json`, and set
   `"use_websocket": true`.

This part is marked experimental in the code (`ws_client.py`) because the
exact message format isn't fully nailed down in the public docs and may
need small adjustments once you see real traffic — it prints every raw
message it receives so you can see what's coming through and I can help
you adjust the parsing if needed.

## Files

| File | Purpose |
| --- | --- |
| `main.py` | Entry point / CLI |
| `wfm_client.py` | REST client (rate-limited) |
| `ws_client.py` | Optional real-time WebSocket client |
| `order_utils.py` | Parses raw order data, finds lowest prices |
| `history.py` | Local JSON history for rolling averages / all-time lows |
| `tracker.py` | Decides what counts as notify-worthy |
| `notifier.py` | Sends Discord embeds |
| `config.example.json` | Copy to `config.json` and fill in |
