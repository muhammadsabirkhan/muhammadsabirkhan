#!/usr/bin/env python3
"""Generates live SVG dashboards + recent-activity block for a GitHub profile README.
No third-party dependencies. Uses the GitHub REST API."""
import os, re, json, html, collections
import urllib.request
import datetime as dt

USER = os.environ.get("GH_USER", "muhammadsabirkhan")
TOKEN = os.environ.get("GITHUB_TOKEN", "")
OUT = "assets"
DAYS = 14

LANG_COLORS = {
    "PHP": "#8892BF", "Dart": "#00B4AB", "JavaScript": "#F7DF1E", "HTML": "#E34F26",
    "CSS": "#9b6bff", "Python": "#3776AB", "TypeScript": "#3178C6", "C++": "#f34b7d",
    "Java": "#b07219", "Kotlin": "#A97BFF", "Swift": "#F05138", "Blade": "#F7523F",
    "Shell": "#89e051", "C": "#aaaaaa", "SCSS": "#c6538c",
}

BG = '<linearGradient id="bg" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#0f0c29"/><stop offset="0.55" stop-color="#1a1442"/><stop offset="1" stop-color="#24243e"/></linearGradient>'
ACC = '<linearGradient id="acc" x1="0" y1="0" x2="1" y2="0"><stop offset="0" stop-color="#b388ff"/><stop offset="1" stop-color="#6a0dad"/></linearGradient>'
FONT = "font-family=\"'Segoe UI',Ubuntu,Helvetica,Arial,sans-serif\""


def api(path):
    headers = {"Accept": "application/vnd.github+json", "User-Agent": "profile-dashboard"}
    if TOKEN:
        headers["Authorization"] = f"Bearer {TOKEN}"
    req = urllib.request.Request("https://api.github.com" + path, headers=headers)
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.load(r)


def esc(s):
    return html.escape(str(s), quote=True)


def svg(w, h, body, title):
    return (f'<svg xmlns="http://www.w3.org/2000/svg" width="{w}" height="{h}" viewBox="0 0 {w} {h}" role="img" aria-label="{esc(title)}">'
            f'<defs>{BG}{ACC}</defs><rect width="{w}" height="{h}" rx="16" fill="url(#bg)" stroke="#3b2a8c" stroke-width="1.5"/>{body}</svg>')


def write(name, content):
    os.makedirs(OUT, exist_ok=True)
    with open(os.path.join(OUT, name), "w", encoding="utf-8") as f:
        f.write(content)


def event_weight(e):
    if e["type"] == "PushEvent":
        p = e.get("payload", {})
        return p.get("size") or len(p.get("commits", [])) or 1
    return 1


def describe(e):
    repo = e["repo"]["name"]
    link = f"[{repo}](https://github.com/{repo})"
    t, p = e["type"], e.get("payload", {})
    if t == "PushEvent":
        n = event_weight(e)
        return f"⬆️ Pushed **{n}** commit{'s' if n != 1 else ''} to {link}"
    if t == "CreateEvent":
        return f"✨ Created {p.get('ref_type', 'resource')} in {link}"
    if t == "PullRequestEvent":
        return f"🔀 {p.get('action', 'updated').capitalize()} a pull request in {link}"
    if t == "IssuesEvent":
        return f"🐛 {p.get('action', 'updated').capitalize()} an issue in {link}"
    if t == "WatchEvent":
        return f"⭐ Starred {link}"
    if t == "ForkEvent":
        return f"🍴 Forked {link}"
    if t == "ReleaseEvent":
        return f"🚀 Published a release in {link}"
    return None


