# Trollina-
The queen of the new era 
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Trollina — Codepack</title>
  <style>
    body { font-family:system-ui,Segoe UI,Roboto,Arial; background:#0b0b0b; color:#e9e9e9; padding:18px; }
    h1{ color:#6bf13b; }
    .file { margin:18px 0; border-radius:8px; background:#07120b; padding:12px; border:1px solid rgba(255,255,255,0.03); }
    .filename { font-weight:700; color:#9ff67d; margin-bottom:6px; }
    pre { background:#020302; color:#e6ffe6; padding:12px; overflow:auto; border-radius:6px; }
    code { font-family: SFMono-Regular,Menlo,Monaco,Consolas,"Liberation Mono","Courier New",monospace; font-size:13px; }
    .note { color:#bbb; margin-top:6px; font-size:13px; }
  </style>
</head>
<body>
  <h1>Trollina — Codepack</h1>
  <p class="note">This single HTML file contains each repository file shown below. Copy the content of each block into the corresponding file in your repository <strong>Leghethaboss/trollina</strong> (branch main). Follow the README for usage instructions.</p>

  <div class="file">
    <div class="filename">README.md</div>
    <pre><code># Trollina (mirror + replace scaffold)

This repository contains a scaffold and automation to mirror `whaletesticlefart.fun` and replace social links/backgrounds/memes using the first 10 images from MemeDepot (https://memedepot.com/d/trollinacto).

Important notes
- You confirmed ownership of the source site and permission to reuse the MemeDepot images.
- Analytics/tracking is removed by default.
- This repo includes scripts that will perform the mirroring and downloads locally or in CI. Running the scripts will fetch and produce the final mirrored files; commit those produced files into this repo when ready.

Repository contents
- `mirror_and_replace.sh` — main automation script (wget, download memes, optimize images, replace links/backgrounds, inject contract and MemeDepot button, remove analytics)
- `fetch_memes.py` — downloads the first N images from a MemeDepot gallery page (Selenium fallback included)
- `requirements.txt` — python dependencies for meme fetcher
- `index.html` — scaffolded header that includes the contract address (copiable) and the X community link
- `assets/` — placeholder folders (images/css/js)
- `.gitignore`

Quick usage
1. Create the repo locally and copy files into it (or use the GitHub CLI to create the remote).
2. Initialize git:
   git init -b main
3. (Optional) Create GitHub repo then push:
   gh repo create Leghethaboss/trollina --public --source=. --remote=origin --push -y
4. Install dependencies:
   pip3 install -r requirements.txt
5. Make script executable:
   chmod +x mirror_and_replace.sh
6. Run the mirror script:
   ./mirror_and_replace.sh

Primary script behavior
- Mirrors `https://whaletesticlefart.fun` into `./whale-mirror/`
- Downloads first 10 images from `https://memedepot.com/d/trollinacto` into `./whale-mirror/assets/images/`
- Converts images to `.webp` if `cwebp` is available
- Replaces Twitter/X links with your X community URL:
  `https://twitter.com/i/communities/2009975890231603467`
- Injects a "More memes @ MemeDepot" button linking to the gallery
- Injects the contract address into the header as copyable text:
  `ACHEprPEhxibTS5URUtTb44NFA14wpb7vHoTu3rZpump`
- Removes common analytics snippets (best-effort). Audit the mirrored files after running.

If you want me to add a GitHub Actions workflow to run this automatically, tell me and I will prepare the workflow YAML next.
</code></pre>
  </div>

  <div class="file">
    <div class="filename">mirror_and_replace.sh</div>
    <pre><code>#!/usr/bin/env bash
# mirror_and_replace.sh
# Mirrors a site, downloads memes from a gallery, optimizes them, replaces social links/backgrounds,
# injects contract address in header, inserts MemeDepot button, and removes analytics.
#
# Usage: ./mirror_and_replace.sh
# Requirements: wget, python3, pip packages (requests, beautifulsoup4), optional: cwebp, chromium for selenium fallback in CI

set -euo pipefail
SITE_URL="https://whaletesticlefart.fun"
MDEPOT_URL="https://memedepot.com/d/trollinacto"
WORKDIR="./whale-mirror"
IMAGEDIR="$WORKDIR/assets/images"
X_LINK="https://twitter.com/i/communities/2009975890231603467"
CONTRACT_ADDRESS="ACHEprPEhxibTS5URUtTb44NFA14wpb7vHoTu3rZpump"
MEME_COUNT=10
GIT_REMOTE="origin"
BRANCH="main"

echo "[1/9] Preparing directories..."
mkdir -p "$WORKDIR"
mkdir -p "$IMAGEDIR"

echo "[2/9] Mirroring site (this may take a while)..."
wget --mirror --convert-links --adjust-extension --page-requisites --no-parent -P "$WORKDIR" "$SITE_URL"

echo "[3/9] Fetching top $MEME_COUNT memes from MemeDepot..."
python3 fetch_memes.py "$MDEPOT_URL" "$IMAGEDIR" "$MEME_COUNT"

echo "[4/9] Converting images to webp (if cwebp is installed)..."
if command -v cwebp >/dev/null 2>&1; then
  for f in "$IMAGEDIR"/*; do
    if [[ "${f##*.}" == "webp" ]]; then
      continue
    fi
    out="${f%.*}.webp"
    echo "Converting $f -> $out"
    cwebp -q 80 "$f" -o "$out" >/dev/null 2>&1 || echo "cwebp conversion failed for $f"
    rm -f "$f" || true
  done
else
  echo "cwebp not found — leaving original formats."
fi

echo "[5/9] Replacing X/Twitter links across mirrored HTML/CSS/JS files..."
find "$WORKDIR" -type f \( -name "*.html" -o -name "*.htm" -o -name "*.css" -o -name "*.js" \) -print0 | while IFS= read -r -d '' file; do
  perl -i -pe "s#https?://(www\\.)?twitter\\.com/[^\"'\\s>]*#$X_LINK#g" "$file" || true
  perl -i -pe "s#https?://t\\.co/[^\"'\\s>]*#$X_LINK#g" "$file" || true
done

echo "[6/9] Injecting MemeDepot button into header and footer of HTML files..."
INJECT_HTML="<a class=\"meme-depot-btn\" href=\"$MDEPOT_URL\" target=\"_blank\" rel=\"noopener\">More memes @ MemeDepot</a>"
find "$WORKDIR" -type f -name "*.html" -print0 | while IFS= read -r -d '' html; do
  if grep -qi "</header>" "$html"; then
    perl -0777 -i -pe "s#</header>#$INJECT_HTML\n</header>#is" "$html" || true
  fi
  if grep -qi "</footer>" "$html"; then
    perl -0777 -i -pe "s#</footer>#$INJECT_HTML\n</footer>#is" "$html" || true
  fi
done

echo "[7/9] Injecting contract address into header as copyable field..."
CONTRACT_SNIPPET="<div class=\"contract-copy\"><label>Contract:</label><input id=\"contractAddr\" readonly value=\"$CONTRACT_ADDRESS\"><button onclick=\"navigator.clipboard.writeText(document.getElementById('contractAddr').value)\">Copy</button></div>"
find "$WORKDIR" -type f -name "*.html" -print0 | while IFS= read -r -d '' html; do
  if grep -qi "</header>" "$html"; then
    perl -0777 -i -pe "s#</header>#$CONTRACT_SNIPPET\n</header>#is" "$html" || true
  else
    perl -0777 -i -pe "s#<body[^>]*>#\$&\n$CONTRACT_SNIPPET#is" "$html" || true
  fi
done

echo "[8/9] Replacing site background references to use downloaded memes (best-effort)"
FIRST10=()
i=1
for file in "$IMAGEDIR"/*; do
  FIRST10+=("assets/images/$(basename "$file")")
  ((i++))
  if [[ $i -gt $MEME_COUNT ]]; then break; fi
done

COUNT=${#FIRST10[@]}
if [[ $COUNT -gt 0 ]]; then
  find "$WORKDIR" -type f \( -name "*.css" -o -name "*.html" -o -name "*.htm" \) -print0 | while IFS= read -r -d '' f; do
    for new in "${FIRST10[@]}"; do
      perl -i -pe "s#(background(?:-image)?\\s*:\\s*url\\([\"']?)[^\\)\"']+([\"']?\\))#\${1}$new\${2}#g" "$f" || true
      perl -i -pe "s#bg(\\.jpg|\\.png|\\.webp|\\.jpeg)#${new}#gi" "$f" || true
    done
  done
fi

echo "[9/9] Remove common analytics and trackers (Google Analytics, gtag, GTM, Matomo, Hotjar) -- best effort"
find "$WORKDIR" -type f \( -name "*.html" -o -name "*.htm" -o -name "*.js" \) -print0 | while IFS= read -r -d '' f; do
  perl -0777 -i -pe 's#<script[^>]*src=["\'][^"\']*(googletagmanager|google-analytics|gtag|analytics.js|matomo|piwik|hotjar|cdn.segment.com)[^"\']*["\'][^>]*>.*?</script>##gis' "$f" || true
  perl -0777 -i -pe 's#<script[^>]*>[^<]*(ga\\(|gtag\\(|_ga=|window\\.dataLayer|_paq\\[)[:;\\s\\S]*?</script>##gis' "$f" || true
done

echo "Done. Mirrored site is under: $WORKDIR"
echo "Please inspect the files, then commit and push to your repository:"
echo "  git add $WORKDIR"
echo "  git commit -m 'Add mirrored site (with replaced links/backgrounds) and assets'"
echo "  git push $GIT_REMOTE $BRANCH"
</code></pre>
  </div>

  <div class="file">
    <div class="filename">fetch_memes.py</div>
    <pre><code>#!/usr/bin/env python3
"""
fetch_memes.py
Download the first N image assets from a MemeDepot gallery page.

Behavior:
- Try a fast static fetch using requests + BeautifulSoup.
- If fewer than `count` images are found, fall back to a headless Selenium renderer
  (requires `selenium` and `webdriver-manager`) to capture images that require JS.

Usage:
    python3 fetch_memes.py &lt;gallery_url&gt; &lt;dest_dir&gt; &lt;count&gt;
"""
import sys
import os
import re
import time
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse

# Optional selenium imports; used only if fallback is necessary
try:
    from selenium import webdriver
    from selenium.webdriver.chrome.options import Options
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.support import expected_conditions as EC
    SELENIUM_AVAILABLE = True
except Exception:
    SELENIUM_AVAILABLE = False

HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; ImageFetcher/1.0)"}
IMG_EXT_RE = re.compile(r"\.(jpe?g|png|gif|webp)$", re.I)

def download_image(url, dest_dir, index):
    try:
        r = requests.get(url, stream=True, timeout=20, headers=HEADERS)
        r.raise_for_status()
        parsed = urlparse(url)
        ext = os.path.splitext(parsed.path)[1]
        if not ext or not IMG_EXT_RE.search(ext):
            ct = r.headers.get("Content-Type", "")
            if "jpeg" in ct: ext = ".jpg"
            elif "png" in ct: ext = ".png"
            elif "gif" in ct: ext = ".gif"
            elif "webp" in ct: ext = ".webp"
            else: ext = ".jpg"
        fname = f"meme-{index}{ext}"
        path = os.path.join(dest_dir, fname)
        with open(path, "wb") as fh:
            for chunk in r.iter_content(8192):
                fh.write(chunk)
        print(f"Saved {path}")
        return path
    except Exception as e:
        print("Error downloading", url, e)
        return None

def fetch_images_requests(gallery_url, dest_dir, count):
    print("Attempting static fetch (requests + BeautifulSoup)...")
    os.makedirs(dest_dir, exist_ok=True)
    try:
        resp = requests.get(gallery_url, headers=HEADERS, timeout=20)
        resp.raise_for_status()
    except Exception as e:
        print("Static fetch failed:", e)
        return []

    soup = BeautifulSoup(resp.text, "html.parser")
    imgs = soup.find_all("img")
    urls = []
    for img in imgs:
        src = img.get("data-src") or img.get("data-original") or img.get("src") or ""
        if not src:
            continue
        src = urljoin(gallery_url, src)
        if not IMG_EXT_RE.search(src):
            continue
        urls.append(src)
        if len(urls) >= count:
            break
    unique_urls = []
    seen = set()
    for u in urls:
        if u in seen:
            continue
        seen.add(u)
        unique_urls.append(u)
    print(f"Static fetch found {len(unique_urls)} image(s).")
    return unique_urls

def fetch_images_selenium(gallery_url, dest_dir, count, timeout=15):
    if not SELENIUM_AVAILABLE:
        print("Selenium not available in Python environment. Install `selenium` and `webdriver-manager` to enable the fallback.")
        return []

    print("Attempting Selenium fetch (headless Chromium)...")
    os.makedirs(dest_dir, exist_ok=True)
    chrome_options = Options()
    chrome_options.add_argument("--headless=new")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--window-size=1920,1080")

    use_wdm = False
    try:
        from webdriver_manager.chrome import ChromeDriverManager
        use_wdm = True
    except Exception:
        use_wdm = False

    try:
        if use_wdm:
            driver = webdriver.Chrome(ChromeDriverManager().install(), options=chrome_options)
        else:
            driver = webdriver.Chrome(options=chrome_options)
    except Exception as e:
        print("Failed to start Chrome webdriver:", e)
        return []

    urls = []
    try:
        driver.get(gallery_url)
        try:
            WebDriverWait(driver, timeout).until(
                EC.presence_of_element_located((By.TAG_NAME, "img"))
            )
        except Exception:
            pass

        imgs = driver.find_elements(By.TAG_NAME, "img")
        for img in imgs:
            src = ""
            try:
                src = img.get_attribute("data-src") or img.get_attribute("data-original") or img.get_attribute("src") or ""
            except Exception:
                src = img.get_attribute("src") or ""
            if not src:
                continue
            src = urljoin(gallery_url, src)
            if not IMG_EXT_RE.search(src):
                continue
            if src not in urls:
                urls.append(src)
            if len(urls) >= count:
                break

        scroll_tries = 0
        last_len = len(urls)
        while len(urls) < count and scroll_tries < 6:
            driver.execute_script("window.scrollBy(0, window.innerHeight);")
            time.sleep(1.2)
            imgs = driver.find_elements(By.TAG_NAME, "img")
            for img in imgs:
                try:
                    src = img.get_attribute("data-src") or img.get_attribute("data-original") or img.get_attribute("src") or ""
                except Exception:
                    src = img.get_attribute("src") or ""
                if not src: continue
                src = urljoin(gallery_url, src)
                if not IMG_EXT_RE.search(src):
                    continue
                if src not in urls:
                    urls.append(src)
                if len(urls) >= count:
                    break
            if len(urls) == last_len:
                scroll_tries += 1
            else:
                last_len = len(urls)
    finally:
        try:
            driver.quit()
        except Exception:
            pass

    print(f"Selenium fetch found {len(urls)} image(s).")
    return urls

def fetch_images(gallery_url, dest_dir, count):
    urls = fetch_images_requests(gallery_url, dest_dir, count)
    if len(urls) >= count:
        return urls[:count]

    print("Static fetch did not find enough images; attempting Selenium fallback...")
    selenium_urls = fetch_images_selenium(gallery_url, dest_dir, count)
    seen = set(urls)
    merged = list(urls)
    for u in selenium_urls:
        if len(merged) >= count:
            break
        if u not in seen:
            merged.append(u)
            seen.add(u)
    return merged[:count]

def main():
    if len(sys.argv) &lt; 4:
        print("Usage: fetch_memes.py &lt;gallery_url&gt; &lt;dest_dir&gt; &lt;count&gt;")
        sys.exit(1)
    gallery_url = sys.argv[1]
    dest_dir = sys.argv[2]
    try:
        count = int(sys.argv[3])
    except Exception:
        count = 10

    os.makedirs(dest_dir, exist_ok=True)
    urls = fetch_images(gallery_url, dest_dir, count)
    if not urls:
        print("No images found.")
        return

    saved = 0
    idx = 1
    for u in urls:
        if saved &gt;= count:
            break
        out = download_image(u, dest_dir, idx)
        if out:
            saved += 1
            idx += 1

    print(f"Downloaded {saved} image(s) to {dest_dir}")
    if saved &lt; count:
        print("Warning: fewer images were downloaded than requested.")

if __name__ == "__main__":
    main()
</code></pre>
  </div>

  <div class="file">
    <div class="filename">requirements.txt</div>
    <pre><code>requests
beautifulsoup4
selenium
webdriver-manager
</code></pre>
  </div>

  <div class="file">
    <div class="filename">index.html</div>
    <pre><code>&lt;!doctype html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
  &lt;meta charset="utf-8" /&gt;
  &lt;meta name="viewport" content="width=device-width,initial-scale=1" /&gt;
  &lt;title&gt;Trollina — mirrored scaffold&lt;/title&gt;
  &lt;link rel="stylesheet" href="assets/css/style.css" /&gt;
&lt;/head&gt;
&lt;body&gt;
  &lt;header&gt;
    &lt;h1&gt;Trollina&lt;/h1&gt;
    &lt;nav class="social"&gt;
      &lt;!-- X community link --&gt;
      &lt;a href="https://twitter.com/i/communities/2009975890231603467" target="_blank" rel="noopener"&gt;X Community&lt;/a&gt;
    &lt;/nav&gt;

    &lt;!-- Contract address copied into the header and is copyable --&gt;
    &lt;div class="contract-copy"&gt;
      &lt;label for="contractAddr"&gt;Contract:&lt;/label&gt;
      &lt;input id="contractAddr" readonly value="ACHEprPEhxibTS5URUtTb44NFA14wpb7vHoTu3rZpump"&gt;
      &lt;button id="copyContract"&gt;Copy&lt;/button&gt;
    &lt;/div&gt;
  &lt;/header&gt;

  &lt;main&gt;
    &lt;section class="hero"&gt;
      &lt;h2&gt;Mirrored site scaffold&lt;/h2&gt;
      &lt;p&gt;This is the scaffolded homepage. After running the mirror script, the mirrored files will live in ./whale-mirror/&lt;/p&gt;
      &lt;a class="meme-depot-btn" href="https://memedepot.com/d/trollinacto" target="_blank" rel="noopener"&gt;More memes @ MemeDepot&lt;/a&gt;
    &lt;/section&gt;

    &lt;section class="gallery"&gt;
      &lt;h3&gt;Background images (placeholders)&lt;/h3&gt;
      &lt;div class="bg-grid"&gt;
        &lt;div class="bg-item" style="background-image:url('assets/images/meme-1.webp')"&gt;&lt;/div&gt;
        &lt;div class="bg-item" style="background-image:url('assets/images/meme-2.webp')"&gt;&lt;/div&gt;
        &lt;div class="bg-item" style="background-image:url('assets/images/meme-3.webp')"&gt;&lt;/div&gt;
        &lt;div class="bg-item" style="background-image:url('assets/images/meme-4.webp')"&gt;&lt;/div&gt;
      &lt;/div&gt;
    &lt;/section&gt;
  &lt;/main&gt;

  &lt;footer&gt;
    &lt;p&gt;Made for Leghethaboss — repo: &lt;strong&gt;trollina&lt;/strong&gt;&lt;/p&gt;
  &lt;/footer&gt;

  &lt;script src="assets/js/main.js"&gt;&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
</code></pre>
  </div>

  <div class="file">
    <div class="filename">assets/css/style.css</div>
    <pre><code>:root{
  --accent: #6bf13b;
  --bg: #0b0b0b;
  --text: #e9e9e9;
}
*{box-sizing:border-box}
body{
  margin:0;
  font-family:system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial;
  background:var(--bg);
  color:var(--text);
  min-height:100vh;
}
header{
  display:flex;
  align-items:center;
  justify-content:space-between;
  padding:16px 24px;
  background:linear-gradient(90deg,#07120b 0%, #0b1a0b 100%);
}
header h1{margin:0;font-size:1.25rem}
nav.social a{color:var(--accent);text-decoration:none;margin-left:8px}
.contract-copy{display:flex;gap:8px;align-items:center}
.contract-copy input{
  background:#fff;color:#000;border:1px solid #ccc;padding:6px 8px;border-radius:4px;
  font-family:monospace;
}
.contract-copy button{padding:6px 8px;border-radius:4px;background:var(--accent);border:none;color:#000;cursor:pointer}

main{padding:24px}
.hero{padding:24px;background:#081008;border-radius:8px}
.meme-depot-btn{
  display:inline-block;margin-top:12px;padding:10px 14px;background:var(--accent);color:#000;border-radius:6px;text-decoration:none;
}

/* gallery placeholders */
.bg-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:12px;margin-top:12px}
.bg-item{height:160px;background-size:cover;background-position:center;border-radius:6px;border:2px solid rgba(255,255,255,0.03)}
footer{padding:18px;text-align:center;color:#999}
</code></pre>
  </div>

  <div class="file">
    <div class="filename">assets/js/main.js</div>
    <pre><code>// Simple copy-to-clipboard for the contract address
document.addEventListener("DOMContentLoaded", function () {
  var btn = document.getElementById("copyContract");
  var input = document.getElementById("contractAddr");
  if (btn && input) {
    btn.addEventListener("click", function () {
      if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard.writeText(input.value).then(function () {
          btn.textContent = "Copied!";
          setTimeout(() =&gt; (btn.textContent = "Copy"), 1500);
        });
      } else {
        input.select();
        try {
          document.execCommand("copy");
          btn.textContent = "Copied!";
          setTimeout(() =&gt; (btn.textContent = "Copy"), 1500);
        } catch (e) {
          btn.textContent = "Copy failed";
        }
      }
    });
  }
});
</code></pre>
  </div>

  <div class="file">
    <div class="filename">.gitignore</div>
    <pre><code>__pycache__/
*.pyc
.whale-mirror/
whale-mirror/
*.zip
.DS_Store
</code></pre>
  </div>

  <div class="note">Next steps: create the repo locally, copy each file into the repo, install python deps (<code>pip3 install -r requirements.txt</code>), make the script executable (<code>chmod +x mirror_and_replace.sh</code>), then run <code>./mirror_and_replace.sh</code>. If you want I can now generate a GitHub Actions workflow to run the mirror on schedule and commit results back to the repo — say "Add Actions" and I’ll provide it.</div>
</body>
</html>
