# 聚阅/海阔视界 实战规则通用模板速查(2026 Q2)

> **来源:** `~/downloads/ces/jiekou/` 25 个实战规则的浓缩提炼
> **对照:** 《聚阅海阔视界规则开发手册 V3_66》同目录
> **原则:** 实战与手册冲突以**实战为准**;手册 V3.66 经实测已极完善,本文档仅做**最新版本规则的骨架速取**

---

## 一、25 条实战规则归档表

| 文件(时间戳) | 站点类型 | 架构 | host | 模板 |
|---|---|---|---|---|
| 1748962049775 | 影视 | 静态 HTML | agedm.io | A |
| 1750302302324 | 有声/吃瓜 | 自研后端+缓存 | honoong | B |
| 1754050350794 | AV | 自研+AES | 51hlwz04 | C |
| 1754469835407 | 私房 | 自研+发布页 | sfktv16 | C |
| 1761226737871 | 资源/Magnet | sort 排序 | seedhub | A |
| 1763806209876 | 小说 | 动态自愈 | jishuge | C |
| 1771332321279 | 漫画 | 静态+AES | gwuswpd | A+AES |
| 1771338051190 | 小说 | WordPress 中文路径 | gwuswpd | A |
| 1772522394401 | 小说 | WebView 过盾 | bh5326 | C |
| **1772546254802** | **有声(WordPress 标杆)** | WordPress | cnnovel | **A 参考** |
| 1772956449770 | 影/小/图 混合 | 自研+函数库 | 320ios | B |
| 1772960477805 | AV/小说 矩阵 | 自研 | yazhouse8 | B |
| 1772964723237 | AV+Cookie | 自研 | 18av.mm-cg | B+加密 |
| 1772969337618 | AV 9 维矩阵 | 自研 | xojav | B 进阶 |
| **1773539975076** | **H 动漫(矩阵标杆)** | 自研多模式 | hanimeone | **B 参考** |
| 1773875546082 | 吃瓜混合 | 自研矩阵 | bad.news | B |
| 1774220019680 | 漫/小/视 | 静态分类 | jmw1 | A |
| 1774481116613 | 游戏资源 | WordPress | gamer520 | A |
| 1775008234477 | 图集 | Vue SPA+batchFetch | youwu | A+pics |
| 1775186752608 | 吃瓜 | cate:// 多树 | xchina | B |
| 1775966449768 | 漫/图/音 | _multiPageRule | acgxmh | A+pics |
| 1776316084529 | 吃瓜 | 自研+24h 缓存 | haijiaod | B |
| 1777686620127 | 论坛(需登录) | x5_webview | xsijishe | C+Cookie |
| 1778061007319 | 图/视 | 多模式动态 | emonl | A |
| **1778856386878** | **流媒体(域名自愈标杆)** | 自研 V35 | utv7 | **C 参考** |

**标杆推荐(直接复制改名):**
- 模板 A: `1772546254802.txt` (cnnovel.xyz 有声小说,简洁优雅 97 行)
- 模板 B: `1773539975076.txt` (hanimeone V5 旗舰矩阵 644 行)
- 模板 C: `1778856386878.txt` (utv7 V35 域名自愈+GCM 760 行)

---

## 二、模板 A:WordPress 简洁型(适用 80% 中型站点)

**特征:** 静态分类 + fyclass 路径 + `/page/N/` 翻页 + pdfa/pdfh 解析。
**最小骨架(从 1772546254802 提炼):**

