#!/usr/bin/env python3
"""
Stock Portfolio Tracker (CLI)

Setup:
    pip install yfinance

Usage:
    python portfolio.py buy AAPL 10 185.50      # buy 10 shares at 185.50
    python portfolio.py sell AAPL 4 210         # sell 4 shares at 210
    python portfolio.py remove AAPL             # delete a holding entirely
    python portfolio.py show                    # live prices, P/L, allocation
    python portfolio.py history                 # transaction log
    python portfolio.py export out.csv          # export holdings to CSV

Indian stocks: use Yahoo symbols, e.g. RELIANCE.NS, TCS.NS, INFY.BO
Data is stored in portfolio.json next to this script.
"""

import argparse
import csv
import json
import sys
from datetime import datetime
from pathlib import Path

DB_FILE = Path(__file__).with_name("portfolio.json")


# ---------- Storage ----------
def load():
    if DB_FILE.exists():
        return json.loads(DB_FILE.read_text())
    return {"holdings": {}, "transactions": [], "realized_pl": 0.0}


def save(data):
    DB_FILE.write_text(json.dumps(data, indent=2))


# ---------- Trading logic ----------
def log(data, action, sym, qty, price):
    data["transactions"].append({
        "date": datetime.now().strftime("%Y-%m-%d %H:%M"),
        "action": action, "symbol": sym, "shares": qty, "price": price,
    })


def buy(data, sym, qty, price):
    h = data["holdings"].setdefault(sym, {"shares": 0.0, "cost": 0.0})
    h["shares"] += qty
    h["cost"] += qty * price
    log(data, "BUY", sym, qty, price)
    print(f"Bought {qty:g} {sym} @ {price:,.2f}")


def sell(data, sym, qty, price):
    h = data["holdings"].get(sym)
    if not h or h["shares"] < qty - 1e-9:
        owned = h["shares"] if h else 0
        sys.exit(f"Error: you only hold {owned:g} shares of {sym}.")
    avg = h["cost"] / h["shares"]
    pl = (price - avg) * qty
    h["shares"] -= qty
    h["cost"] -= avg * qty
    data["realized_pl"] += pl
    if h["shares"] < 1e-9:
        del data["holdings"][sym]
    log(data, "SELL", sym, qty, price)
    print(f"Sold {qty:g} {sym} @ {price:,.2f}  (realized P/L: {pl:+,.2f})")


# ---------- Prices ----------
def fetch_prices(symbols):
    """Return {symbol: price or None}. Requires yfinance."""
    try:
        import yfinance as yf
    except ImportError:
        print("yfinance not installed (pip install yfinance). Using cost basis.\n")
        return {s: None for s in symbols}

    prices = {}
    for s in symbols:
        try:
            prices[s] = float(yf.Ticker(s).fast_info["last_price"])
        except Exception:
            prices[s] = None
    return prices


def build_rows(data, offline=False):
    symbols = list(data["holdings"])
    prices = {s: None for s in symbols} if offline else fetch_prices(symbols)
    rows = []
    for s, h in data["holdings"].items():
        avg = h["cost"] / h["shares"]
        price = prices[s] if prices[s] is not None else avg
        value = price * h["shares"]
        pl = value - h["cost"]
        rows.append({
            "symbol": s, "shares": h["shares"], "avg_cost": avg,
            "price": price, "live": prices[s] is not None,
            "cost": h["cost"], "value": value, "pl": pl,
            "pl_pct": pl / h["cost"] * 100 if h["cost"] else 0.0,
        })
    total_value = sum(r["value"] for r in rows)
    for r in rows:
        r["weight"] = r["value"] / total_value * 100 if total_value else 0.0
    return rows


# ---------- Commands ----------
def show(data, offline=False):
    if not data["holdings"]:
        print("Portfolio is empty. Add one with: python portfolio.py buy AAPL 10 185.5")
        return
    rows = sorted(build_rows(data, offline), key=lambda r: -r["value"])
    hdr = f"{'Symbol':<12}{'Shares':>9}{'Avg Cost':>12}{'Price':>12}{'Value':>14}{'P/L':>13}{'P/L %':>9}{'Weight':>8}"
    print(hdr)
    print("-" * len(hdr))
    for r in rows:
        flag = "" if r["live"] else "*"
        print(f"{r['symbol']:<12}{r['shares']:>9g}{r['avg_cost']:>12,.2f}"
              f"{r['price']:>11,.2f}{flag or ' '}{r['value']:>14,.2f}"
              f"{r['pl']:>+13,.2f}{r['pl_pct']:>+8.2f}%{r['weight']:>7.1f}%")
    print("-" * len(hdr))

    cost = sum(r["cost"] for r in rows)
    value = sum(r["value"] for r in rows)
    pl = value - cost
    print(f"{'TOTAL':<12}{'':>9}{'':>12}{'':>12}{value:>14,.2f}"
          f"{pl:>+13,.2f}{(pl / cost * 100 if cost else 0):>+8.2f}%")
    print(f"\nInvested: {cost:,.2f}   Realized P/L: {data['realized_pl']:+,.2f}")
    if any(not r["live"] for r in rows):
        print("* live price unavailable, showing cost basis")


def history(data):
    if not data["transactions"]:
        print("No transactions yet.")
        return
    for t in data["transactions"]:
        print(f"{t['date']}  {t['action']:<5}{t['symbol']:<12}"
              f"{t['shares']:>9g} @ {t['price']:,.2f}")


def export_csv(data, path, offline=False):
    rows = build_rows(data, offline)
    if not rows:
        sys.exit("Nothing to export.")
    with open(path, "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0].keys()))
        w.writeheader()
        w.writerows(rows)
    print(f"Exported {len(rows)} holdings to {path}")


# ---------- CLI ----------
def main():
    p = argparse.ArgumentParser(description="Stock portfolio tracker")
    sub = p.add_subparsers(dest="cmd", required=True)

    for name in ("buy", "sell"):
        sp = sub.add_parser(name)
        sp.add_argument("symbol")
        sp.add_argument("shares", type=float)
        sp.add_argument("price", type=float)

    sp = sub.add_parser("remove")
    sp.add_argument("symbol")

    sp = sub.add_parser("show")
    sp.add_argument("--offline", action="store_true", help="skip live price lookup")

    sub.add_parser("history")

    sp = sub.add_parser("export")
    sp.add_argument("path")
    sp.add_argument("--offline", action="store_true")

    args = p.parse_args()
    data = load()

    if args.cmd in ("buy", "sell"):
        if args.shares <= 0 or args.price <= 0:
            sys.exit("Shares and price must be positive.")
        sym = args.symbol.upper()
        (buy if args.cmd == "buy" else sell)(data, sym, args.shares, args.price)
        save(data)
    elif args.cmd == "remove":
        sym = args.symbol.upper()
        if data["holdings"].pop(sym, None) is None:
            sys.exit(f"{sym} not in portfolio.")
        save(data)
        print(f"Removed {sym}.")
    elif args.cmd == "show":
        show(data, args.offline)
    elif args.cmd == "history":
        history(data)
    elif args.cmd == "export":
        export_csv(data, args.path, args.offline)


if __name__ == "__main__":
    main()
