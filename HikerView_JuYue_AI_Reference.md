# HikerView / 聚阅（JuYue）Rule Development — AI Quick Reference

> **What is this?** A condensed, AI-readable reference distilled from the V3.47 internal development manual (49 chapters, ~11,000 lines). Covers architecture, all critical APIs, gotchas (called "Laws"), and common patterns. When writing rules for this platform, treat every "⚠️ Law N" as a hard constraint — violating them causes silent failures or crashes.

---

## 1. Platform Architecture

| Dimension | HikerView (base layer) | JuYue (wrapper) |
|---|---|---|
| JS Engine | Rhino (ES5 strict) | Identical — same engine |
| Built-in APIs | `fetch / request / pd / pdfh / pdfa / log` | Identical |
| Pagination variable | `MY_PAGE` / `MY_URL` | Only injected when `type:"主页"` + `url:"fypage"` |
| Video protocol | `video://url;{Header@Val}` | Same |
| Image/manga | `pics://` protocol | `解析` function must `return`, never `setResult` |
| Novel/reading | (none native) | Append `#isJiexi=1#readTheme##autoPage#` to URL |
| JS syntax | New HikerView (2026+): ES6 safe subset (const/let/arrow OK) | **Latest versions also support ES6** (see Section 20 & 24) |

**Key rule:** Latest JuYue versions support ES6 (`let`, `const`, `=>`, template literals). ES5 (`var`/`function(){}`) still works everywhere and is safe for maximum compatibility. See Section 24 for the full divergence record.

---

## 2. Rule Skeleton (parse object)

All code must live inside a single `parse` object. No globals.

```javascript
var parse = {
    作者: "dev",
    版本: "20260101.V1",
    host: "https://example.com",
    UA: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",

    // 页码: which functions support pagination
    页码: { "主页": 1, "分类": 1, "搜索": 1 },

    // 静态分类: category/filter config (see Section 3)
    静态分类: { type:"主页", url:"fypage", class_name:"...", class_url:"..." },

    // Core functions
    主页: function() { /* build card array, return d */ },
    二级: function(url) { /* return { vod_name, vod_pic, line, list } */ },
    解析: function(url) { /* return "video://..." string */ },
    搜索: function(name) { /* return d */ },

    // Utility helpers (call with this._base64Decode(...) from main functions)
    _base64Decode: function(str) {
        if (!str) return "";
        str = str.replace(/-/g,"+").replace(/_/g,"/");
        while (str.length % 4) str += "=";
        try {
            var b = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
            return new java.lang.String(b, "UTF-8") + "";
        } catch(e) {
            try { return decodeURIComponent(escape(atob(str))); }
            catch(e2) { log("Base64 decode failed"); return ""; }
        }
    },

    _autoDeshell: function(html) {
        if (!html) return html;
        for (var i = 0; i < 3; i++) {
            if (html.indexOf('decodeURIComponent("') > -1) {
                var m = html.match(/decodeURIComponent\("([^"]+)"\)/);
                if (m) html = decodeURIComponent(m[1]);
            } else break;
        }
        return html;
    }
};
```

---

## 3. Static Category Config (静态分类)

### 3.1 Three modes

| Mode | type | url | When to use |
|---|---|---|---|
| Standard paginated | `"主页"` | `"fypage"` | Single-dimension categories with pagination |
| With dropdown + pagination | `"主页"` | `"hiker://empty##fyclass##fypage"` | Top dropdown + pagination |
| Matrix mode | `"主页"` | `"0"` | Complex multi-row filtering (see Section 6) |

**⚠️ Law: Never use `type:0`.** `MY_PAGE` is not injected at all with `type:0` — the engine errors at parse time. Always use `type:"主页"` + `url:"fypage"`.

### 3.2 Five placeholder dimensions

```javascript
静态分类: {
    type: "主页",
    url: "https://example.com/list/fyclass-fyarea-fysort---fyyear---fypage.html",
    class_name: "电影&电视剧&动漫",  class_url: "1&2&3",
    area_name:  "全部&大陆&日本",     area_url:  "&大陆&日本",
    year_name:  "全部&2026&2025",     year_url:  "&2026&2025",
    sort_name:  "时间&热度",          sort_url:  "time&hits"
}
```

| Placeholder | Field pair | Notes |
|---|---|---|
| `fyclass` | `class_name` / `class_url` | Main category |
| `fyarea` | `area_name` / `area_url` | Region filter |
| `fyyear` | `year_name` / `year_url` | Year filter |
| `fysort` | `sort_name` / `sort_url` | Sort order |
| `fypage` | (auto-managed) | Page number — never define manually |

**Iron rule:** `name` and `url` arrays must have **identical item counts** (separated by `&`). "All" option has an empty string for its url value.

### 3.3 fypage advanced syntax

```javascript
// Page starts at 0, step 20 → 0, 20, 40...
"url": "https://api.example.com/list?start=fypage@-1@*20@&size=20"

// Page starts at 0, step 1 → 0, 1, 2...
"url": "https://api.example.com/list?page=fypage@-1@"

// Different URL for page 1
"url": "https://example.com/list/fypage/[firstPage=https://example.com/list/]"

// fypage must NOT be at end of URL — append dummy param if needed
"url": "https://example.com/fypage.html?_t=0"
```

---

## 4. Built-in HTML Parsing Functions (pd / pdfh / pdfa)