```javascript
let parse = {
    作者: 'NULL',
    版本: '2026.Vx',
    host: 'https://example.com',
    UA: 'Mozilla/5.0 (Linux; Android 10) ... Mobile Safari/537.36',

    页码: { 主页: 1, 分类: 1, 搜索: 1 },

    静态分类: {
        type: '主页',
        url: 'fyclass',
        class_name: '分类A&分类B&分类C',
        class_url: '路径A/&/路径B/&/路径C/',
    },

    // ★ 资源解析 lazyRule 工厂(闭包模式,主页/分类/搜索共享)
    _resRule: function() {
        return $('').lazyRule(() => {
            let html = request(input, { headers: { 'User-Agent': MOBILE_UA, 'Referer': input } });
            let src = pdfh(html, 'audio&&src') || pdfh(html, 'source&&src');
            if (!src) {
                let m = html.match(/https?:[^"'\s]+\.mp3[^"'\s]*/);
                if (m) src = m[0];
            }
            if (src) {
                if (src.startsWith('//')) src = 'https:' + src;
                return src + ';{Referer@' + input + '}#isMusic=true#';
            }
            return 'webRule://' + input + '@' + $.toString(() => {
                let urls = _getUrls();
                return urls.find(u => u.includes('.mp3')) || '';
            });
        });
    },

    // ★ 列表抓取(主页/分类/搜索共享)
    _抓取列表: function(url) {
        let d = [];
        let resLazy = this._resRule();

        let realUrl = url;
        if (MY_PAGE > 1) {
            realUrl = url.replace(/\/+$/, '') + '/page/' + MY_PAGE + '/';
        }
        realUrl = encodeURI(realUrl);   // ★ 中文路径必须

        let html = fetch(realUrl, { headers: { 'User-Agent': this.UA } });
        if (!html || html.length < 100) return d;

        pdfa(html, 'body&&article').forEach(item => {
            try {
                let title = pdfh(item, 'h2&&Text') || pdfh(item, 'a&&title');
                let link  = pd(item,  'h2 a&&href') || pd(item, 'a&&href');
                let pic   = pd(item,  'img&&data-src') || pd(item, 'img&&src');
                let desc  = pdfh(item, '.entry-excerpt&&Text') || '';
                if (title && link) {
                    d.push({
                        title: title.trim(),
                        desc:  desc.trim(),
                        pic_url: pic ? pic + '@Referer=' + this.host + '/' : '',
                        url:   link + resLazy,
                        col_type: 'movie_2',
                    });
                }
            } catch (e) {}
        });
        return d;
    },

    主页:   function() { return this._抓取列表(this.host + (MY_URL === 'fyclass' ? '/' : MY_URL)); },
    分类:   function() { return this.主页(); },
    搜索:   function(name) {
        let url = this.host + '/';
        url += MY_PAGE > 1 ? 'page/' + MY_PAGE + '/?s=' + encodeURIComponent(name)
                           : '?s=' + encodeURIComponent(name);
        return this._抓取列表(url);
    },
};
```

**关键点(实战提炼,手册各章已述,此处汇总):**
- L17 (Law 17): 中文路径翻页**必须** `encodeURI()`,GBK 站点搜索用 `encodeStr`
- 闭包工厂 `_resRule` 返回的 lazyRule 字符串可附加到任意 url 后面,实现统一资源解析
- `request(input, ...)` 在 lazyRule 内自动注入 `input` 变量,这是聚阅约定
- pdfh 选择器用 `||` 串联多个降级方案,处理同站点多种主题
- pic_url 防盗链统一用 `@Referer=...` 后缀

---

## 三、模板 B:多模式矩阵筛选型(复杂筛选站点)

**特征:** 顶层模式切换 + 多维筛选行(单选/多选)+ getMyVar/putMyVar 状态机 + refreshPage 刷新。
**关键架构(从 1773539975076 V5 提炼,完整版 644 行):**

