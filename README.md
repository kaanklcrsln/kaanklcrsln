### Hey, I'm Kaan👋 

- Geomatic Engineering student at Hacettepe University.

- Check out my personal ([Blog](https://kaanklcrsln.github.io/)) to learn more about me.
- Stay updated with my travel/photography journey on [Instagram](https://www.instagram.com/kaanklcrsln)

<!--LANGUAGE-START-->
name: Update language stats in README

on:
  push:
    branches: [ main, master ]
  schedule:
    - cron: '0 3 * * *'  # her gün UTC 03:00'te çalışır (isteğe göre değiştir)

permissions:
  contents: write

jobs:
  update-readme:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: python -m pip install --upgrade pip requests

      - name: Run language gatherer and update README
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_REPOSITORY: ${{ github.repository }}
        run: |
          mkdir -p .github/scripts
          cat > .github/scripts/update_readme.py <<'PY'
#!/usr/bin/env python3
import os, sys, requests, json, math, subprocess

TOKEN = os.environ.get("GITHUB_TOKEN")
if not TOKEN:
    print("GITHUB_TOKEN yok, çıkılıyor.")
    sys.exit(1)

# Kullanıcı/organizasyon adı — profil README repo'su için GITHUB_REPOSITORY -> "username/username"
repo_full = os.environ.get("GITHUB_REPOSITORY")
if not repo_full:
    print("GITHUB_REPOSITORY bulunamadı.")
    sys.exit(1)

# Profil sahibi kullanıcı adını al
owner = repo_full.split("/")[0]

session = requests.Session()
session.headers.update({
    "Authorization": f"token {TOKEN}",
    "Accept": "application/vnd.github.v3+json",
    "User-Agent": "readme-language-stats-action"
})

def paged_get(url, params=None):
    params = params or {}
    items = []
    while url:
        r = session.get(url, params=params)
        r.raise_for_status()
        items.extend(r.json())
        # next link
        if 'next' in r.links:
            url = r.links['next']['url']
            params = None
        else:
            url = None
    return items

# 1) Kullanıcının tüm public reposunu listele
repos_url = f"https://api.github.com/users/{owner}/repos"
repos = paged_get(repos_url, params={"per_page": 100, "type": "owner", "sort": "full_name"})

if not repos:
    print("Repo bulunamadı.")
    sys.exit(0)

lang_totals = {}

for r in repos:
    # ignore forks optionally
    # if r.get("fork"):
    #     continue
    name = r["name"]
    langs_url = r["languages_url"]
    r2 = session.get(langs_url)
    if r2.status_code != 200:
        print(f"Uyarı: {name} için language fetch başarısız: {r2.status_code}")
        continue
    langs = r2.json()  # { "Python": 12345, "HTML": 234 }
    for lang, bytes_count in langs.items():
        lang_totals[lang] = lang_totals.get(lang, 0) + bytes_count

if not lang_totals:
    print("Hiç dil bilgisi toplanamadı.")
    sys.exit(0)

# Toplam byte
total_bytes = sum(lang_totals.values())

# Yüzdeye çevir ve sırala
lang_list = sorted(
    [(lang, bytes_count, bytes_count/total_bytes*100) for lang, bytes_count in lang_totals.items()],
    key=lambda x: x[1],
    reverse=True
)

# Markdown tablo oluştur
table_lines = []
table_lines.append("| Dil | Bytes | Yüzde |")
table_lines.append("|---:|---:|---:|")
for lang, b, pct in lang_list:
    # yüzdeyi 1 ondalık göster
    table_lines.append(f"| {lang} | {b:,} | {pct:.1f}% |")

table_md = "\n".join(table_lines)

# Özet satırı (ör: diğerleri)
# (isteğe göre ilk N dil + 'Other' şeklinde de yapabilirsin)

insert_block = f"""<!--LANGUAGE-START-->
Aşağıda repolarımdaki programlama dilleri ve kullanım yüzdeleri yer almaktadır (otomatik güncellenir).

{table_md}

<!--LANGUAGE-END-->"""

# README dosyasını oku ve marker arasını değiştir
readme_path = "README.md"
if not os.path.exists(readme_path):
    print("README.md yok, oluşturuluyor.")
    with open(readme_path, "w", encoding="utf-8") as f:
        f.write(insert_block + "\n")
    changed = True
else:
    with open(readme_path, "r", encoding="utf-8") as f:
        content = f.read()
    start_marker = "<!--LANGUAGE-START-->"
    end_marker = "<!--LANGUAGE-END-->"
    if start_marker in content and end_marker in content:
        pre = content.split(start_marker)[0]
        post = content.split(end_marker)[1]
        new_content = pre + insert_block + post
    else:
        # marker yoksa dosyanın sonuna ekle
        if content.endswith("\n"):
            new_content = content + "\n" + insert_block + "\n"
        else:
            new_content = content + "\n\n" + insert_block + "\n"
    changed = new_content != content

if changed:
    with open(readme_path, "w", encoding="utf-8") as f:
        f.write(new_content)
    # commit değişiklikleri
    subprocess.run(["git", "config", "user.name", os.environ.get("GITHUB_ACTOR", "github-actions")], check=True)
    subprocess.run(["git", "config", "user.email", f"{os.environ.get('GITHUB_ACTOR','github-actions')}@users.noreply.github.com"], check=True)
    subprocess.run(["git", "add", readme_path], check=True)
    commit_msg = "chore: update language stats in README"
    subprocess.run(["git", "commit", "-m", commit_msg], check=True)
    # push (GITHUB_TOKEN kullanarak)
    repo = os.environ.get("GITHUB_REPOSITORY")
    remote = f"https://x-access-token:{TOKEN}@github.com/{repo}.git"
    subprocess.run(["git", "push", remote, "HEAD:HEAD"], check=True)
    print("README güncellendi ve pushlandı.")
else:
    print("Değişiklik yok; README güncellenmedi.")
PY
          python .github/scripts/update_readme.py

<!--LANGUAGE-END-->