| Function | Alias | Returns | Notes |
|---|---|---|---|
| `parseDom(html, rule, baseUrl)` | `pd` | String (auto-completes domain) | Use for href/src that need domain prepended |
| `parseDomForHtml(html, rule)` | `pdfh` | String (raw) | No domain processing |
| `parseDomForArray(html, rule)` | `pdfa` | Array of strings | For iterating lists |

### Selector syntax

| Syntax | Meaning | Example |
|---|---|---|
| `&&` | Descend into child | `"body&&.list&&li"` |
| `--` | Exclude element | `"body--script&&a&&href"` |
| `tag,N` | Get Nth element (0-indexed) | `"body&&a,2"` (3rd `<a>`) |
| `tag,-1` | Get last | `"body&&li,-1"` |
| `Text` | Get text content | `"h2&&Text"` |
| `Html` | Get inner HTML | `"div&&Html"` |
| `attrName` | Get attribute | `"img&&src"`, `"a&&href"` |
| `[attr*="val"]` | Fuzzy class match | `[class*="item"]` |
| `\|\|` | OR (try first, fallback) | `"#app\|\|#app2&&Text"` |
| `.js:code` | JS post-processing | `"a&&href.js:input.replace('\\\\','/')"` |
| `js:code` | Full JS rule | `"js:parseDom(html,'...')+'/page/1'"` |

---

## 5. Network Request Functions

| Function | Purpose | Key options |
|---|---|---|
| `fetch(url, opts)` | Standard GET/POST | `{headers, body, method, timeout, withHeaders, withStatusCode, redirect}` |
| `fetchPC(url, opts)` | Same with PC UA | same |
| `request(url, opts)` | Alias for fetch | same |
| `post(url, {body})` | POST with form encoding | `{body: {key: val}}` |
| `batchFetch(arr)` | Concurrent requests (**max 16 per batch**) | `[{url, options}, ...]` |
| `fetchCookie(url, opts)` | Get Set-Cookie headers | Returns JSON string of cookie array |
| `fetchCodeByWebView(url, opts)` | WebView-rendered source | `{headers, blockRules, timeout, checkJs}` |

```javascript
// With custom headers
var html = fetch(url, { headers: { "User-Agent": parse.UA, "Referer": parse.host+"/" } });

// Get response headers (e.g. Set-Cookie)
var res = JSON.parse(fetch(url, { withHeaders: true }));
var ck = res.headers["Set-Cookie"][0];

// GBK site
var html = fetch(url, { headers: { "content-type": "text/html; charset=GBK" } });

// Disable auto-cookie
var html = fetch(url, { headers: { Cookie: "#noCookie#" } });
```

---

## 6. Variable Storage

| Function | Scope | Lifetime | Use for |
|---|---|---|---|
| `setItem(k,v)` / `getItem(k,def)` | Current rule | Persistent (survives app restart) | CF cookies, tokens, domain cache |
| `putMyVar(k,v)` / `getMyVar(k,def)` | Current rule | Session (lost on restart) | Filter state, page index, temp selection |
| `putVar(k,v)` / `getVar(k,def)` | Global (cross-rule) | Session | One-shot cross-rule data |
| `storage0.putMyVar(k,obj)` | Current rule | Session | Store JSON objects |
| `storage0.setItem(k,obj)` | Current rule | Persistent | Persist JSON objects |

**⚠️ Law:** Keys for `getMyVar`/`putMyVar` must be **pure alphanumeric + underscore only**. Keys containing `://` or `/` (e.g., using `host` as prefix) will silently fail — values are stored but never retrieved. Use site abbreviation prefix: `hm_cindex0`, `cz_page`, etc.

**⚠️ Law:** CF cookies must use `setItem` (persistent), not `putMyVar` (session). `putMyVar` is lost on restart, forcing re-verification every time.

---

## 7. URL Tags (append to URL strings to modify behavior)

These tags are stripped by the engine before the actual request.

| Tag | Effect |
|---|---|
| `#noLoading#` | No loading spinner |
| `#noHistory#` | Don't record in history |
| `#readTheme#` | Reading mode (eye-care background, font, progress) |
| `#autoPage#` | Auto-load next page on scroll (novel chapters) |
| `#isJiexi=1#` | Trigger JuYue built-in reader/player |
| `#isVideo=true#` | Force treat as video |
| `#ignoreVideo=true#` | Force treat as non-video |
| `#isMusic=true#` | Force treat as audio |
| `#pre#` / `#noPre#` | Force pre-load / disable pre-load |
| `#concat#` | Merge multiple video segments |
| `#fastPlayMode#` | Multi-thread streaming playback |
| `#background#` | Background playback |

```javascript
// Novel chapter URL standard pattern
var chapterUrl = pd(it, "a&&href") + "#isJiexi=1#readTheme##autoPage#";

// Multi-segment video
var url = "video://seg1.m3u8#concat#seg2.m3u8";
```

---

## 8. Card Layout Types (col_type)

| col_type | Description |
|---|---|
| `movie_3` / `movie_3_marquee` | 3-column grid, rounded image + title (default) |
| `movie_2` / `movie_1` | 2-col / 1-col grid |
| `movie_1_vertical_pic` | Vertical image left, text right (books/manga) |
| `text_1` ~ `text_5` | Text-only 1–5 columns |
| `rich_text` | HTML-formatted text |
| `long_text` | Unlimited length text |
| `pic_1_full` | Full-width image, auto height |
| `pic_3` / `pic_2` | Image grid 3/2 columns |
| `scroll_button` | Horizontal scrolling filter button (no HTML in title) |
| `flex_button` | Flow-wrap filter button (supports HTML title) |
| `blank_block` | 1dp spacer |
| `line` | Divider line |
| `input` | Single-line text input |
| `x5_webview_single` | X5 WebView (max one per page) |
| `avatar` | Avatar style (image + title + right desc) |