```javascript
var parse = {
    作者: 'NULL',
    版本: '2026.V5',
    host: 'https://example.com',
    页码: { 主页: 1 },

    静态分类: { type: '主页', url: 'fypage', class_name: '旗舰矩阵', class_url: '0' },

    主页: function() {
        var _res = [];
        var host = this.host;
        var _ua  = '...';
        var headers = { 'User-Agent': _ua, 'Referer': host + '/' };
        var H = 'app_';   // ★ 状态空间前缀,跨规则隔离

        // ── 三种渲染器(粘贴到每条规则即可,不需要全局函数)─────

        // 单选行(切换 Row0 时清空所有后级状态)
        var renderRow = function(rowIdx, titleStr, idStr, d, h) {
            var tArr = titleStr.split('&'), iArr = idStr.split('&');
            var curIdx = getMyVar(h + 'cindex' + rowIdx, '0');
            for (var i = 0; i < tArr.length; i++) {
                var isSel = (i == curIdx);
                d.push({
                    title: isSel ? '✦ ' + tArr[i] + ' ✦' : tArr[i],
                    url: $('#noLoading#').lazyRule(function(rIdx, idx, val, h) {
                        if (rIdx == 0) {
                            // ★ 模式切换:清所有后级状态
                            for (var k = 1; k <= 5; k++) {
                                putMyVar(h + 'cindex' + k, '0');
                                putMyVar(h + 'curl' + k, '');
                            }
                        }
                        putMyVar(h + 'cindex' + rIdx, idx + '');
                        putMyVar(h + 'curl'   + rIdx, val);
                        refreshPage(false);
                        return 'hiker://empty';
                    }, rowIdx, i, iArr[i], h),
                    col_type: 'scroll_button'
                });
            }
            d.push({ col_type: 'blank_block' });
        };

        // 单选行(独立 key,不参与 row 联动)
        var renderSingle = function(key, titleStr, valueStr, d, h) {
            var tArr = titleStr.split('&'), vArr = valueStr.split('&');
            var curIdx = getMyVar(h + key + '_idx', '0');
            for (var i = 0; i < tArr.length; i++) {
                var isSel = (i == curIdx);
                d.push({
                    title: isSel ? '✦ ' + tArr[i] + ' ✦' : tArr[i],
                    url: $('#noLoading#').lazyRule(function(k, idx, val, h) {
                        putMyVar(h + k + '_idx', idx + '');
                        putMyVar(h + k, val);
                        refreshPage(false);
                        return 'hiker://empty';
                    }, key, i, vArr[i], h),
                    col_type: 'scroll_button'
                });
            }
            d.push({ col_type: 'blank_block' });
        };

        // 多选行(Law 57:末尾必须加清空)
        var renderMulti = function(key, titleStr, valueStr, d, h) {
            var tArr = titleStr.split('&'), vArr = valueStr.split('&');
            var selStr = getMyVar(h + key, '');
            var selArr = selStr ? selStr.split(',').filter(s => s !== '') : [];
            for (var i = 0; i < tArr.length; i++) {
                var isSel = selArr.indexOf(vArr[i]) > -1;
                d.push({
                    title: isSel ? '✅ ' + tArr[i] : tArr[i],
                    url: $('#noLoading#').lazyRule(function(k, val, h) {
                        var cur = getMyVar(h + k, '');
                        var arr = cur ? cur.split(',').filter(s => s !== '') : [];
                        var idx = arr.indexOf(val);
                        if (idx > -1) arr.splice(idx, 1); else arr.push(val);
                        putMyVar(h + k, arr.join(','));
                        refreshPage(false); return 'hiker://empty';
                    }, key, vArr[i], h),
                    col_type: 'scroll_button'
                });
            }
            d.push({   // ★ 清空按钮
                title: '🗑️ 清空',
                url: $('#noLoading#').lazyRule(function(k, h) {
                    putMyVar(h + k, ''); refreshPage(false); return 'hiker://empty';
                }, key, h),
                col_type: 'scroll_button'
            });
            d.push({ col_type: 'blank_block' });
        };

        // ── Row0:模式切换(仅第一页渲染,避免翻页时重复绘制)
        if (MY_PAGE == 1) {
            renderRow(0, '🎬视频&📖漫画&🏷️标签', 'video&comic&tags', _res, H);
        }

        var mode = getMyVar(H + 'curl0', 'video');

        // ── 分支:视频模式
        if (mode == 'video') {
            if (MY_PAGE == 1) {
                renderRow(1, '排序:最新&热门', 'latest&hot', _res, H);
                // renderSingle / renderMulti 按需追加
            }
            var c1 = getMyVar(H + 'curl1', 'latest');
            var url = host + '/?sort=' + c1 + '&page=' + MY_PAGE;
            var html = fetch(url, { headers: headers });
            pdfa(html, 'body&&a[href*="/watch"]').forEach(it => {
                var link = pd(it, 'a&&href', host).split('#')[0];
                if (!link) return;
                _res.push({
                    title:   pdfh(it, 'div&&Text'),
                    pic_url: pd(it, 'img&&src') + '@Referer=' + host + '/',
                    url:     $(link).lazyRule((u, ua, h) => {
                        var html = request(u, { headers: { 'User-Agent': ua, 'Referer': h+'/' } });
                        var m = html.match(/<source\s+src="([^"]+)"[^>]+size="1080"/i)
                             || html.match(/<source\s+src="([^"]+)"/i);
                        if (!m) return u + '#嗅探';
                        var src = m[1].split(String.fromCharCode(92)).join('');
                        return 'video://' + src + ';{User-Agent@' + ua + '&&Referer@' + h + '/}';
                    }, link, _ua, host),
                    col_type: 'movie_2'
                });
            });
        }
        // else if (mode == 'comic') { ... 漫画走 pics:// }
        // else if (mode == 'tags')  { ... 多维矩阵 }

        return _res;
    },

    // 分类 = 主页(矩阵全在主页驱动)
    分类: function() { return this.主页(); },
};
```