def main():
    user = api(f"/users/{USER}")
    repos = [r for r in api(f"/users/{USER}/repos?per_page=100&type=owner") if not r["fork"]]
    events = api(f"/users/{USER}/events/public?per_page=100")

    stars = sum(r["stargazers_count"] for r in repos)
    today = dt.datetime.now(dt.timezone.utc).date()

    # per-day activity
    days = [today - dt.timedelta(days=i) for i in range(DAYS - 1, -1, -1)]
    counts = collections.OrderedDict((d, 0) for d in days)
    for e in events:
        d = dt.datetime.strptime(e["created_at"], "%Y-%m-%dT%H:%M:%SZ").date()
        if d in counts:
            counts[d] += event_weight(e)
    total14 = sum(counts.values())

    # languages
    langs = collections.Counter()
    for r in repos:
        try:
            for k, v in api(f"/repos/{USER}/{r['name']}/languages").items():
                langs[k] += v
        except Exception:
            pass

    stamp = dt.datetime.now(dt.timezone.utc).strftime("%d %b %Y, %H:%M UTC")

    # ---------- stats.svg ----------
    cards = [("Repositories", len(repos)), ("Stars Earned", stars),
             ("Followers", user["followers"]), (f"Activity ({DAYS}d)", total14)]
    body = f'<text x="30" y="40" {FONT} font-size="20" font-weight="700" fill="#ffffff">📊 {esc(USER)} · live stats</text>'
    cw = 180
    for i, (label, val) in enumerate(cards):
        x = 30 + i * (cw + 14)
        body += (f'<rect x="{x}" y="64" width="{cw}" height="84" rx="12" fill="#ffffff" fill-opacity="0.06" stroke="#6a0dad" stroke-opacity="0.6"/>'
                 f'<text x="{x + cw / 2}" y="108" text-anchor="middle" {FONT} font-size="32" font-weight="800" fill="url(#acc)">{val}</text>'
                 f'<text x="{x + cw / 2}" y="132" text-anchor="middle" {FONT} font-size="13" fill="#c9bfff">{esc(label)}</text>')
    body += f'<text x="30" y="178" {FONT} font-size="11" fill="#8f86c9">Updated {stamp}</text>'
    write("stats.svg", svg(820, 192, body, "GitHub stats"))

    # ---------- activity.svg ----------
    W, H = 820, 280
    body = f'<text x="30" y="40" {FONT} font-size="20" font-weight="700" fill="#ffffff">📈 Activity · last {DAYS} days</text>'
    mx = max(max(counts.values()), 1)
    chart_h, base_y = 150, 232
    bw = (W - 60) / DAYS
    for i, (d, v) in enumerate(counts.items()):
        h = max(4, v / mx * chart_h) if v else 4
        x = 30 + i * bw + 5
        op = "1" if v else "0.25"
        body += (f'<rect x="{x:.1f}" y="{base_y - h:.1f}" width="{bw - 10:.1f}" height="{h:.1f}" rx="5" fill="url(#acc)" opacity="{op}"/>')
        if v:
            body += f'<text x="{x + (bw - 10) / 2:.1f}" y="{base_y - h - 6:.1f}" text-anchor="middle" {FONT} font-size="11" fill="#e0d4ff">{v}</text>'
        body += f'<text x="{x + (bw - 10) / 2:.1f}" y="{base_y + 18}" text-anchor="middle" {FONT} font-size="10" fill="#8f86c9">{d.day}</text>'
    body += f'<text x="30" y="{H - 12}" {FONT} font-size="11" fill="#8f86c9">Commits, PRs, issues &amp; repo events · {stamp}</text>'
    write("activity.svg", svg(W, H, body, "Recent activity"))

    # ---------- languages.svg ----------
    top = langs.most_common(6)
    tot = sum(v for _, v in top) or 1
    H = 70 + len(top) * 34 + 24
    body = f'<text x="30" y="40" {FONT} font-size="20" font-weight="700" fill="#ffffff">🧬 Top languages</text>'
    for i, (name, v) in enumerate(top):
        y = 66 + i * 34
        pct = v / tot * 100
        col = LANG_COLORS.get(name, "#b388ff")
        body += (f'<text x="30" y="{y + 14}" {FONT} font-size="13" fill="#e0d4ff">{esc(name)}</text>'
                 f'<rect x="150" y="{y}" width="540" height="14" rx="7" fill="#ffffff" fill-opacity="0.08"/>'
                 f'<rect x="150" y="{y}" width="{max(6, 540 * pct / 100):.1f}" height="14" rx="7" fill="{col}"/>'
                 f'<text x="705" y="{y + 12}" {FONT} font-size="12" fill="#c9bfff">{pct:.1f}%</text>')
    if not top:
        body += f'<text x="30" y="80" {FONT} font-size="13" fill="#8f86c9">No language data yet</text>'
    write("languages.svg", svg(820, H, body, "Top languages"))

    # ---------- README recent activity ----------
    lines = []
    for e in events:
        txt = describe(e)
        if txt:
            lines.append(f"- {txt} · `{e['created_at'][:10]}`")
        if len(lines) == 6:
            break
    block = "\n".join(lines) if lines else "_No public activity yet. Time to ship something!_"
    block += f"\n\n<sub>Last updated: {stamp}</sub>"
    with open("README.md", encoding="utf-8") as f:
        readme = f.read()
    new = re.sub(r"(<!--RECENT_ACTIVITY:START-->).*?(<!--RECENT_ACTIVITY:END-->)",
                 lambda m: f"{m.group(1)}\n{block}\n{m.group(2)}", readme, flags=re.S)
    with open("README.md", "w", encoding="utf-8") as f:
        f.write(new)
    print("Done:", len(repos), "repos,", total14, "activity")


if __name__ == "__main__":
    main()