**⚠️ Law:** `scroll_button` does NOT render HTML in titles — tags display as raw text. Use plain text with symbols (e.g., `❆ Selected ❆`) for selected state. `flex_button` supports HTML.

---

## 9. The `$` Utility Namespace

The `$` object is the core runtime utility. It generates dynamic URL strings that drive navigation, lazy evaluation, and image decryption.

### 9.1 Method overview

| Method | Runs when | Must return | Use case |
|---|---|---|---|
| `$().rule(fn, ...args)` | User taps → opens new page | call `setResult(d)` inside | CF bypass sandbox, novel content page |
| `$().lazyRule(fn, ...args)` | User taps → returns new URL | URL string (`video://`, `pics://`, etc.) | Lazy video URL resolution |
| `$().image(fn, ...args)` | Image loads (async) | InputStream | Anti-hotlink covers, AES-encrypted images |
| `$().x5Lazy()` / `$().webLazy()` | X5/Webkit WebView sniff | (auto) | JS-rendered pages that can't be fetched |

### 9.2 lazyRule — critical rules

**⚠️ Law 2 (Scope isolation):** `lazyRule` / `rule` / `image` callbacks run in an **isolated scope**. They have NO access to outer variables. All values must be passed explicitly as arguments.

**⚠️ Law 52 (No objects in lazyRule params):** Only pass `string`, `number`, `boolean`. Passing an object serializes it to `"[object Object]"`, silently breaking request headers.