**关键点(2026 Q2 实战凝练):**
1. **状态空间前缀:** 所有 `getMyVar/putMyVar` 用 `H = 'app_'` 之类前缀隔离,跨规则不冲突,跨实例同前缀**复用**——按需选择
2. **Row0 清场原则:** 切换主模式必须把所有 Row1+ 的 `cindex`/`curl` 重置回默认值,否则筛选状态会粘附
3. **`if (MY_PAGE == 1)` 只在首页渲染选择行:** 翻页时不再画一次,否则界面闪烁
4. **`refreshPage(false)`:** 局部刷新,不重新走 fetch
5. **lazyRule 显式传参:** `$.lazyRule(func, ...args)`,**绝不依赖闭包变量**——闭包变量在 lazyRule 上下文里访问不到(手册 49 章已澄清,本模板严守)
6. **`String.fromCharCode(92)` 替代 `'\\\\'`:** 避免转义层级混乱(实战 hanime 用此招处理 m3u8 反斜杠)

---

## 四、模板 C:域名自愈+加密通信型(高对抗站点)

**特征:** 发布页 + 多备用 host + Cookie 自动获取 + AES-GCM 解密 API + juItem TTL 缓存。
**核心架构(从 1778856386878 V35 提炼):**

```javascript
var parse = {
    版本: '2026.Vx',
    PUBLISH_PAGE: 'https://publish.example/',     // 公告新域名的发布页
    __hosts: [                                     // 兜底硬编码列表
        'https://api1.example:port',
        'https://api2.example:port',
    ],
    UA: '...',
    CK_TTL_MS:   25 * 60 * 1000,                  // Cookie 缓存 25 分钟
    HOST_TTL_MS:  6 * 3600 * 1000,                // 域名缓存 6 小时

    // ── 三级域名自愈 ───────────────────────────────

    _resolveHosts: function() {                   // 1) 从发布页抓最新域名
        try {
            var html = fetch(this.PUBLISH_PAGE, { headers: {'User-Agent': this.UA}, timeout: 8000 });
            if (!html) return [];
            var re = /<div[^>]*class="btnLink"[^>]*>\s*(https?:\/\/[^\s<]+?)\s*<\/div>/g;
            var hosts = [], m;
            while ((m = re.exec(html)) !== null) {
                var h = m[1].trim().replace(/\/+$/, '');
                if (hosts.indexOf(h) < 0) hosts.push(h);
            }
            return hosts;
        } catch (e) { return []; }
    },

    _pingHost: function(h) {                      // 2) 试测响应是否存活
        try {
            var r = fetch(h + '/api/index/notice', { headers: {'User-Agent': this.UA}, timeout: 5000 });
            return r && r.length > 10;
        } catch (e) { return false; }
    },

    _getHost: function() {                        // 3) 决策当前 host(带 TTL)
        var ts     = parseInt(juItem.get('app_host_ts', '0'));
        var cached = juItem.get('app_host', '');
        if (cached && (Date.now() - ts) < this.HOST_TTL_MS) return cached;
        if (cached && this._pingHost(cached)) {
            juItem.set('app_host_ts', String(Date.now()));
            return cached;
        }
        var pub = this._resolveHosts();
        var all = pub.concat(this.__hosts).filter((x, i, a) => a.indexOf(x) === i);
        for (var i = 0; i < all.length; i++) {
            if (this._pingHost(all[i])) {
                juItem.set('app_host', all[i]);
                juItem.set('app_host_ts', String(Date.now()));
                juItem.set('app_ck', '');         // ★ 换 host 同时废 Cookie
                juItem.set('app_ck_ts', '0');
                return all[i];
            }
        }
        return cached || this.__hosts[0];
    },

    // ── Cookie 自动获取(WebView 过盾)──────────

    _ensureSession: function() {
        var ts = parseInt(juItem.get('app_ck_ts', '0'));
        var ck = juItem.get('app_ck', '');
        if (ck && (Date.now() - ts) < this.CK_TTL_MS) return ck;

        var host = juItem.get('app_host', this.__hosts[0]);
        try {
            fetchCodeByWebView(host + '/', {
                timeout: 15000, ua: this.UA,
                blockRules: ['.png', '.jpg', '.jpeg', '.gif', '.webp', 'analytics'],
                checkJs: "document.cookie.indexOf('js_challenge_passed')>=0"
            });
        } catch (e) { log('WebView 过盾: ' + e.message); }

        // 优先 CookieManager(过盾后最稳)
        try {
            var cm = android.webkit.CookieManager.getInstance();
            var s  = cm.getCookie(host);
            if (s && String(s).length > 0) ck = String(s);
        } catch (e) {}

        // 兜底 fetchCookie(必须 raw 校验,空响应 JSON.parse 会炸)
        if (!ck) {
            try {
                var raw = fetchCookie(host + '/');
                if (raw && String(raw).trim().length > 2 && String(raw).indexOf('[') === 0) {
                    ck = JSON.parse(raw).map(c => String(c).split(';')[0].trim()).filter(x => x).join('; ');
                }
            } catch (e) {}
        }

        if (ck) {
            juItem.set('app_ck', ck);
            juItem.set('app_ck_ts', String(Date.now()));
        }
        return ck;
    },

    _defaultHeaders: function(extra) {
        var host = juItem.get('app_host', this.__hosts[0]);
        var h = {
            'User-Agent': this.UA,
            'Accept':     'application/json, text/plain, */*',
            'Referer':    host + '/',
            'Origin':     host,
            'Cookie':     this._ensureSession() || '#noCookie#'
        };
        if (extra) for (var k in extra) h[k] = extra[k];
        return h;
    },

    // ── AES-GCM 解密(java.crypto 反射,处理 PBKDF2 + 12B IV + 16B tag 信封)

    _toJavaCharArray: function(str) {
        var arr = java.lang.reflect.Array.newInstance(java.lang.Character.TYPE, str.length);
        for (var i = 0; i < str.length; i++) arr[i] = str.charAt(i);
        return arr;
    },

    _pbkdf2: function(password, salt, iter) {
        var SKF = javax.crypto.SecretKeyFactory.getInstance('PBKDF2WithHmacSHA256');
        var spec = new javax.crypto.spec.PBEKeySpec(this._toJavaCharArray(password), salt, iter || 1000, 256);
        return SKF.generateSecret(spec).getEncoded();
    },

    _decryptGCM: function(b64, keyB64) {
        try {
            var B = android.util.Base64;
            var key = B.decode(keyB64, B.DEFAULT);
            var dec = B.decode(b64, B.DEFAULT);
            if (dec.length < 12) return null;
            var iv = java.util.Arrays.copyOfRange(dec, 0, 12);
            var ct = java.util.Arrays.copyOfRange(dec, 12, dec.length);
            var cipher = javax.crypto.Cipher.getInstance('AES/GCM/NoPadding');
            cipher.init(javax.crypto.Cipher.DECRYPT_MODE,
                new javax.crypto.spec.SecretKeySpec(key, 'AES'),
                new javax.crypto.spec.GCMParameterSpec(128, iv));
            return new java.lang.String(cipher.doFinal(ct), 'UTF-8') + '';
        } catch (e) { return null; }
    },

    // 凯撒解码 + Base64 + GCM 解密信封(站点常见组合)
    _decryptEnvelope: function(encData, password) {
        try {
            var caesar = '';
            for (var i = 0; i < encData.length; i++) caesar += String.fromCharCode(encData.charCodeAt(i) - 3);
            var bytes = android.util.Base64.decode(
                caesar.replace(/[\r\n\s]+/g,'').replace(/-/g,'+').replace(/_/g,'/'),
                android.util.Base64.DEFAULT);
            if (!bytes || bytes.length < 28) return null;
            var salt = java.util.Arrays.copyOfRange(bytes, 0, 16);
            var iv   = java.util.Arrays.copyOfRange(bytes, 16, 28);
            var ct   = java.util.Arrays.copyOfRange(bytes, 28, bytes.length);
            var dk   = this._pbkdf2(password || 'test', salt, 1000);
            var combined = java.util.Arrays.copyOf(iv, iv.length + ct.length);
            java.lang.System.arraycopy(ct, 0, combined, iv.length, ct.length);
            var b64 = android.util.Base64.encodeToString(combined, android.util.Base64.NO_WRAP);
            var dkb = android.util.Base64.encodeToString(dk,       android.util.Base64.NO_WRAP);
            var plain = this._decryptGCM(b64, dkb);
            try { return JSON.parse(plain); } catch (e) { return plain; }
        } catch (e) { return null; }
    },

    // 主页/分类等用 _getHost() + _defaultHeaders() 即可
};
```

**关键点(实战凝练):**
1. **三层缓存(Cookie / Host / 数据)各有独立 TTL**,Cookie 短(25min),Host 中(6h),数据可加 24h
2. **换 host 时连带废 Cookie**(`juItem.set('app_ck', '')`),否则带旧 ck 访问新 host 会 401
3. **CookieManager 优先于 fetchCookie**:WebView 过盾后 Cookie 写入 Android CookieManager,直读最稳;Android 12+ 严格模式 fallback fetchCookie 时**必须校验 raw 长度和首字符**
4. **PBEKeySpec 必须传 `char[]`**(用 `java.lang.reflect.Array.newInstance(java.lang.Character.TYPE, ...)` 构造),JS 字符串会 ClassCastException
5. **GCM tag 长度 128 bit** 是标准信封,改成 96/120 会失败
6. **凯撒+Base64+PBKDF2+GCM** 组合是 2026 主流站点 API 加密标准,值得抄一份留底

---

## 五、25 个实战规则 vs 手册章节对照表

| 实战技术点 | 手册章节 | 备注 |
|---|---|---|
| 中文路径 encodeURI 翻页 | 第 3 章 L667 / 第 15 章 Law 17 (L3047) / 第 17 章 L4035 | 已完整 |
| host 前缀 getMyVar 隔离 | 第 6.3 章 L1050 / 第 11 章 L498 | 已完整 |
| renderRow 唯一正确版本 | **6.3 节** L1050 | 已完整 |
| renderMulti 多选 + 清空 | **18.7 节** L4673 / Law 57 | 已完整 |
| fetchCodeByWebView checkJs | 第 3 章 L426 / Law 72 (L9286) | 已完整 |
| juItem 持久化(CF Cookie/host) | **3.4 节** L475 + L494 警告 | 已完整 |
| batchFetch ≤ 16 / 自动分批 | 第 3 章 L421 / 第 30 章 L8249 | 已完整 |
| AES-GCM + PBKDF2 反射 | **第 10 章 §10.3 补充** L2491-2517 | 已完整 |
| CookieManager.getCookie | L2447 三种 Cookie 获取方式 | 已完整 |
| fetchCookie 空响应校验 | L2452 Table | 已完整 |
| 凯撒+atob 解密 | 第 ?? 章 L7463 | 已完整 |
| cate:// 虚拟协议 | **第 42 章** L10554 | 已完整 |
| pics:// 多页聚合 | 第 24/25/40 章 | 已完整 |
| webRule:// 兜底嗅探 | 第 8 章 / 第 9 章 | 已完整 |
| WordPress 完整骨架 | **第 46 章** L11271 + 46.8 完整骨架 L11706 | 已完整 |
| 自研矩阵筛选 | **第 28 章** L7745 + 第 18 章 | 已完整 |
| 域名自愈四步 | **第 13 章** L2880 + 第 45.8 节 L11142 | 已完整 |
| 工具对象封装 | **第 41 章** L10397 / 第 48 章 L12631 | 已完整 |
| 二级页 noShow + extenditems | **第 43 章** L10697 / 49.补充 3 L12960 | 已完整 |
| lazyRule 显式传参/$.toString | 49 章 澄清 2 L13218 + 补充 2 L12923 | 已完整 |