**⚠️ Law 61 (Don't concatenate lazyRule):** `url + $('').lazyRule(fn)` does NOT work. The engine treats it as a plain URL. Always use `$(detailUrl).lazyRule(fn, ...args)` — `input` inside the callback equals `detailUrl`.

```javascript
// ✅ CORRECT
var _ua = this.UA;
var _host = this.host;
url: $(detailUrl).lazyRule(function(ua, host) {
    // input = detailUrl (automatically)
    var html = fetch(input, { headers: { "User-Agent": ua, "Referer": host+"/" } });
    var m = html.match(/https?:\/\/[^\s'"]+\.m3u8/);
    if (m) return "video://" + m[0] + ";{Referer@" + host + "/}";
    return input + "#嗅探";
}, _ua, _host)

// ❌ WRONG: concatenation
url: detailUrl + $("").lazyRule(fn, ua)  // does nothing useful

// ❌ WRONG: passing object
url: $(link).lazyRule(function(link, headers) {
    fetch(link, { headers: headers });  // headers = "[object Object]"
}, link, { "User-Agent": ua })
```

### 9.3 Modular development

```javascript
// In dependency page (hiker://page/utils):
$.exports.formatTitle = function(t) { return t.replace(/\s+/g,'').trim(); };

// In main rule:
var utils = $.require('hiker://page/utils');
var title = utils.formatTitle(rawTitle);

// ⚠️ $.require inside rule()/lazyRule() sandbox: the name must match
// the exact interface name the rule is saved under in JuYue, not a file path.
// When in doubt: inline the logic instead of using $.require.
```

---

## 10. Image Handling

### 10.1 Anti-hotlink / large images — FileUtil byte stream (Law 28)

**Never process Base64 images as JS strings** — causes Error 36 (OOM). Use the byte stream path:

```javascript
pic_url: $(picUrl).image({
    image: function() {
        var FileUtil = com.example.hikerview.utils.FileUtil;
        try {
            // input = the URL passed to $()
            var raw = FileUtil.readBytes(input, {
                "User-Agent": parse.UA,
                "Referer": parse.host + "/"
            });
            return FileUtil.toInputStream(raw);
        } catch(e) { log("image stream failed: " + e); return null; }
    }
})
```

### 10.2 XOR decryption (first N bytes)

```javascript
var raw = FileUtil.readBytes(input, { "User-Agent": parse.UA });
var key = "2019ysapp";
for (var i = 0; i < Math.min(raw.length, 100); i++) {
    var res = (raw[i] ^ key.charCodeAt(i % key.length)) & 0xff;
    raw[i] = res > 127 ? res - 256 : res;
}
return FileUtil.toInputStream(raw);
```

### 10.3 AES-encrypted images — CryptoUtil

```javascript
// Define ONCE before loop, reuse by concatenating to pic_url
var tlazy = $("").image(function(k, iv) {
    var CryptoUtil = $.require("hiker://assets/crypto-java.js");
    var dec = CryptoUtil.AES.decrypt(
        CryptoUtil.Data.parseInputStream(input),
        CryptoUtil.Data.parseUTF8(k),
        { mode: "AES/CBC/PKCS7Padding", iv: CryptoUtil.Data.parseUTF8(iv) }
    );
    return dec.toInputStream();
}, KEY, IV);

// In loop:
d.push({ pic_url: imgSrc + tlazy, ... });  // append, don't call again
```

**⚠️ Law:** Never call `_getImageDecrypt()` or `$().image()` inside a loop. Call once, reuse the string.

---

## 11. Video URL Protocols

```javascript
// Direct video
"video://https://cdn.example.com/video.m3u8"

// With headers
"video://https://cdn.example.com/video.m3u8;{User-Agent@Mozilla/5.0&&Referer@https://example.com/}"

// Sniffer fallback
detailUrl + "#嗅探"

// Manga/image set
"pics://https://img1.com/1.jpg&&https://img2.com/2.jpg&&..."

// Image with Referer
"pics://https://img.com/1.jpg@Referer=https://site.com/&&..."

// Novel chapter content
richTextContent  // return string directly from 解析 function
```

**⚠️ Law 30 (Backslash isolation):** Never write `\\` directly in code — use `String.fromCharCode(92)` to avoid Rhino escape bugs.

```javascript
var slash = String.fromCharCode(92);
var cleanUrl = realUrl.split(slash).join("");
return "video://" + cleanUrl + ";{Referer@" + host + "/}";
```

---

## 12. Secondary Page (二级) Return Formats

### Standard video/anime (with episode list)
```javascript
二级: function(url) {
    var html = fetch(url, { headers: { "User-Agent": this.UA } });
    var tabs = ["线路1", "线路2"];
    var lists = [
        [ { title:"EP1", url:"$(ep1Url).lazyRule(...)" }, ... ],
        [ { title:"EP1", url:"$(ep1Url).lazyRule(...)" }, ... ]
    ];
    return { vod_name: pdfh(html,"h1&&Text"), vod_pic: pd(html,"img&&src"), line: tabs, list: lists };
}
```

### Custom layout (actor profile, article detail)
```javascript
return {
    vod_name: name,
    vod_pic:  avatar,
    noShow:   { 封面: true, 简介: true, 排序: true, 选集: true },
    list:     [['']]  // ⚠️ REQUIRED: must be [['']] not [] or [[]]
    extenditems: d    // custom content rendered below
};
```

**⚠️ Law 94:** When using `extenditems`, `list` must be `[['']]` (a 2D array with one empty string). An empty `[]` or `[[]]` causes a structure error.

---

## 13. MACCMS player_aaaa Decryption

Many Chinese video sites (MACCMS v10) use this pattern:

```javascript
_parseVideo: function(playUrl) {
    var _ua = this.UA;
    var _host = this.host;
    var html = this._autoDeshell(request(playUrl, { headers: { "User-Agent": _ua } }));
    
    // encrypt: 0=plain, 1=URL-encoded, 2=Base64+URL-encoded
    var match = html.match(/var player_aaaa\s*=\s*(\{[\s\S]*?\})(?=;\s*<\/script|}\|\]\]>)/);
    if (!match) return playUrl + "#嗅探";
    
    try {
        var cfg = JSON.parse(match[1]);
        var encUrl = cfg.url;
        var realUrl = cfg.encrypt == 2 ? unescape(this._base64Decode(encUrl))
                    : cfg.encrypt == 1 ? unescape(encUrl)
                    : encUrl;
        var slash = String.fromCharCode(92);
        return "video://" + realUrl.split(slash).join("") + ";{User-Agent@" + _ua + "&&Referer@" + _host + "/}";
    } catch(e) {
        log("parse failed: " + e);
        return playUrl + "#嗅探";
    }
}
```

---

## 14. Cloudflare Bypass Standard Pattern

```javascript
UA: "Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36",

_fetch: function(url) {
    var d = [];
    var _host = this.host, _ua = this.UA;
    
    // Step1: migrate cookie from session var → persistent storage
    var tempCk = getVar("site_ck", "");
    if (tempCk !== "") { setItem("site_cookie", tempCk); clearVar("site_ck"); }
    
    var _cookie = getItem("site_cookie", "");
    var html = fetch(url, { headers: { "User-Agent": _ua, "Referer": _host+"/", "Cookie": _cookie } });
    
    // ✅ Check by content presence, NOT html.length
    if (html.indexOf("video-card-class") === -1) {
        d.push({
            title: "🛡️ CF verification — tap to bypass",
            col_type: "text_center_1",
            url: $().rule(function(targetUrl, ua) {
                var res = [{
                    title: "Verify (auto-returns on success)",
                    url: targetUrl,
                    col_type: "x5_webview_single",
                    desc: "float&&100%",
                    extra: {
                        ua: ua,
                        js: $.toString(function() {
                            function check() {
                                var c = fba.getCookie(location.href);
                                var nodes = document.querySelectorAll(".video-card-class");
                                // ✅ BOTH conditions must be true before returning
                                if (nodes && nodes.length > 0 && c && c.indexOf("cf_clearance") > -1) {
                                    fba.putVar("site_ck", c);  // temp hand-off
                                    fba.toast("✅ Bypass success!");
                                    fba.back();
                                } else { setTimeout(check, 1000); }
                            }
                            check();
                        })
                    }
                }];
                setResult(res);
            }, url, _ua)
        });
        return d;
    }
    return this._parseList(html);
}
```

---

## 15. Matrix Classification (矩阵方案) — Multi-row Filter

Use this pattern when a site has many categories in multiple independent filter dimensions (e.g., 9 rows × 30+ items each).

```javascript
静态分类: { type:"主页", url:"0", class_name:"matrix", class_url:"0" },
页码: { "主页": 1 },

主页: function() {
    var d = [];
    var h = "site_";  // namespace prefix — pure alphanumeric (Law 51)
    
    var dataClass = [
        { t:"All&Action&Comedy&Drama", i:"0&/action&/comedy&/drama" },
        { t:"All&HD&4K",              i:"0&hd&4k" },
    ];
    
    var renderRow = function(rowIdx, titleStr, idStr, d, h) {
        var tArr = titleStr.split("&"), iArr = idStr.split("&");
        var curIdx = getMyVar(h + "cindex" + rowIdx, "0");
        for (var i = 0; i < tArr.length; i++) {
            var isSel = (i == curIdx);
            d.push({
                title: isSel ? "❆ " + tArr[i] + " ❆" : tArr[i],
                url: $("#noLoading#").lazyRule(function(rIdx, idx, val, h) {
                    putMyVar(h+"cindex"+rIdx, idx+"");
                    putMyVar(h+"curl"+rIdx, val);
                    if (rIdx == 0) {
                        for (var n = 1; n < 10; n++) {
                            putMyVar(h+"cindex"+n, "0");
                            putMyVar(h+"curl"+n, "");
                        }
                    }
                    refreshPage(false);
                    return "hiker://empty";
                }, rowIdx, i, iArr[i], h),
                col_type: "scroll_button"
            });
        }
        d.push({ col_type: "blank_block" });
    };
    
    if (MY_PAGE == 1) {
        // Row 0 always shown; lower rows shown conditionally
        renderRow(0, "Latest&Hot&Categories", "/&/hot&cat", d, h);
        if (getMyVar(h+"cindex0","0") == "2") {
            for (var j = 0; j < dataClass.length; j++) {
                renderRow(j+1, dataClass[j].t, dataClass[j].i, d, h);
            }
        }
    }
    
    // Build URL from selected state
    var c0 = getMyVar(h+"cindex0","0");
    var fetchUrl = host;
    if (c0 == "0") {
        fetchUrl = host + "/latest/" + MY_PAGE;
    } else if (c0 == "1") {
        fetchUrl = host + "/hot/" + MY_PAGE;
    } else {
        var slug = "/default/";
        for (var k = 1; k < 10; k++) {
            if (getMyVar(h+"cindex"+k,"0") != "0") {
                slug = getMyVar(h+"curl"+k, slug);
                break;
            }
        }
        fetchUrl = host + slug + "?page=" + MY_PAGE;
    }
    
    var html = fetch(fetchUrl, { headers: { "User-Agent": this.UA } });
    // ... parse cards, return d.concat(cards)
    return d;
}
```

---

## 16. Category Architecture Decision Tree

```
Q1: Does the site have multiple content types (video / actor / playlist / login)?
→ YES: Use categoryList + switch(type) pattern (complex, requires $.require or inline)
→ NO : Go to Q2

Q2: Do categories exceed 4 dimensions, or is the URL structure irregular?
→ YES: Use Matrix pattern (Section 15) — getMyVar rows, manual URL build
→ NO : Go to Q3

Q3: Few categories, fixed URL pattern?
→ YES: Use 静态分类 placeholder method (Section 3) — zero code, engine auto-manages
→ NO : Use Matrix pattern
```

**Additional constraint — Law 67 (sandbox HTTP opens browser):**
Inside a `$().rule()` sandbox, HTTP card URLs always open the external browser — the engine doesn't know which rule function to call. If the site needs a 二级 function (detail page), you **must** use the Matrix pattern (cards in main page context). The sandbox `$().rule()` approach only works for sites that return `video://`, `pics://`, or `hiker://` directly (no 二级 needed).

---

## 17. Engine Error Reference

| Error code | Cause | Fix |
|---|---|---|
| Error 14 | Undefined variable / used `let`/`const` / scope leak | Use `var`; pass values as params; check for undeclared vars |
| Error 31 | Syntax error (often in regex) | Use `new RegExp()` for dynamic regex |
| Error 35 | Complex object nested in lazyRule | Simplify lazyRule body |
| Error 36 | String too long / OOM | Use FileUtil byte stream for images; avoid huge string concat |
| Error 38 | Unclosed string (has newline) | Escape newlines as `\n` |
| Error 43 | Illegal backslash | Use `String.fromCharCode(92)` everywhere |

---

## 18. High-Frequency Bug Patterns

| Symptom | Root cause | Fix |
|---|---|---|
| `MY_PAGE` undefined | `type:0` doesn't inject it | Use `type:"主页"` + `url:"fypage"` |
| Video taps open browser | lazyRule returns http URL from sandbox | Add `#嗅探` fallback or fix URL extraction |
| `scroll_button` selected state unstable | HTML in title (unsupported) | Use plain text symbols: `❆ Title ❆` |
| `getMyVar` key silently fails | Key contains `://` or `/` | Use pure alphanumeric prefix like `site_cindex0` |
| Manga images blank | Lazy-load: `src` is placeholder | Extract `data-original` or `data-src` |
| Double domain in URL | `putMyVar` stored full URL | Strip host before storing: `.replace(host,"")` |
| CF cookie re-verifies every time | Used `putMyVar` for CF cookie | Use `setItem` (persistent) |
| `lazyRule` variable undefined | Closure isolation | Pass all needed values as explicit args |
| `this.UA` fails in lazyRule | `this` context lost | `var _ua = this.UA;` before, pass `_ua` as param |
| `atob()` not defined | Rhino has no browser globals | Use `android.util.Base64.decode(str, android.util.Base64.DEFAULT)` |

---

## 19. Encoding & Crypto

### Base64 (Law 62)
```javascript
// Rhino has NO atob()/btoa()
// Decode:
var raw = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
var decoded = new java.lang.String(raw, "UTF-8") + "";  // + "" forces JS string

// Encode:
android.util.Base64.encodeToString(bytes, android.util.Base64.NO_WRAP)
```

### Caesar cipher on byte array (Law 63)
```javascript
// ❌ WRONG: split("") breaks multi-byte UTF-8 characters
// ✅ CORRECT: operate on byte[] directly
var bytes = android.util.Base64.decode(ev.d, android.util.Base64.DEFAULT);
var k = ev.k & 0xff;
for (var i = 0; i < bytes.length; i++) {
    bytes[i] = ((bytes[i] & 0xff) - k + 256) & 0xff;
}
var decoded = new java.lang.String(bytes, "UTF-8") + "";
```

### AES via CryptoUtil
```javascript
var CryptoUtil = $.require("hiker://assets/crypto-java.js");
var key = CryptoUtil.Data.parseUTF8("your_key_16chars");
var iv  = CryptoUtil.Data.parseUTF8("your_iv__16chars");
var dec = CryptoUtil.AES.decrypt(
    CryptoUtil.Data.parseInputStream(input),  // or parseBytes / parseBase64
    key,
    { mode: "AES/CBC/PKCS7Padding", iv: iv }  // also: ECB, NoPadding
);
// Returns InputStream (for images) or call .toText() for string
```

**⚠️ Law 93 (padding must be confirmed from source):** Never assume `PKCS7Padding`. Check the site's JS source for `NoPadding`, `ZeroPadding`, etc. Wrong padding produces garbage silently.

### 19.4 Built-in getCryptoJS() (latest JuYue)

Latest JuYue versions ship CryptoJS as a built-in. No `$.require` or remote loading needed:

```javascript
// Load CryptoJS via built-in function (latest JuYue only)
eval(getCryptoJS());

// Then use the standard CryptoJS API directly:
var decrypted = CryptoJS.AES.decrypt(
    encryptedStr,
    CryptoJS.enc.Utf8.parse(key),
    {
        iv:   CryptoJS.enc.Utf8.parse(iv),
        mode: CryptoJS.mode.CBC,
        padding: CryptoJS.pad.Pkcs7
    }
);
var plaintext = decrypted.toString(CryptoJS.enc.Utf8);
```

If `getCryptoJS()` is unavailable (older engine), fall back to `$.require("hiker://assets/crypto-java.js")` (see Section 19.3).

---

## 20. ES5 Safety Rules (Law 34) — partially superseded in latest JuYue

> **⚠️ Updated (see Section 24):** The latest JuYue versions allow ES6 syntax. The table below remains valid for old-engine compatibility. If you know you're targeting a recent JuYue build, ES6 is fine. When in doubt or when targeting the widest audience, ES5 is still the safer choice.

| ❌ ES6+ (old-engine banned) | ✅ ES5 safe equivalent | Latest JuYue |
|---|---|---|
| `let x = 1` / `const x = 1` | `var x = 1` | ✅ OK |
| `x => x * 2` | `function(x) { return x * 2; }` | ✅ OK |
| `` `Hello ${name}` `` | `"Hello " + name` | ✅ OK |
| `obj?.prop` | `obj && obj.prop` | ✅ OK |
| `[...arr]` / `{...obj}` | `arr.slice()` / `$.extend({}, obj)` | ✅ OK |
| `class Foo {}` | Standard prototype pattern | ✅ OK |
| `async/await` | Callbacks or synchronous flow | ❌ Still unsupported |

**Variable naming (Law 23) — still applies regardless of ES version:**
```javascript
// ❌ var title, var videoUrl  (can trigger Rhino regex flag issues)
// ✅ var _title, var _videoUrl
```

---

## 21. Complete Laws Index (quick lookup)

| Law | Core rule |
|---|---|
| Law 2 | lazyRule/rule/image are isolated scopes — pass all values as explicit params |
| Law 16 | Line/episode index (SID) must be matched to its list — never mix up order |
| Law 23 | Local variable names: use `_` prefix to avoid Rhino regex flag conflicts |
| Law 26 | `$().lazyRule`: always explicit params, never `this` |
| Law 28 | Image byte stream: must use `FileUtil.readBytes + toInputStream`, no JS string processing |
| Law 30 | Backslash: always `String.fromCharCode(92)`, never write `\\` directly |
| Law 34 | ES5 only for old JuYue engine — **latest JuYue supports ES6** (let/const/arrow/template literals OK); `async/await` still unsupported |
| Law 39 | MACCMS player_aaaa: handle encrypt 0/1/2 correctly |
| Law 51 | `getMyVar`/`putMyVar` keys: pure alphanumeric + underscore only |
| Law 52 | lazyRule params: only string/number/boolean — objects become `"[object Object]"` |
| Law 53 | URL param order: must match actual captured request exactly |
| Law 54 | Empty params must still be present: `sort=&date=` not omitted |
| Law 55 | Multi-select array params: `tags[]=val` not `tags%5B%5D=val` |
| Law 56 | Filter ad links: skip cards where `link.indexOf(host) === -1` |
| Law 57 | Multi-select button rows must include a clear/reset button |
| Law 58 | Tag values: use `input.value` from HTML source, never guess from display text |
| Law 60 | In fyclass mode, `MY_URL` is the path string, NOT a numeric index |
| Law 61 | lazyRule must be `$(url).lazyRule(fn)` — string concatenation doesn't trigger it |
| Law 62 | Rhino has no `atob`/`btoa` — use `android.util.Base64` |
| Law 63 | Multi-byte decryption: operate on `byte[]`, don't `split("")` first |
| Law 67 | `$().rule()` sandbox: HTTP card URLs always open browser — incompatible with 二级 |
| Law 83 | `$.require("name")` inside sandbox: `name` must match the JuYue save interface name exactly — OR use built-in paths like `hiker://assets/crypto-java.js` |
| Law 84 | Sub-category state: clear when top-level category changes to prevent cross-category residue |
| Law 88 | No remote loading: never use `rc()`/`gfd()`/`fc()` — inline all dependencies |
| Law 89 | Domain cache must include timestamp — 1-hour TTL |
| Law 93 | AES padding: confirm from site JS, never assume PKCS7 |
| Law 94 | `extenditems`: `list` must be `[['']]`, not `[]` |
| Law 95 | Interleaved image+text: iterate all child nodes `pdfa('container&&*')`, don't extract separately |
| Law 96 | `eval` is last resort per manual — **latest JuYue uses `eval(getCryptoJS())` as built-in pattern**; `eval` for dynamic lib loading is now acceptable |

---

## 22. Domain Self-Healing Pattern

For sites that change domains frequently:

```javascript
_getHost: function() {
    var cached   = getItem("site_host", "");
    var cachedTs = parseInt(getItem("site_host_time", "0"));
    if (cached && (Date.now() - cachedTs) < 3600000) return cached;  // 1-hour cache

    try {
        var html = fetchCodeByWebView("https://announcement-page.com/", { timeout: 8000 });
        var urls = pdfa(html, "#list-wrap&&.btnLink").map(function(item) {
            return pdfh(item, "Text");
        });
        var tasks = urls.map(function(u) { return { url: "https://"+u, options: { timeout: 3000 } }; });
        var results = batchFetch(tasks);
        for (var i = 0; i < results.length; i++) {
            if (results[i] && results[i].indexOf(".post-card") > -1) {
                var host = "https://" + urls[i].replace(/\/$/, "");
                setItem("site_host", host);
                setItem("site_host_time", Date.now() + "");
                return host;
            }
        }
    } catch(e) { log("domain discovery failed: " + e); }

    setItem("site_host", this._defaultHost);
    setItem("site_host_time", Date.now() + "");
    return this._defaultHost;
},
```

---

## 23. Useful Miscellaneous APIs

```javascript
log(msg)                         // Debug output
toast(msg)                       // Short popup toast
refreshPage(false)               // Refresh page (false = don't scroll to top)
back(true)                       // Close page; true = also refresh parent
setPageTitle("New Title")        // Dynamically change page title
updateItem(id, obj)              // Update a card in-place without full refresh
addItemAfter(id, obj)            // Insert card after id
deleteItem(id)                   // Remove card by id

// Inject variables (read-only — never reassign)
MY_PAGE   // current page number (number type, no parseInt needed)
MY_URL    // current request URL (after placeholder substitution)
MY_HOME   // root domain extracted from MY_URL
MY_RULE.title  // rule display name
MOBILE_UA // built-in mobile UA
PC_UA     // built-in PC UA

// Encoding
base64Encode(s) / base64Decode(s)
buildUrl(url, paramsObject)      // Appends GET params safely
```

---

*Distilled from HikerView / JuYue Rule Development Manual V3.47, 2026. 49 chapters. Verified against Anime7, CZBooks, hanime1.me, and other production rules.*

---

## 24. Divergences: Manual vs. Actual Latest JuYue Behavior

> This section records confirmed differences between what the V3.47 manual specifies and what the current latest JuYue version actually supports. When they conflict, **actual behavior wins**. The manual constraints remain valid for old-engine / maximum-compatibility targets.

### 24.1 Full Divergence Table

| # | Topic | Manual says | Actual latest JuYue | Impact |
|---|---|---|---|---|
| 1 | **ES6 syntax** | Law 34: ES5 mandatory — no `let`/`const`/`=>`/template literals | ✅ All ES6 syntax works: `let`, `const`, arrow functions, template literals | Can write cleaner, shorter code |
| 2 | **`eval` usage** | Law 96: `eval` is last resort | ✅ `eval(getCryptoJS())` is the canonical built-in pattern for loading CryptoJS | `eval` for dynamic lib loading is now standard |
| 3 | **CryptoJS source** | Not mentioned; manual uses `$.require("hiker://assets/crypto-java.js")` | ✅ Built-in `getCryptoJS()` function — no `$.require` needed | Simpler one-liner; no path dependency |
| 4 | **`$.require` paths** | Law 83: name must match the JuYue-saved interface name (fragile) | ✅ Built-in asset paths work directly: `hiker://assets/crypto-js.js`, `hiker://assets/crypto-java.js` | No need to save rules under specific interface names for crypto |
| 5 | **`post()` body format** | Section 5: `post(url, {body: {key: val}})` | ✅ Both object body and manually-joined string body work | Either style is fine |

### 24.2 Laws Confirmed Still Valid

These constraints have been verified to still apply in the latest version:

| Law | Rule | Status |
|---|---|---|
| Law 2 | `lazyRule`/`rule`/`image` scope isolation — must pass all values as params | ✅ Still enforced |
| Law 30 | Never write `\\` directly — use `String.fromCharCode(92)` | ✅ Still required |
| Law 51 | `getMyVar`/`putMyVar` keys: alphanumeric + underscore only | ✅ Still silently fails with `://` or `/` |
| Law 61 | `$(url).lazyRule(fn)` — concatenation does not trigger it | ✅ Still true |
| Law 67 | `$().rule()` sandbox: HTTP URLs open browser, not rule 二级 | ✅ Still true |
| Law 94 | `extenditems` requires `list: [['']]` | ✅ Still required |

### 24.3 Practical Guidance for New Rules

```javascript
// ✅ Modern style — safe for latest JuYue
const parse = {
    host: "https://example.com",
    UA: MOBILE_UA,

    主页() {
        const d = [];
        // getCryptoJS() for AES — no $.require needed
        eval(getCryptoJS());
        const key = "abcdef1234567890";
        const decrypted = CryptoJS.AES.decrypt(encStr, CryptoJS.enc.Utf8.parse(key), {
            mode: CryptoJS.mode.CBC,
            padding: CryptoJS.pad.Pkcs7,
            iv: CryptoJS.enc.Utf8.parse(key)
        }).toString(CryptoJS.enc.Utf8);

        // Arrow functions in loops are fine
        const items = pdfa(html, "body&&.list&&li");
        items.forEach(item => {
            const title = pdfh(item, "a&&Text");
            const url   = pd(item, "a&&href", this.host);
            d.push({ title, url, col_type: "movie_3" });
        });
        return d;
    },

    // Law 2 still applies — self/params still required in lazyRule
    _buildLazy(detailUrl) {
        const _ua   = this.UA;
        const _host = this.host;
        return $(detailUrl).lazyRule((ua, host) => {
            // `input` = detailUrl; arrow function is fine here now
            const html = fetch(input, { headers: { "User-Agent": ua, "Referer": host + "/" } });
            const m = html.match(/https?:\/\/[^\s'"]+\.m3u8/);
            return m ? `video://${m[0]};{Referer@${host}/}` : `${input}#嗅探`;
        }, _ua, _host);
    }
};
```

```javascript
// ✅ Conservative style — ES5, works on ALL versions
var parse = {
    host: "https://example.com",
    UA: MOBILE_UA,

    主页: function() {
        var d = [];
        eval(getCryptoJS());  // getCryptoJS() available on latest; $.require fallback for old
        // ...
        return d;
    }
};
```

---

## 25. Case Study: 私房KTV (sfktv) Rule — Full Technical Summary

> Verified production rule covering AES-encrypted search API, AES-encrypted image display, domain self-healing, and tag-page two-level navigation.

### 25.1 Feature Checklist

| Feature | Status | Notes |
|---|---|---|
| Homepage categories | ✅ | 首页/往期/标签/热门/原创 |
| Homepage pagination | ✅ | `/page/2/` format |
| 往期 (archive) page | ✅ | Date-keyed listing |
| 标签 (tag) page | ✅ | Tag buttons → filtered list |
| Detail page | ✅ | Text + image + video |
| Image decryption | ✅ | AES-CBC via CryptoJS + FileUtil |
| Video playback | ✅ | m3u8 direct link |
| Search | ✅ | Encrypted POST API, paginated |
| Domain self-healing | ✅ | Probes publish pages, caches 1 hour |

### 25.2 Additional Divergences Confirmed (extends Section 24)

| # | Topic | Manual says | Actual (sfktv verified) |
|---|---|---|---|
| 6 | **`image()` callback + FileUtil** | Manual: FileUtil byte stream only | ✅ CryptoJS + FileUtil together: decrypt bytes in CryptoJS, pass result to `FileUtil.toInputStream()` |
| 7 | **`rule()` vs `lazyRule()` input** | Manual: `lazyRule` auto-injects `input`; `rule` behavior unclear | ✅ Confirmed: `rule` does NOT auto-inject `input` — must pass URL as explicit param |

### 25.3 Key Patterns

**AES-encrypted search POST**
```javascript
// Endpoint — hardcoded, never interpolate host prefix
var url = 'https://apiv1.sacmbcf.com/api.php/api/tcontent/searchContents';
// Pipeline: body → AES-encrypt → Base64 → append signature → POST → AES-decrypt response → parse JSON
```

**Image decryption**
```javascript
$(cleanUrl).image(function() {
    var raw = FileUtil.readBytes(input, { headers: { /*...*/ } });
    eval(getCryptoJS());
    var decrypted = CryptoJS.AES.decrypt(raw, key, { iv, mode: CBC, padding: Pkcs7 });
    return FileUtil.toInputStream(decrypted.words); // return stream, not string
});
```

**Tag page → sub-list (rule with explicit params)**
```javascript
// Law 2 applies: rule sandbox never receives `input` automatically
url: $(tagUrl).rule(function(url, host, ua) {
    var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });
    // parse and setResult(items)
}, tagUrl, host, ua)
```

**Domain self-healing**
```javascript
_getHost: function() {
    // 1. Read cache: getMyVar('sfktv_host') + getMyVar('sfktv_host_time'), TTL = 3600s
    // 2. Probe publish pages: sfktv15.com → sfktv25.com
    // 3. Extract domain list from publish page HTML
    // 4. Test each domain; cache and return first live one
}
```

### 25.4 Secrets & Endpoints Quick Reference

| Item | Value |
|---|---|
| Search API | `https://apiv1.sacmbcf.com/api.php/api/tcontent/searchContents` |
| Signature key | `5589d41f92a597d016b037ac37db243d` |
| Image AES key | `f5d965df75336270` |
| Image AES iv | `97b60394abc2fbe1` |
| Search AES key | `2acf7e91e9864673` |
| Search AES iv | `1c29882d3ddfcfd6` |
| Cache keys | `sfktv_host`, `sfktv_host_time`, `sfktv_publish` |

### 25.5 Maintenance Checklist (if rule breaks)

1. Check if `apiv1.sacmbcf.com` has moved
2. Check if signature key changed
3. Check if AES key/iv rotated
4. Check if publish page range (`sfktv15~25.com`) shifted

---

*Updated to V3.49 · 2026 · Added Section 25: 私房KTV case study*