**结论:** 手册 V3.66(尤其第 49 章勘误补充 + 第 46/47/48 章实战 + 第 49 章澄清 + 第 73/75/76 章扩展)**已完整覆盖**这 25 个实战规则中出现的所有技术点。无需再大改手册。

---

## 六、本轮分析新增/强调铁律(候选 Law 163-167)

> 这些点手册零散提到,但未单独列为铁律。建议补入手册"铁律速查全表":

- **Law 163:** 模板 B 矩阵规则的状态前缀必须用 `H = 'app_'` 之类显式变量,**不要直接拼 host**——host 含 `://` 和 `.` 会污染 key
- **Law 164:** 模板 C 域名自愈中,**换 host 必须同步清 Cookie**(host_ts 与 ck_ts 联动失效),否则跨域名带旧 ck 必 401
- **Law 165:** `if (MY_PAGE == 1)` 包裹筛选行渲染,**翻页不再画**(避免界面闪烁、避免状态被默认值覆盖)
- **Law 166:** lazyRule 内反斜杠用 `String.fromCharCode(92)` 替代字面 `'\\\\'`,**避免多层转义**(m3u8/JSON 字符串中常用)
- **Law 167:** PBEKeySpec 第一参数必须 `char[]`(通过 `java.lang.reflect.Array.newInstance(java.lang.Character.TYPE, n)` 构造),JS 字符串报 ClassCastException

---

## 七、使用建议

1. **新站点适配:** 先判定类型 → 选标杆文件 → 复制到 `开发专用/` → 改 host/UA/selector → 测
   - WordPress / Drupal / 单页博客 → **1772546254802.txt**
   - 自研后端 + 复杂筛选 → **1773539975076.txt**
   - 加密通信 + 域名漂移 → **1778856386878.txt**

2. **遇到加密/Cookie 问题:** 直接搬 1778856386878 的 `_decryptEnvelope` + `_ensureSession`,改 PUBLISH_PAGE 和正则即可

3. **多模式聚合站:** 抄 1773539975076 的 `renderRow/renderSingle/renderMulti` 三件套,按需扩展 mode 分支

4. **手册检索路线:** 遇到具体 API 不会用 → 查手册第 26/45/48 章工具函数完全参考 → 查第 49 章勘误补充
