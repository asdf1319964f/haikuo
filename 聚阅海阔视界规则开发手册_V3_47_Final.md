**██████████████████████**

◆ HIKER VIEW & JU YUE ◆

**📱 觚阅 / 海阔视界**

**规则开发完全手册**

─────────────◆─────────────

V3.47 · 2026 · 50 章 · 新版海阔 ES6 安全子集 / 聚阅旧版 ES5

*基于海阔视界底层 \| 已通过 Anime7 · CZBooks · hanime1.me 实战验证*

> **第零章：两款 App 的关系与底层架构**

聚阅 是基于 海阔视界（HikerView）底层引擎
封装的二次开发平台。两者共用同一套 Rhino JS
引擎和相同的内置函数库（fetch/request/pd/pdfa/pdfh
等）。新版海阔视界引擎（2026+）支持 ES6
安全子集（const/let/箭头函数/模板字符串）；聚阅及旧版引擎仍须使用
ES5（全量 var）。开发时统一以海阔视界 API
规范为底层基准，再叠加聚阅特有的变量注入规则和协议扩展。

  ------------------------------------------------------------------------------------------------------------------------------
  **维度**         **海阔视界（底层）**                                           **聚阅（上层）**
  ---------------- -------------------------------------------------------------- ----------------------------------------------
  JS 引擎          自研 Rhino-ES5 引擎                                            继承，完全相同

  内置函数         fetch/request/pd/pdfh/pdfa/log                                 继承，完全相同

  分页变量         MY_PAGE / MY_URL                                               type:&quot;主页&quot;+url:&quot;fypage&quot;
                                                                                  才能注入

  视频协议         video:// + ;{Header@Val}                                       相同

  图片/漫画        pics:// 协议                                                   解析函数必须 return，禁用 setResult

  小说/阅读        无专属                                                         #isJiexi=1#readTheme##autoPage# 后缀

  语法限制         新版海阔：ES6                                                  完全相同，全量用 var
                   安全子集（const/let/箭头函数可用）；聚阅/旧版引擎：ES5（全量   
                   var）。见第三十一章。                                          
  ------------------------------------------------------------------------------------------------------------------------------

> **第一章：parse 对象标准骨架**

所有规则必须封装在 parse 对象中，严禁在全局作用域散放变量。

> var parse = {
>
> 作者: &quot;dev&quot;,
>
> 版本: &quot;20260101.V1&quot;,
>
> host: &quot;https://example.com&quot;,
>
> UA: &quot;Mozilla/5.0 (Windows NT 10.0; Win64; x64)
> AppleWebKit/537.36&quot;,
>
> 页码: { &quot;主页&quot;: 1, &quot;分类&quot;: 1, &quot;搜索&quot;: 1
> },
>
> 静态分类: { type:&quot;主页&quot;, url:&quot;fypage&quot;,
> class_name:&quot;\...&quot;, class_url:&quot;\...&quot; },
>
> 主页: function() { /\* return d; \*/ },
>
> 二级: function(url){ /\* return {\...}; \*/ },
>
> 解析: function(url){ /\* return &quot;video://\...&quot;; \*/},
>
> 搜索: function(name){ /\* return d; \*/ },
>
> // ── 工具函数 ────────────────────────────────────────
>
> // 双保险 Base64 解码
>
> \_base64Decode: function(str) {
>
> if (!str) return &quot;&quot;;
>
> str = str.replace(/-/g,&quot;+&quot;).replace(/\_/g,&quot;/&quot;);
>
> while (str.length % 4) str += &quot;=&quot;;
>
> try {
>
> var b = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
>
> return new java.lang.String(b, &quot;UTF-8&quot;);
>
> } catch(e) {
>
> try { return decodeURIComponent(escape(atob(str))); }
>
> catch(e2) { log(&quot;Base64 解码失败&quot;); return &quot;&quot;; }
>
> }
>
> },
>
> // 自动脱壳（最多 3 层，防死循环）
>
> \_autoDeshell: function(html) {
>
> if (!html) return html;
>
> for (var i = 0; i \< 3; i++) {
>
> if (html.indexOf(&quot;decodeURIComponent(\\&quot;&quot;) \> -1) {
>
> var m =
> html.match(/decodeURIComponent\\(\\&quot;(\[\^\\&quot;\]+)\\&quot;\\)/);
>
> if (m) html = decodeURIComponent(m\[1\]);
>
> } else break;
>
> }
>
> return html;
>
> }
>
> // ⚠️ 注意：parse 骨架中不应定义 \_safeRequest 等未实现的占位函数
>
> // 如需带重试的请求，在各自函数内用 try/catch + log 自行实现
>
> };
>
> **第二章：静态分类------三种写法与五维占位符**

**2.1 三种写法对比**

  -------------------------------------------------------------------------------------------------------------
  **写法**        **type**           **url**                                      **适用场景**
  --------------- ------------------ -------------------------------------------- -----------------------------
  纯矩阵自管理    &quot;主页&quot;   &quot;fypage&quot;                           主页内自渲染分类行，MY_PAGE
                                                                                  可用

  带选单+翻页     &quot;主页&quot;   &quot;hiker://empty##fyclass##fypage&quot;   顶部有选单，MY_URL 带分类路径

  无选单净模式    0                  &quot;fypage&quot;                           分类全在主页内渲染（⚠ MY_PAGE
                                                                                  不注入，不推荐）
  -------------------------------------------------------------------------------------------------------------

> ⚠️ **警告:** type:0 时 MY_PAGE 不会被注入，连 typeof MY_PAGE !==
> &quot;undefined&quot; 也救不了------聚阅在词法分析阶段就报错。统一使用
> type:&quot;主页&quot; + url:&quot;fypage&quot;。

**2.2 五维多级筛选（完整写法）**

> &quot;静态分类&quot;: {
>
> &quot;type&quot;: &quot;主页&quot;,
>
> &quot;url&quot;:
> &quot;https://example.com/list/fyclass-fyarea-fysort\-\--fyyear\-\--fypage.html&quot;,
>
> &quot;class_name&quot;: &quot;电影&电视剧&动漫&quot;,
> &quot;class_url&quot;: &quot;1&2&3&quot;,
>
> &quot;area_name&quot;: &quot;全部&大陆&日本&美国&quot;,
> &quot;area_url&quot;: &quot;&大陆&日本&美国&quot;,
>
> &quot;year_name&quot;: &quot;全部&2026&2025&2024&quot;,
> &quot;year_url&quot;: &quot;&2026&2025&2024&quot;,
>
> &quot;sort_name&quot;: &quot;时间&热度&评分&quot;,
> &quot;sort_url&quot;: &quot;time&hits&score&quot;
>
> }

**2.3 五大占位符对照表**

  ------------------------------------------------------------------------------
  **占位符**   **对应 name 字段**    **对应 url       **说明**
                                     字段**           
  ------------ --------------------- ---------------- --------------------------
  fyclass      class_name            class_url        主分类

  fyarea       area_name             area_url         地区筛选

  fyyear       year_name             year_url         年份筛选

  fysort       sort_name             sort_url         排序方式

  fypage       ---（引擎自动管理）   ---              页码，自动递增，无需定义
  ------------------------------------------------------------------------------

> 📌 **铁律:** name 与 url 数量必须严格一一对应（&
> 分隔），&quot;全部&quot;选项对应的 url 留空字符串。fypage
> 由引擎自动管理，只需放在模板中即可。

**2.4 fypage 高级语法------偏移量与步长**

当站点分页参数不是从 1 递增而是从 0 开始或以固定步长递增（如
0,20,40\...）时，使用偏移量语法：

> // 语法：fypage@偏移量@\*步长@
>
> // 偏移量：fypage 的初始值基础上加减的数字
>
> // 步长：乘以页码的倍数（默认步长=1）
>
> // 示例1：从 0 开始，步长 20（即 0,20,40\...）
>
> &quot;url&quot;:
> &quot;https://api.example.com/list?start=fypage@-1@\*20@&size=20&quot;
>
> // 第1页 → start=0，第2页 → start=20，第3页 → start=40
>
> // 示例2：从 0 开始，步长 1（即 0,1,2\...）
>
> &quot;url&quot;:
> &quot;https://api.example.com/list?page=fypage@-1@&quot;
>
> // 注意：fypage 不能放在 URL 最末尾，否则可能翻页异常
>
> // 解决方法：末尾加无效参数，如 ?\_t=0
>
> &quot;url&quot;: &quot;https://example.com/fypage.html?\_t=0&quot;
>
> // 示例3：第一页使用不同地址（部分站点第一页无 /1/）
>
> &quot;url&quot;:
> &quot;https://example.com/list/fypage/\[firstPage=https://example.com/list/\]&quot;
>
> 💡 **说明:** fyAll
> 是一个特殊占位符：当分类只有一个维度但选项太多时，class/area/year
> 的替换词都会替换 fyAll，有 fyAll 就不能再有其他替换词。

**2.5 URL 模板写法示例**

**示例1：标准 MACCMS 路径型**

> &quot;url&quot;:
> &quot;https://example.com/vod-show/fyclass-fyarea-fysort\-\--fyyear\-\--fypage\-\--.html&quot;

**示例2：查询参数型（API 站点）**

> &quot;url&quot;:
> &quot;https://api.example.com?type=fyclass&area=fyarea&year=fyyear&sort=fysort&page=fypage&\_t=0&quot;

**示例3：纯路径拼接型**

> &quot;url&quot;:
> &quot;https://example.com/list/fyclass/fyarea/fyyear/fypage/&quot;
>
> **第三章：海阔视界底层内置函数参考**

**3.1 HTML 解析三件套**

  -----------------------------------------------------------------------------------------------
  **函数**                    **缩写**   **返回值**           **说明**
  --------------------------- ---------- -------------------- -----------------------------------
  parseDom(html, rule,        pd         String（智能补全）   自动补全域名/http，返回第一个匹配
  baseUrl)                                                    

  parseDomForHtml(html, rule) pdfh       String（原始）       不处理域名，原始返回

  parseDomForArray(html,      pdfa       Array\<String\>      返回匹配列表，用于循环
  rule)                                                       
  -----------------------------------------------------------------------------------------------

> var items = pdfa(html, &quot;body&&.list&&li&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var \_title = pdfh(items\[i\], &quot;a&&Text&quot;);
>
> var \_href = pd(items\[i\], &quot;a&&href&quot;, host); //
> 第三参数=baseUrl，自动补全域名
>
> var \_pic = pd(items\[i\], &quot;img&&data-original&quot;);
>
> }

**3.2 选择器语法速查**

  ----------------------------------------------------------------------------------------------------------------------------------------
  **语法**                  **说明**                     **示例**
  ------------------------- ---------------------------- ---------------------------------------------------------------------------------
  &&                        取子元素                     &quot;body&&.list&&li&quot;

  \--                       排除元素                     &quot;body\--script&&a&&href&quot;

  tag,N                     取第 N 个（0起）             &quot;body&&a,2&quot; 取第3个 a

  tag,-1                    倒数取                       &quot;body&&li,-1&quot; 取最后一个 li

  Text                      获取文本内容                 &quot;h2&&Text&quot;

  Html                      获取含标签的 HTML            &quot;div&&Html&quot;

  attr                      获取属性                     &quot;img&&src&quot; / &quot;a&&href&quot;

  \[attr\*=&quot;&quot;\]   模糊类名匹配                 \[class\*=&quot;item&quot;\]（比固定类名稳健）

  \|\|                      或语法（先找前者再找后者）   &quot;#app\|\|#app2&&Text&quot;

  .js:expr                  对结果进行 JS 处理           &quot;a&&href.js:input.replace(\\&quot;\\\\\\\\\\&quot;,\\&quot;\\&quot;)&quot;

  js:代码                   完全用 JS 写规则             &quot;js:parseDom(html,\'\...\') + \'/page/1\'&quot;
  ----------------------------------------------------------------------------------------------------------------------------------------

**3.3 网络请求函数**

  ----------------------------------------------------------------------------------------------------
  **函数**                  **说明**                                    **常用参数**
  ------------------------- ------------------------------------------- ------------------------------
  fetch(url, opts)          标准 GET/POST 请求                          {headers, body, method,
                                                                        timeout, withHeaders,
                                                                        withStatusCode, redirect}

  fetchPC(url, opts)        PC UA 请求                                  同 fetch

  request(url, opts)        fetch 的别名                                同 fetch

  post(url, {body})         POST，自动表单编码                          {body: {key: val}}

  batchFetch(arr)           并发请求（最多16个/批，超出自动分批串行）   \[{url, options}, \...\]，缩写
                                                                        bf

  fetchCookie(url, opts)    只获取 Set-Cookie                           返回 Cookie 数组的 JSON 字符串

  fetchCodeByWebView(url,   用 WebView 加载获取源码                     {headers, blockRules, timeout,
  opts)                                                                 checkJs}
  ----------------------------------------------------------------------------------------------------

> // 带 Header 请求
>
> var html = fetch(url, { headers: { &quot;User-Agent&quot;: parse.UA,
> &quot;Referer&quot;: parse.host+&quot;/&quot; } });
>
> // 获取返回 Header（如 Set-Cookie）
>
> var res = JSON.parse(fetch(url, { withHeaders: true }));
>
> var ck = res.headers\[&quot;Set-Cookie&quot;\]\[0\];
>
> // GBK 站点
>
> var html = fetch(url, { headers: { &quot;content-type&quot;:
> &quot;text/html; charset=GBK&quot; } });
>
> // 禁止自动 Cookie（避免串 Cookie）
>
> var html = fetch(url, { headers: { Cookie: &quot;#noCookie#&quot; }
> });
>
> ⚠️ **警告:** batchFetch 并发上限为 16。传入超过 16 个 URL
> 时会自动分批串行执行，并非全部同时发出。

**3.4 变量存取------setItem vs putMyVar 的关键区别**

  -------------------------------------------------------------------------------------------------------------
  **函数**                   **作用域**       **生命周期**                       **典型用途**
  -------------------------- ---------------- ---------------------------------- ------------------------------
  setItem(k,v) /             当前规则         持久化（重启不丢，规则删除才丢）   CF Cookie、Token、动态域名
  getItem(k,def)                                                                 

  putMyVar(k,v) /            当前规则         会话级（App 重启后丢失）           筛选状态、页码、临时选中索引
  getMyVar(k,def)                                                                

  putVar(k,v) /              全局（跨规则）   会话级（App 重启后丢失）           规则间传递一次性数据
  getVar(k,def)                                                                  

  storage0.putMyVar(k,obj)   当前规则         会话级                             存储 JSON 对象

  storage0.setItem(k,obj)    当前规则         持久化                             持久存储 JSON 对象
  -------------------------------------------------------------------------------------------------------------

> ⚠️ **警告:** CF Cookie 必须用 setItem 持久化，putMyVar 在 App
> 重启后会丢失，导致每次都需重新验证。
>
> 📌 **命名空间:** 用 host
> 字符串作前缀（getMyVar(host+&quot;cindex0&quot;,&quot;0&quot;)），但部分版本不允许
> key 含 &quot;://&quot;，改用纯字母前缀如
> &quot;sitename_cindex0&quot;。

**3.5 其他常用内置**

  ----------------------------------------------------------------------------------------
  **函数/变量**                             **说明**
  ----------------------------------------- ----------------------------------------------
  log(msg)                                  打印调试日志

  MY_PAGE                                   当前页码（需
                                            type:&quot;主页&quot;+url:&quot;fypage&quot;
                                            才注入）

  MY_URL                                    当前请求 URL（含静态分类替换后的完整 URL）

  MY_HOME                                   从 MY_URL 计算出的根域名，如 https://a.com

  MY_RULE.title / MY_RULE.find_rule         当前规则信息对象

  MOBILE_UA / PC_UA                         内置移动端/PC UA 变量

  base64Encode(s) / base64Decode(s)         Base64 编解码

  aesDecode(key,data) / aesEncode(key,data) AES 加解密

  buildUrl(url, params)                     GET 参数拼接

  refreshPage(false)                        刷新页面，false=不滚动到顶部

  back(true/false)                          关闭当前页，true=同时刷新上一页（仅限二级）

  toast(msg)                                弹出短提示

  confirm({title,content,confirm,cancel})   弹出确认框

  setPageTitle(title)                       动态修改当前页标题

  updateItem(id, obj)                       动态更新某个列表项（无需整页刷新）

  addItemAfter(id, obj)                     在指定 id 的项后插入新项

  deleteItem(id)                            删除指定 id 的列表项

  setPreResult(d) + setResult(d)            先返回顶部元素再追加列表（海阔用）
  ----------------------------------------------------------------------------------------

> **第四章：URL 标签速查**

在页面 URL 或按钮 URL
中追加以下标签可改变行为。请求时引擎会自动剥除这些标签，不影响实际请求地址。

  -------------------------------------------------------------------------------------------
  **标签**              **说明**                               **常用场景**
  --------------------- -------------------------------------- ------------------------------
  #noLoading#           不显示 loading 弹窗                    筛选按钮点击（避免闪屏）

  #noHistory#           不记录足迹                             纯导航按钮

  #noRecordHistory#     不记录历史记录                         临时跳转页

  #readTheme#           阅读模式（护眼背景/字体/进度记忆）     小说正文页

  #autoPage#            自动翻页（滚动到底部自动加载下一章）   小说章节页的每一项 URL

  #autoCache#           自动缓存页面（秒开）                   电子书章节目录

  #cacheOnly#           有缓存则只用缓存，不发网络请求         离线阅读

  #memoryPage#          自动记忆翻页页数，下次直接跳回         小说目录

  #noRefresh#           禁止下拉刷新                           不需要刷新的固定页

  #isJiexi=1            触发聚阅内置阅读/播放器                漫画/小说章节 URL 必须有此标签

  #isVideo=true#        强制识别为视频资源                     链接无视频扩展名但确实是视频

  #ignoreVideo=true#    强制识别为非视频                       链接含 mp4 字样但实际是页面

  #isMusic=true#        强制识别为音频资源                     音频链接

  #ignoreImg=true#      强制识别为非图片                       链接含 jpg 字样但不是图片

  #pre# / #noPre#       强制预加载 / 禁止预加载                时效性 URL 用 #noPre#

  #concat#              连接多段视频（用此标签拼接多地址）     分段视频合并播放

  #immersiveTheme#      沉浸式页面（标题栏不占位）             二级/子页面

  #fullTheme#           全屏页面                               游戏/全屏展示

  #fastPlayMode#        极速播放（多线程边下边播）             超大文件播放

  #background#          后台播放，通知栏显示前台通知           音频页面
  -------------------------------------------------------------------------------------------

> 📌 **使用示例:** // 聚阅小说章节 URL 标准写法 chapterUrl =
> pd(it,&quot;a&&href&quot;) +
> &quot;#isJiexi=1#readTheme##autoPage#&quot; // 分段视频 url =
> &quot;video://seg1.m3u8#concat#seg2.m3u8&quot;
>
> **第五章：col_type 显示样式速查**

  -----------------------------------------------------------------------------
  **col_type**                **说明**
  --------------------------- -------------------------------------------------
  movie_3 / movie_3_marquee   一行3列，圆角图片+标题（默认样式）

  movie_2                     一行2列，圆角图片+标题

  movie_1                     一行1列，标题描述较多时使用

  movie_1_left_pic            一行1列，图片在左

  movie_1_vertical_pic        竖向图片在左，适合书籍/漫画封面

  movie_1_vertical_pic_blur   同上，背景高斯模糊

  text_1 \~ text_5            文本1\~5列，支持红色&quot;&quot;和橙色\'\'混排

  long_text                   不限长度长文本，extra:{textSize:18}

  rich_text                   富文本，支持 HTML
                              标签，extra:{textSize,lineSpacing}

  pic_1_full                  全宽图片，高度按比例自适应

  pic_3 / pic_2               纯图片布局，3列/2列

  scroll_button               横向滚动按钮（支持 HTML），多个连续 push 自动聚合

  flex_button                 流式自适应按钮（支持 HTML），多个连续 push
                              自动聚合

  blank_block                 空白块（高度1dp）

  line                        分割线

  input                       单行输入框，extra:{type,defaultValue,onChange}

  x5_webview_single           X5 浏览器组件，desc 控制高度/模式

  avatar                      头像样式（图片+标题+右侧 desc）

  card_pic_2 / card_pic_1     方形卡片，可配合 card_pic_2_2 组合

  icon_4 / icon_round_4       桌面图标样式，4列

  video                       视频组件（与 x5_webview_single 不能共存）
  -----------------------------------------------------------------------------

> ⚠️ **警告:** x5_webview_single 一个页面只能有一行使用。⚠️
> scroll_button 不支持 HTML 渲染（fontcolor/bold
> 会直接显示原始标签字符串）；flex_button 支持 HTML 标题。scroll_button
> 选中状态只能靠 backgroundColor + 纯文本符号（▶ title 或 ●
> title）标记，不能用 fontcolor().bold()。
>
> **第六章：主页函数------矩阵静态分类完整方案**

矩阵静态分类是将所有分类数据硬编码在规则内部，通过多行 scroll_button
实现多维筛选的方案。优点是零网络请求、点击即开、支持多行联动。以下内容基于
XOJAV V34 实战代码提炼。

**6.1 静态分类声明------矩阵模式固定写法**

矩阵模式不依赖引擎的静态分类替换，仅用它声明入口。url 固定写
&quot;0&quot;，class_name/class_url
填任意占位值，主页函数自己全权管理所有分类渲染和 URL 构建。

> // ✅ 矩阵模式固定写法：url 写 &quot;0&quot;，引擎不替换任何占位符
>
> // 主页函数内通过 getMyVar 读取 cindex/curl 自行构建请求 URL
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;,
>
> &quot;url&quot;: &quot;0&quot;,
>
> &quot;class_name&quot;: &quot;静态矩阵旗舰版&quot;,
>
> &quot;class_url&quot;: &quot;0&quot;
>
> },
>
> 页码: { &quot;主页&quot;: 1 },
>
> 📌 **与 fypage 方案的区别:** fypage
> 方案（第二章）依赖引擎替换占位符，适合单维/多维但分类较少、URL
> 规律固定的站点。矩阵方案适合分类极多（如 9 行×数十列）、URL
> 需要按行单选拼接的复杂站点。两种方案 MY_PAGE 都可用。

**6.2 分类数据字典------物理硬编码**

将所有分类以数组形式硬编码在主页函数内。每一项对应一行筛选条，t
是显示标题，i 是对应的 URL 路径值，用 &
分隔，数量严格一一对应。第一项固定为&quot;全部&quot;，i 值为
&quot;0&quot;（表示不选）。

> // 每个对象 = 一行筛选条
>
> // t：显示标题（& 分隔）
>
> // i：对应 URL 路径值（& 分隔，与 t 数量严格一一对应）
>
> // 第一项固定为全部/重置项，i 值用 &quot;0&quot; 表示不选中
>
> var dataClass = \[
>
> // 第 1 行（cindex1）
>
> { t: &quot;按主題&中文字幕&無碼解放&台灣AV&日本AV&quot;,
>
> i:
> &quot;0&/categories/chinese-subtitle&/categories/uncensored&/categories/taiwan-av&/categories/jav&quot;
> },
>
> // 第 2 行（cindex2）
>
> { t: &quot;按衣着&婚紗&水着&吊帶襪&校服&quot;,
>
> i:
> &quot;0&/tags/wedding-dress&/tags/swimsuit&/tags/stockings&/tags/school-uniform&quot;
> },
>
> // 第 3 行（cindex3）\... 以此类推，最多支持到 cindex9
>
> \];
>
> ⚠️ **警告:** t 和 i 中 &
> 分隔的项数必须严格相等，数量不匹配会导致越界取到 undefined，点击后
> curl 写入空值。

**6.3 renderRow------多行筛选渲染函数（唯一正确版本）**

renderRow
在主页函数内部定义为局部函数，参数：rowIdx（行号）、titleStr（标题串）、idStr（值串）、d（结果数组）、h（命名空间前缀，传
host 字符串）。

> var renderRow = function(rowIdx, titleStr, idStr, d, h) {
>
> var tArr = titleStr.split(&quot;&&quot;);
>
> var iArr = idStr.split(&quot;&&quot;);
>
> var curIdx = getMyVar(h + &quot;cindex&quot; + rowIdx, &quot;0&quot;);
>
> for (var i = 0; i \< tArr.length; i++) {
>
> var isSel = (i == curIdx);
>
> d.push({
>
> title: isSel ? &quot;❆ &quot; + tArr\[i\] + &quot; ❆&quot; :
> tArr\[i\],
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(rIdx, idx, val, h)
> {
>
> putMyVar(h + &quot;cindex&quot; + rIdx, idx + &quot;&quot;);
>
> putMyVar(h + &quot;curl&quot; + rIdx, val);
>
> // ── 联动重置规则 ──────────────────────────────
>
> if (rIdx == 0) {
>
> // Row0（主切换）：重置所有子行
>
> for (var n = 1; n \< 10; n++) {
>
> putMyVar(h + &quot;cindex&quot; + n, &quot;0&quot;);
>
> putMyVar(h + &quot;curl&quot; + n, &quot;&quot;);
>
> }
>
> } else {
>
> // 分类行：其他分类行互斥单选（可选，按需开启）
>
> for (var m = 1; m \< 10; m++) {
>
> if (m != rIdx) putMyVar(h + &quot;cindex&quot; + m, &quot;0&quot;);
>
> }
>
> }
>
> // ────────────────────────────────────────────
>
> refreshPage(false);
>
> return &quot;hiker://empty&quot;;
>
> }, rowIdx, i, iArr\[i\], h),
>
> col_type: &quot;scroll_button&quot;
>
> });
>
> }
>
> d.push({ col_type: &quot;blank_block&quot; });
>
> };

  --------------------------------------------------------------------------------------------
  **联动规则**         **行为**                   **适用场景**
  -------------------- -------------------------- --------------------------------------------
  Row0 切换            重置 cindex1\~9 全部归 0   主模式切换（如：最近更新 / 热门 / 分类）

  分类行互斥（可选）   点击某行时，其他分类行归 0 各行分类互不干扰，同一时刻只有一行有效选中

  不互斥（各行独立）   各行 cindex 互不影响       多维筛选（如分类+地区+年份同时选）
  --------------------------------------------------------------------------------------------

> 💡 **两种联动策略的选择:**
> 如果各行分类是&quot;或&quot;关系（选了按主题就不能同时选按衣着），用互斥单选。如果各行是独立维度（分类+地区+年份可叠加），去掉
> else 分支，各行 cindex 互不影响。

**6.4 分类行渲染时机------仅第一页渲染**

分类行只在 MY_PAGE == 1 时渲染，翻页时不重复渲染。Row0 固定渲染；Row1\~N
是否渲染取决于当前 Row0
的选中状态（例如只有选中&quot;分类模式&quot;才展开 9 行矩阵）。

> if (MY_PAGE == 1) {
>
> // Row0：主模式切换，始终渲染
>
> renderRow(0, &quot;🏠最近更新&🔥热门影片&📂视频分类&quot;,
>
> &quot;/&hot&categories&quot;, \_resList, host);
>
> var index0 = getMyVar(host + &quot;cindex0&quot;, &quot;0&quot;);
>
> if (index0 == &quot;2&quot;) {
>
> // 仅在&quot;视频分类&quot;模式下展开 9 行矩阵
>
> for (var j = 0; j \< dataClass.length; j++) {
>
> renderRow(j + 1, dataClass\[j\].t, dataClass\[j\].i, \_resList, host);
>
> }
>
> }
>
> }

**6.5 URL 构建------读取 cindex/curl 拼接请求地址**

URL 构建完全由主页函数自己负责。读取 cindex0
判断当前主模式，分类模式下回溯 cindex1\~9 找到被选中的那一行，取其 curl
值拼接请求 URL。

> var \_fetchUrl = host;
>
> var c0 = getMyVar(host + &quot;cindex0&quot;, &quot;0&quot;);
>
> if (c0 == &quot;0&quot;) {
>
> // 模式0：最近更新
>
> \_fetchUrl = host + &quot;/?sort_by=release_at&from=&quot; + MY_PAGE;
>
> } else if (c0 == &quot;1&quot;) {
>
> // 模式1：热门影片
>
> \_fetchUrl = host + &quot;/?sort_by=views&from=&quot; + MY_PAGE;
>
> } else {
>
> // 模式2：矩阵分类------回溯 cindex1\~9，找到第一个非 0 的选中项
>
> var \_slug = &quot;/categories/default/&quot;; // 兜底默认分类
>
> for (var k = 1; k \< 10; k++) {
>
> var \_idx = getMyVar(host + &quot;cindex&quot; + k, &quot;0&quot;);
>
> if (\_idx != &quot;0&quot;) {
>
> \_slug = getMyVar(host + &quot;curl&quot; + k, \_slug);
>
> break; // 找到即停，单选
>
> }
>
> }
>
> // 拼接 URL，兼容路径已含 ? 的情况
>
> \_fetchUrl = host + \_slug
>
> \+ (\_slug.indexOf(&quot;?&quot;) \> -1 ? &quot;&&quot; :
> &quot;?&quot;)
>
> \+ &quot;sort_by=release_at&from=&quot; + MY_PAGE;
>
> }
>
> 📌 **变量命名:** 注意主页函数内要用 var \_fetchUrl
> 等自定义变量，不要直接给 MY_URL 赋值------MY_URL
> 是引擎注入的只读变量，覆写它在部分版本会导致引擎异常。

**6.6 多分类行互不干扰------完整工作流程**

以下是用户点击一个分类按钮后，各行状态如何变化的完整流程图解：

> 用户点击 Row3（按身材）中的&quot;巨乳&quot;
>
> → lazyRule 执行：
>
> putMyVar(host+&quot;cindex3&quot;, &quot;10&quot;) // 记录 Row3
> 选中索引
>
> putMyVar(host+&quot;curl3&quot;, &quot;/tags/big-tits&quot;) // 记录
> Row3 选中值
>
> // 互斥模式：其他行归0
>
> putMyVar(host+&quot;cindex1&quot;, &quot;0&quot;) // Row1 重置
>
> putMyVar(host+&quot;cindex2&quot;, &quot;0&quot;) // Row2 重置
>
> // cindex4\~9 同理\...
>
> → refreshPage(false)
>
> 主页函数重新执行：
>
> → c0 == &quot;2&quot;（仍在分类模式）
>
> → 渲染 Row0（✦ 视频分类 ✦），Row1（全部），Row2（全部），Row3（✦ 巨乳
> ✦）\...
>
> → URL 构建：回溯 k=1（cindex1=0 跳过），k=2（跳过），k=3（cindex3=10
> 命中）
>
> \_slug = &quot;/tags/big-tits&quot;
>
> \_fetchUrl = host +
> &quot;/tags/big-tits?sort_by=release_at&from=1&quot;
>
> → 请求并渲染列表

**6.7 矩阵模式主页函数完整骨架**

> 主页: function() {
>
> var \_resList = \[\];
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> // ── 1. renderRow 定义（主页函数内局部，不污染全局）─────
>
> var renderRow = function(rowIdx, titleStr, idStr, d, h) {
>
> var tArr = titleStr.split(&quot;&&quot;);
>
> var iArr = idStr.split(&quot;&&quot;);
>
> var curIdx = getMyVar(h + &quot;cindex&quot; + rowIdx, &quot;0&quot;);
>
> for (var i = 0; i \< tArr.length; i++) {
>
> var isSel = (i == curIdx);
>
> d.push({
>
> title: isSel ? &quot;❆ &quot; + tArr\[i\] + &quot; ❆&quot; :
> tArr\[i\],
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(rIdx, idx, val, h)
> {
>
> putMyVar(h + &quot;cindex&quot; + rIdx, idx + &quot;&quot;);
>
> putMyVar(h + &quot;curl&quot; + rIdx, val);
>
> if (rIdx == 0) {
>
> for (var n = 1; n \< 10; n++) putMyVar(h + &quot;cindex&quot; + n,
> &quot;0&quot;);
>
> } else {
>
> for (var m = 1; m \< 10; m++) {
>
> if (m != rIdx) putMyVar(h + &quot;cindex&quot; + m, &quot;0&quot;);
>
> }
>
> }
>
> refreshPage(false);
>
> return &quot;hiker://empty&quot;;
>
> }, rowIdx, i, iArr\[i\], h),
>
> col_type: &quot;scroll_button&quot;
>
> });
>
> }
>
> d.push({ col_type: &quot;blank_block&quot; });
>
> };
>
> // ── 2. 分类数据字典（硬编码）────────────────────────────
>
> var dataClass = \[
>
> { t: &quot;行1标题A&行1标题B&行1标题C&quot;, i:
> &quot;0&/path/b&/path/c&quot; },
>
> { t: &quot;行2标题A&行2标题B&quot;, i: &quot;0&/path/b2&quot; },
>
> // \... 最多到 dataClass\[8\]（对应 cindex1\~cindex9）
>
> \];
>
> // ── 3. 分类行渲染（仅第一页）────────────────────────────
>
> if (MY_PAGE == 1) {
>
> renderRow(0, &quot;最近更新&热门&分类&quot;,
> &quot;/&/hot&/categories&quot;, \_resList, \_host);
>
> if (getMyVar(\_host + &quot;cindex0&quot;, &quot;0&quot;) ==
> &quot;2&quot;) {
>
> for (var j = 0; j \< dataClass.length; j++) {
>
> renderRow(j + 1, dataClass\[j\].t, dataClass\[j\].i, \_resList,
> \_host);
>
> }
>
> }
>
> }
>
> // ── 4. URL 构建 ──────────────────────────────────────────
>
> var \_url = \_host;
>
> var c0 = getMyVar(\_host + &quot;cindex0&quot;, &quot;0&quot;);
>
> if (c0 == &quot;0&quot;) {
>
> \_url = \_host + &quot;/?sort_by=release_at&from=&quot; + MY_PAGE;
>
> } else if (c0 == &quot;1&quot;) {
>
> \_url = \_host + &quot;/?sort_by=views&from=&quot; + MY_PAGE;
>
> } else {
>
> var \_slug = &quot;/categories/default/&quot;;
>
> for (var k = 1; k \< 10; k++) {
>
> if (getMyVar(\_host + &quot;cindex&quot; + k, &quot;0&quot;) !=
> &quot;0&quot;) {
>
> \_slug = getMyVar(\_host + &quot;curl&quot; + k, \_slug);
>
> break;
>
> }
>
> }
>
> \_url = \_host + \_slug + (\_slug.indexOf(&quot;?&quot;) \> -1 ?
> &quot;&&quot; : &quot;?&quot;) + &quot;from=&quot; + MY_PAGE;
>
> }
>
> // ── 5. 请求与列表解析 ────────────────────────────────────
>
> var \_html = fetch(\_url, { headers: { &quot;User-Agent&quot;: \_ua }
> });
>
> // \... 解析 \_html，push 到 \_resList \...
>
> return \_resList; // ✅ Law 43：绝对不能漏
>
> },

**6.8 fypage 方案 vs 矩阵方案------如何选择**

  ---------------------------------------------------------------------------------
  **对比维度**        **fypage 方案（第二章）**      **矩阵方案（本章）**
  ------------------- ------------------------------ ------------------------------
  静态分类 url 写法   &quot;fypage&quot;             &quot;0&quot;（固定）
                      或含占位符的完整 URL           

  分类数据来源        写在 class_name/area_name      硬编码在主页函数内的 dataClass
                      等字段                         数组

  URL 构建            引擎自动替换占位符             主页函数完全自己构建

  适用行数            1\~4 行，结构规律              1\~9 行，结构复杂或 URL 不规律

  适用分类数          每行几个到十几个               每行可以几十个

  互斥单选逻辑        引擎自动处理                   renderRow lazyRule 内手动
                                                     putMyVar 重置

  多行叠加筛选        受限（引擎只替换对应占位符）   完全自由，主页函数自己读多行
                                                     curl 拼接

  MY_PAGE 可用        ✅（type:&quot;主页&quot;）    ✅（type:&quot;主页&quot;）
  ---------------------------------------------------------------------------------

> **第七章：二级函数------详情页**

**7.1 视频源格式（有播放列表）**

> 二级: function(url) {
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> var \_html = request(url, { headers: { &quot;User-Agent&quot;: \_ua }
> });
>
> var tabs = \[\];
>
> var lists = \[\];
>
> // MACCMS SID 强关联：用 data-sid 匹配对应的集数面板（Law 16）
>
> var tabNodes = pdfa(\_html,
> &quot;body&&.module-tab-item\[data-sid\]&quot;);
>
> for (var a = 0; a \< tabNodes.length; a++) {
>
> var sid = tabNodes\[a\].match(/data-sid=&quot;(\\d+)&quot;/)\[1\];
>
> var epItems = pdfa(\_html, &quot;body&&#pane&quot; + sid +
> &quot;&&.module-play-list-link&quot;);
>
> if (epItems.length \> 0) {
>
> // 集数下标美化（Law 21）
>
> var cnt = epItems.length;
>
> var subNum = (cnt + &quot;&quot;).split(&quot;&quot;).map(function(n)
> {
>
> return String.fromCharCode(parseInt(n) + 8320);
>
> }).join(&quot;&quot;);
>
> tabs.push(pdfh(tabNodes\[a\], &quot;span&&Text&quot;) + subNum);
>
> var subList = \[\];
>
> for (var i = 0; i \< epItems.length; i++) {
>
> subList.push({
>
> title: pdfh(epItems\[i\], &quot;Text&quot;),
>
> // ✅ 直接构造 video:// 协议（V102 最稳定方案）
>
> url: &quot;video://&quot; + pd(epItems\[i\], &quot;a&&href&quot;,
> \_host)
>
> \+ &quot;;{User-Agent@&quot; + \_ua + &quot;&&Referer@&quot; +
> \_host + &quot;/}&quot;
>
> });
>
> }
>
> lists.push(subList);
>
> }
>
> }
>
> return {
>
> vod_name: pdfh(\_html, &quot;h1&&Text&quot;),
>
> vod_pic: pd(\_html, &quot;.cover&&img&&src&quot;, \_host),
>
> vod_content: pdfh(\_html, &quot;.intro&&Text&quot;),
>
> line: tabs,
>
> list: lists
>
> };
>
> },

**7.2 漫画/小说格式**

> 二级: function(url) {
>
> var \_host = this.host;
>
> var \_html = request(url);
>
> var chapters = pdfa(\_html, &quot;ul.chapter&&li&quot;);
>
> var list = \[\];
>
> for (var i = 0; i \< chapters.length; i++) {
>
> list.push({
>
> title: pdfh(chapters\[i\], &quot;a&&Text&quot;),
>
> // ✅ 聚阅铁律：必须有 #isJiexi=1 才能触发内置阅读器
>
> url: pd(chapters\[i\], &quot;a&&href&quot;, \_host) +
> &quot;#isJiexi=1#readTheme##autoPage#&quot;
>
> });
>
> }
>
> return { detail1: pdfh(\_html,&quot;h1&&Text&quot;), img:
> pd(\_html,&quot;.cover&&img&&src&quot;,\_host),
>
> desc: pdfh(\_html,&quot;.desc&&Text&quot;), list: \[list\] };
>
> },

**7.3 二级返回结构完整字段**

  ------------------------------------------------------------------------------------------
  **字段**            **类型**                        **说明**
  ------------------- ------------------------------- --------------------------------------
  vod_name / detail1  String                          标题

  vod_pic / img       String                          封面图 URL

  vod_remarks         String                          简短备注（如&quot;更新至09集&quot;）

  vod_content / desc  String                          详细简介

  line                Array\<String\>                 线路名称数组

  list                Array\<Array\<{title,url}\>\>   与 line 一一对应的选集二维数组

  noShow              Object                          {封面:false, 简介:false, 选集:false}
                                                      控制不显示的组件

  moreitems           Array                           选集上方的扩展项

  extenditems         Array                           选集下方的扩展项（图文详情常用）

  novel               Boolean                         是否为小说模式
  ------------------------------------------------------------------------------------------

**7.4 erji 沙箱 vs 标准二级的选择依据**

**标准二级机制**

规则对象里有 二级 函数，引擎在用户点击列表卡片时自动调用，input 是卡片的
url 字段值。适用条件：详情页是独立 URL，可以直接 fetch/request 拿到完整
HTML；返回结构是标准的 vod_name / img / line / list
对象；不需要在详情页内部再跳转到其他同类详情页。

**erji 沙箱机制**

没有 二级 函数，卡片的 url 字段末尾拼接
\$('').rule(parse.erji)，点击后进入独立的列表沙箱页面，getResCode()
拿详情页 HTML，setResult(d)
输出渲染结果。适用条件：（1）详情页内部存在嵌套跳转------详情页里还有指向其他详情页的链接，需要递归进入同一套解析逻辑；（2）详情页里混合了图文、视频、关联跳转等多种内容类型，标准二级的
line/list
结构装不下；（3）图片需要解密，且解密逻辑（tlazy）需要在沙箱内重新初始化。

**判断流程**

> 详情页内部有没有指向同类详情页的链接？
>
> ├── 有 → 用 erji 沙算（递归拼 \$('').rule(parse.erji)）
>
> └── 没有
>
> 详情页是纯章节列表 + 简介？
>
> ├── 是 → 用标准二级（返回 line/list 结构）
>
> └── 否（图文混排/视频/复杂布局） → 用 erji 沙算

以海角吃瓜为例，选 erji
沙箱的原因是前两条同时成立：详情页里有相关帖子链接（/archives/xxx）需要再次进入同一解析逻辑，且内容是图文混排加视频，标准二级的
list 二维数组无法表达这种结构。

> **第八章：解析函数------播放/阅读协议**

  ----------------------------------------------------------------------------
  **内容类型**    **返回格式**                          **说明**
  --------------- ------------------------------------- ----------------------
  视频直链        return \'video://\' + url +           分号后为 Header 配置
                  \';{User-Agent@UA&&Referer@host/}\'   

  漫画/图集       return \'pics://\' + url1 + \'&&\' +  每个 URL 可追加
                  url2 + \'\...\'                       \@Referer=xxx

  嗅探兜底        return url + \'#嗅探\'                无法解析时触发 App
                                                        内置嗅探器
  ----------------------------------------------------------------------------

> ⚠️ **警告:** 解析函数（parse.解析）只认 return 协议字符串，严禁使用
> setResult()，否则漫画/小说内容无法显示！（Law 47a）⚠️
> 注意：\$().rule() 沙箱内是列表场景，必须用 setResult(d)，不能
> return；两者场景不同，请勿混淆。

**8.1 图片防盗链写法（两种格式均可用）**

> // ✅ 推荐新格式（支持多 header）
>
> pics.push(picUrl +
> &quot;@headers={\\&quot;Referer\\&quot;:\\&quot;https://cdn.example.com/\\&quot;,\\&quot;User-Agent\\&quot;:\\&quot;&quot; +
> parse.UA + &quot;\\&quot;}&quot;);
>
> // 简写格式（实测可靠；Referer 填图片 CDN 域名，非站点主域名）
>
> pics.push(picUrl + &quot;@Referer=https://cdn.example.com/&quot;);
>
> // nhentai 专用（t2 → i2 域名 + 去文件名末尾 t）
>
> var big = src.split(&quot; &quot;)\[0\]
>
> .replace(&quot;t2.nhentai.net&quot;, &quot;i2.nhentai.net&quot;)
>
> .replace(/(\\d+)t(\\.\[a-z\]+)\$/, &quot;\$1\$2&quot;);
>
> pics.push(big +
> &quot;@headers={\\&quot;Referer\\&quot;:\\&quot;https://nhentai.net/\\&quot;}&quot;);

**8.2 三种 lazyRule 播放模式**

**模式1：直接 video:// 协议（最稳定，优先选用）**

> // ✅ V102 验证最稳定：能直接构造协议就绝不用 lazyRule
>
> url: &quot;video://&quot; + epHref + &quot;;{User-Agent@&quot; +
> parse.UA + &quot;&&Referer@&quot; + parse.host + &quot;/}&quot;

**模式2：lazyRule 内联（需进一步解析时）**

> // ✅ 用于需要请求详情页解密的场景
>
> // 注意：lazyRule 内无法访问外部变量，所有需要的值必须通过参数传入
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(pUrl, ua, host) {
>
> var \_html = request(pUrl, { headers: { &quot;User-Agent&quot;: ua }
> });
>
> var \_slash = String.fromCharCode(92); // ✅ 反斜杠隔离术
>
> var m =
> \_html.match(/https?:\\/\\/\[\^\\s\'&quot;\]+\\.m3u8\[\^\\s\'&quot;.\]\*/);
>
> if (m) return &quot;video://&quot; +
> m\[0\].split(\_slash).join(&quot;&quot;) + &quot;;{User-Agent@&quot; +
> ua + &quot;&&Referer@&quot; + host + &quot;/}&quot;;
>
> return pUrl + &quot;#嗅探&quot;;
>
> }, detailUrl, parse.UA, parse.host)

**模式3：MACCMS player_aaaa 双重解密（见第九章）**

> **第九章：MACCMS V10 播放加密穿透（Law 39）**

**9.1 加密特征识别**

> // 播放页源码特征：player_aaaa 配置块
>
> // encrypt: 0=不加密, 1=URL编码, 2=Base64后再URL编码
>
> var player_aaaa = {
>
> &quot;flag&quot;: &quot;play&quot;,
>
> &quot;encrypt&quot;: 2,
>
> &quot;url&quot;: &quot;JTY4JTc0\...&quot;,
>
> &quot;from&quot;: &quot;ddao&quot;
>
> }

**9.2 完整解密实现**

> \_parseVideo: function(playUrl) {
>
> var \_ua = this.UA;
>
> var \_host = this.host;
>
> var html = request(playUrl, { headers: { &quot;User-Agent&quot;: \_ua
> } });
>
> html = this.\_autoDeshell(html);
>
> var match = html.match(/var
> player_aaaa\\s\*=\\s\*(\\{\[\\s\\S\]\*?\\})(?=;\\s\*\<\\/script\|}\|\\\]\\\]\>)/);
>
> if (!match) { log(&quot;player_aaaa 未找到，嗅探&quot;); return
> playUrl + &quot;#嗅探&quot;; }
>
> try {
>
> var cfg = JSON.parse(match\[1\]);
>
> var encUrl = cfg.url;
>
> var realUrl = &quot;&quot;;
>
> if (cfg.encrypt == 2) realUrl = unescape(this.\_base64Decode(encUrl));
>
> else if (cfg.encrypt == 1) realUrl = unescape(encUrl);
>
> else realUrl = encUrl;
>
> var slash = String.fromCharCode(92);
>
> realUrl = realUrl.split(slash).join(&quot;&quot;);
>
> return &quot;video://&quot; + realUrl + &quot;;{User-Agent@&quot; +
> \_ua + &quot;&&Referer@&quot; + \_host + &quot;/}&quot;;
>
> } catch(e) {
>
> log(&quot;解析失败: &quot; + e);
>
> return playUrl + &quot;#嗅探&quot;;
>
> }
>
> }
>
> **第十章：图片处理------字节流优先（Law 28）**

JS 层处理 Base64 图片或大图时极易触发内存溢出（Error 36）。必须使用
FileUtil 字节流方案：

> var \_pic = pd(it, &quot;img&&data-original&quot;) \|\| pd(it,
> &quot;img&&src&quot;);
>
> if (\_pic && \_pic.indexOf(&quot;http&quot;) === 0) {
>
> \_pic = \$(\_pic).image({
>
> image: function() {
>
> var FileUtil = com.example.hikerview.utils.FileUtil;
>
> try {
>
> var raw = FileUtil.readBytes(input, {
>
> &quot;User-Agent&quot;: parse.UA,
>
> &quot;Referer&quot;: parse.host + &quot;/&quot;
>
> });
>
> return FileUtil.toInputStream(raw);
>
> } catch(e) {
>
> log(&quot;图片字节流失败: &quot; + e);
>
> return null;
>
> }
>
> }
>
> });
>
> }
>
> ⚠️ **警告:** 严禁在 JS 层用字符串处理 Base64 大图。必须走
> FileUtil.readBytes + toInputStream 字节流通道。

如需图片异或解密（如 Maomi/欢乐谷站点），在 readBytes 后、toInputStream
前加解密逻辑：

> var raw = FileUtil.readBytes(input, { &quot;User-Agent&quot;: parse.UA
> });
>
> var key = &quot;2019ysapp&quot;;
>
> for (var i = 0; i \< Math.min(raw.length, 100); i++) {
>
> var res = (raw\[i\] \^ key.charCodeAt(i % key.length)) & 0xff;
>
> raw\[i\] = res \> 127 ? res - 256 : res;
>
> }
>
> return FileUtil.toInputStream(raw);

**10.2 图片 AES 解密------\$(url).image() + CryptoUtil 方案**

当图片本身是 AES 加密的二进制数据（下载后不能直接显示）时，需要用
\$(url).image() 语法在下载完成后解密。这是和字节流防盗链完全不同的场景。

  ----------------------------------------------------------------------------------------------
  **场景**                      **方案**                  **核心 API**
  ----------------------------- ------------------------- --------------------------------------
  图片防盗链/大图，内容未加密   FileUtil.readBytes +      com.example.hikerview.utils.FileUtil
                                toInputStream             

  图片内容 AES                  \$(url).image(() =\>      hiker://assets/crypto-java.js
  加密，下载后需解密            CryptoUtil.AES.decrypt)   

  图片前 N 字节异或加密         FileUtil.readBytes +      同字节流方案
                                手动异或循环              
  ----------------------------------------------------------------------------------------------

**方案一：作为图片 lazyRule 字符串（拼接到 pic_url 末尾）**

\_getImageDecrypt() 返回一段 lazyRule 字符串，直接拼接到图片 URL
末尾，适合列表页封面批量解密。提前调用一次赋给变量，循环内复用。

> // 定义解密 lazyRule 生成函数
>
> \_getImageDecrypt: function() {
>
> return \$(&quot;&quot;).image(function() {
>
> try {
>
> var CryptoUtil =
> \$.require(&quot;hiker://assets/crypto-java.js&quot;);
>
> var key = CryptoUtil.Data.parseUTF8(&quot;your_aes_key_here&quot;);
>
> var iv = CryptoUtil.Data.parseUTF8(&quot;your_aes_iv_here&quot;);
>
> var rawData = CryptoUtil.Data.parseInputStream(input);
>
> var dec = CryptoUtil.AES.decrypt(rawData, key, {
>
> mode: &quot;AES/CBC/PKCS7Padding&quot;, iv: iv
>
> });
>
> return dec.toInputStream();
>
> } catch(e) { log(&quot;解密失败: &quot;+e); return input; }
>
> });
>
> },
>
> // ✅ 使用：提前调用一次，循环内复用（禁止在循环内每次调用）
>
> \_parseList: function(html) {
>
> var tlazy = this.\_getImageDecrypt(); // 只调用一次
>
> var items = pdfa(html, &quot;ul.list&&li&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var imgSrc = pd(items\[i\], &quot;img&&data-src&quot;);
>
> d.push({ pic_url: imgSrc + tlazy }); // 直接拼接
>
> }
>
> },

**方案二：在 lazyRule 内部重新定义（用于阅读页解析）**

lazyRule 内部无法访问外部的 \_getImageDecrypt()
返回值（作用域隔离），需要在 lazyRule
内部重新定义解密逻辑，再拼接到每张图片 URL 末尾。

> \_getComicReaderRule: function() {
>
> return \$(&quot;&quot;).lazyRule(function() {
>
> var host = &quot;https://example.com&quot;;
>
> // lazyRule 内部重新定义解密 lazyRule（不能从外部传入）
>
> var tlazy = \$(&quot;&quot;).image(function() {
>
> try {
>
> var CryptoUtil =
> \$.require(&quot;hiker://assets/crypto-java.js&quot;);
>
> var key = CryptoUtil.Data.parseUTF8(&quot;your_aes_key&quot;);
>
> var iv = CryptoUtil.Data.parseUTF8(&quot;your_aes_iv&quot;);
>
> var dec = CryptoUtil.AES.decrypt(
>
> CryptoUtil.Data.parseInputStream(input), key,
>
> { mode: &quot;AES/CBC/PKCS7Padding&quot;, iv: iv }
>
> );
>
> return dec.toInputStream();
>
> } catch(e) { log(&quot;解密失败: &quot;+e); return input; }
>
> });
>
> var html = request(input, { headers: { &quot;User-Agent&quot;:
> &quot;pc&quot;, &quot;Referer&quot;: host+&quot;/&quot; } });
>
> var pics = \[\];
>
> var items = pdfa(html, &quot;.reader&&img&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var src = pdfh(items\[i\], &quot;img&&data-src&quot;) \|\|
> pdfh(items\[i\], &quot;img&&src&quot;);
>
> if (src && src.indexOf(&quot;loading&quot;) == -1) {
>
> if (src.indexOf(&quot;http&quot;) !== 0) src = host + src;
>
> pics.push(src + tlazy); // 每张图拼上解密 lazyRule
>
> }
>
> }
>
> if (pics.length == 0) return &quot;toast://未找到图片&quot;;
>
> return &quot;pics://&quot; + pics.join(&quot;&&quot;);
>
> });
>
> },
>
> ⚠️ **警告:** \_getImageDecrypt()
> 每次调用都生成一段新字符串，严禁在循环内每次调用。必须提前 var tlazy =
> this.\_getImageDecrypt() 调用一次，循环内直接用 tlazy。
>
> 💡 **CryptoUtil 支持的加密模式:**
> AES/CBC/PKCS7Padding（最常用）、AES/ECB/PKCS7Padding、AES/CBC/NoPadding
> 等。key/iv 通常在站点 JS 源码中搜索 key/iv/secret 关键词找到。
>
> **第十一章：JS 引擎报错速查与修复**

  ----------------------------------------------------------------------------
  **报错码**   **错误原因**             **解决方案**
  ------------ ------------------------ --------------------------------------
  Error 14     变量未定义 / 用了        直接
               let/const / 作用域泄漏   video://，或将变量显式传参；全量改用
                                        var

  Error 31     语法错误（含正则拼接）   用 new RegExp() 动态创建正则

  Error 35     lazyRule 内嵌套复杂对象  简化 lazyRule 内代码

  Error 36     字符串过长/内存溢出      图片走字节流；弃用长字符串拼接
                                        lazyRule

  Error 38     代码未闭合               检查字符串中的换行符，用 \\n 转义

  Error 43     非法反斜杠               用 String.fromCharCode(92)，禁止直接写
                                        \\
  ----------------------------------------------------------------------------

**11.1 高频陷阱对照表**

  --------------------------------------------------------------------------------
  **问题现象**        **原因**                     **解决方案**
  ------------------- ---------------------------- -------------------------------
  MY_PAGE 未定义      type:0 不注入 MY_PAGE        改用 type:&quot;主页&quot; +
                                                   url:&quot;fypage&quot;

  视频点击跳网页      fetch 返回空，#嗅探 被触发   加 log 调试，检查 html.length

  scroll_button       部分机型 HTML 渲染不一致     用纯文本符号 ✦ title ✦ 标记选中
  选中状态不稳定                                   

  getMyVar key 报错   key 含特殊字符 ://           改用纯字母前缀如
                                                   sitename_cindex0

  漫画图片空白        图片懒加载，src 是占位符     提取 data-original / data-src

  双域名拼接          putMyVar 存了完整 URL        存前
                                                   .replace(host,&quot;&quot;)
                                                   去域名

  返回列表停在末页    页码未重置                   返回按钮同时 putMyVar
                                                   page=&quot;1&quot;

  CF Cookie           用了 putMyVar 存 Cookie      改用 setItem 持久化
  每次重新验证                                     

  lazyRule 内变量     闭包隔离，无法访问外部变量   显式传参或硬编码到 lazyRule 内
  undefined                                        

  this.UA 在          this 指向变了                提前 var \_ua = this.UA; 再传参
  rule/lazyRule                                    
  内报错                                           
  --------------------------------------------------------------------------------

**11.2 安全编码规范**

**变量命名避锋（Law 23）**

> // ❌ 以 t/v/g 开头可能触发正则标志
>
> var title = &quot;\...&quot;; var videoUrl = &quot;\...&quot;;
>
> // ✅ 以下划线开头
>
> var \_title = &quot;\...&quot;; var \_videoUrl = &quot;\...&quot;;

**反斜杠隔离术（Law 30）**

> // ❌ 直接写反斜杠会触发转义 Bug
>
> var cleanUrl = realUrl.replace(/\\\\/g, &quot;&quot;);
>
> // ✅ 动态生成 ASCII 92
>
> var slash = String.fromCharCode(92);
>
> var cleanUrl = realUrl.split(slash).join(&quot;&quot;);

**ES5 全量降级（Law 34）**

> // ❌ 禁用
>
> let a = 1; const b = 2; c?.d; arr.map(x =\> x); \`\${a}\`;
>
> // ✅ 只能用
>
> var a = 1; var b = 2; if(c && c.d){} arr.map(function(x){return x;});
> a+&quot;&quot;;

**this 指向保护**

> // ❌ 错误：在 \$().rule 或 lazyRule 中直接用 this
>
> url: \$().rule(function(targetUrl) {
>
> var html = fetch(targetUrl, { headers: { &quot;User-Agent&quot;:
> this.UA } }); // this 指向错误！
>
> }, url)
>
> // ✅ 正确：提前取出，通过参数传入
>
> var \_ua = this.UA;
>
> var \_host = this.host;
>
> url: \$().rule(function(targetUrl, ua, host) {
>
> var html = fetch(targetUrl, { headers: { &quot;User-Agent&quot;: ua }
> });
>
> }, url, \_ua, \_host)
>
> **第十二章：Cloudflare 过盾标准方案**

CF 过盾有两个关键细节，错一个就失效：①检测条件要用页面内容判断（不用
html.length）；②过盾成功条件必须同时满足&quot;页面有真实内容&quot;和&quot;Cookie
有 cf_clearance&quot;------只有 Cookie
不等页面加载完，返回太早下次请求还是被 CF 拦截。

**12.1 完整标准模板（实战验证版）**

> // ── CF 过盾标准模板 ──────────────────────────────────────
>
> // UA 必须用 Android Chrome，和系统 WebView 默认 UA 一致
>
> // cf_clearance Cookie 与验证时的 UA+IP 绑定，UA 不匹配则 Cookie 无效
>
> UA: &quot;Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36
> (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36&quot;,
>
> \_fetch: function(url) {
>
> var d = \[\];
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> // Step1: putVar(临时中转) → setItem(持久化) 两步流程
>
> var tempCk = getVar(&quot;site_ck&quot;, &quot;&quot;);
>
> if (tempCk !== &quot;&quot;) {
>
> setItem(&quot;site_cookie&quot;, tempCk);
>
> clearVar(&quot;site_ck&quot;);
>
> }
>
> var \_cookie = getItem(&quot;site_cookie&quot;, &quot;&quot;);
>
> var html = fetch(url, {
>
> headers: { &quot;User-Agent&quot;: \_ua, &quot;Referer&quot;:
> \_host+&quot;/&quot;, &quot;Cookie&quot;: \_cookie }
>
> });
>
> // ✅ 检测条件：用页面内容判断，不用 html.length（长度不稳定）
>
> // 把 &quot;列表元素的 class&quot; 替换为实际站点的视频卡片 class
>
> if (html.indexOf(&quot;视频卡片class&quot;) === -1) {
>
> d.push({
>
> title: &quot;🛡️ CF 验证 -\> 点击过盾&quot;,
>
> col_type: &quot;text_center_1&quot;,
>
> url: \$().rule(function(targetUrl, ua) {
>
> var res = \[{
>
> title: &quot;验证沙箱（过盾后自动返回）&quot;,
>
> url: targetUrl,
>
> col_type: &quot;x5_webview_single&quot;,
>
> desc: &quot;float&&100%&quot;,
>
> extra: {
>
> ua: ua,
>
> js: \$.toString(function() {
>
> function check() {
>
> var c = fba.getCookie(location.href);
>
> // ✅ 核心：必须同时满足两个条件才返回
>
> // 1. 页面已加载出真实内容（视频列表节点存在）
>
> // 2. Cookie 里有 cf_clearance
>
> // 只检查 Cookie 会返回太早，下次请求还被拦截
>
> var nodes = document.querySelectorAll(&quot;.视频卡片class&quot;);
>
> if (nodes && nodes.length \> 0 &&
>
> c && c.indexOf(&quot;cf_clearance&quot;) \> -1) {
>
> fba.putVar(&quot;site_ck&quot;, c); // putVar 临时中转
>
> fba.toast(&quot;✅ 过盾成功！&quot;);
>
> fba.back();
>
> } else {
>
> setTimeout(check, 1000);
>
> }
>
> }
>
> check();
>
> })
>
> }
>
> }\];
>
> setResult(res);
>
> }, url, \_ua) // ✅ 传 \_ua 字符串，不传 this
>
> });
>
> return d;
>
> }
>
> return this.\_parseList(html);
>
> },

**12.2 四个关键点**

  -------------------------------------------------------------------------------------------------------------
  **关键点**      **错误做法**       **正确做法**                          **原因**
  --------------- ------------------ ------------------------------------- ------------------------------------
  UA              用 iPhone/PC UA    用 Android Chrome UA                  cf_clearance 与验证时的 UA
                                                                           绑定，不匹配则 Cookie 无效

  CF 检测条件     html.length \<     html.indexOf(&quot;视频class&quot;)   页面长度不稳定，内容判断更准确
                  1500               === -1                                

  过盾成功条件    只检查             节点存在 AND cf_clearance 存在        只有 Cookie
                  cf_clearance 存在                                        会返回太早，页面未渲染完下次还被拦

  Cookie 持久化   putVar 直接存      putVar 临时中转 → 主页函数 setItem    putVar 是会话级，App 重启后丢失
                                     持久化                                
  -------------------------------------------------------------------------------------------------------------

**12.3 this.UA 在 \$().rule 内无效**

\$().rule 内部是独立作用域，this 指向不是 parse 对象。UA 必须在调用
\$().rule 之前取出赋给局部变量，通过参数传入。

> // ❌ 错误：\$().rule 内 this.UA 是 undefined
>
> url: \$().rule(function(targetUrl) {
>
> var res = \[{ extra: { ua: this.UA } }\]; // this.UA = undefined
>
> }, url)
>
> // ✅ 正确：提前取出，通过参数传入
>
> var \_ua = this.UA; // 提前取出
>
> url: \$().rule(function(targetUrl, ua) {
>
> var res = \[{ extra: { ua: ua } }\]; // ✅ 正常
>
> }, url, \_ua) // 通过参数传入
>
> **第十三章：域名自愈系统（Law 14）**
>
> 预处理: function() {
>
> var \_host = getItem(&quot;current_host&quot;, this.host);
>
> var \_test = fetch(\_host + &quot;/&quot;, { headers: {
> &quot;User-Agent&quot;: this.UA }, timeout: 5000 });
>
> if (!\_test \|\| \_test.indexOf(&quot;特征关键词&quot;) === -1) {
>
> log(&quot;域名失效，重置为默认&quot;);
>
> setItem(&quot;current_host&quot;, this.host);
>
> } else {
>
> setItem(&quot;current_host&quot;, \_host);
>
> }
>
> },
>
> // 所有函数中统一获取（不要硬编码 this.host）
>
> var \_host = getItem(&quot;current_host&quot;, this.host);
>
> **第十四章：聚阅专属规则**

**14.1 变量注入规律（核心必读）**

> ⚠️ **警告:** type:0 时 MY_PAGE 不会被注入。连 typeof MY_PAGE !==
> &quot;undefined&quot; 也救不了------聚阅在词法分析阶段就报错，和 JS
> 运行时检查无关。
>
> // ✅ 唯一正确方案
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;,
>
> &quot;url&quot;: &quot;fypage&quot;,
>
> &quot;class_name&quot;: &quot;全部&分类1&分类2&quot;,
>
> &quot;class_url&quot;: &quot;0&1&2&quot;
>
> },
>
> 页码: { &quot;主页&quot;: 1 },
>
> 主页: function() {
>
> var \_url = this.host + &quot;/?page=&quot; + MY_PAGE; // MY_PAGE
> 直接用，无需判断
>
> }

**14.2 解析函数协议对照**

  -----------------------------------------------------------------------
  **内容类型**        **返回格式**
  ------------------- ---------------------------------------------------
  视频直链            return \'video://\' + url +
                      \';{User-Agent@UA&&Referer@host/}\'

  漫画/图集           return \'pics://\' + url1 + \'&&\' + url2

  嗅探兜底            return url + \'#嗅探\'

  小说章节 URL 后缀   #isJiexi=1#readTheme##autoPage#
  -----------------------------------------------------------------------

> ⚠️ **警告:** 解析函数（parse.解析）只认 return，严禁 setResult()（Law
> 47a）。lazyRule 中也一样，lazyRule 返回的字符串就是解析结果。⚠️
> 区分：\$().rule() 列表沙箱内必须用 setResult(d)（Law 47b），不能
> return。

**14.3 putMyVar 存路径而非完整 URL（Law 45）**

> // ❌ 错误：存完整 URL，之后拼接时出现双域名
>
> var \_bHref = pd(btn, &quot;a&&href&quot;, host);
>
> putMyVar(vk, \_bHref);
>
> var \_detailUrl = host + \_selItem; // 变成
> https://host/https://host/path
>
> // ✅ 正确：存入前去掉域名
>
> var \_bPath = \_bHref.replace(host, &quot;&quot;);
>
> putMyVar(vk, \_bPath);
>
> var \_detailUrl = host + \_selItem; // 正确

**14.4 返回列表同时重置页码（Law 46）**

> // 返回按钮：清除选中项，同时重置页码为 1，防止停留在末页
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(vk, h) {
>
> putMyVar(vk, &quot;&quot;);
>
> putMyVar(h + &quot;page&quot;, &quot;1&quot;);
>
> refreshPage(false);
>
> return &quot;hiker://empty&quot;;
>
> }, \_varKey, host)
>
> **第十五章：开发铁律速查全表**

  ----------------------------------------------------------------------------------------------------
  **编号**   **铁律名称**       **核心要义**                       **关键口诀**
  ---------- ------------------ ---------------------------------- -----------------------------------
  Law 1      入口固定化         type:&quot;主页&quot; +            禁用 type:0，禁用 fyclass 作为唯一
                                url:&quot;fypage&quot;             url

  Law 2      作用域隔离         lazyRule 内通过参数传值，禁用 this 显式传参：fn(a,b,c), val1, val2,
                                                                   val3

  Law 3      静默嗅探           返回直链用 video:// 协议           video://直链;{Referer@host/}

  Law 4      防盗链补全         资源必须挂载 Referer               Referer 漏了就 403

  Law 6      模糊选择器         类名随机变化时用                   \[class\*=&quot;item&quot;\]
                                \[class\*=&quot;&quot;\]           比固定类名稳健

  Law 8      Header 零空格      ; 和 && 前后无空格                 &quot;video://url;{UA@val}&quot;
                                                                   有空格就崩

  Law 11     CF 破盾            X5 沙箱 + fba.getCookie            过盾后必须 setItem 持久化

  Law 14     域名自愈           预处理探测域名，setItem 持久化     getItem(&quot;host&quot;,default)

  Law 16     SID 强关联         data-sid 匹配 paneID               #pane+sid 防线路错位

  Law 17     路径编码           非英文路径执行 encodeURI           忘了就 404

  Law 18     持久化存储         动态数据存入 setItem               CF Cookie 必须 setItem

  Law 19     禁止内部循环分页   禁止 lazyRule 内 while 循环抓多页  用&quot;加载更多&quot;按钮

  Law 21     UI 字符增强        Unicode 下标美化集数               String.fromCharCode(n+8320)

  Law 22     指纹二次同步       getVar 转 setItem 持久化           clearVar 清临时值

  Law 23     变量名避锋         不以 t/v/g 开头，用下划线          var \_title, var \_url

  Law 26     传参透传           \$().lazyRule 显式传参             不能用 this，必须传参

  Law 28     字节流图片         图片走 FileUtil.readBytes          \$(pic).image({image:fn})

  Law 30     反斜杠隔离         String.fromCharCode(92)            禁止直接写 \\

  Law 34     ES5 全量降级       禁用 let/const/=\>/?./             全量 var + function

  Law 37     Base64 修正        用 Java android.util.Base64        双保险：Java → atob 降级

  Law 39     自动脱壳           decodeURIComponent 最多脱 3 层     \_autoDeshell 防死循环

  Law 41     聚阅变量注入       type:&quot;主页&quot;+fypage       不能用 type:0
                                才注入 MY_PAGE                     

  Law 42     scroll_button      建议纯文本标记选中（兼容性最好）   \'✦ \'+title+\' ✦\'

  Law 43     主页必须 return    函数末尾 return \_res 不能漏       写完后检查 return

  Law 44     命名空间隔离       多模式各用独立 key                 host + \'sel\_\' + mode

  Law 45     存纯路径           putMyVar 存路径，不存完整 URL      .replace(host,\'\')

  Law 46     返回重置页码       返回列表同时 putMyVar page=1       防止末页空白

  Law 47     解析函数 return    禁用 setResult，只认 return        return \'pics://\'

  Law 48     nhentai 大图       t2→i2 域名 + 去末尾 t              .replace(\'t2\',\'i2\')

  Law 49     五维占位对应       name 与 url 数量一一对应           数量不匹配→越界 undefined

  Law 50     空值即全部         &quot;全部&quot;选项 url           &quot;area_url&quot;:
                                留空字符串                         &quot;&国&日&quot;
  ----------------------------------------------------------------------------------------------------

> **第十六章：完整规则骨架模板（可直接复用）**
>
> var parse = {
>
> 作者: &quot;dev&quot;, 版本: &quot;20260101.V1&quot;,
>
> host: &quot;https://example.com&quot;,
>
> UA: &quot;Mozilla/5.0 (Windows NT 10.0; Win64; x64)
> AppleWebKit/537.36&quot;,
>
> 页码: { &quot;主页&quot;: 1, &quot;分类&quot;: 1, &quot;搜索&quot;: 1
> },
>
> 静态分类: {
>
> type: &quot;主页&quot;, url: &quot;fypage&quot;,
>
> class_name: &quot;全部&分类1&分类2&quot;,
>
> class_url: &quot;0&1&2&quot;
>
> },
>
> 主页: function() {
>
> var \_host = this.host; var \_ua = this.UA;
>
> var \_h = \_host; // getMyVar 命名空间前缀
>
> var \_res = \[\];
>
> if (MY_PAGE == 1) {
>
> renderRow(0, &quot;全部&分类1&分类2&quot;, &quot;0&1&2&quot;, \_res,
> \_h);
>
> }
>
> var \_c0 = getMyVar(\_h + &quot;curl0&quot;, &quot;0&quot;);
>
> var \_url = \_host + &quot;/list/&quot; + \_c0 + &quot;/page/&quot; +
> MY_PAGE + &quot;/&quot;;
>
> var \_html = fetch(\_url, { headers: { &quot;User-Agent&quot;: \_ua }
> });
>
> var \_items = pdfa(\_html, &quot;body&&.list&&li&quot;);
>
> for (var i = 0; i \< \_items.length; i++) {
>
> \_res.push({
>
> title: pdfh(\_items\[i\], &quot;a&&title&quot;),
>
> url: pd(\_items\[i\], &quot;a&&href&quot;, \_host),
>
> pic_url: pd(\_items\[i\], &quot;img&&data-original&quot;),
>
> col_type: &quot;movie_3&quot;
>
> });
>
> }
>
> return \_res; // ✅ Law 43：绝对不能漏
>
> },
>
> 二级: function(url) {
>
> var \_host = this.host; var \_ua = this.UA;
>
> var \_html = request(url, { headers: { &quot;User-Agent&quot;: \_ua }
> });
>
> var \_eps = pdfa(\_html, &quot;.episode&&a&quot;);
>
> var \_list = \[\];
>
> for (var i = 0; i \< \_eps.length; i++) {
>
> \_list.push({
>
> title: pdfh(\_eps\[i\], &quot;Text&quot;),
>
> url: &quot;video://&quot; + pd(\_eps\[i\], &quot;href&quot;, \_host)
>
> \+ &quot;;{User-Agent@&quot; + \_ua + &quot;&&Referer@&quot; +
> \_host + &quot;/}&quot;
>
> });
>
> }
>
> return {
>
> vod_name: pdfh(\_html, &quot;h1&&Text&quot;),
>
> vod_pic: pd(\_html, &quot;.cover&&img&&src&quot;, \_host),
>
> desc: pdfh(\_html, &quot;.intro&&Text&quot;),
>
> line: \[&quot;默认线路&quot;\],
>
> list: \[\_list\]
>
> };
>
> },
>
> 解析: function(url) {
>
> var \_url = url.split(&quot;#&quot;)\[0\];
>
> var \_ua = this.UA;
>
> var \_host = this.host;
>
> var \_html = request(\_url, { headers: { &quot;User-Agent&quot;: \_ua
> } });
>
> \_html = this.\_autoDeshell(\_html);
>
> var m =
> \_html.match(/https?:\\/\\/\[\^\\s\'&quot;\]+\\.m3u8\[\^\\s\'&quot;.\]\*/);
>
> if (m) {
>
> var \_slash = String.fromCharCode(92);
>
> return &quot;video://&quot; + m\[0\].split(\_slash).join(&quot;&quot;)
>
> \+ &quot;;{User-Agent@&quot; + \_ua + &quot;&&Referer@&quot; +
> \_host + &quot;/}&quot;;
>
> }
>
> return \_url + &quot;#嗅探&quot;;
>
> },
>
> 搜索: function(name) {
>
> var \_host = this.host; var \_ua = this.UA;
>
> var \_url = \_host + &quot;/search/&quot; + encodeURIComponent(name) +
> &quot;/page/&quot; + MY_PAGE + &quot;/&quot;;
>
> var \_html = fetch(\_url, { headers: { &quot;User-Agent&quot;: \_ua }
> });
>
> var \_res = \[\];
>
> var \_items = pdfa(\_html, &quot;body&&.search-item&quot;);
>
> for (var i = 0; i \< \_items.length; i++) {
>
> \_res.push({
>
> title: pdfh(\_items\[i\], &quot;h3&&Text&quot;),
>
> url: pd(\_items\[i\], &quot;a&&href&quot;, \_host),
>
> pic_url: pd(\_items\[i\], &quot;img&&src&quot;),
>
> col_type: &quot;movie_1_vertical_pic_blur&quot;
>
> });
>
> }
>
> return \_res;
>
> },
>
> \_base64Decode: function(str) {
>
> if (!str) return &quot;&quot;;
>
> str = str.replace(/-/g,&quot;+&quot;).replace(/\_/g,&quot;/&quot;);
>
> while (str.length % 4) str += &quot;=&quot;;
>
> try {
>
> var b = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
>
> return new java.lang.String(b, &quot;UTF-8&quot;);
>
> } catch(e) {
>
> try { return decodeURIComponent(escape(atob(str))); }
>
> catch(e2) { log(&quot;Base64 fail&quot;); return &quot;&quot;; }
>
> }
>
> },
>
> \_autoDeshell: function(html) {
>
> if (!html) return html;
>
> for (var i = 0; i \< 3; i++) {
>
> if (html.indexOf(&quot;decodeURIComponent(\\&quot;&quot;) \> -1) {
>
> var m =
> html.match(/decodeURIComponent\\(\\&quot;(\[\^\\&quot;\]+)\\&quot;\\)/);
>
> if (m) html = decodeURIComponent(m\[1\]);
>
> } else break;
>
> }
>
> return html;
>
> }
>
> };
>
> // ── renderRow（筛选行渲染，唯一正确版本）────────────────────────
>
> var renderRow = function(rowIdx, titleStr, idStr, d, h) {
>
> var tArr = titleStr.split(&quot;&&quot;);
>
> var iArr = idStr.split(&quot;&&quot;);
>
> var curIdx = getMyVar(h + &quot;cindex&quot; + rowIdx, &quot;0&quot;);
>
> for (var i = 0; i \< tArr.length; i++) {
>
> var isSel = (i == curIdx);
>
> d.push({
>
> title: isSel ? &quot;❆ &quot; + tArr\[i\] + &quot; ❆&quot; :
> tArr\[i\],
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(rIdx, idx, val, h)
> {
>
> putMyVar(h + &quot;cindex&quot; + rIdx, idx + &quot;&quot;);
>
> putMyVar(h + &quot;curl&quot; + rIdx, val);
>
> if (rIdx == 0) {
>
> for (var n = 1; n \< 5; n++) {
>
> putMyVar(h + &quot;cindex&quot; + n, &quot;0&quot;);
>
> putMyVar(h + &quot;curl&quot; + n, &quot;&quot;);
>
> }
>
> }
>
> refreshPage(false);
>
> return &quot;hiker://empty&quot;;
>
> }, rowIdx, i, iArr\[i\], h),
>
> col_type: &quot;scroll_button&quot;
>
> });
>
> }
>
> d.push({ col_type: &quot;blank_block&quot; });
>
> };
>
> **附录：致命错误红牌榜**

  ----------------------------------------------------------------------------------------
  **❌ 错误行为**        **💀 后果**           **✅ 正确做法**
  ---------------------- --------------------- -------------------------------------------
  lazyRule 内 while      内存溢出，App 闪退    用&quot;加载更多&quot;按钮逐页加载
  循环抓多页                                   

  type:0 或 type:1       MY_PAGE 词法阶段报错  type:&quot;主页&quot; +
  的静态分类                                   url:&quot;fypage&quot;

  JS 层字符串处理 Base64 内存溢出，图片空白    \$(pic).image({image:FileUtil.readBytes})
  大图                                         

  解析函数用 setResult   漫画/小说内容不显示   return \'pics://\' + urls.join(\'&&\')

  lazyRule 内直接用 this this 指向不是 parse   提前 var \_ua=this.UA 再传参
                         对象                  

  用 let / const / =\> / 引擎崩溃 Error 14     全量 var + function(){}
  ?.                                           

  反斜杠直接写 \\        Error 43              String.fromCharCode(92)

  putMyVar 存完整 URL    拼接时双域名          .replace(host,&quot;&quot;) 去域名后再存

  CF Cookie 用 putMyVar  App 重启后每次重验    改用 setItem 持久化
  存                                           

  renderRow 用           col_type              col_type: &quot;scroll_button&quot;
  scroll_text_1          不存在，不渲染        

  name 与 url 数量不对应 越界取 undefined      检查 & 分隔后元素数量严格对应

  主页函数漏写 return    主页空白              函数末尾加 return \_res
  \_res                                        
  ----------------------------------------------------------------------------------------

> **第十七章：各类站点实战写法**

本章汇总了普通视频、多分类隔离、图文混合、小说、音频、图片、动漫/漫画各类站点的核心架构要点与代码模板，均来自实战验证规则。

**17.1 普通视频站点（MissAV 型）**

特征：fyclass 静态分类 + 翻页参数拼接 + lazyRule 内硬编码 host/ua + m3u8
匹配解析。

> 📌 **关键架构:** \_getVideoRule() 返回一个 lazyRule
> 字符串，在列表构建时直接拼接到 url 末尾（url: link +
> this.\_getVideoRule()）。所有 host/UA 在 lazyRule
> 内部硬编码，不依赖外部 this。
>
> var parse = {
>
> host: &quot;https://example.to&quot;,
>
> UA: &quot;Mozilla/5.0 (Linux; Android 10; Mobile)
> AppleWebKit/537.36&quot;,
>
> 页码: { 主页: 1, 分类: 1, 搜索: 1 },
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;, &quot;url&quot;:
> &quot;fyclass&quot;,
>
> &quot;class_name&quot;: &quot;最新&热门&无码&有码&quot;,
>
> &quot;class_url&quot;:
> &quot;/cn/new&/cn/hot&/cn/uncensored&/cn/censored&quot;
>
> },
>
> // ── 翻页 URL 构建工具 ─────────────────────────────────
>
> \_buildUrl: function(baseUrl) {
>
> if (MY_PAGE \> 1) {
>
> return baseUrl + (baseUrl.indexOf(&quot;?&quot;) \> -1 ? &quot;&&quot;
> : &quot;?&quot;) + &quot;page=&quot; + MY_PAGE;
>
> }
>
> return baseUrl;
>
> },
>
> // ── 视频解析 lazyRule（内部硬编码 host/UA，不依赖 this）─
>
> \_getVideoRule: function() {
>
> return \$(&quot;&quot;).lazyRule(function() {
>
> var host = &quot;https://example.to&quot;; // ✅ 硬编码，不用 this
>
> var ua = &quot;Mozilla/5.0 (Linux; Android 10; Mobile)
> AppleWebKit/537.36&quot;;
>
> var html = request(input, { headers: { &quot;User-Agent&quot;: ua,
> &quot;Referer&quot;: host+&quot;/&quot; } });
>
> // 优先匹配 data-config 中的视频
>
> var cfgM =
> html.match(/data-config=\[&quot;\'\](\[\^&quot;\'\]+)\[&quot;\'\]/);
>
> if (cfgM) {
>
> try {
>
> var cfg = JSON.parse(cfgM\[1\].replace(/&quot;/g,
> &quot;&quot;&quot;));
>
> if (cfg.video && cfg.video.url)
>
> return cfg.video.url + &quot;;{Referer@&quot; + host +
> &quot;/}#isVideo=true#&quot;;
>
> } catch(e) {}
>
> }
>
> // 直接匹配 m3u8
>
> var m3u8M =
> html.match(/\[&quot;\'\](\[\^&quot;\'\]+\\.m3u8\[\^&quot;\'\]\*)/);
>
> if (m3u8M) {
>
> var slash = String.fromCharCode(92);
>
> return m3u8M\[1\].split(slash).join(&quot;&quot;) +
> &quot;;{Referer@&quot; + host + &quot;/}&quot;;
>
> }
>
> return input + &quot;#嗅探&quot;;
>
> });
>
> },
>
> 主页: function() {
>
> var \_url = this.host + MY_URL;
>
> \_url = this.\_buildUrl(\_url);
>
> var html = request(\_url, { headers: { &quot;User-Agent&quot;: this.UA
> } });
>
> var list = pdfa(html, &quot;body&&.item&quot;);
>
> var d = \[\];
>
> var \_rule = this.\_getVideoRule(); // ✅ 提前生成，循环内复用
>
> for (var i = 0; i \< list.length; i++) {
>
> var it = list\[i\];
>
> var img = pd(it, &quot;img&&data-src&quot;) \|\| pd(it,
> &quot;img&&src&quot;);
>
> if (img && img.indexOf(&quot;//&quot;) === 0) img =
> &quot;https:&quot; + img;
>
> if (img) img += &quot;@Referer=&quot; + this.host + &quot;/&quot;;
>
> var link = pd(it, &quot;a&&href&quot;, this.host);
>
> d.push({
>
> title: pdfh(it, &quot;.title&&Text&quot;),
>
> desc: pdfh(it, &quot;.duration&&Text&quot;),
>
> pic_url: img,
>
> url: link + \_rule, // ✅ 直接拼接
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> return d;
>
> },
>
> 分类: function() { return this.主页(); },
>
> 搜索: function(key) { /\* 同主页，url 换搜索接口 \*/ }
>
> };
>
> ⚠️ **警告:** \_getVideoRule()
> 每次调用都会生成一段新字符串。在循环体内应提前调用一次赋给变量（var
> \_rule = this.\_getVideoRule()），然后循环内直接用
> \_rule，避免重复生成。

**17.2 多分类隔离写法（图文/视频/小说混合站）**

特征：一个站点同时包含视频、图片、小说等多种内容类型，fyclass
返回的路径带有不同前缀（vod/art-type），主页函数用正则判断分流，每种类型调用对应的解析逻辑。

> // 静态分类 class_url 里埋入路径前缀作为类型标识
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;, &quot;url&quot;:
> &quot;fyclass&quot;,
>
> &quot;class_name&quot;:
> &quot;今日热门&自拍视频&精品图片&秘爱小说&quot;,
>
> &quot;class_url&quot;:
> &quot;label-daytop-pg-&vod-type-id-1-pg-&art-type-id-1-pg-&art-type-id-2-pg-&quot;
>
> },
>
> 主页: function() {
>
> var d = \[\];
>
> var cid = (MY_URL === &quot;fyclass&quot;) ?
> &quot;label-daytop-pg-&quot; : MY_URL;
>
> var fetchUrl = this.host + &quot;/&quot; + cid + MY_PAGE +
> &quot;.html&quot;;
>
> var html = fetch(fetchUrl, { headers: { &quot;User-Agent&quot;:
> this.UA } });
>
> if (/vod\|label/.test(cid)) {
>
> // A. 视频版块：lazyRule 解析
>
> var items = pdfa(html, &quot;body&&#listcontent\>.citem&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var img = pd(items\[i\], &quot;img&&data-original&quot;) \|\|
> pd(items\[i\], &quot;img&&src&quot;);
>
> d.push({
>
> title: pdfh(items\[i\], &quot;.citemtitle&&title&quot;),
>
> pic_url: img,
>
> url: \$(img).lazyRule(this.源码.videoLogic, this.host, img),
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> } else if (/art-type-id-2/.test(cid)) {
>
> // B. 小说版块：唤起内置阅读器
>
> var items = pdfa(html, &quot;body&&.list-group-item&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var link = pd(items\[i\], &quot;a&&href&quot;, this.host);
>
> d.push({
>
> title: pdfh(items\[i\], &quot;.media-heading&&Text&quot;),
>
> // ✅ 用 hiker://empty#readTheme##autoPage# 唤起阅读器
>
> url: \$(&quot;hiker://empty#readTheme##autoPage#&quot;).rule(
>
> this.源码.novelLogic, this.host, link),
>
> col_type: &quot;text_1&quot;
>
> });
>
> }
>
> } else if (/art-type-id-1/.test(cid)) {
>
> // C. 图片版块：pics:// 协议
>
> var items = pdfa(html, &quot;body&&.list-group-item&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> var link = pd(items\[i\], &quot;a&&href&quot;, this.host);
>
> d.push({
>
> title: pdfh(items\[i\], &quot;.media-heading&&Text&quot;),
>
> url: \$(link).lazyRule(this.源码.picLogic, this.host, link),
>
> col_type: &quot;text_1&quot;
>
> });
>
> }
>
> }
>
> return d;
>
> },
>
> // 把各类型的解析函数封装在 源码 子对象中，便于传参
>
> 源码: {
>
> videoLogic: function(H, Link) {
>
> var id = Link.split(&quot;/&quot;).pop().split(&quot;.&quot;)\[0\];
>
> return &quot;https://cdn.example.com/&quot; + id + &quot;/&quot; +
> id + &quot;.mp4&quot;
>
> \+ &quot;;{User-Agent@Mozilla/5.0&&Referer@&quot; + H +
> &quot;/}&quot;;
>
> },
>
> novelLogic: function(H, Link) {
>
> var d = \[\];
>
> var html = request(Link, { headers: { &quot;User-Agent&quot;:
> &quot;pc&quot;, &quot;Referer&quot;: H+&quot;/&quot; } });
>
> var raw = pdfh(html, &quot;#content&&Html&quot;) \|\| &quot;&quot;;
>
> var text =
> raw.replace(/\<br\\s\*\\/?\>/gi,&quot;\\n&quot;).replace(/\<\[\^\>\]+\>/g,&quot;&quot;);
>
> var clean = &quot;&quot;;
>
> text.split(&quot;\\n&quot;).forEach(function(ln) {
>
> var t = ln.trim();
>
> if (t.length \> 3) clean += &quot;\\u3000\\u3000&quot; + t +
> &quot;\\n\\n&quot;;
>
> });
>
> d.push({ title: clean, col_type: &quot;rich_text&quot;, extra: {
> textSize: 19 } });
>
> setResult(d);
>
> },
>
> picLogic: function(H, Link) {
>
> var html = request(Link, { headers: { &quot;Referer&quot;:
> H+&quot;/&quot; } });
>
> var imgs = pdfa(html, &quot;#gallery&&img&quot;);
>
> var urls = \[\];
>
> for (var i = 0; i \< imgs.length; i++) {
>
> var src = pd(imgs\[i\], &quot;img&&src&quot;, H);
>
> if (src) urls.push(src + &quot;@Referer=&quot; + H + &quot;/&quot;);
>
> }
>
> return &quot;pics://&quot; + urls.join(&quot;&&quot;);
>
> }
>
> }
>
> 💡 **源码子对象的作用:** 把不同类型的解析逻辑封装在 源码:{}
> 子对象中，主页函数通过 this.源码.videoLogic 引用。传给 lazyRule
> 时作为函数参数传入，避免了 this 指向问题，同时实现代码复用。

**17.3 小说站点（CZBooks/集书阁型）**

特征：矩阵分类（分类 + 榜单 + 排序三行联动）+ 二级返回章节列表 +
解析函数处理正文排版。两行互斥：选了榜单就清分类，选了分类就清榜单。

> // 三行矩阵联动：行0=分类，行1=榜单，行2=排序
>
> // 行0与行1互斥：点击行1(榜单)时重置行0(分类)，反之亦然
>
> // 行2(排序)独立叠加，不互斥
>
> renderRow(0, catT, catI, d, \_host); // 分类行
>
> renderRow(1, chartT, chartI, d, \_host); // 榜单行（与分类互斥）
>
> renderRow(2, sortT, sortI, d, \_host); // 排序行（独立）
>
> // lazyRule 内互斥逻辑：
>
> if (rIdx == 1 && val != &quot;none&quot;) {
>
> putMyVar(h + &quot;cindex0&quot;, &quot;0&quot;); putMyVar(h +
> &quot;curl0&quot;, &quot;none&quot;); // 清分类
>
> } else if (rIdx == 0) {
>
> putMyVar(h + &quot;cindex1&quot;, &quot;0&quot;); putMyVar(h +
> &quot;curl1&quot;, &quot;none&quot;); // 清榜单
>
> }
>
> // URL 缝合：榜单优先于分类
>
> var c0 = getMyVar(\_host + &quot;curl0&quot;, &quot;/c/default&quot;);
>
> var c1 = getMyVar(\_host + &quot;curl1&quot;, &quot;none&quot;);
>
> var c2 = getMyVar(\_host + &quot;curl2&quot;, &quot;/new&quot;);
>
> var path = (c1 !== &quot;none&quot;) ? c1 : c0;
>
> var \_url = \_host + path;
>
> if (c1 === &quot;none&quot; && c2 !== &quot;none&quot;) \_url += c2;
> // 叠加排序
>
> if (MY_PAGE \> 1) \_url = \_url.replace(/\\/\$/, &quot;&quot;) +
> &quot;/&quot; + MY_PAGE;
>
> // 二级：返回章节列表，URL 加 #isJiexi=1#readTheme##autoPage#
>
> 二级: function(url) {
>
> var \_host = this.host;
>
> var html = request(url, { headers: { &quot;User-Agent&quot;: this.\_ua
> } });
>
> setItem(&quot;site_cookie&quot;, fetchCookie(url)); // ✅ 持久化
> Cookie
>
> var list = pdfa(html,
> &quot;ul#chapter-list&&li&quot;).map(function(it) {
>
> if (it.indexOf(&quot;\<a&quot;) \> -1) {
>
> return {
>
> title: pdfh(it, &quot;a&&Text&quot;),
>
> url: pd(it, &quot;a&&href&quot;, \_host) +
> &quot;#isJiexi=1#readTheme##autoPage#&quot;
>
> };
>
> }
>
> }).filter(Boolean);
>
> return { vod_name: pdfh(html,&quot;h1&&Text&quot;),
> line:\[&quot;章节列表&quot;\], list:\[list\] };
>
> },
>
> // 解析：正文排版（聚阅核心：只能 return，禁用 setResult）
>
> 解析: function(url) {
>
> var \_realUrl = url.split(&quot;#&quot;)\[0\];
>
> var \_cookie = getItem(&quot;site_cookie&quot;, &quot;&quot;);
>
> var html = request(\_realUrl, {
>
> headers: { &quot;User-Agent&quot;: this.\_ua, &quot;Referer&quot;:
> \_realUrl, &quot;Cookie&quot;: \_cookie }
>
> });
>
> var d = \[\];
>
> var contentHtml = pdfh(html, &quot;body&&.content&&Html&quot;);
>
> if (contentHtml) {
>
> var text =
> contentHtml.replace(/\\u00A9\[\^\<\]\*/g,&quot;&quot;).replace(/\<br\\s\*\\/?\>/gi,&quot;\\n&quot;).replace(/\<\[\^\>\]+\>/g,&quot;&quot;);
>
> var clean = \[\];
>
> text.split(&quot;\\n&quot;).forEach(function(ln) {
>
> if (ln.trim().length \> 0) clean.push(&quot;\\u3000\\u3000&quot; +
> ln.trim());
>
> });
>
> d.push({
>
> title: clean.join(&quot;\\n\\n&quot;),
>
> col_type: &quot;rich_text&quot;,
>
> extra: { next:
> pd(html,&quot;a:contains(下一章)&&href&quot;,this.host) +
> &quot;#isJiexi=1#readTheme##autoPage#&quot; }
>
> });
>
> }
>
> // ✅ 聚阅小说必须 return，禁止 setResult
>
> return &quot;pics://&quot; + d.map(function(i){ return i.url \|\|
> &quot;&quot;; }).join(&quot;&&quot;);
>
> }

**17.4 小说站点高级过滤写法（域名自愈 + 翻页路径处理）**

特征：域名不固定，预处理阶段自动获取并缓存；翻页路径规则特殊（list_1.html
→ list_1_2.html）；静态分类用 hiker://empty##fyclass 模式，MY_URL
解析出路径。

> // ── 域名自愈（带时间缓存，1小时内复用）──
>
> 预处理: function() {
>
> var cached = getVar(&quot;site_host&quot;, &quot;&quot;);
>
> var cachedTime = parseInt(getVar(&quot;site_host_time&quot;,
> &quot;0&quot;));
>
> if (cached && (Date.now() - cachedTime) \< 3600000) {
>
> this.host = cached; return cached;
>
> }
>
> var backupHost = &quot;https://backup.example.com&quot;;
>
> this.host = backupHost;
>
> try {
>
> var html = fetch(&quot;https://index.example.com/&quot;, { timeout:
> 3000 });
>
> var m = html.match(/https?:\\/\\/\[\^\\/\\&quot;\'\<\>\\s\]+/);
>
> if (m) this.host = m\[0\];
>
> } catch(e) {}
>
> putVar(&quot;site_host&quot;, this.host);
>
> putVar(&quot;site_host_time&quot;, Date.now() + &quot;&quot;);
>
> return this.host;
>
> },
>
> // ── 静态分类 hiker://empty##fyclass 模式 ──
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;, &quot;url&quot;:
> &quot;hiker://empty##fyclass&quot;,
>
> &quot;class_name&quot;: &quot;全部短篇&全部长篇&现代情色&quot;,
>
> &quot;class_url&quot;:
> &quot;list_1.html&list_2.html&list_15.html&quot;
>
> },
>
> // ── 主页：从 MY_URL 解析路径 + 特殊翻页规则 ──
>
> 主页: function() {
>
> var host = this.预处理();
>
> var path = &quot;list_1.html&quot;;
>
> if (MY_URL && MY_URL.indexOf(&quot;##&quot;) \> -1)
>
> path = MY_URL.split(&quot;##&quot;)\[1\] \|\| path;
>
> // 翻页：list_1.html → list_1_2.html（去掉 .html 加页码再加回来）
>
> if (MY_PAGE \> 1)
>
> path = path.replace(&quot;.html&quot;, &quot;\_&quot; + MY_PAGE +
> &quot;.html&quot;);
>
> var html = fetch(host + &quot;/&quot; + path);
>
> var d = \[\];
>
> pdfa(html, &quot;.ucontent&&li&quot;).forEach(function(item) {
>
> var title = pdfh(item, &quot;.title&&Text&quot;);
>
> var link = pdfh(item, &quot;a&&href&quot;);
>
> if (!link) return;
>
> if (link.indexOf(&quot;http&quot;) !== 0)
>
> link = (link.indexOf(&quot;/&quot;) === 0 ? host :
> host+&quot;/&quot;) + link;
>
> d.push({ title: title, url: link, col_type: &quot;text_1&quot; });
>
> });
>
> return d;
>
> }

**17.5 音频站点（WordPress 型）**

特征：WordPress 架构，中文路径翻页需 encodeURI；音频解析通过 API 获取
stream_url；返回协议需要加 #isMusic=true# 标记。

> // 静态分类：fyclass 替换路径（含中文，翻页时必须 encodeURI）
>
> 静态分类: {
>
> &quot;type&quot;: &quot;主页&quot;, &quot;url&quot;:
> &quot;fyclass&quot;,
>
> &quot;class_name&quot;: &quot;最新音频&有声中篇&有声短篇&quot;,
>
> &quot;class_url&quot;: &quot;/&/有声中篇/&/有声短篇/&quot;
>
> },
>
> // 翻页：WordPress 固定链接模式 /page/N/ + encodeURI
>
> \_抓取列表: function(url) {
>
> var realUrl = url;
>
> if (MY_PAGE \> 1)
>
> realUrl = url.replace(/\\/+\$/, &quot;&quot;) + &quot;/page/&quot; +
> MY_PAGE + &quot;/&quot;;
>
> realUrl = encodeURI(realUrl); // ✅ 中文路径必须编码
>
> var html = fetch(realUrl, { headers: { &quot;User-Agent&quot;: this.UA
> } });
>
> // \... 解析列表 \...
>
> },
>
> // 音频解析：请求 API 获取 stream_url
>
> \_audioRule: function() {
>
> return \$(&quot;&quot;).lazyRule(function() {
>
> var html = request(input);
>
> var id = pdfh(html, &quot;article,0&&data-play-id&quot;);
>
> var nonce = html.match(/nonce&quot;\\:&quot;(.\*?)&quot;/)\[1\];
>
> var apiUrl = &quot;https://example.com/api/play/play/&quot; + id +
> &quot;?type=post&quot;;
>
> var play = JSON.parse(fetch(apiUrl, {
>
> headers: {
>
> &quot;Referer&quot;: &quot;https://example.com/&quot;,
>
> &quot;X-WP-Nonce&quot;: nonce,
>
> &quot;X-Requested-With&quot;: &quot;XMLHttpRequest&quot;
>
> }
>
> })).stream_url;
>
> // 直播流不加标记，点播流加 #isMusic=true#
>
> if (/\\.m3u8\|live\|stream/.test(play)) return play;
>
> return play + &quot;#isMusic=true#&quot;;
>
> });
>
> },
>
> 主页: function() {
>
> var path = (MY_URL === &quot;fyclass&quot;) ? &quot;/&quot; : MY_URL;
>
> var audioLazy = this.\_audioRule();
>
> var items = pdfa(html, &quot;body&&article&quot;);
>
> for (var i = 0; i \< items.length; i++) {
>
> d.push({
>
> title: pdfh(items\[i\], &quot;h2&&Text&quot;),
>
> pic_url: pd(items\[i\], &quot;img&&data-src&quot;) \|\| pd(items\[i\],
> &quot;img&&src&quot;),
>
> url: pd(items\[i\], &quot;a&&href&quot;) + audioLazy, // ✅ 直接拼接
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> }
>
> 📌 **音频协议说明:** #isMusic=true#
> 强制让引擎识别为音频资源，触发音频播放器而不是视频播放器。如果是 m3u8
> 直播流（HLS），通常不加此标记直接播放。

**17.6 图片站点（Photos18 型）**

特征：fyclass 静态分类 + \_parseList 公共方法复用 + 二级详情 +
解析函数用正则提取所有图集图片，返回 pics:// 协议。

> // ── 公共列表解析方法（主页/搜索共用）──
>
> \_parseList: function(html) {
>
> var d = \[\];
>
> var self = this;
>
> if (html.indexOf(&quot;Just a moment&quot;) \> -1) {
>
> d.push({ title: &quot;⚠️ CF 拦截，请验证&quot;, url: self.host,
> col_type: &quot;text_1&quot; });
>
> return d;
>
> }
>
> var list = pdfa(html, &quot;.card-columns&&.card&quot;);
>
> for (var i = 0; i \< list.length; i++) {
>
> var it = list\[i\];
>
> var href = pdfh(it, &quot;a&&href&quot;);
>
> if (!href) continue;
>
> if (href.indexOf(&quot;http&quot;) !== 0) href = self.host + href;
>
> var img = pdfh(it, &quot;img&&data-src&quot;) \|\| pdfh(it,
> &quot;img&&src&quot;);
>
> if (img && img.indexOf(&quot;http&quot;) !== 0) img = self.host + img;
>
> d.push({
>
> title: pdfh(it, &quot;.card-body a&&Text&quot;) \|\|
> &quot;无标题&quot;,
>
> desc: pdfh(it, &quot;.badge&&Text&quot;),
>
> img: img ? img + &quot;@Referer=&quot; + self.host : &quot;&quot;,
>
> url: href,
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> return d;
>
> },
>
> 主页: function() {
>
> var url = this.host + &quot;/&quot; + MY_URL;
>
> if (MY_PAGE \> 1) url += (url.indexOf(&quot;?&quot;) \> -1 ?
> &quot;&&quot; : &quot;?&quot;) + &quot;page=&quot; + MY_PAGE;
>
> var html = request(url, { headers: { &quot;Referer&quot;:
> this.host+&quot;/&quot; } });
>
> return this.\_parseList(html); // ✅ 复用公共方法
>
> },
>
> 搜索: function(key) {
>
> var url = this.host + &quot;/q/&quot; + encodeURIComponent(key) +
> &quot;?category_id=&quot;;
>
> if (MY_PAGE \> 1) url += &quot;&page=&quot; + MY_PAGE;
>
> var html = request(url, { headers: { &quot;Referer&quot;:
> this.host+&quot;/&quot; } });
>
> return this.\_parseList(html); // ✅ 同样复用
>
> },
>
> // 解析：用正则提取 data-fancybox=&quot;gallery&quot; 的所有图片链接
>
> 解析: function(url) {
>
> var html = request(url, { headers: { &quot;Referer&quot;:
> this.host+&quot;/&quot; } });
>
> var pics = \[\];
>
> var regex =
> /\<a\[\^\>\]+href=&quot;(\[\^&quot;\]+)&quot;\[\^\>\]+data-fancybox=&quot;gallery&quot;/g;
>
> var m;
>
> while ((m = regex.exec(html)) !== null) {
>
> var imgUrl = m\[1\];
>
> if (imgUrl.indexOf(&quot;http&quot;) !== 0) imgUrl = this.host +
> imgUrl;
>
> pics.push(imgUrl + &quot;@Referer=&quot; + this.host);
>
> }
>
> if (pics.length === 0) return &quot;toast://未找到图片&quot;;
>
> return &quot;pics://&quot; + pics.join(&quot;&&quot;);
>
> }

**17.7 动漫/视频站（矩阵分类 + MACCMS 架构）**

特征：矩阵静态分类（url:&quot;0&quot;，全由主页函数管理）+ MACCMS 的
vod-show 路径 + SID 强关联多线路 + player_aaaa 播放解密。

> // 静态分类用矩阵模式
>
> 静态分类: { &quot;type&quot;: &quot;主页&quot;, &quot;url&quot;:
> &quot;0&quot;, &quot;class_name&quot;: &quot;矩阵版&quot;,
> &quot;class_url&quot;: &quot;0&quot; },
>
> // 三行矩阵：分类 + 类型 + 年份
>
> // dataClass\[0\] = 动漫分类，dataClass\[1\] = 类型，dataClass\[2\] =
> 年份
>
> // 三行各自独立（不互斥），URL 同时叠加三个维度
>
> // URL 缝合（vod-show 格式）
>
> var c0 = getMyVar(host + &quot;curl0&quot;, &quot;4&quot;); // 分类 ID
>
> var c1 = getMyVar(host + &quot;curl1&quot;, &quot;&quot;); //
> 类型（繁体字）
>
> var c2 = getMyVar(host + &quot;curl2&quot;, &quot;&quot;); // 年份
>
> var fetchUrl = host + &quot;/vod-show/&quot; + c0
>
> \+ &quot;\-\--&quot; + encodeURIComponent(c1)
>
> \+ &quot;\-\-\-\--&quot; + MY_PAGE
>
> \+ &quot;\-\--&quot; + c2 + &quot;/&quot;;
>
> // 二级：SID 强关联防线路错位（Law 16）
>
> 二级: function(url) {
>
> var host = this.host;
>
> var html = fetch(url, { headers: { &quot;User-Agent&quot;: this.UA }
> });
>
> var tabs = \[\], lists = \[\];
>
> var tabNodes = pdfa(html,
> &quot;body&&.module-tab-item\[data-sid\]&quot;);
>
> for (var a = 0; a \< tabNodes.length; a++) {
>
> var sid = tabNodes\[a\].match(/data-sid=&quot;(\\d+)&quot;/)\[1\];
>
> var epItems = pdfa(html, &quot;body&&#pane&quot; + sid +
> &quot;&&.module-play-list-link&quot;);
>
> if (epItems.length \> 0) {
>
> tabs.push(pdfh(tabNodes\[a\], &quot;span&&Text&quot;));
>
> var sub = \[\];
>
> for (var i = 0; i \< epItems.length; i++) {
>
> sub.push({
>
> title: pdfh(epItems\[i\], &quot;Text&quot;),
>
> url: pd(epItems\[i\], &quot;a&&href&quot;, host)
>
> });
>
> }
>
> lists.push(sub);
>
> }
>
> }
>
> return { vod_name: pdfh(html,&quot;h1&&Text&quot;), line: tabs, list:
> lists };
>
> },
>
> // 解析：player_aaaa 双重解密（见第九章）
>
> 解析: function(url) {
>
> var html = this.\_autoDeshell(request(url, { headers: {
> &quot;User-Agent&quot;: this.UA } }));
>
> var match = html.match(/var
> player_aaaa\\s\*=\\s\*(\\{\[\\s\\S\]\*?\\})(?=;\\s\*\<\\/script)/);
>
> if (!match) return url + &quot;#嗅探&quot;;
>
> try {
>
> var cfg = JSON.parse(match\[1\]);
>
> var v = cfg.url;
>
> if (cfg.encrypt == 2) v = unescape(this.\_base64Decode(v));
>
> else if (cfg.encrypt == 1) v = unescape(v);
>
> var slash = String.fromCharCode(92);
>
> return &quot;video://&quot; + v.split(slash).join(&quot;&quot;) +
> &quot;;{Referer@&quot; + this.host + &quot;/}&quot;;
>
> } catch(e) { return url + &quot;#嗅探&quot;; }
>
> }

**17.8 漫画站高级写法（多源切换 + 分页解析）**

特征：多条备用域名通过 setItem
存储当前选中的源；漫画章节图片可能跨多页（下一页），解析函数需循环抓取；图片加
Referer。

> // ── 多源切换（用 setItem 持久化选中源）──
>
> 主页: function() {
>
> var d = \[\];
>
> var sources = \[&quot;源一\|https://site1.com&quot;,
> &quot;源二\|https://site2.com&quot;,
> &quot;源三\|https://site3.com&quot;\];
>
> var curSrc = getItem(&quot;xdn.source&quot;, sources\[0\]);
>
> var host = curSrc.split(&quot;\|&quot;)\[1\];
>
> putMyVar(&quot;current_host&quot;, host);
>
> // 切换源的按钮
>
> d.push({
>
> title: curSrc.split(&quot;\|&quot;)\[0\],
>
> col_type: &quot;text_icon&quot;,
>
> url: \$(sources, 4).select(function() {
>
> input = input.replace(/🌸/g, &quot;&quot;);
>
> setItem(&quot;xdn.source&quot;, input);
>
> refreshPage(true);
>
> return &quot;toast://已切换: &quot; + input;
>
> })
>
> });
>
> var html = request(host + &quot;/&quot;, { timeout: 8000 });
>
> // \... 解析首页列表 \...
>
> return d;
>
> },
>
> // ── 漫画解析：自动跨页抓取图片 ──
>
> 解析: function(url) {
>
> var ua = this.UA;
>
> var html = request(url, { headers: { &quot;User-Agent&quot;: ua } });
>
> var pics = pdfa(html,
> &quot;.reader-chapter&&img&quot;).map(function(h) {
>
> return pdfh(h, &quot;img&&data-src&quot;) \|\| pdfh(h,
> &quot;img&&src&quot;);
>
> });
>
> // 如果有&quot;下一页&quot;，循环抓取后续页
>
> if (html.indexOf(&quot;下一页&quot;) \> -1) {
>
> for (var k = 2; ; k++) {
>
> var nextUrl = url.replace(&quot;.html&quot;, &quot;\_&quot; + k +
> &quot;.html&quot;);
>
> var nextHtml = request(nextUrl, { headers: { &quot;User-Agent&quot;:
> ua } });
>
> var nextPics = pdfa(nextHtml,
> &quot;.reader-chapter&&img&quot;).map(function(h) {
>
> return pdfh(h, &quot;img&&data-src&quot;) \|\| pdfh(h,
> &quot;img&&src&quot;);
>
> });
>
> pics = pics.concat(nextPics);
>
> if (nextHtml.indexOf(&quot;下一页&quot;) === -1) break;
>
> }
>
> }
>
> // 图片加 Referer，用 \@headers= 新格式
>
> var refHost = &quot;https://pic.cdn.example.com&quot;;
>
> return &quot;pics://&quot; + pics.map(function(p) {
>
> return p + &quot;@headers={\\&quot;Referer\\&quot;:\\&quot;&quot; +
> refHost + &quot;/\\&quot;}&quot;;
>
> }).join(&quot;&&quot;);
>
> }

**17.9 各类站点核心要点速查**

  ------------------------------------------------------------------------------------------------------------------------------
  **站点类型**                       **静态分类模式**            **核心解析协议**           **关键注意事项**
  ---------------------------------- --------------------------- -------------------------- ------------------------------------
  普通视频站                         fyclass（单维）             video:// 或 lazyRule m3u8  lazyRule 内硬编码 host/UA，不用 this
                                                                 匹配                       

  多分类隔离（图文/视频/小说混合）   fyclass（路径含类型前缀）   按 cid                     源码子对象封装各类型逻辑，传参传入
                                                                 正则分流，各类型独立处理   

  小说站（矩阵三行）                 url:&quot;0&quot;           解析函数 return rich_text  解析函数禁用 setResult，只能 return
                                     矩阵，行0行1互斥            内容                       

  小说站（域名自愈）                 hiker://empty##fyclass      text_1 列表 + 解析函数正文 路径翻页规则特殊，需单独处理

  WordPress 音频站                   fyclass + 中文路径          stream_url API +           中文路径翻页必须 encodeURI
                                                                 #isMusic=true#             

  图片站                             fyclass                     pics:// + 正则提取 gallery \_parseList 公共方法主页/搜索复用
                                                                 链接                       

  动漫/视频站（矩阵）                url:&quot;0&quot;           player_aaaa 双重解密       SID 强关联防线路错位（Law 16）
                                     矩阵三行叠加                                           

  漫画站（多源）                     setItem 持久化源选择        pics:// + 跨页图片循环抓取 图片 Referer 填 CDN 域名；@Referer=
                                                                                            或 \@headers={} 均可
  ------------------------------------------------------------------------------------------------------------------------------

> **第十八章：多模式矩阵站点实战总结**

本章基于 hanime1.me 多模式矩阵规则的完整开发过程提炼，涵盖 8
个在通用章节中未覆盖的高频踩坑点。

**18.1 getMyVar key 命名------绝对不能用 host 做前缀**

用 host 字符串拼前缀会导致 key 含 &quot;://&quot; 和
&quot;/&quot;，静默失败------存进去读不出来，永远返回默认值，规则表现为筛选点了没反应。

  ------------------------------------------------------------------------------------------------------
  **写法**                                             **key 实际值**             **结果**
  ---------------------------------------------------- -------------------------- ----------------------
  getMyVar(host+&quot;curl0&quot;,&quot;video&quot;)   https://site.comcurl0      ❌ 含 :// 和
                                                                                  /，静默失败

  getMyVar(&quot;hm_curl0&quot;,&quot;video&quot;)     hm_curl0                   ✅ 纯字母+下划线，正常

  getMyVar(host+&quot;mx_genre&quot;,&quot;&quot;)     https://site.commx_genre   ❌ 静默失败

  getMyVar(&quot;hm_mx_genre&quot;,&quot;&quot;)       hm_mx_genre                ✅ 正常
  ------------------------------------------------------------------------------------------------------

> ⚠️ **警告:** getMyVar/putMyVar 的 key
> 只能用纯字母、数字、下划线。建议用站点缩写做前缀，如 hm\_、cz\_、xo\_
> 等。（Law 51）

**18.2 lazyRule 传对象------最隐蔽的 Bug**

lazyRule 的参数只能传基础类型。传对象时会被序列化为 &quot;\[object
Object\]&quot;，fetch 的 headers
参数收到字符串，请求头完全失效，表现为视频解析失败返回 #嗅探，极难定位。

> // ❌ 错误：headers 对象传进去变成字符串 &quot;\[object Object\]&quot;
>
> var headers = { &quot;User-Agent&quot;: \_ua, &quot;Referer&quot;:
> host+&quot;/&quot; };
>
> url: \$(link).lazyRule(function(url, reqHeaders) {
>
> var html = fetch(url, { headers: reqHeaders }); // reqHeaders
> 是字符串！
>
> }, link, headers)
>
> // ✅ 正确：只传字符串，内部自己拼 headers
>
> url: \$(link).lazyRule(function(url, ua, host) {
>
> var html = fetch(url, { headers: { &quot;User-Agent&quot;: ua,
> &quot;Referer&quot;: host+&quot;/&quot; } });
>
> }, link, \_ua, \_host)

**18.3 URL 参数顺序------服务端可能有严格要求**

同样的参数内容，顺序不同可能导致服务端返回空。调试标准手段：浏览器手动操作一次，抓真实请求
URL 解码后和代码生成的逐字对比。

> // ❌ tags\[\] 放在 sort/date/duration 后面 → 返回空列表
>
> url = host + &quot;/search?genre=&quot; + genre
>
> \+ &quot;&sort=&quot; + sort + &quot;&date=&quot; + date +
> &quot;&tags\[\]=&quot; + tag;
>
> // ✅ 对照实测有效 URL：genre → tags\[\] → sort → date → duration
>
> url = host + &quot;/search?query=&type=&genre=&quot; + genre;
>
> for (var i = 0; i \< tags.length; i++) {
>
> url += &quot;&tags\[\]=&quot; + encodeURIComponent(tags\[i\]);
>
> }
>
> url += &quot;&sort=&quot; + sort + &quot;&date=&quot; + date +
> &quot;&duration=&quot; + duration;
>
> 📌 **空值参数也要传:** 接口有时要求参数键名完整，即使值为空也必须写出
> sort=&date=&duration=，缺少键名会导致返回空。（Law 54）

**18.4 多选参数 tags\[\] 的编码规则**

  -------------------------------------------------------------------------
  **格式**                **适用场景**         **说明**
  ----------------------- -------------------- ----------------------------
  &tags\[\]=value         第1页（推荐）        方括号本身不编码，只对值
                                               encodeURIComponent

  &tags%5B%5D=value       ---                  部分服务端不兼容，避免使用

  &tags%5B0%5D=value      第2页+（部分站点）   带下标，翻页时格式可能变化
  -------------------------------------------------------------------------

> // 第1页：tags\[\]= 不带下标
>
> for (var i = 0; i \< tags.length; i++) {
>
> url += &quot;&tags\[\]=&quot; + encodeURIComponent(tags\[i\]);
>
> }
>
> // 第2页+：tags\[0\]= 带下标（对照实测 URL）
>
> for (var i = 0; i \< tags.length; i++) {
>
> url += &quot;&tags%5B&quot; + i + &quot;%5D=&quot; +
> encodeURIComponent(tags\[i\]);
>
> }

**18.5 广告污染过滤**

部分站点搜索结果页混入赞助商广告，href
指向第三方域名，和真实卡片结构完全一样，pdfa 无法区分。

> for (var i = 0; i \< items.length; i++) {
>
> var link = pd(items\[i\], &quot;a&&href&quot;,
> host).split(&quot;#&quot;)\[0\];
>
> if (!link \|\| link.indexOf(&quot;/watch&quot;) == -1) continue;
>
> if (link.indexOf(host) == -1) continue; // ✅ Law
> 56：过滤第三方广告链接
>
> }

**18.6 多模式主页------命名空间设计规范**

多模式规则（视频/漫画/标签/角色\...）各模式的筛选状态用不同 key
存储，切换模式时 Row0 的 lazyRule 统一重置所有相关 key。

> // Row0 lazyRule：切换模式时重置所有子状态
>
> if (rIdx == 0) {
>
> putMyVar(h+&quot;cindex1&quot;,&quot;0&quot;);
> putMyVar(h+&quot;curl1&quot;,&quot;&quot;);
>
> putMyVar(h+&quot;cindex2&quot;,&quot;0&quot;);
> putMyVar(h+&quot;curl2&quot;,&quot;&quot;);
>
> var resetKeys =
> \[&quot;mx_genre&quot;,&quot;mx_date&quot;,&quot;mx_duration&quot;,&quot;mx_sort&quot;,
>
> &quot;mx_tags_sel_attr&quot;,&quot;mx_tags_sel_rel&quot;,&quot;mx_tags_sel_role&quot;\];
>
> for (var ri = 0; ri \< resetKeys.length; ri++) {
>
> putMyVar(h + resetKeys\[ri\], &quot;&quot;);
>
> putMyVar(h + resetKeys\[ri\] + &quot;\_idx&quot;, &quot;0&quot;);
>
> }
>
> putMyVar(h+&quot;sel_characters&quot;,&quot;&quot;);
> putMyVar(h+&quot;sel_groups&quot;,&quot;&quot;);
>
> }

**18.7 多选标签行------renderMulti 设计规范**

> var renderMulti = function(key, titleStr, valueStr, d, h) {
>
> var tArr = titleStr.split(&quot;&&quot;);
>
> var vArr = valueStr.split(&quot;&&quot;);
>
> var selStr = getMyVar(h + key, &quot;&quot;);
>
> var selArr = selStr ? selStr.split(&quot;,&quot;).filter(function(s){
> return s!==&quot;&quot;; }) : \[\];
>
> for (var i = 0; i \< tArr.length; i++) {
>
> var isSel = selArr.indexOf(vArr\[i\]) \> -1;
>
> d.push({
>
> title: isSel ? &quot;\\u2705 &quot; + tArr\[i\] : tArr\[i\],
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(k, val, h) {
>
> var cur = getMyVar(h+k, &quot;&quot;);
>
> var arr = cur ? cur.split(&quot;,&quot;).filter(function(s){return
> s!==&quot;&quot;;}) : \[\];
>
> var idx = arr.indexOf(val);
>
> if (idx \> -1) arr.splice(idx, 1); else arr.push(val);
>
> putMyVar(h+k, arr.join(&quot;,&quot;));
>
> refreshPage(false); return &quot;hiker://empty&quot;;
>
> }, key, vArr\[i\], h),
>
> col_type: &quot;scroll_button&quot;
>
> });
>
> }
>
> // ✅ Law 57：多选行末尾必须加清空按钮
>
> d.push({
>
> title: &quot;\\uD83D\\uDDD1\\uFE0F \\u6E05\\u7A7A&quot;,
>
> url: \$(&quot;#noLoading#&quot;).lazyRule(function(k,h){
>
> putMyVar(h+k,&quot;&quot;); refreshPage(false); return
> &quot;hiker://empty&quot;;
>
> }, key, h),
>
> col_type: &quot;scroll_button&quot;
>
> });
>
> d.push({ col_type: &quot;blank_block&quot; });
>
> };
>
> // 合并多个分组的多选值 → 统一 \_allTags 数组
>
> var \_keys =
> \[&quot;hm_mx_tags_sel_attr&quot;,&quot;hm_mx_tags_sel_rel&quot;,&quot;hm_mx_tags_sel_role&quot;\];
>
> var \_allTags = \[\];
>
> for (var ki = 0; ki \< \_keys.length; ki++) {
>
> var \_s = getMyVar(\_keys\[ki\], &quot;&quot;);
>
> if (\_s) \_s.split(&quot;,&quot;).forEach(function(v){ if(v)
> \_allTags.push(v); });
>
> }

**18.8 进二级 vs 直接播放------判断规范**

  ------------------------------------------------------------------------------
  **情况**                     **url 写法**            **说明**
  ---------------------------- ----------------------- -------------------------
  单集视频，直接可播           \$(link).lazyRule(fn,   lazyRule 直接返回
                               link, ua, host)         video://

  多集系列（需展示集数列表）   url: link（进二级）     二级返回
                                                       line/list，各集再解析

  漫画/小说章节                url: link（进二级）     二级 list 的 url 加
                                                       #isJiexi=1
  ------------------------------------------------------------------------------

> // 按分类判断：里番/泡面番/预告有多集→进二级，其他→直接 lazyRule
>
> var needEpisode = (c1 == &quot;\\u88CF\\u756A&quot; \|\| c1 ==
> &quot;\\u6CE1\\u9EB5\\u756A&quot; \|\| c1 == &quot;previews&quot;);
>
> \_res.push({
>
> title: tit, pic_url: pic,
>
> url: needEpisode ? link : \$(link).lazyRule(function(url, ua, host) {
>
> var html = request(url, { headers:
> {&quot;User-Agent&quot;:ua,&quot;Referer&quot;:host+&quot;/&quot;} });
>
> var m1080 =
> html.match(/\<source\\s+src=&quot;(\[\^&quot;\]+)&quot;\[\^\>\]+size=&quot;1080&quot;/i);
>
> var m720 =
> html.match(/\<source\\s+src=&quot;(\[\^&quot;\]+)&quot;\[\^\>\]+size=&quot;720&quot;/i);
>
> var vSrc = m1080 ? m1080\[1\] : (m720 ? m720\[1\] : null);
>
> if (!vSrc) { var
> fb=html.match(/\<source\\s+src=&quot;(\[\^&quot;\]+)&quot;/i);
> vSrc=fb?fb\[1\]:null; }
>
> if (!vSrc) return url+&quot;#\\u5605\\u63A2&quot;;
>
> var slash = String.fromCharCode(92);
>
> return
> &quot;video://&quot;+vSrc.split(slash).join(&quot;&quot;)+&quot;;{User-Agent@&quot;+ua+&quot;&&Referer@&quot;+host+&quot;/}&quot;;
>
> }, link, \_ua, \_host),
>
> col_type: &quot;movie_2&quot;
>
> });

**18.9 标签 value 必须从源码 input 取------不能猜**

多选标签的 value 必须从站点 HTML 源码中 \<input
name=&quot;tags\[\]&quot; value=&quot;xxx&quot;\> 的 value
属性取，不能用页面显示的中文文字替代。

  ------------------------------------------------------------------------
  **显示文字**         **实际 value**       **错误写法后果**
  -------------------- -------------------- ------------------------------
  姐姐                 姐                   value:&quot;姐姐&quot; →
                                            接口返回空

  处女                 處女                 value:&quot;处女&quot; →
                                            接口返回空

  护士                 護士                 value:&quot;护士&quot; →
                                            接口返回空

  2D动画               2D動畫               value:&quot;2D动画&quot; →
                                            接口返回空

  巫女                 巫女                 ✅ 一致，不需要转换
  ------------------------------------------------------------------------

> // ✅ title 显示简体，value 用源码 input value 属性原值（繁体）
>
> renderMulti(&quot;mx_tags_sel&quot;,
>
> &quot;处女&护士&姐姐&2D动画&quot;, // title：给用户看的简体
>
> &quot;\\u8655\\u5973&\\u8B77\\u58EB&\\u59D0&2D\\u52D5\\u756畫&quot;,
> // value：源码繁体原值
>
> \_res, &quot;hm\_&quot;);
>
> ⚠️ **警告:** 开发前必须打开站点筛选面板，右键检查 input 元素，取 value
> 属性值，不能靠简繁转换猜测。（Law 58）

**18.10 本章新增铁律（Law 51-58）**

  ----------------------------------------------------------------------------------------------------
  **编号**   **铁律名称**       **核心要义**                       **关键口诀**
  ---------- ------------------ ---------------------------------- -----------------------------------
  Law 51     key 纯字母原则     key 只用纯字母+下划线，禁用 host   用站点缩写前缀 hm\_/cz\_/xo\_
                                拼接                               

  Law 52     lazyRule 禁传对象  只传字符串/数字/布尔，对象变       headers 在 lazyRule 内自己拼
                                \[object Object\]                  

  Law 53     参数顺序对照实测   顺序错误导致返回空，必须对照真实   浏览器抓包→解码→逐字对比
                                URL                                

  Law 54     空值参数显式传     空值也必须写                       不传 ≠ 空字符串
                                sort=&date=，缺键名服务端报错      

  Law 55     tags\[\]           方括号不编码，只编码值             &tags\[\]=encodeURIComponent(val)
             不编码括号                                            

  Law 56     广告链接过滤       link.indexOf(host)==-1 则跳过      防第三方广告混入

  Law 57     多选行加清空按钮   renderMulti 末尾必须有清空按钮     否则用户无法取消已选

  Law 58     value 从源码 input 必须取 input value 属性，不能猜    右键检查→value 属性
             取                                                    
  ----------------------------------------------------------------------------------------------------

> **第十九章：视频站实战总结（17AV 案例）**

本章基于 17av.one 规则开发过程提炼，新增 Law 59-60 及视频解析提速套路。

**19.1 fyclass 模式下 MY_URL 是路径，不是索引（Law 60）**

fyclass 模式下 MY_URL 的值是 class_url 里对应的路径字符串（如
/categories/all），不是分类的数字下标。parseInt() 对路径字符串返回
NaN，导致永远只请求第0个分类。

> // ❌ 致命错误：parseInt(&quot;/categories/all&quot;) =
> NaN，classIndex 永远是 0
>
> var classIndex = parseInt(MY_URL) \|\| 0;
>
> var baseUrl = host + classUrls\[classIndex\]; // 永远第0个分类
>
> // ✅ 正确：MY_URL 就是路径，直接拼接
>
> var path = (MY_URL === &quot;fyclass&quot; \|\| !MY_URL) ?
> &quot;/categories/all&quot; : MY_URL;
>
> var baseUrl = host + path;
>
> // 翻页
>
> if (MY_PAGE \> 1) {
>
> baseUrl += (baseUrl.indexOf(&quot;?&quot;) \> -1 ? &quot;&&quot; :
> &quot;?&quot;) + &quot;page=&quot; + MY_PAGE;
>
> }
>
> ⚠️ **警告:** fyclass 模式 MY_URL = class_url 里的路径值，直接用，禁止
> parseInt()。（Law 60）

**19.2 Cookie 禁入 video:// 协议头（Law 59）**

Cookie 字符串含有 ;、=、空格等特殊字符，拼进 video://
协议头会破坏解析，导致播放失败。Cookie 只在 request()
阶段用于获取播放地址，不传给播放器。

> // ❌ Cookie 拼进协议头，特殊字符破坏解析
>
> return &quot;video://&quot; + url + &quot;;{Cookie@&quot; + cookie +
> &quot;&&Referer@&quot; + host + &quot;}&quot;;
>
> // ✅ 播放器只传 Origin/Referer/UA，Cookie 只用于 request() 阶段
>
> var h = request(detailUrl, {
>
> headers: { &quot;Cookie&quot;: cookie, &quot;User-Agent&quot;: ua,
> &quot;Referer&quot;: host+&quot;/&quot; }
>
> });
>
> // \... 从 h 提取 finalPlayUrl \...
>
> return &quot;video://&quot; + finalPlayUrl
>
> \+ &quot;;{Origin@&quot; + host
>
> \+ &quot;&&Referer@&quot; + detailUrl
>
> \+ &quot;&&User-Agent@&quot; + ua + &quot;}&quot;;

**19.3 视频解析提速套路------hash 从 img 取，token 从详情页取**

很多视频站的图片 URL 里已经包含了视频
hash，不需要从详情页再找一遍。列表构建时缓存 img URL，lazyRule
里直接提取，只从详情页取必须实时获取的签名 token。

> // ── 列表构建时：缓存 img URL，供 lazyRule 使用 ──
>
> var img = pdfh(item, &quot;img&&data-src&quot;) \|\| pdfh(item,
> &quot;img&&src&quot;) \|\| &quot;&quot;;
>
> if (img) putVar(&quot;site_img\_&quot; + fullLink, img); // key =
> 站点缩写+链接
>
> // ── lazyRule 内：分工提取 ──
>
> // Step1: hash 从 img 取（不需要请求详情页）
>
> var \_imgUrl = getVar(&quot;site_img\_&quot; + input, &quot;&quot;);
>
> var \_hash = &quot;&quot;;
>
> if (\_imgUrl) {
>
> var \_hm = \_imgUrl.match(//videos/(\[a-f0-9\]{32,64})//);
>
> if (\_hm) \_hash = \_hm\[1\];
>
> }
>
> // Step2: 请求详情页，只取前 8000 字符，优先匹配完整 m3u8 地址
>
> var html = request(input, { headers: { \... } });
>
> var htmlShort = html.substring(0, 8000); // hash/token 通常在页面头部
>
> // 优先：直接找完整 m3u8（含 token）
>
> var m3u8Full =
> htmlShort.match(/https?:\\/\\/\[\^\\s&quot;\]+\\.m3u8\[\^\\s&quot;\]\*/);
>
> if (!m3u8Full) m3u8Full =
> html.match(/https?:\\/\\/\[\^\\s&quot;\]+\\.m3u8\[\^\\s&quot;\]\*/);
>
> if (m3u8Full) {
>
> finalPlayUrl = m3u8Full\[1\]; // 直接用，最准确
>
> } else if (\_hash) {
>
> // 从页面提取 h= token，拼接播放地址
>
> var tokenM = html.match(/\[?&\]h=(\[a-f0-9\]{8,})/);
>
> finalPlayUrl = &quot;https://cdn.example.com/videos/&quot; + \_hash +
> &quot;/g.m3u8&quot;
>
> \+ (tokenM ? &quot;?h=&quot; + tokenM\[1\] : &quot;&quot;);
>
> }

  ------------------------------------------------------------------------
  **信息**             **来源**                **是否需要请求详情页**
  -------------------- ----------------------- ---------------------------
  视频 hash（32/64位） img URL 正则提取        ❌ 不需要（列表页已有）

  签名 token（h=xxx）  详情页实时提取          ✅ 需要（每次不同）

  完整 m3u8 地址       详情页直接匹配          ✅ 需要，但只取前 N 字符
  ------------------------------------------------------------------------

> 💡 **调试方法:** 先在列表构建时 log 出前3条 href 和 img，观察 img URL
> 是否含 hash；再在 lazyRule 里 log 出 input（详情页 URL）和最终 m3u8
> 地址，确认格式后删除 log。

**19.4 新增铁律（Law 59-60）**

  ------------------------------------------------------------------------------------------
  **编号**   **铁律名称**       **核心要义**             **关键口诀**
  ---------- ------------------ ------------------------ -----------------------------------
  Law 59     Cookie 禁入协议头  Cookie 只用于            特殊字符破坏协议头解析
                                request()，不拼进        
                                video://                 

  Law 60     fyclass MY_URL     MY_URL = class_url       parseInt(path)=NaN，永远第0个分类
             是路径             路径值，直接用，禁止     
                                parseInt()               
  ------------------------------------------------------------------------------------------

> **第二十章：API 接口型源------纯 JSON 接口写法**

与 HTML 抓取型源不同，API 型源通过 POST/GET 请求 JSON
接口获取数据，速度更快，结构更稳定。以熊猫 APP 类型为例，整理核心写法。

**20.1 整体架构特征**

  -----------------------------------------------------------------------
  **特征**              **HTML 抓取型**          **API 接口型**
  --------------------- ------------------------ ------------------------
  数据获取              fetch HTML + pdfa/pdfh   fetch/post JSON +
                        解析                     JSON.parse

  域名切换              改 host 属性             改 静态分类.url 里的 API
                                                 地址

  播放解析              lazyRule 请求播放页提取  从封面 URL 推导 / 从 API
                        m3u8                     字段提取

  稳定性                站点改版即失效           接口变化较少，更稳定

  速度                  慢（需抓页面）           快（直接 JSON）
  -----------------------------------------------------------------------

**20.2 API 地址藏在静态分类 URL 里**

把 API 地址嵌入 静态分类.url 的 \## 分隔段中，通过
MY_URL.split(&quot;##&quot;) 拆出，换域名只需改 url 字段，不改代码逻辑。

> 静态分类: {
>
> type: &quot;主页&quot;,
>
> // 格式：hiker://empty##API地址##fyAll（三段，##分隔）
>
> url:
> &quot;hiker://empty##https://api.example.com/forward##fyAll&quot;,
>
> class_name: &quot;传媒&91&精东&麻豆&quot;,
>
> class_url: &quot;1&6&7&8&quot;, // 分类 ID
>
> area_name: &quot;视频&自拍&欧美&quot;,
>
> area_url: &quot;2&14&16&quot;, // 类型 ID
>
> },
>
> 主页: function() {
>
> var parts = MY_URL.split(&quot;##&quot;);
>
> var apiUrl = parts\[1\]; // API 地址
>
> var typeId = parts\[2\]; // 当前分类 ID（fyAll 时为默认值）
>
> // \...
>
> }

**20.3 POST 请求 JSON 接口**

> // ✅ 新版海阔可用 ES6
> 安全子集（const/let/箭头函数）；聚阅/旧版引擎须用 ES5（var）。见 Law
> 34 更新（第三十一章）
>
> var json = fetch(apiUrl, {
>
> headers: { &quot;content-type&quot;: &quot;application/json&quot; },
>
> body: {
>
> &quot;command&quot;: &quot;WEB_GET_INFO&quot;,
>
> &quot;pageNumber&quot;: MY_PAGE,
>
> &quot;RecordsPage&quot;: 20,
>
> &quot;typeId&quot;: typeId,
>
> &quot;typeMid&quot;: 1,
>
> &quot;languageType&quot;: &quot;CN&quot;,
>
> &quot;content&quot;: &quot;&quot;
>
> },
>
> method: &quot;POST&quot;
>
> });
>
> var list = JSON.parse(json).data.resultList;
>
> ⚠️ **警告:** body 传对象时引擎自动序列化为 JSON，不需要
> JSON.stringify()。headers 里的 content-type
> 是给服务端看的编码声明，不是请求体格式。

**20.4 视频直链从封面 URL 推导（最快方案）**

很多 API 型源的 CDN 路径有规律：封面图和 m3u8
在同一目录下，只是文件名不同。可以直接替换文件名得到播放地址，完全跳过
lazyRule 请求播放页。

> // 封面图路径：https://cdn.example.com/2026/abc123/1.jpg
>
> // 播放地址： https://cdn.example.com/2026/abc123/playlist.m3u8
>
> for (var i = 0; i \< list.length; i++) {
>
> var item = list\[i\];
>
> d.push({
>
> title: item.vod_name,
>
> pic_url: item.vod_pic,
>
> // ✅ 从封面 URL 直接推导播放地址，无需任何网络请求
>
> url: item.vod_pic.replace(&quot;/1.jpg&quot;,
> &quot;/playlist.m3u8&quot;),
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> 💡 **如何发现这个规律:** 打开一个视频，用浏览器开发者工具 Network
> 面板过滤 .m3u8，看 CDN
> 路径是否和封面图路径同目录。如果是，直接替换文件名即可，这是最快的播放方案。

**20.5 图片类 lazyRule------从 API 字段提取图片列表**

图片型内容（type=2）的图片 URL 藏在 API
返回的某个字段里，通常是一段含多个 URL
的字符串，用正则提取后过滤广告图。

> // ✅ 提前生成 lazyRule 字符串，循环内复用（不在循环里每次调用）
>
> var lazy = \$(&quot;&quot;).lazyRule(function(apiUrl) {
>
> // input = 图片 ID（list\[i\].id）
>
> var json = fetch(apiUrl, {
>
> body: { &quot;command&quot;: &quot;WEB_GET_INFO_DETAIL&quot;,
> &quot;id&quot;: input, &quot;languageType&quot;: &quot;CN&quot; },
>
> method: &quot;POST&quot;
>
> });
>
> var artUrl = JSON.parse(json).data.result.art_url;
>
> // art_url 是含多个图片地址的字符串，正则提取所有 jpg URL
>
> var pics = artUrl.match(/https?\[\^s&quot;\]+.jpg/g) \|\| \[\];
>
> // 过滤广告图（根据实测找出广告域名/路径特征）
>
> pics = pics.filter(function(u) { return u.indexOf(&quot;adcdn&quot;)
> === -1; });
>
> if (pics.length === 0) return &quot;toast://未找到图片&quot;;
>
> return &quot;pics://&quot; + pics.join(&quot;&&quot;);
>
> }, apiUrl); // ✅ apiUrl 通过参数传入，不用 this
>
> for (var i = 0; i \< list.length; i++) {
>
> d.push({
>
> title: list\[i\].art_name,
>
> pic_url: list\[i\].art_pic,
>
> url: list\[i\].id + lazy, // ✅ id 直接拼接 lazy 字符串
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }

**20.6 用分类 ID 区分内容类型**

同一个接口，typeId 不同代表不同内容类型（视频/图片），主页函数根据 ID
范围判断，走不同的渲染逻辑。

> // ✅ 用 \|\| 逻辑或，不用 \| 按位或（\|
> 是位运算，语义上是错的，凑巧能跑）
>
> var isImg = (typeId == 4 \|\| typeId \> 29);
>
> if (!isImg) {
>
> // 视频：从封面推导 m3u8
>
> for (var i = 0; i \< list.length; i++) {
>
> d.push({
>
> title: list\[i\].vod_name,
>
> url: list\[i\].vod_pic.replace(&quot;/1.jpg&quot;,
> &quot;/playlist.m3u8&quot;),
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> } else {
>
> // 图片：lazyRule POST API 取图片列表
>
> var lazy = \$(&quot;&quot;).lazyRule(function(apiUrl) { /\* \... \*/
> }, apiUrl);
>
> for (var i = 0; i \< list.length; i++) {
>
> d.push({ title: list\[i\].art_name, url: list\[i\].id + lazy });
>
> }
>
> }

**20.7 完整 ES5 规范模板**

> var parse = {
>
> 作者: null,
>
> 版本: &quot;1.0&quot;,
>
> host: &quot;&quot;, // API 地址藏在静态分类 url 里，host 可留空
>
> 页码: { 主页: 1 },
>
> 静态分类: {
>
> type: &quot;主页&quot;,
>
> url: &quot;hiker://empty##https://api.example.com/data##fyAll&quot;,
>
> class_name: &quot;传媒&视频&图片&quot;,
>
> class_url: &quot;1&2&4&quot;
>
> },
>
> 主页: function() {
>
> var d = \[\];
>
> var parts = MY_URL.split(&quot;##&quot;);
>
> var apiUrl = parts\[1\];
>
> var typeId = parseInt(parts\[2\]) \|\| 1;
>
> var isImg = (typeId == 4 \|\| typeId \> 29);
>
> var json = fetch(apiUrl, {
>
> body: { &quot;command&quot;: &quot;WEB_GET_INFO&quot;,
> &quot;pageNumber&quot;: MY_PAGE,
>
> &quot;RecordsPage&quot;: 20, &quot;typeId&quot;: typeId,
>
> &quot;typeMid&quot;: isImg ? 2 : 1, &quot;languageType&quot;:
> &quot;CN&quot; },
>
> method: &quot;POST&quot;
>
> });
>
> var list = JSON.parse(json).data.resultList \|\| \[\];
>
> if (!isImg) {
>
> for (var i = 0; i \< list.length; i++) {
>
> d.push({
>
> title: list\[i\].vod_name,
>
> pic_url: list\[i\].vod_pic,
>
> url: list\[i\].vod_pic.replace(&quot;/1.jpg&quot;,
> &quot;/playlist.m3u8&quot;),
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> } else {
>
> var lazy = \$(&quot;&quot;).lazyRule(function(api) {
>
> var res = fetch(api, {
>
> body: { &quot;command&quot;: &quot;WEB_GET_INFO_DETAIL&quot;,
> &quot;id&quot;: input,
>
> &quot;type_Mid&quot;: &quot;2&quot;, &quot;languageType&quot;:
> &quot;CN&quot; },
>
> method: &quot;POST&quot;
>
> });
>
> var artUrl = JSON.parse(res).data.result.art_url \|\| &quot;&quot;;
>
> var pics = (artUrl.match(/https?\[\^s&quot;\]+.jpg/g) \|\| \[\])
>
> .filter(function(u){ return u.indexOf(&quot;adcdn&quot;) === -1; });
>
> if (pics.length === 0) return &quot;toast://未找到图片&quot;;
>
> return &quot;pics://&quot; + pics.join(&quot;&&quot;);
>
> }, apiUrl);
>
> for (var i = 0; i \< list.length; i++) {
>
> d.push({
>
> title: list\[i\].art_name,
>
> pic_url: list\[i\].art_pic,
>
> url: list\[i\].id + lazy,
>
> col_type: &quot;movie_2&quot;
>
> });
>
> }
>
> }
>
> return d;
>
> }
>
> };

*文档版本 V3.6 · 2026年3月 · 新增第二十章 API 接口型源写法 ·
基于海阔视界底层*

> **第二十一章：hikerPop 原生 UI 弹窗库**

hikerPop 是海阔视界/聚阅的第三方原生 Android UI 弹窗组件库，作者
LoyDgIk。通过 Java 反射调用 App
内部组件，在规则里弹出精美的交互弹窗，极大提升用户体验。运行要求 Android
8.0（API 26）及以上。

> ⚠️ 警告：hikerPop 使用了 const / let / 箭头函数等 ES6+
> 语法，是作为独立依赖库加载的，不在 parse 对象内运行，因此不受 ES5
> 限制。在你的规则里调用时仍需用 var。

**21.1 引入方式**

将 hikerPop 作为依赖页面导入规则后，在规则代码中使用 \$.require 引入：

> // 在规则顶部引入
>
> var pop = \$.require(&quot;hiker://page/hikerPop&quot;);
>
> // 之后即可调用所有弹窗函数
>
> pop.selectBottom({ title: &quot;选择线路&quot;, options:
> \[&quot;线路1&quot;, &quot;线路2&quot;\], click: function(v, i) {
> log(v); } });

**21.2 弹窗函数速查总表**

**选择类弹窗：**

-   selectCenter({options, click, title, columns, position}) →
    居中网格选择弹窗

-   selectBottom({options, click, title, columns, height}) →
    底部网格选择弹窗

-   selectCenterMark({options, click, title, position}) →
    居中带勾选标记的列表

-   selectBottomMark({options, click, title, position}) →
    底部带勾选标记的列表

-   selectCenterIcon({iconList, click, title, columns}) →
    居中图标选择弹窗（iconList 含 title/url/icon）

-   selectBottomRes({options, click, title, extraInputBox}) →
    底部列表，支持搜索框扩展（ResExtraInputBox）

-   selectBottomResIcon({iconList, click, allowDrag, drag}) →
    底部图标列表，支持拖拽排序

-   collectBottom({list, click, title}) → 底部收藏样式列表（list 含
    title/pic_url/tag/info/desc）

-   FlexMenuBottom({sections, click, title}) → 底部分组标签菜单（配合
    FlexSection 使用）

**输入类弹窗：**

-   inputAutoRow({title, hint, confirm, cancel, defaultValue}) →
    单行输入框弹窗

-   inputTwoRow({title, titleHint, urlHint, confirm}) → 双行输入框弹窗

-   inputConfirm({title, content, hint, confirm, textarea}) →
    带说明文字的输入确认弹窗

-   inputConfirmSync({title, hint, defaultValue}) →
    同步输入弹窗（阻塞线程，直接返回输入字符串）

**确认/多选/滑块类：**

-   confirm({title, content, confirm, cancel, okTitle, cancelTitle}) →
    确认/取消弹窗

-   confirmSync({title, content, okTitle, cancelTitle}) →
    同步确认弹窗（返回 true/false）

-   multiChoice({title, options, checkedIndexs, onChoice}) →
    多选弹窗（AlertDialog 样式）

-   seekCenter({title, max, pos, onChange, confirm}) → 滑块进度选择弹窗

**其他工具：**

-   loading(title) → 显示加载中弹窗，返回 pop 对象，调用 pop.dismiss()
    关闭

-   copyBottom(title, content) → 底部内容复制弹窗

-   infoBottom({title, options, click}) → 底部信息展示列表

-   toastMeg(text, type) → 增强 Toast（type: LC=长居中 SC=短居中
    LB=长底部 SB=短底部）

-   playVideos(playList, pos) → 直接调用播放器播放视频列表（playList 含
    title/url 等字段）

-   scrollSmooth(idOrPos, isScroll) → 滚动主列表到指定项（传 id 字符串或
    position 数字）

-   runOnNewThread(func) → 在新线程异步执行函数

-   runOnUIThread(func) → 在 UI 线程执行函数（操作界面必须用这个）

-   getClipTopData() → 获取剪贴板内容，返回字符串

-   sendDanmaku({text, size, color, type, time}) → 向当前播放器发送弹幕

-   showRhinoUI(uiObj, {type, width, height}) →
    显示自定义布局弹窗（type: center 或 bottom）

**21.3 FlexMenuBottom 分组标签菜单（详解）**

FlexMenuBottom
是功能最强大的弹窗，适合多维度切换场景（如同时选择画质+线路）。每个
FlexSection 是一个分组，分组内包含多个按钮标签。

> var pop = \$.require(&quot;hiker://page/hikerPop&quot;);
>
> var FlexSection = pop.FlexMenuBottom.FlexSection;
>
> pop.FlexMenuBottom({
>
> title: &quot;播放设置&quot;,
>
> sections: \[
>
> new FlexSection(&quot;画质选择&quot;, \[&quot;1080P&quot;,
> &quot;720P&quot;, &quot;480P&quot;\]),
>
> new FlexSection(&quot;播放线路&quot;, \[&quot;线路A&quot;,
> &quot;线路B&quot;, &quot;线路C&quot;\]),
>
> \],
>
> click: function(button, sectionIndex, buttonIndex) {
>
> // button.title = 按钮文字
>
> log(&quot;点击分组&quot; + sectionIndex + &quot;按钮&quot; +
> buttonIndex + &quot;: &quot; + button.title);
>
> },
>
> longClick: function(button, sectionIndex, buttonIndex) {
>
> log(&quot;长按: &quot; + button.title);
>
> },
>
> height: 0.6 // 弹窗高度占屏幕比例，默认 0.75
>
> });

**21.4 同步弹窗（在 lazyRule 中使用）**

confirmSync 和 inputConfirmSync
会阻塞当前线程直到用户操作完成，非常适合在 lazyRule 的 .js:
代码块中做交互式选择。

> // 在二级页面的 url 的 lazyRule 中使用同步弹窗选择线路
>
> url: videoUrl + &quot;@lazyRule=.js:&quot; +
>
> &quot;var pop = \$.require(\'hiker://page/hikerPop\');&quot; +
>
> &quot;var ok =
> pop.confirmSync({title:\'提示\',content:\'需要付费，是否继续？\'});&quot;
> +
>
> &quot;if (!ok) return \'toast://已取消\';&quot; +
>
> &quot;return input;&quot;

**21.5 带搜索框的底部列表（ResExtraInputBox）**

selectBottomRes 支持附加一个搜索/输入框到弹窗内，通过 ResExtraInputBox
实现：

> var pop = \$.require(&quot;hiker://page/hikerPop&quot;);
>
> var ResExtraInputBox = pop.ResExtraInputBox; // 注意从 exports 取
>
> var searchBox = new ResExtraInputBox({
>
> hint: &quot;搜索集数\...&quot;,
>
> title: &quot;跳转&quot;,
>
> click: function(inputText, manage) {
>
> // inputText = 用户输入内容
>
> // manage.scrollToPosition(Number(inputText) - 1, true);
>
> log(&quot;跳转到: &quot; + inputText);
>
> }
>
> });
>
> pop.selectBottomRes({
>
> title: &quot;选择集数&quot;,
>
> options: \[&quot;第1集&quot;, &quot;第2集&quot;, &quot;第3集&quot;\],
>
> columns: 4,
>
> extraInputBox: searchBox,
>
> click: function(value, index, manage) {
>
> log(&quot;选择: &quot; + value);
>
> }
>
> });

**21.6 playVideos 直接调用播放器**

无需跳转页面，直接在弹窗回调中启动播放器，适合多线路切换：

> var pop = \$.require(&quot;hiker://page/hikerPop&quot;);
>
> // playList 格式
>
> var playList = \[
>
> { title: &quot;线路1&quot;, url:
> &quot;https://example.com/video1.m3u8&quot;, header: &quot;Referer:
> https://example.com/&quot; },
>
> { title: &quot;线路2&quot;, url:
> &quot;https://example.com/video2.m3u8&quot; },
>
> \];
>
> // 直接播放第0条
>
> pop.playVideos(playList, 0);
>
> // 配合选择弹窗使用
>
> pop.selectCenter({
>
> title: &quot;选择线路&quot;,
>
> options: playList.map(function(v) { return v.title; }),
>
> click: function(value, index) {
>
> pop.playVideos(playList, index);
>
> }
>
> });

**21.7 注意事项**

> ⚠️ 节流保护：所有弹窗函数内置 200ms
> 节流，短时间内重复点击会被忽略。如需调整，在调用前执行
> pop.setNextThrottle(500) 设置下次节流时长（毫秒）。
>
> ⚠️ UI 线程：直接操作界面的代码（如修改按钮文字）必须在 runOnUIThread
> 内执行，否则崩溃。
>
> ⚠️ 嗅觉浏览器适配：hikerPop 会自动检测并适配嗅觉浏览器环境，copyBottom
> 在嗅觉浏览器下返回 null。
>
> 📌 Android 版本要求：hikerPop 需要 Android 8.0（API
> 26）及以上，低版本会直接抛错。
>
> **第二十二章：RhinoUI 原生布局渲染库**

RhinoUI 是海阔视界/聚阅的高级 UI 框架，通过 E4X XML 语法在规则 JS
环境中直接描述 Android 原生布局，动态创建并操作 View 控件。配合 hikerPop
的 showRhinoUI() 可弹出任意自定义界面，是目前最灵活的规则 UI 方案。

> ⚠️ RhinoUI 使用 E4X XML 语法（XML 作为 JS 一等公民），布局 XML
> 不能被代码格式化工具处理，否则会乱码。规则文件请关闭自动格式化。
>
> ⚠️ 所有 UI 操作必须在 UI 线程执行，使用 RhinoUI.runOnUI(fn)
> 包裹，否则崩溃。

**22.1 引入与基本调用**

> const RhinoUI = \$.require(&quot;UI&quot;); // 引入 RhinoUI 依赖页面
>
> // render() 渲染布局，返回 idMap（key=XML中id属性，value=Wrapper对象）
>
> let ui = RhinoUI.render(
>
> \<vertical bg=&quot;#FFFFFF&quot; padding=&quot;16&quot;\>
>
> \<text id=&quot;tvTitle&quot; text=&quot;Hello&quot;
> textSize=&quot;18sp&quot; textStyle=&quot;bold&quot;/\>
>
> \<button id=&quot;btn&quot; text=&quot;点我&quot; margin=&quot;0
> 10&quot;/\>
>
> \</vertical\>
>
> );
>
> ui.tvTitle.text(&quot;修改标题&quot;); // 通过 id 访问并修改
>
> ui.btn.click(() =\> toast(&quot;点击了&quot;));
>
> // 配合 hikerPop 显示为底部弹窗（必须在 UI 线程执行）
>
> RhinoUI.runOnUI(() =\> {
>
> const pop =
> \$.require(&quot;hiker://page/hikerPop.js?rule=hikerPop&quot;);
>
> let popup = pop.showRhinoUI(ui, { type: &quot;bottom&quot;, height:
> 400 });
>
> ui.btn.click(() =\> popup.dismiss());
>
> });

**22.2 支持的控件标签**

**布局容器：**

-   \<vertical\> / \<linear\> → 垂直线性布局（LinearLayout）

-   \<horizontal\> → 水平线性布局

-   \<frame\> → 帧布局（FrameLayout），子控件可叠加

-   \<scroll\> / \<scrollview\> → 垂直滚动容器

-   \<hscroll\> → 水平滚动容器，自动隐藏滚动条

-   \<card\> → 卡片容器，支持 cardElevation / cardCornerRadius 属性

-   \<drawer\> / \<drawerlayout\> → 抽屉布局，配合
    layout_gravity=&quot;start&quot; 实现侧边栏

-   \<toolbar\> → 顶部工具栏，支持 title / titleTextColor 属性

-   \<grad\> / \<gradient\> → 纯色/渐变背景容器，支持 color / radius /
    stroke 属性

**基础控件：**

-   \<text\> → 文本（TextView），支持
    text/textSize/textColor/textStyle/maxLines/ellipsize/typeface

-   \<btn\> / \<button\> → 按钮（Button），继承 text 全部属性

-   \<input\> / \<edit\> → 输入框（EditText），支持
    hint/password/inputType/digits/hintColor

-   \<img\> / \<image\> → 图片（ImageView），支持
    src/scaleType/circle/radius/tint，自动使用 Glide 加载

-   \<fab\> → 悬浮按钮（FloatingActionButton）

-   \<check\> / \<checkbox\> → 复选框，支持 checked 属性

-   \<switch\> → 开关，支持
    thumbTint/trackTint（格式：未选色/选中色，用/分隔）

-   \<radio\> / \<radiogroup\> → 单选按钮 / 单选组

-   \<spinner\> → 下拉菜单，entries=&quot;A\|B\|C&quot; 快速设置选项

-   \<progressbar\> → 进度条，支持 max/progress/indeterminate 属性

-   \<seekbar\> → 拖动条（SeekBar），on(&quot;change&quot;, fn) 监听拖动

-   \<webview\> → 内嵌网页，url/src 属性直接加载 URL

-   \<list\> / \<recyclerview\> →
    列表（RecyclerView），内嵌一个子元素作为 item 模板，span 设置列数

-   \<viewpager\> / \<viewpager2\> →
    左右滑动分页（ViewPager2），内嵌一个子元素作为页面模板

-   \<viewswitcher\> / \<switcher\> → 视图切换器

-   \<canvas\> → 自定义画布，on(&quot;draw&quot;, canvas) 获取绘制回调

-   \<blank\> / \<space\> → 空白占位控件

-   \<divider\> / \<line\> → 分割线（默认 1px 高 / #E0E0E0 颜色 /
    宽度填满）

**22.3 通用属性速查**

**布局属性：**

-   w / width → 宽度：&quot;match_parent&quot; /
    &quot;wrap_content&quot; / &quot;\*&quot; / &quot;-1&quot; /
    &quot;200dp&quot;

-   h / height → 高度，同上

-   margin → 外边距，1/2/4个值（左上右下），如 &quot;8dp&quot; /
    &quot;8dp 4dp&quot; / &quot;左 上 右 下&quot;

-   padding → 内边距，格式同 margin

-   layout_weight / weight → 线性布局权重（配合 w=&quot;0&quot; 或
    h=&quot;0&quot; 使用）

-   layout_gravity → 在父容器中的对齐：center / start / end / bottom 等

-   gravity → 内容对齐（对 LinearLayout / TextView 有效）

**外观属性：**

-   bg / background → 背景色（#RRGGBB）/ ?attr/资源 / \@drawable/资源名

-   radius / cornerRadius → 全圆角，如 &quot;8dp&quot;

-   corners → 四角独立圆角（左上 右上 右下 左下），如 &quot;16dp 16dp 0
    0&quot;

-   stroke → 边框，&quot;宽度 颜色&quot;，如 &quot;1dp #CCCCCC&quot;

-   ripple → 点击水波纹，&quot;true&quot; 或直接写颜色
    &quot;#33000000&quot;

-   alpha → 透明度 0.0\~1.0

-   elevation → 阴影高度（Android 5.0+）

-   rotation → 旋转角度（度）

-   visibility → 可见性：&quot;visible&quot; / &quot;invisible&quot; /
    &quot;gone&quot;

**22.4 Wrapper 对象方法速查**

**通用方法（所有控件）：**

-   .attr(key, value) → 设置属性（等价于 XML 属性）；传入对象可批量设置

-   .attr(key) → 读取已设置的属性值

-   .click(fn) → 设置点击监听

-   .longClick(fn) → 设置长按监听

-   .on(event, fn) → 通用事件监听（\'click\' / \'long_click\' 等）

-   .visibility(v) → 设置可见性：&quot;visible&quot; /
    &quot;invisible&quot; / &quot;gone&quot;

-   .width(w) / .height(h) → 快捷设置尺寸

-   .margin(m) / .padding(p) → 快捷设置边距

-   .gravity(g) → 快捷设置对齐

-   .addView(viewOrXml) → 动态添加子控件（支持 E4X XML / 原生 View /
    Wrapper 对象）

-   .removeAllViews() → 清空所有子控件

-   .post(fn) → 在 View 消息队列中执行（UI 线程安全的异步操作）

-   .raw() → 获取底层 Android 原生 View 对象

**TextWrapper 特有（text/button/input）：**

-   .text(val) → 设置/获取文字内容（无参数为 getter）

-   .textColor(colorStr) → 设置文字颜色

-   .error(msg) → 设置输入框错误提示（msg=null 清除）

-   .setSelection(start, end) → 设置光标/选区位置

**InputWrapper 特有事件（on 方法）：**

-   .on(&quot;change&quot;, fn) → 文字变化后回调，fn(text)，最常用

-   .on(&quot;before_text_change&quot;, fn) → 文字变化前回调

-   .on(&quot;on_text_change&quot;, fn) → 文字变化中回调

-   .off(event) → 移除监听；&quot;all&quot; 移除全部

**CheckWrapper 特有（checkbox/switch/radio）：**

-   .checked(val) → 设置/获取选中状态（无参数为 getter）

-   .on(&quot;change&quot;, fn) → 选中状态变化回调，fn(isChecked)

**ImageWrapper 特有：**

-   .loadUrl(url) → 加载图片（自动 Glide，支持 HTTP/hiker://@drawable/）

-   .setTint(colorStr) → 设置图片着色滤镜

**RecyclerWrapper（list）特有：**

-   .setDataSource(arr) → 设置数据源并触发渲染

-   .setSpanCount(n) → 设置列数（1=列表，n\>1=网格）

-   .addItem(item) → 末尾追加一条数据并刷新

-   .insertItem(index, item) → 指定位置插入数据

-   .removeItem(index) → 删除指定位置数据

-   .updateItem(index, item) → 更新指定位置数据

-   .notifyDataSetChanged() → 全量刷新列表

-   .on(&quot;item_bind&quot;, fn) → item 绑定回调，fn(itemWrapper,
    itemData, position)

-   .on(&quot;item_click&quot;, fn) → 点击回调，fn(itemData, position,
    itemWrapper, listWrapper)

-   .on(&quot;item_long_click&quot;, fn) → 长按回调，fn(event, itemData,
    position)；event.consumed=true 消费事件

**PagerWrapper（viewpager）特有：**

-   .setDataSource(arr) → 设置页面数据源

-   .setCurrentItem(index, smooth) → 跳转到指定页，smooth=false 禁用动画

-   .notifyDataSetChanged() → 全量刷新

-   .on(&quot;item_bind&quot;, fn) → 页面绑定回调，fn(pageWrapper,
    pageData, position)

-   .on(&quot;change&quot;, fn) → 页面切换回调，fn(position)

**22.5 模板插值（{{key}} 语法）**

在 item 模板的 text 属性中使用 {{key}}
占位符，数据绑定时自动替换，无需手动 item_bind 赋值：

> let ui = RhinoUI.render(
>
> \<vertical\>
>
> \<list id=&quot;list&quot;\>
>
> \<vertical padding=&quot;12&quot;\>
>
> \<!\-- 自动从数据对象中取 name / age 字段 \--\>
>
> \<text text=&quot;{{name}}&quot; textSize=&quot;16sp&quot;
> textStyle=&quot;bold&quot;/\>
>
> \<text text=&quot;年龄: {{age}} 岁&quot; textSize=&quot;13sp&quot;
> textColor=&quot;#666&quot;/\>
>
> \<!\-- {{this}} 代表 item 本身，适合字符串数组 \--\>
>
> \</vertical\>
>
> \</list\>
>
> \</vertical\>
>
> );
>
> ui.list.setDataSource(\[{name:&quot;张三&quot;, age:25},
> {name:&quot;李四&quot;, age:30}\]);
>
> // 无需 item_bind，自动渲染

**22.6 三种显示方式**

-   RhinoUI.render(xml) → 只渲染不显示，配合 hikerPop.showRhinoUI()
    最灵活（推荐）

-   RhinoUI.showPage(xml, isFull) → 全屏 Dialog，isFull=true
    时真正全屏无状态栏，ui.dialog.dismiss() 关闭

-   RhinoUI.showDialog(xml, options) → AlertDialog 样式，支持
    title/positive/negative/neutral 按钮配置

> // showRhinoUI 配置项（最常用方式）
>
> let popup = hikerPop.showRhinoUI(ui, {
>
> type: &quot;bottom&quot;, // &quot;center&quot; 居中 或
> &quot;bottom&quot; 底部
>
> height: 400, // dp，底部弹窗高度
>
> maxHeight: 500, // dp，最大高度（bottom 类型）
>
> borderRadius: 16, // 圆角 dp
>
> dismissOnTouchOutside: true,
>
> dismissOnBackPressed: true,
>
> onShow: () =\> {}, // 显示后回调
>
> onDismiss: () =\> {} // 关闭后回调
>
> });
>
> popup.dismiss(); // 手动关闭
>
> // getPercentageHeight 计算屏幕百分比高度（dp）
>
> let h = RhinoUI.getPercentageHeight(0.75); // 屏幕高度的 75%（dp）

**22.7 fromStr 动态构建布局**

当布局需要插入 JS 变量（颜色、文字等）时，用模板字符串拼接后再通过
fromStr 转为 E4X 对象：

> var C = { bg: &quot;#1E1E1E&quot;, text: &quot;#E0E0E0&quot;, btn:
> &quot;#333333&quot; };
>
> var layoutXml = RhinoUI.fromStr(\`
>
> \<vertical bg=&quot;\${C.bg}&quot; corners=&quot;16dp 16dp 0 0&quot;
> w=&quot;match_parent&quot;\>
>
> \<text id=&quot;tvTitle&quot; textColor=&quot;\${C.text}&quot;
> textSize=&quot;16sp&quot; textStyle=&quot;bold&quot;/\>
>
> \<list id=&quot;rvList&quot; w=&quot;match_parent&quot;
> h=&quot;0&quot; weight=&quot;1&quot;/\>
>
> \</vertical\>
>
> \`);
>
> var itemXml = RhinoUI.fromStr(\`
>
> \<frame w=&quot;match_parent&quot; h=&quot;wrap_content&quot;
> padding=&quot;5dp&quot;\>
>
> \<text id=&quot;tvItem&quot; w=&quot;match_parent&quot;
> h=&quot;40dp&quot; gravity=&quot;center&quot;
>
> textSize=&quot;13sp&quot; radius=&quot;8dp&quot;
> bg=&quot;\${C.btn}&quot; textColor=&quot;\${C.text}&quot;/\>
>
> \</frame\>
>
> \`);
>
> var ui = RhinoUI.render(layoutXml);
>
> ui.rvList.setTemplate(itemXml); // 手动设置模板（fromStr
> 方式必须手动调用）
>
> ui.rvList.setDataSource(dataArr);

**22.8 自定义组件注册（registerWidget）**

将常用组合控件封装为自定义标签，在 XML 中像原生标签一样使用：

> RhinoUI.registerWidget(&quot;my-card&quot;, {
>
> template: (
>
> \<vertical padding=&quot;12&quot; bg=&quot;#FFFFFF&quot;
> radius=&quot;8dp&quot;\>
>
> \<text id=&quot;\_title&quot; textSize=&quot;16sp&quot;
> textStyle=&quot;bold&quot;/\>
>
> \<text id=&quot;\_sub&quot; textSize=&quot;12sp&quot;
> textColor=&quot;#999&quot; marginTop=&quot;4dp&quot;/\>
>
> \</vertical\>
>
> ),
>
> attrHandler: function(wrapper, attrName, value) {
>
> if (attrName === &quot;title&quot;) wrapper.\_title.text(value);
>
> if (attrName === &quot;sub&quot;) wrapper.\_sub.text(value);
>
> },
>
> methods: {
>
> getTitle: function() { return this.\_title.text(); }
>
> }
>
> });
>
> // 使用自定义标签
>
> let ui = RhinoUI.render(
>
> \<vertical\>
>
> \<my-card id=&quot;card1&quot; title=&quot;仙逆&quot;
> sub=&quot;125集已完结&quot;/\>
>
> \</vertical\>
>
> );
>
> toast(ui.card1.getTitle()); // 调用自定义方法
>
> **第二十三章：KVS 播放器 Hash 解密算法（x1hub 实战）**

KernelVideoSharing（KVS）是一套广泛使用的成人视频站建站程序，其播放器对视频直链进行了
Hash 重排加密保护。本章记录逆向还原过程与完整 ES5 实现，算法源自 yt-dlp
项目移植。

> 📌 适用站点：使用 KVS 程序搭建、embed 页含 flashvars.license_code 和
> video_url: \'function/0/\...\' 字段的站点，如 x1hub.com 等。

**23.1 加密原理**

**KVS 播放器的视频直链保护分两层：**

-   video_url 字段：格式为
    \'function/0/https://cdn\.../hash路径/video.mp4?embed=true\'，hash
    是经过重排的字符串

-   license_code 字段：格式为 \'\$361701411933162\'，去掉 \$
    后的数字用于生成 token 数组

-   token 数组：用于指导 hash 字符的还原顺序（swap 操作）

-   最终 URL：还原后的 hash 路径 + ?rnd=毫秒时间戳（去掉 embed=true）

**23.2 算法步骤详解**

**Step 1：从 license_code 生成 token 数组**

> // license_code 示例: \'\$361701411933162\'
>
> var lc = licCode.replace(/\\\$/g, \'\'); // \'361701411933162\'
>
> var lcVals = lc.split(\'\').map(Number); //
> \[3,6,1,7,0,1,4,1,1,9,3,3,1,6,2\]
>
> // 把 0 替换成 1，防止除数为0
>
> var modLc = lc.replace(/0/g, \'1\');
>
> var center = Math.floor(modLc.length / 2);
>
> // 前后半段相减，乘以4，截取前 center+1 位
>
> var front = parseInt(modLc.substring(0, center + 1));
>
> var back = parseInt(modLc.substring(center));
>
> var modStr = (4 \* Math.abs(front - back) + \'\').substring(0,
> center + 1);
>
> // 双重循环生成 token 数组（长度约32）
>
> var token = \[\];
>
> for (var ti = 0; ti \< modStr.length; ti++) {
>
> var cur = parseInt(modStr.charAt(ti));
>
> for (var off = 0; off \< 4; off++) {
>
> if (ti + off \< lcVals.length) {
>
> token.push((lcVals\[ti + off\] + cur) % 10);
>
> }
>
> }
>
> }

**Step 2：用 token 还原 hash（字符重排）**

> // 从 video_url 提取路径，找到长度 \>= 32 的 hash 段
>
> var url = rawUrl.replace(\'function/0/\', \'\');
>
> var parts = url.split(\'/\');
>
> var hashIdx = parts.findIndex(p =\> p.length \>= 32); // ES5用for循环
>
> var HL = 32;
>
> var hash = parts\[hashIdx\].substring(0, HL);
>
> // 构建下标数组 \[0,1,2,\...,31\]
>
> var indices = \[\];
>
> for (var ii = 0; ii \< HL; ii++) indices.push(ii);
>
> // 从尾到头做 swap（关键：从后往前，和加密时相反）
>
> var accum = 0;
>
> for (var src = HL - 1; src \>= 0; src\--) {
>
> accum += token\[src\] \|\| 0;
>
> var dest = (src + accum) % HL;
>
> var tmp = indices\[src\];
>
> indices\[src\] = indices\[dest\];
>
> indices\[dest\] = tmp;
>
> }
>
> // 用还原后的 indices 重组 hash 字符
>
> var newHash = \'\';
>
> for (var ni = 0; ni \< HL; ni++) {
>
> newHash += hash.charAt(indices\[ni\]);
>
> }
>
> parts\[hashIdx\] = newHash + parts\[hashIdx\].substring(HL);

**Step 3：拼接最终 URL**

> // 去掉 embed=true，rnd 用毫秒时间戳（服务器接受任意毫秒时间戳）
>
> var finalUrl = parts.join(\'/\') + \'?rnd=\' + new Date().getTime();
>
> return \'video://\' + finalUrl + \';{User-Agent@\' + ua +
> \'&&Referer@\' + host + \'/}\';

**23.3 完整 ES5 实现（可直接复用）**

以下是可直接用于海阔/聚阅规则的完整 lazyRule 实现：

> \_getVideoRule: function() {
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> return \$(\'\').lazyRule(function(host, ua) {
>
> var \_embedUrl = input;
>
> if (\_embedUrl.indexOf(\'/videos/\') \> -1) {
>
> var \_vid = \_embedUrl.match(/\\/videos\\/(\\d+)/);
>
> if (\_vid) \_embedUrl = host + \'/embed/\' + \_vid\[1\] + \'/\';
>
> }
>
> var \_html = request(\_embedUrl, {
>
> headers: { \'User-Agent\': ua, \'Referer\': host + \'/\' }
>
> });
>
> if (!\_html \|\| \_html.length \< 200) return \_embedUrl + \'#嗅探\';
>
> // 提取 flashvars 两个关键字段
>
> var \_mUrl =
> \_html.match(/video_url\\s\*\[\':\]\\s\*\[\'&quot;\](\[\^\'&quot;\]+)\[\'&quot;\]/);
>
> var \_mLic =
> \_html.match(/license_code\\s\*\[\':\]\\s\*\[\'&quot;\](\[\^\'&quot;\]+)\[\'&quot;\]/);
>
> if (!\_mUrl \|\| !\_mLic) return \_embedUrl + \'#嗅探\';
>
> var \_rawUrl = \_mUrl\[1\];
>
> var \_licCode = \_mLic\[1\];
>
> // Step1: 生成 token 数组
>
> var \_lc = \_licCode.replace(/\\\$/g, \'\');
>
> var \_lcVals = \[\];
>
> for (var \_i = 0; \_i \< \_lc.length; \_i++)
> \_lcVals.push(parseInt(\_lc.charAt(\_i)));
>
> var \_modLc = \_lc.replace(/0/g, \'1\');
>
> var \_center = Math.floor(\_modLc.length / 2);
>
> var \_front = parseInt(\_modLc.substring(0, \_center + 1));
>
> var \_back = parseInt(\_modLc.substring(\_center));
>
> var \_modStr = (4 \* Math.abs(\_front - \_back) + \'\').substring(0,
> \_center + 1);
>
> var \_token = \[\];
>
> for (var \_ti = 0; \_ti \< \_modStr.length; \_ti++) {
>
> var \_cur = parseInt(\_modStr.charAt(\_ti));
>
> for (var \_off = 0; \_off \< 4; \_off++) {
>
> if (\_ti + \_off \< \_lcVals.length)
>
> \_token.push((\_lcVals\[\_ti + \_off\] + \_cur) % 10);
>
> }
>
> }
>
> // Step2: 还原 hash
>
> var \_isFunc = \_rawUrl.indexOf(\'function/0/\') === 0;
>
> var \_url = \_isFunc ? \_rawUrl.substring(11) : \_rawUrl;
>
> \_url = \_url.split(String.fromCharCode(92)).join(\'\');
>
> var \_qIdx = \_url.indexOf(\'?\');
>
> var \_base = \_qIdx \> -1 ? \_url.substring(0, \_qIdx) : \_url;
>
> if (\_isFunc) {
>
> var \_parts = \_base.split(\'/\');
>
> var \_hi = -1;
>
> for (var \_pi = 0; \_pi \< \_parts.length; \_pi++) {
>
> if (\_parts\[\_pi\].length \>= 32) { \_hi = \_pi; break; }
>
> }
>
> if (\_hi \> -1) {
>
> var \_HL = 32;
>
> var \_hash = \_parts\[\_hi\].substring(0, \_HL);
>
> var \_suf = \_parts\[\_hi\].substring(\_HL);
>
> var \_idx = \[\];
>
> for (var \_ii = 0; \_ii \< \_HL; \_ii++) \_idx.push(\_ii);
>
> var \_acc = 0;
>
> for (var \_src = \_HL - 1; \_src \>= 0; \_src\--) {
>
> \_acc += \_token\[\_src\] \|\| 0;
>
> var \_dst = (\_src + \_acc) % \_HL;
>
> var \_t = \_idx\[\_src\]; \_idx\[\_src\] = \_idx\[\_dst\];
> \_idx\[\_dst\] = \_t;
>
> }
>
> var \_nh = \'\';
>
> for (var \_ni = 0; \_ni \< \_HL; \_ni++) \_nh +=
> \_hash.charAt(\_idx\[\_ni\]);
>
> \_parts\[\_hi\] = \_nh + \_suf;
>
> \_base = \_parts.join(\'/\');
>
> }
>
> }
>
> // Step3: 拼接最终 URL
>
> var \_final = \_base + \'?rnd=\' + new Date().getTime();
>
> log(\'\[KVS\] \' + \_final.substring(0, 100));
>
> return \'video://\' + \_final
>
> \+ \';{User-Agent@\' + ua + \'&&Referer@\' + host + \'/}\';
>
> }, \_host, \_ua);
>
> },

**23.4 关键注意事项**

> 📌 rnd 用 Date.now() 毫秒时间戳即可，服务器只校验 hash 是否正确，rnd
> 任意毫秒值均可通过。
>
> 📌 swap 方向必须从尾到头（src = HL-1 →
> 0），这是还原操作，与加密时方向相反。
>
> 📌 license_code 中的 \$ 前缀必须去掉再处理，否则 parseInt 会返回 NaN。
>
> ⚠️ embed 页通常无 CF 保护，用普通 request 即可，无需
> fetchCodeByWebView，速度更快。
>
> ⚠️ 此算法适用于 KVS 程序站点，非 KVS 站点（如使用 MACCMS
> 的站点）无需此算法。
>
> **第二十四章：内嵌视频算法总结与 VidHide 实战**

内嵌视频（iframe
嵌套播放器）是目前成人视频站最常见的播放方式。本章汇总所有常见算法类型，并重点记录
VidHide 播放器的逆向实战过程（代表站：kbjus.com / playrecord.biz）。

**24.1 内嵌视频算法类型总览**

-   直接 m3u8/mp4 直链 --- 无加密，正则直接匹配

-   KVS Hash 重排（第二十三章） --- 特征：video_url:
    \'function/0/\...\' + license_code；代表站：x1hub.com

-   VidHide split 数组拼接（本章） --- 特征：eval(function(p,a,c,k\...))
    混淆 JS + split(\'\|\')
    字符串数组末尾固定结构；代表站：playrecord.biz / vidhide.com

-   MACCMS player_aaaa 加密（第九章） --- 特征：player_aaaa 配置块 +
    encrypt:1/2；encrypt=1 URL编码，encrypt=2 Base64+URL双重编码

-   AES/XOR 图片/视频加密（第二十一章） ---
    特征：图片灰色占位或视频黑屏，站点 worker.js 含
    CryptoJS.AES.decrypt；代表站：91short.com

-   iframe 二次请求 --- 特征：视频页源码含 \<iframe
    src=\'https://外部域名/embed/xxx\'\>，真实地址在 iframe 页里；需两步
    fetch

-   WebView 嗅探兜底 --- 以上方法均失败时，用 fetchCodeByWebView 或
    #嗅探 触发 App 内置嗅探器

> 📌 识别技巧：先看站点 Network 面板，找视频请求 URL 格式；再看 embed 页
> JS 特征（eval混淆、player_aaaa、flashvars）来判断使用哪种算法。

**24.2 iframe 二次请求通用写法**

视频页嵌套外部 iframe 时，需要两步请求才能拿到真实播放地址：

> // Step1：请求视频页，提取 iframe src
>
> var \_html = fetchPC(input, { headers: { \'Referer\': host + \'/\' }
> });
>
> // 方式A：从 hidden input 的 base64 value 解码（更可靠）
>
> // \<input type=\'hidden\' id=\'links\'
> value=\'aHR0cHM6Ly9wbGF5cmVjb3JkLmJpei9lbWJlZC8\...\'\>
>
> var \_b64m =
> \_html.match(/id=\[&quot;\'\]links\[&quot;\'\]\[\^\>\]\*value=\[&quot;\'\](\[A-Za-z0-9+\\/=\]+)\[&quot;\'\]/);
>
> if (\_b64m) {
>
> var \_bytes = android.util.Base64.decode(\_b64m\[1\],
> android.util.Base64.DEFAULT);
>
> \_iframeSrc = new java.lang.String(\_bytes, \'UTF-8\');
>
> }
>
> // 方式B：直接正则匹配 iframe src
>
> if (!\_iframeSrc) {
>
> var \_im =
> \_html.match(/\<iframe\[\^\>\]\*src=\[&quot;\'\](https?:\\/\\/\[\^&quot;\'\]+)\[&quot;\'\]/);
>
> if (\_im) \_iframeSrc = \_im\[1\];
>
> }
>
> // Step2：请求 embed 页提取播放地址
>
> var \_ih = fetchPC(\_iframeSrc, { headers: { \'Referer\': host + \'/\'
> } });
>
> var \_m =
> \_ih.match(/(https?:\[\^\\s&quot;\'\<\>\\\\\]+\\.m3u8\[\^\\s&quot;\'\<\>\\\\\]\*)/);
>
> if (\_m) return \'video://\' + \_m\[1\] + \';{User-Agent@\' + ua +
> \'&&Referer@\' + \_iframeSrc + \'}\';
>
> ⚠️ atob() 在 Rhino JS 引擎里未定义，base64 解码必须用
> android.util.Base64.decode()。

**24.3 VidHide 播放器算法（playrecord.biz / vidhide.com）**

VidHide 是一个广泛使用的视频托管播放器，embed 页使用
eval(function(p,a,c,k,\...)) 混淆 JS。通过分析解混淆后的字符串
split(\'\|\') 数组，可以从末尾固定位置提取播放地址的组成部分。

**split 数组末尾固定结构：**

-   arr\[-1\] = key （base64字符串，含大写字母，如
    VoFAMLOu95r4YO7Rc8KKWA）

-   arr\[-2\] = token （纯小写字母，如 hjkrhuihghfvu）

-   arr\[-3\] = 时间戳 （10位Unix时间戳，如 1774045296）

-   arr\[-4\] = key1 （另一个base64，hls2备用地址用）

-   arr\[-5\] = CDN子域 （如 acek）

-   arr\[-6\] = \'cdn\'

-   arr\[-7\] = hashid （5位数字，如 07671）

-   arr\[-8\] = 文件名 （小写字母+下划线，如 rzagv3vtqm2t\_）

-   arr\[-9\] = \'urlset\'

-   arr\[-10\] = token2 （大写base64，hls2地址用）

-   arr\[-11\] = 时间戳2

**最终 stream URL 拼接格式：**

> //
> https://cdn域名/stream/arr\[-1\]/arr\[-2\]/arr\[-3\]/file_id/master.m3u8
>
> // cdn域名直接从 iframeSrc 提取（如 playrecord.biz）
>
> // file_id 从 embed 页 \$.cookie(\'file_id\', \'xxx\') 提取
>
> // 示例：
>
> //
> https://playrecord.biz/stream/VoFAMLOu95r4YO7Rc8KKWA/hjkrhuihghfvu/1774045296/38359663/master.m3u8

**24.4 完整 ES5 实现（可直接复用）**

> \_getVideoRule: function() {
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> return \$(\'\').lazyRule(function(host, ua) {
>
> // Step1: 视频页 → 提取 embed URL
>
> var \_html = fetchPC(input, { headers: { \'Referer\': host + \'/\' }
> });
>
> var \_iframeSrc = \'\';
>
> var \_b64m =
> \_html.match(/id=\[&quot;\'\]links\[&quot;\'\]\[\^\>\]\*value=\[&quot;\'\](\[A-Za-z0-9+\\/=\]+)\[&quot;\'\]/);
>
> if (\_b64m) {
>
> try {
>
> var \_bytes = android.util.Base64.decode(\_b64m\[1\],
> android.util.Base64.DEFAULT);
>
> \_iframeSrc = new java.lang.String(\_bytes, \'UTF-8\');
>
> } catch(e) {}
>
> }
>
> if (!\_iframeSrc) {
>
> var \_im =
> \_html.match(/\<iframe\[\^\>\]\*src=\[&quot;\'\](https?:\\/\\/\[\^&quot;\'\]+)\[&quot;\'\]/);
>
> if (\_im) \_iframeSrc = \_im\[1\];
>
> }
>
> if (!\_iframeSrc) return input + \'#嗅探\';
>
> // Step2: embed 页 → 提取播放地址
>
> var \_ih = fetchPC(\_iframeSrc, { headers: { \'Referer\': host + \'/\'
> } });
>
> var \_sl = String.fromCharCode(92);
>
> // 优先：直接匹配完整 m3u8
>
> var \_r1 =
> \_ih.match(/(https?:\\/\\/\[\^&quot;\'\\s\\\\\]+\\.m3u8\[\^&quot;\'\\s\\\\\]\*)/);
>
> if (\_r1) return \'video://\' + \_r1\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> // VidHide：从 split 数组末尾拼接 stream URL
>
> var \_arrm =
> \_ih.match(/\'(\[\^\'\]{500,})\'\\.split\\(\'\\\|\'\\)\\)\\)/);
>
> if (\_arrm) {
>
> var \_arr = \_arrm\[1\].split(\'\|\');
>
> var \_len = \_arr.length;
>
> var \_key = \_arr\[\_len - 1\];
>
> var \_token = \_arr\[\_len - 2\];
>
> var \_ts = \_arr\[\_len - 3\];
>
> var \_fidm =
> \_ih.match(/file_id\[\'&quot;\],\\s\*\[\'&quot;\](\\d{5,10})\[\'&quot;\]/);
>
> var \_fid = \_fidm ? \_fidm\[1\] : \'\';
>
> var \_cdn = \_iframeSrc.match(/https?:\\/\\/(\[\^\\/\]+)/)\[1\];
>
> if (\_key && \_token && \_ts && \_fid) {
>
> var \_m3u8 = \'https://\' + \_cdn + \'/stream/\' +
>
> \_key + \'/\' + \_token + \'/\' + \_ts + \'/\' + \_fid +
> \'/master.m3u8\';
>
> return \'video://\' + \_m3u8 +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> }
>
> }
>
> // 兜底 mp4
>
> var \_r3 =
> \_ih.match(/(https?:\[\^\\s&quot;\'\<\>\\\\\]+\\.mp4\[\^\\s&quot;\'\<\>\\\\\]\*)/);
>
> if (\_r3) return \'video://\' + \_r3\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> return \_iframeSrc + \'#嗅探\';
>
> }, \_host, \_ua);
>
> },

**24.5 注意事项**

> 📌 split 数组长度各站不同，但末尾相对位置固定：arr\[-1\]=key,
> arr\[-2\]=token, arr\[-3\]=时间戳。
>
> 📌 CDN域名直接从 iframeSrc
> 提取，不同视频可能用不同域名（playrecord.biz / recordplay.biz
> 等），不要硬编码。
>
> 📌 file_id 在 embed 页 JS 里以 \$.cookie(\'file_id\', \'38359663\')
> 形式出现，正则：/file_id\[\'&quot;\],\\s\*\[\'&quot;\](\\d{5,10})\[\'&quot;\]/
>
> ⚠️ fetchPC 比 fetch 更稳定，服务器对移动 UA
> 可能返回不同内容（甚至不返回 iframe）。
>
> ⚠️ pdfh 无法解析 iframe 标签的 src 属性（iframe
> 无闭合标签），必须用正则或 base64 解码方式提取。
>
> **第二十五章：VidHide Dean Edwards Unpacker 解密（进阶）**

第二十四章介绍了通过 split 数组末尾提取 stream URL 的方案，适用于
playrecord.biz 的标准格式。但 VidHide 有多种 CDN
后端（podcastguestfinder.sbs、seamistboutiquecreations.cfd
等），这些后端使用 hls3 格式而非 stream 格式，需要通过 Dean Edwards
Unpacker 解码 eval 混淆代码来提取真实地址。

**25.1 两种 VidHide URL 格式对比**

**VidHide embed 页使用 eval(function(p,a,c,k,e,d){\...}) 混淆，解码后 o
对象包含两个播放地址字段：**

-   o.hls4 / o.1j → stream
    格式（旧）：https://cdn域名/stream/key/token/ts/fid/master.m3u8

-   o.hls3 / o.1a → hls3
    格式（新）：https://host.cfd/fid/hls3/01/hashid/filename\_,l,n,h,.urlset/master.txt

-   两种格式都需要先通过 Unpacker 解码 eval 混淆代码，再从解码结果里提取
    URL

**25.2 Dean Edwards Unpacker 核心算法**

eval 混淆格式：eval(function(p,a,c,k,e,d){\...})(模板字符串, 进制, 词数,
词典字符串.split(\'\|\'))

**解码步骤：**

> // 1. 定位 eval 代码块，提取模板字符串
>
> var \_evalIdx = html.indexOf(\'eval(function(p,a,c,k,e,d)\');
>
> var \_pidx = html.indexOf(&quot;}(\'&quot;, \_evalIdx); // 找 \'}(\'
> 分隔符
>
> var \_tmpl = html.substring(\_pidx + 3, \...); // 模板字符串
>
> // 2. 提取参数：base进制, count词数, keywords词典
>
> var \_pm =
> after.match(/\',\\s\*(\\d+)\\s\*,\\s\*(\\d+)\\s\*,\\s\*\'(\[\^\'\]+)\'\\s\*\\.split/);
>
> var \_base = parseInt(\_pm\[1\]); // 36 或 62
>
> var \_cnt = parseInt(\_pm\[2\]); // 词典大小
>
> var \_kws = \_pm\[3\].split(\'\|\'); // 词典数组
>
> // 3. 替换模板中每个 token（36进制时）
>
> \_code = \_tmpl.replace(/\\b(\\w+)\\b/g, function(m) {
>
> var n = parseInt(m, \_base);
>
> return (!isNaN(n) && n \< \_cnt && \_kws\[n\]) ? \_kws\[n\] : m;
>
> });
>
> // 4. 从解码结果里提取播放地址（按优先级）
>
> // A. 绝对 stream URL
>
> var \_su =
> \_code.match(/(https?:\\/\\/\[\^&quot;\'\\s\]+\\/stream\\/\[\^&quot;\'\\s\]+master\\.m3u8)/);
>
> // B. 相对路径（o对象里是相对路径时）
>
> var \_rp =
> \_code.match(/&quot;(\\/\[\^&quot;\]+\\/master\\.(?:m3u8\|txt)\[\^&quot;\]\*)&quot;/);
>
> if (\_rp) return embedHost + \_rp\[1\]; // 拼上 embed 域名
>
> // C. 绝对 hls3/hls2 CDN URL
>
> var \_hu =
> \_code.match(/(https?:\\/\\/\[\^&quot;\'\\s\]+(?:hls3\|hls2)\[\^&quot;\'\\s\]+\\.(?:m3u8\|txt)\[\^&quot;\'\\s\]\*)/);
>
> 📌 关键细节：embed 域名（embedHost）用于拼接相对路径。有时 vidhide.com
> 的 iframe 会重定向到 playrecord.biz，需要从 embed
> 页内容里检测真实域名。
>
> 📌 master.txt 和 master.m3u8 都是有效的 HLS
> 播放列表，海阔/聚阅播放器都能识别，不需要额外处理。
>
> ⚠️ 62进制（base=62）时用 \'0-9a-zA-Z\' 字符集，不能直接用 parseInt(m,
> 62)，需要自己实现转换。

**25.3 完整通用解析函数（兼容两种格式）**

以下实现同时支持 stream 格式和 hls3 格式，适用于所有 VidHide/PlayRecord
embed 页：

> \_getVideoRule: function() {
>
> var \_host = this.host;
>
> var \_ua = this.UA;
>
> return \$(\'\').lazyRule(function(host, ua) {
>
> var \_html = fetch(input, { headers: { \'Referer\': host + \'/\',
> \'User-Agent\': ua } });
>
> if (!\_html \|\| \_html.length \< 200) return input + \'#嗅探\';
>
> // Step1: 提取 embed URL（base64解码 → iframe正则）
>
> var \_iframeSrc = \'\';
>
> var \_b64m =
> \_html.match(/id=\[&quot;\'\]links\[&quot;\'\]\[\^\>\]\*value=\[&quot;\'\](\[A-Za-z0-9+\\/=\]+)\[&quot;\'\]/);
>
> if (\_b64m) {
>
> try {
>
> var \_b = android.util.Base64.decode(\_b64m\[1\],
> android.util.Base64.DEFAULT);
>
> \_iframeSrc = new java.lang.String(\_b, \'UTF-8\');
>
> } catch(e) {}
>
> }
>
> if (!\_iframeSrc) {
>
> var \_im =
> \_html.match(/iframe\[\^\>\]+src=\[&quot;\'\](https?:\\/\\/\[\^&quot;\'\\s\>\]+\\/embed\\/\[\^&quot;\'\\s\>\]+)\[&quot;\'\]/i);
>
> if (\_im) \_iframeSrc = \_im\[1\];
>
> }
>
> if (!\_iframeSrc) return input + \'#嗅探\';
>
> // Step2: 请求 embed 页
>
> var \_ih = fetchPC(\_iframeSrc, { headers: { \'Referer\': host + \'/\'
> } });
>
> if (!\_ih \|\| \_ih.length \< 100) return \_iframeSrc + \'#嗅探\';
>
> var \_sl = String.fromCharCode(92);
>
> // 获取真实 embed 域名（处理 vidhide→playrecord 跳转）
>
> var \_embedHost = \_iframeSrc.match(/https?:\\/\\/\[\^\\/\]+/)\[0\];
>
> var \_rhm =
> \_ih.match(/src=\[&quot;\'\](https?:\\/\\/\[\^&quot;\'\\/\]\*(?:playrecord\|recordplay)\[\^&quot;\'\\/\]\*)\[\\/&quot;\'\]/i);
>
> if (\_rhm) \_embedHost = \_rhm\[1\];
>
> // ① 直接匹配完整 stream m3u8
>
> var \_rs =
> \_ih.match(/(https?:\\/\\/\[\^&quot;\'\\s\\\\\]+\\/stream\\/\[\^&quot;\'\\s\\\\\]+\\/master\\.m3u8)/);
>
> if (\_rs) return \'video://\' + \_rs\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> // ② Dean Edwards Unpacker 解码 eval 混淆
>
> var \_evalIdx = \_ih.indexOf(\'eval(function(p,a,c,k,e,d)\');
>
> if (\_evalIdx \> -1) {
>
> var \_pidx = \_ih.indexOf(&quot;}(\'&quot;, \_evalIdx);
>
> if (\_pidx \> -1) {
>
> // 提取模板字符串（处理转义）
>
> var \_ti = \_pidx + 3, \_te = -1, \_esc = false;
>
> for (var \_ci = \_ti; \_ci \< \_ih.length; \_ci++) {
>
> if (\_esc) { \_esc = false; continue; }
>
> if (\_ih.charAt(\_ci) === \_sl) { \_esc = true; continue; }
>
> if (\_ih.charAt(\_ci) === &quot;\'&quot;) { \_te = \_ci; break; }
>
> }
>
> if (\_te \> \_ti) {
>
> var \_tmpl = \_ih.substring(\_ti, \_te);
>
> var \_after = \_ih.substring(\_te, Math.min(\_te + 10000,
> \_ih.length));
>
> var \_pm =
> \_after.match(/\',\\s\*(\\d+)\\s\*,\\s\*(\\d+)\\s\*,\\s\*\'(\[\^\'\]+)\'\\s\*\\.split/);
>
> if (\_pm) {
>
> var \_base = parseInt(\_pm\[1\]);
>
> var \_cnt = parseInt(\_pm\[2\]);
>
> var \_kws = \_pm\[3\].split(\'\|\');
>
> var \_code = \_tmpl.replace(/\\b(\\w+)\\b/g, function(tok) {
>
> var n = parseInt(tok, \_base);
>
> return (!isNaN(n) && n \>= 0 && n \< \_cnt && \_kws\[n\]) ? \_kws\[n\]
> : tok;
>
> });
>
> // A. 绝对 stream URL
>
> var \_su =
> \_code.match(/(https?:\\/\\/\[\^&quot;\'\\s\]+\\/stream\\/\[\^&quot;\'\\s\]+master\\.m3u8)/);
>
> if (\_su) return \'video://\' + \_su\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> // B. 相对路径（拼 embedHost）
>
> var \_rp =
> \_code.match(/&quot;(\\/\[\^&quot;\]+\\/master\\.(?:m3u8\|txt)\[\^&quot;\]\*)&quot;/);
>
> if (\_rp) return \'video://\' + \_embedHost +
> \_rp\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> // C. 绝对 hls3/hls2
>
> var \_hu =
> \_code.match(/(https?:\\/\\/\[\^&quot;\'\\s\]+(?:hls3\|hls2)\[\^&quot;\'\\s\]+\\.(?:m3u8\|txt)\[\^&quot;\'\\s\]\*)/);
>
> if (\_hu) return \'video://\' + \_hu\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> // D. 任意 m3u8
>
> var \_mu =
> \_code.match(/(https?:\\/\\/\[\^&quot;\'\\s\]+\\.m3u8\[\^&quot;\'\\s\]\*)/);
>
> if (\_mu) return \'video://\' + \_mu\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> }
>
> }
>
> }
>
> }
>
> // ③ 通用兜底
>
> var \_r1 =
> \_ih.match(/(https?:\\/\\/\[\^&quot;\'\\s\\\\\]+\\.m3u8\[\^&quot;\'\\s\\\\\]\*)/);
>
> if (\_r1) return \'video://\' + \_r1\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> var \_r3 =
> \_ih.match(/(https?:\[\^\\s&quot;\'\<\>\\\\\]+\\.mp4\[\^\\s&quot;\'\<\>\\\\\]\*)/);
>
> if (\_r3) return \'video://\' + \_r3\[1\].split(\_sl).join(\'\') +
>
> \';{User-Agent@\' + ua + \'&&Referer@\' + \_iframeSrc + \'}\';
>
> return \_iframeSrc + \'#嗅探\';
>
> }, \_host, \_ua);
>
> },

**25.4 注意事项**

> 📌 此方案已在 sexbjcam.com（host: playrecord.biz）和 kbjus.com
> 上验证可用，两种格式均能正确解析。
>
> 📌 embedHost 的作用：o 对象里的 hls3 URL 有时是相对路径（如
> /fid/hls3/\...），需要拼上 embed 页的域名才能播放。
>
> ⚠️ 不同视频可能使用不同
> CDN（playrecord.biz、podcastguestfinder.sbs、seamistboutiquecreations.cfd
> 等），域名会变但 URL 结构固定，Unpacker 方案不受域名影响。
>
> ⚠️ 第二十四章的 split 数组末尾方案（arr\[-1\]=key）对 hls3
> 格式无效，应优先使用本章的 Unpacker 方案，split 方案作为兜底。
>
> **第二十六章：\$ 工具函数完全参考**

\$
是海阔视界/聚阅内置的核心工具命名空间，提供动态规则生成、模块化开发、交互弹窗等功能。本章内容整合自官方《海阔视界工具文档》，与手册其他章节形成互补------其他章节侧重规则结构与实战铁律，本章专注于
\$ 命名空间下所有可用方法的完整语法与参数说明。

**26.1 模块化开发（\$.exports + \$.require）**

\$.exports 和 \$.require
是海阔视界的模块系统，用于将规则代码拆分为多个依赖页面，实现逻辑复用。

  ----------------------------------------------------------------------------------------------------------------------
  **方法**             **说明**                               **注意事项**
  -------------------- -------------------------------------- ----------------------------------------------------------
  \$.exports           导出模块，默认值为                     赋值必须在页面顶层执行，不能延迟。
                       {}。在依赖页面中赋值，供其他页面通过   
                       \$.require() 引入。                    

  \$.require(path,     导入模块。先执行路径文件代码，再返回   调用 \$.exports
  param, headers,      \$.exports。若路径以 .json 结尾，自动  中的函数时，无法直接使用函数外部变量，必须通过传参传入。
  time)                JSON.parse 后赋给 \$.exports。         

  \$.importRequire()   将模块引入到指定的 scope 环境中运行。  较少使用，适合沙箱隔离场景。
  ----------------------------------------------------------------------------------------------------------------------

依赖页面（hiker://page/utils）：

> // 依赖页面中定义并导出
>
> \$.exports.formatTitle = function(title) {
>
> return title.replace(/\\s+/g, \'\').trim();
>
> };
>
> \$.exports.buildUrl = function(host, path) {
>
> return host + path;
>
> };

主规则中引入使用：

> var utils = \$.require(\'hiker://page/utils\');
>
> var title = utils.formatTitle(rawTitle); // ✅ 正确
>
> // ❌ 错误：utils 内部函数不能访问外部的 host 变量
>
> // 必须通过参数传入：utils.buildUrl(host, path)
>
> 📌 \$.require 适合封装 \_base64Decode、\_autoDeshell
> 等工具函数，在多个规则之间复用，避免每个规则都粘贴一遍。

**26.2 动态规则生成（核心方法）**

\$ 最核心的能力是生成动态 URL
字符串，驱动页面跳转、懒加载解析、图片解密。以下四个方法构成了海阔视界规则的骨干。

  -------------------------------------------------------------------------------------------
  **方法**              **用途**                       **典型场景**
  --------------------- ------------------------------ --------------------------------------
  \$().rule(func,       生成用于新开页面的 URL         CF
  \...args)             字符串。func                   过盾沙箱页、小说正文页、自定义设置页
                        在新页面中执行，通常用         
                        setResult() 设置内容。         

  \$().lazyRule(func,   生成懒加载解析                 列表页构建 url
  \...args)             URL。用户点击后才执行          字段时，延迟提取真实视频地址
                        func，返回值作为真实 URL（如   
                        video://）。                   

  \$().image(func,      生成图片 URL。func 中 input    防盗链封面、Base64 解码图片、AES
  \...args)             为图片标识，必须返回           加密图片解密
                        InputStream，引擎自动渲染。    

  \$().x5Lazy() /       生成网页资源嗅探 URL，分别使用 无法直接解析的视频页，嗅探兜底
  \$().webLazy()        X5 内核和 Webkit               
                        内核加载页面并提取视频地址。   
  -------------------------------------------------------------------------------------------

▶ \$().lazyRule 详解

> 📌 Law 2（作用域隔离）：lazyRule
> 内部是独立作用域，外部变量一律无法访问。所有需要的值必须通过参数显式传入------包括
> host、UA、任何局部变量。
>
> ⚠️ lazyRule
> 参数只能传基础类型（字符串、数字、布尔）。传入对象时会被序列化为
> \"\[object Object\]\"，导致请求头失效（Law 52）。
>
> // ✅ 正确：只传字符串，内部自己拼 headers
>
> url: \$(\'\').lazyRule(function(link, ua, host) {
>
> var html = fetch(link, {
>
> headers: { \'User-Agent\': ua, \'Referer\': host + \'/\' }
>
> });
>
> var m = html.match(/https?:\\/\\/\[\^\\s\'\"\]+\\.m3u8/);
>
> if (m) return \'video://\' + m\[0\] + \';{Referer@\' + host + \'/}\';
>
> return link + \'#嗅探\';
>
> }, detailUrl, parse.UA, parse.host)
>
> // ❌ 错误：传对象，headers 变成字符串 \"\[object Object\]\"
>
> var headers = { \'User-Agent\': ua };
>
> url: \$(\'\').lazyRule(function(link, reqHeaders) {
>
> fetch(link, { headers: reqHeaders }); // reqHeaders 是字符串！
>
> }, link, headers)

▶ \$().image 详解（封面解密标准方案）

\$().image() 的 func 中，input 是拼接在 \$()
括号内的字符串（如封面路径）。func 必须返回
InputStream，引擎才能渲染图片。这是处理防盗链封面和 Base64
编码封面的唯一正确方案（手册 Law 28）。

> // 通用封面解密方案（base64 txt 文件）
>
> \_getImageDecodeRule: function(tupath, host) {
>
> return \$(\'\').image(function(tp, h) {
>
> // input = 封面路径（\$() 括号内的字符串）
>
> var txtUrl = tp + \'cover_64_2/\' + input + \'.txt\';
>
> var b64 = fetch(txtUrl, {
>
> headers: { \'Referer\': h + \'/\' }
>
> });
>
> if (!b64) return null;
>
> b64 = b64.replace(/\\\\/g, \'/\').trim();
>
> var raw = android.util.Base64.decode(
>
> b64, android.util.Base64.DEFAULT
>
> );
>
> var FileUtil = com.example.hikerview.utils.FileUtil;
>
> return FileUtil.toInputStream(raw); // ✅ 必须返回 InputStream
>
> }, tupath, host);
>
> },
>
> // 使用：提前调用一次，循环内复用
>
> var imgLazy = this.\_getImageDecodeRule(cfg.tupath, this.host);
>
> for (var i = 0; i \< items.length; i++) {
>
> d.push({
>
> pic_url: coverPath + imgLazy, // ✅ 直接拼接
>
> });
>
> }
>
> ⚠️ \$().image()
> 每次调用都生成一段新字符串，严禁在循环内每次调用。必须提前 var imgLazy
> = this.\_getImageDecodeRule() 调用一次，循环内直接用 imgLazy（与
> \_getVideoRule 同理，参见第十七章 Law 说明）。
>
> 📌 \$().image 与 \$(url).image({image:fn}) 的区别：\$().image(fn,
> \...args) 是工具文档中的函数式写法，func
> 通过参数传值；\$(url).image({image:fn})
> 是手册第十章的对象式写法，两者功能等同，选一种风格保持一致即可。

**26.3 \$().b64()------Base64 编解码辅助**

\$().b64() 对字符串进行 Base64 编解码，主要用于解决 lazyRule
等方法无法嵌套使用时的传值问题。当需要把一段含特殊字符的字符串安全地传入
lazyRule 时，可以先 b64 编码，内部再解码。

> // 场景：把含特殊字符的 JSON 字符串安全传入 lazyRule
>
> var encoded = \$(\'\').b64(JSON.stringify(configObj)); // 编码
>
> url: \$(\'\').lazyRule(function(enc) {
>
> var cfg = JSON.parse(\$(\'\').b64(enc, true)); // 第二参数 true = 解码
>
> // 使用 cfg\...
>
> }, encoded)

**26.4 辅助工具方法**

  ---------------------------------------------------------------------------------------------------------------
  **方法**              **说明**                                                   **示例**
  --------------------- ---------------------------------------------------------- ------------------------------
  \$.type(param)        比 typeof 更强大的类型判断，可识别 Array、null、Date 等。  \$.type(\[\]) ===
                                                                                   \'array\'（typeof \[\] 返回
                                                                                   \'object\'）

  \$.dateFormat(date,   日期格式化。text 支持 yyyy/MM/dd/HH/mm/ss 占位符。         \$.dateFormat(new Date(),
  text)                                                                            \'yyyy-MM-dd HH:mm\')

  \$.extend             扩展 \$                                                    \$.extend.myHelper =
                        工具（添加静态属性或方法），仅在当前规则生命周期内有效。   function(){\...}
  ---------------------------------------------------------------------------------------------------------------

**26.5 交互类方法（弹窗）**

以下方法触发系统级交互弹窗，通常用在按钮的 url 字段中，通过 \$().rule 或
\$().lazyRule 包裹调用。注意：这些是 \$
基础层的轻量弹窗，功能比第二十一章 hikerPop 简单，但无需额外依赖。

  ------------------------------------------------------------------------------------------
  **方法**                **说明**                          **典型用法**
  ----------------------- --------------------------------- --------------------------------
  \$().input(title, hint, 弹出单行输入框。用户确认后执行    搜索框、自定义 Cookie 输入
  func)                   func，input 为用户输入内容。      

  \$().confirm(title,     弹出确认/取消框。confirm/cancel   删除操作二次确认、清空缓存确认
  content, confirm,       为对应回调函数。                  
  cancel)                                                   

  \$(options,             弹出下拉/网格选择框。options      多线路切换、分辨率选择、源切换
  columns).select(func)   为字符串数组，columns             
                          控制列数，func 中 input           
                          为选中值。                        
  ------------------------------------------------------------------------------------------

> // 示例：用 \$().select 实现源切换
>
> d.push({
>
> title: \'🔄 切换源\',
>
> col_type: \'text_1\',
>
> url: \$(\[\'源一\|https://site1.com\', \'源二\|https://site2.com\'\],
> 1)
>
> .select(function() {
>
> // input = 用户选中的字符串，如 \'源一\|https://site1.com\'
>
> input = input.replace(/🌸/g, \'\'); // 去除已选标记
>
> setItem(\'current_source\', input);
>
> refreshPage(true);
>
> return \'toast://已切换: \' + input.split(\'\|\')\[0\];
>
> })
>
> });

**26.6 rule / lazyRule / image / x5Lazy 选型速查**

  ------------------------------------------------------------------------------------------------------------
  **方法**          **执行时机**                       **返回值要求**              **适用场景**
  ----------------- ---------------------------------- --------------------------- ---------------------------
  \$().rule()       用户点击后，新开页面时执行         内部调用 setResult(d)       自定义内容页、CF
                                                                                   沙箱、小说正文

  \$().lazyRule()   用户点击后立即执行，返回值作为新   返回 URL                    列表页懒加载解析视频/图集
                    URL                                字符串（video://、pics://   
                                                       等）                        

  \$().image()      图片加载时执行（异步）             必须返回 InputStream        防盗链封面、Base64/AES
                                                                                   图片解密

  \$().x5Lazy()     X5 WebView 加载并嗅探              自动提取页面内视频地址      JS 渲染页面，无法直接 fetch
                                                                                   解析时
  ------------------------------------------------------------------------------------------------------------

**26.7 本章相关铁律速查**

  ---------------------------------------------------------------------------------------
  **编号**   **铁律名称**         **核心要义**                   **关键口诀**
  ---------- -------------------- ------------------------------ ------------------------
  Law 2      作用域隔离           lazyRule/rule/image            显式传参：fn(a,b,c),
                                  内通过参数传值，禁用 this      val1, val2, val3

  Law 26     传参透传             \$().lazyRule 显式传参，不能用 不能用 this，必须传参
                                  this                           

  Law 28     字节流图片           图片走 FileUtil.readBytes +    \$().image({image:fn})
                                  toInputStream                  或 \$().image(fn)

  Law 34     ES5 全量降级         lazyRule/rule/image 内禁用     全量 var + function(){}
                                  let/const/=\>/?.等             

  Law 52     lazyRule 禁传对象    只传字符串/数字/布尔，对象变   headers 在 lazyRule
                                  \[object Object\]              内自己拼
  ---------------------------------------------------------------------------------------

> 📌 本章知识来源：整合自语雀《海阔视界工具文档》（\$.exports /
> \$.require / \$.type / \$().rule / \$().lazyRule / \$().image
> 等官方说明），与手册其他章节的实战代码形成互补。
>
> **第二十七章：Next.js 站点实战------加密视频解析与分类独立页**

本章基于 rou.video（Next.js SSR + 客户端渲染）规则开发过程提炼，记录了
\$(url).lazyRule 触发机制、Rhino 环境下 Base64
解密、多字节字节流操作等实战踩坑，以及分类独立标签页的标准写法。

**27.1 \$(url).lazyRule 触发机制------字符串拼接不触发**

lazyRule 必须用 \$(detailUrl).lazyRule(fn, \...args)
形式，引擎才会识别并在点击时触发。直接把 lazyRule 字符串拼接到 URL
后面，引擎会把整体当普通 URL 处理，直接跳转网页。

> ⚠️ 致命错误：url + \$(\'\').lazyRule() 字符串拼接，引擎不触发
> lazyRule，直接跳网页。（Law 61）
>
> // ❌ 错误：字符串拼接，lazyRule 不触发
>
> url: host + \'/v/\' + id + \$(\'\').lazyRule(function(ua) {\...}, ua)
>
> // ✅ 正确：\$(detailUrl).lazyRule()，input = detailUrl
>
> var detailUrl = host + \'/v/\' + id;
>
> url: \$(detailUrl).lazyRule(function(ua, host) {
>
> // input 就是 detailUrl
>
> var html = fetch(input, { headers: { \'User-Agent\': ua } });
>
> // \...
>
> }, \_ua, \_host)

**27.2 Rhino 无 atob------用 android.util.Base64 替换**

Rhino JS 引擎（ES5）没有浏览器环境的 atob()/btoa() 函数，调用会报
ReferenceError: \"atob\" 未定义。必须用 Android 原生 Base64 类替换。

> ⚠️ Rhino 引擎不支持 atob()/btoa()，调用直接报 ReferenceError。（Law
> 62）
>
> // ❌ 错误：Rhino 没有 atob
>
> var decoded = atob(base64Str);
>
> // ✅ 正确：android.util.Base64 + java.lang.String
>
> var raw = android.util.Base64.decode(base64Str,
> android.util.Base64.DEFAULT);
>
> var decoded = new java.lang.String(raw, \'UTF-8\') + \'\'; // + \'\'
> 强转 JS 字符串
>
> 📌 同理，btoa() 用 android.util.Base64.encodeToString(bytes,
> android.util.Base64.NO_WRAP) 替换。

**27.3 多字节字符解密------必须按字节操作，不能按字符 split**

站点 JS 的凯撒解密逻辑是：先 atob()
得到二进制字符串（每字符=1字节），再对每个字节的 charCode 减去偏移量
k。用 android.util.Base64 解码后得到的是 byte\[\]，如果先转字符串再
.split(\'\') 切割，多字节 UTF-8 字符会被拆散，导致解密结果乱码（出现 ￣
等乱码字符）。正确做法是直接对 byte\[\] 按字节减 k，再整体转字符串。

> ⚠️ 对多字节编码字符串按字符 split(\'\') 再操作
> charCode，会拆散多字节序列导致乱码。必须在字节层面操作。（Law 63）
>
> // ❌ 错误：先转字符串再 split，多字节字符被拆散
>
> var str = new java.lang.String(bytes, \'UTF-8\') + \'\';
>
> var dec = str.split(\'\').map(function(c) {
>
> return String.fromCharCode(c.charCodeAt(0) - k);
>
> }).join(\'\');
>
> // ✅ 正确：直接在 byte\[\] 上操作，每字节减 k，再整体转字符串
>
> var k = \_ev.k & 0xff;
>
> for (var i = 0; i \< bytes.length; i++) {
>
> bytes\[i\] = ((bytes\[i\] & 0xff) - k + 256) & 0xff;
>
> }
>
> var dec = new java.lang.String(bytes, \'UTF-8\') + \'\';
>
> 📌 & 0xff 把 Java 有符号 byte（-128\~127）转为无符号值，+ 256 再 &
> 0xff 防止减法结果为负数溢出。

**27.4 Next.js \_\_NEXT_DATA\_\_ 加密字段解析完整方案**

部分 Next.js 站点为防止直接爬取视频地址，在 \_\_NEXT_DATA\_\_ 的
pageProps 中不直接暴露视频 URL，而是放一个加密字段（如 ev.d +
ev.k）。解密逻辑藏在 JS bundle 中，需逆向分析后在 lazyRule 内复现。

以 rou.video 为例，解密链路如下：

> // 1. fetch 拿 SSR HTML（\_\_NEXT_DATA\_\_ 是服务端渲染，fetch
> 即可，无需 WebView）
>
> var html = fetch(input, { headers: { \'User-Agent\': ua, \'Referer\':
> host + \'/\' } });
>
> // 2. 提取 \_\_NEXT_DATA\_\_
>
> var dm = html.match(/\<script id=\"\_\_NEXT_DATA\_\_\"
> type=\"application\\/json\"\>(\[\\s\\S\]\*?)\<\\/script\>/);
>
> var pp = JSON.parse(dm\[1\]).props.pageProps;
>
> // 3. 取加密字段 ev.d（base64）和 ev.k（偏移量）
>
> var ev = pp.ev; // { d: \'base64\...\', k: 36 }
>
> // 4. Base64 解码得到 byte\[\]
>
> var bytes = android.util.Base64.decode(ev.d,
> android.util.Base64.DEFAULT);
>
> // 5. 按字节减 k（凯撒解密）
>
> var k = ev.k & 0xff;
>
> for (var i = 0; i \< bytes.length; i++) {
>
> bytes\[i\] = ((bytes\[i\] & 0xff) - k + 256) & 0xff;
>
> }
>
> // 6. 转字符串并 JSON.parse
>
> var obj = JSON.parse(new java.lang.String(bytes, \'UTF-8\') + \'\');
>
> // 7. 从 URL 字段推导真实视频地址
>
> // videoUrl 是封面图（index.jpg），替换扩展名得到 m3u8
>
> var src = obj.videoUrl.replace(\'index.jpg\', \'index.m3u8\');
>
> return \'video://\' + src + \';{User-Agent@\' + ua + \'&&Referer@\' +
> host + \'/}\';
>
> 📌 \_\_NEXT_DATA\_\_ 是 Next.js SSR 注入的 JSON，用普通 fetch
> 就能拿到，不需要 fetchCodeByWebView。需要 WebView
> 的是客户端渲染（CSR）页面------没有 \_\_NEXT_DATA\_\_ 或数据在 JS
> 执行后才有。

**27.5 从 URL 结构推导视频地址------扩展名替换**

部分站点的封面图 URL 和视频 m3u8 URL 只有文件名不同，路径和 token
参数完全一样。这种情况可以直接替换扩展名，完全跳过 lazyRule
内的额外请求，是最快的解析方案。

> // 封面:
> https://cdn.example.com/hls/xxxid/xxxid-720/index.jpg?exp=\...&auth=\...
>
> // 视频:
> https://cdn.example.com/hls/xxxid/xxxid-720/index.m3u8?exp=\...&auth=\...
>
> var src = obj.videoUrl.replace(\'index.jpg\', \'index.m3u8\');
>
> // ✅ auth/exp token 不变，直接可用
>
> 📌 发现这个规律的方法：打开浏览器开发者工具 Network 面板，过滤
> .m3u8，看 CDN
> 路径是否和封面图同目录。如果是，直接替换文件名，无需额外请求（参见第二十章
> 20.4 节）。

**27.6 分类独立标签页------\$(\'hiker://empty##fypage\').rule()
翻页方案**

当分类数量多、结构复杂，不适合用 fyclass
静态分类时，用独立标签页方案：主页展示分类列表，每个 tag
点击后新开独立页面，页面内部用 MY_PAGE 翻页。

> // 分类按钮
>
> d.push({
>
> title: cat.name,
>
> col_type: \'scroll_button\',
>
> url: \$(\'hiker://empty##fypage\').rule(function(tid, host, ua) {
>
> // MY_PAGE 自动注入，支持翻页
>
> var d = \[\];
>
> var url = host + \'/t/\' + encodeURIComponent(tid);
>
> if (MY_PAGE \> 1) url += \'?page=\' + MY_PAGE;
>
> var html = fetch(url, { headers: { \'User-Agent\': ua } });
>
> // \... 解析列表 \...
>
> setResult(d); // ✅ \$().rule 内用 setResult，不 return
>
> }, cat.id, host, ua)
>
> });
>
> ⚠️ \$().rule() 内必须用 setResult(d)，不能 return d。\$().lazyRule()
> 才是 return 字符串。（Law 47 延伸）

**27.7 本章新增铁律（Law 61-63）**

  ------------------------------------------------------------------------------
  **编号**   **铁律名称**         **核心要义**             **关键口诀**
  ---------- -------------------- ------------------------ ---------------------
  Law 61     lazyRule 触发方式    必须用                   拼接 =
                                  \$(url).lazyRule()       跳网页，\$(url) =
                                  形式，字符串拼接不触发   触发

  Law 62     禁用 atob/btoa       Rhino 无 atob/btoa，用   atob →
                                  android.util.Base64      Base64.decode；btoa →
                                                           encodeToString

  Law 63     字节层面解密         多字节编码字符串必须在   & 0xff
                                  byte\[\] 上操作，不能    转无符号，字节减
                                  split 后操作 charCode    k，整体转字符串
  ------------------------------------------------------------------------------

> 📌 本章内容来自 rou.video（Next.js SSR）规则开发实战，适用于所有使用
> \_\_NEXT_DATA\_\_ 加密字段的 Next.js 站点。
>
> **第二十八章：静态分类多维筛选补充------URL 模板四种写法**

第二章已介绍静态分类的五维占位符和矩阵方案的核心逻辑。本章作为补充，整理四种常见
URL 模板写法和注意事项，供快速查阅。

**28.1 四种 URL 模板写法**

根据站点 URL 结构不同，静态分类的 url 字段有以下四种常见形态：

▶ 写法一：标准 MACCMS 路径型

> \"静态分类\": {
>
> \"type\": \"主页\",
>
> \"url\":
> \"https://example.com/vod-show/fyclass-fyarea-fysort\-\--fyyear\-\--fypage\-\--.html\",
>
> \"class_name\": \"电影&电视剧&综艺&动漫\", \"class_url\": \"1&2&3&4\",
>
> \"area_name\": \"全部&大陆&香港&台湾&美国&韩国&日本\",
>
> \"area_url\": \"&大陆&香港&台湾&美国&韩国&日本\",
>
> \"year_name\": \"全部&2026&2025&2024&2023\",
>
> \"year_url\": \"&2026&2025&2024&2023\",
>
> \"sort_name\": \"时间&人气&评分\", \"sort_url\": \"time&hits&score\"
>
> }

▶ 写法二：纯路径拼接型

> \"静态分类\": {
>
> \"type\": \"主页\",
>
> \"url\": \"https://example.com/list/fyclass/fyarea/fyyear\",
>
> \"class_name\": \"电影&电视剧&综艺&动漫\", \"class_url\": \"1&2&3&4\",
>
> \"area_name\": \"全部&大陆&香港&台湾\", \"area_url\":
> \"all&mainland&hongkong&taiwan\",
>
> \"year_name\": \"全部&2026&2025&2024\", \"year_url\":
> \"all&2026&2025&2024\"
>
> }

▶ 写法三：单维仅分页型

> \"静态分类\": {
>
> \"type\": \"主页\",
>
> \"url\": \"https://example.com/type/fyclass/page/fypage\",
>
> \"class_name\": \"推荐&最新&热门&排行\",
>
> \"class_url\": \"recommend&latest&hot&rank\"
>
> }

▶ 写法四：查询参数型（API 站点）

> \"静态分类\": {
>
> \"type\": \"主页\",
>
> \"url\":
> \"https://example.com/api?type=fyclass&area=fyarea&year=fyyear&sort=fysort&page=fypage\",
>
> \"class_name\": \"电影&电视剧&综艺&动漫\", \"class_url\": \"1&2&3&4\",
>
> \"area_name\": \"全部&大陆&香港&台湾\", \"area_url\":
> \"&mainland&hongkong&taiwan\",
>
> \"year_name\": \"全部&2026&2025&2024\", \"year_url\":
> \"&2026&2025&2024\",
>
> \"sort_name\": \"最新&最热\", \"sort_url\": \"new&hot\"
>
> }

**28.2 使用静态分类的主页函数写法**

启用静态分类后，引擎自动将占位符替换为用户选择的值，并把完整 URL 注入
MY_URL。主页函数只需直接用 MY_URL 请求即可，无需手动拼接分类参数。

> var parse = {
>
> \"页码\": { \"主页\": 1 },
>
> \"静态分类\": {
>
> \"type\": \"主页\",
>
> \"url\": \"https://example.com/list/fyclass_fypage.html\",
>
> \"class_name\": \"电影&电视剧&动漫\", \"class_url\": \"1&2&3\"
>
> },
>
> \"主页\": function() {
>
> var \_ua = this.UA;
>
> var \_url = MY_URL; // ✅ 引擎已自动替换所有占位符
>
> var \_html = fetch(\_url, { headers: { \"User-Agent\": \_ua } });
>
> var \_d = \[\];
>
> var \_items = pdfa(\_html, \"body&&.list-item\");
>
> for (var i = 0; i \< \_items.length; i++) {
>
> var \_t = \_items\[i\];
>
> \_d.push({
>
> title: pdfh(\_t, \"a&&Text\"),
>
> url: pd(\_t, \"a&&href\", this.host),
>
> pic_url: pd(\_t, \"img&&src\"),
>
> col_type: \"movie_3\"
>
> });
>
> }
>
> return \_d;
>
> }
>
> };
>
> 📌 静态分类的 type 必须为 \"主页\"，聚阅引擎通过 type
> 识别入口类型。fypage 由引擎自动管理，开发者只需在 url
> 模板中放置占位符即可，无需在函数内处理页码。
>
> ⚠️ name 与 url 数量必须严格一一对应（& 分隔），\"全部\" 选项对应的 url
> 留空字符串。数量不匹配会导致越界取到 undefined（Law 49 / Law 50）。
>
> **第二十九章：聚阅测试仓核心调用机制**

聚阅测试仓（聚阅）是阔界小程序的专用测试环境，所有源接口（JS
代码）都在壳子内解析执行。本章说明各入口函数的调用流程和数据结构规范，帮助开发者理解壳子如何驱动
parse 对象。

**29.1 一级（主页）调用流程**

> 用户打开小程序
>
> → yiji() 被调用
>
> → 读取当前选中的源接口信息 jkdata
>
> → 渲染顶部菜单栏（切源/频道/搜索/收藏/管理）
>
> → 调用 getYiData(\'主页\', jkdata, d) 加载主页内容
>
> → 内部执行 parse\[\'主页\'\]() 获取列表数据
>
> 📌 jkdata 包含接口的 name、url（代码文件路径）、type、group、id
> 等信息。getObjCode(jkdata, \'yi\') 解析出 parse
> 对象。静态分类由壳子自动渲染，开发者只需定义好字段即可。

**29.2 二级（详情页）调用流程**

> 用户点击列表项
>
> → erji() 被调用
>
> → 从 MY_PARAMS 获取附加信息（含 jkdata）
>
> → 执行 parse\[\'二级\'\](MY_URL) 获取详情数据
>
> → 渲染封面、简介、线路、选集

二级函数完整返回结构（手册第七章基础上的补充字段）：

> return {
>
> // ── 封面信息 ────────────────────────────────────────
>
> img: \'封面图片URL\', // 必填
>
> desc: \'简介文字\', // 可选
>
> detailurl: \'点击封面跳转URL\', // 可选
>
> detailObj: {}, // 可选，完整封面对象，优先级最高
>
> // ── 线路与选集（核心）──────────────────────────────
>
> line: \[\'线路1\', \'线路2\'\], // 线路名称数组
>
> list: \[ // 与 line 一一对应的二维数组
>
> \[ {title:\'第01集\', url:\'链接1\'}, {title:\'第02集\',
> url:\'链接2\'} \],
>
> \[ {title:\'第01集\', url:\'链接1\'}, {title:\'第02集\',
> url:\'链接2\'} \]
>
> \],
>
> // ── 高级控制字段 ────────────────────────────────────
>
> type: \'video\', // video/comic/novel/audio/aggregate
>
> noShow: { 封面:false, 简介:false, 排序:false, 选集:false },
>
> moreitems: \[\], // 选集上方扩展项
>
> extenditems: \[\], // 选集下方扩展项（图文详情常用）
>
> extra: {}, // 选集扩展对象
>
> novel: true, // 是否为小说模式
>
> // ── 分页支持 ────────────────────────────────────────
>
> pageparse: function(url) {}, // 分页动态解析函数
>
> listparse: function(lineid, linename) {} // 列表动态解析函数
>
> };
>
> ⚠️ line 与 list 必须严格一一对应，list
> 是二维数组（外层按线路，内层按集数）。line.length !== list.length
> 会导致线路错位（Law 51）。

**29.3 搜索 / 解析 / 最新 / 频道 / 换源调用流程**

  -------------------------------------------------------------------------------------------
  **入口**        **调用方式**                 **说明**
  --------------- ---------------------------- ----------------------------------------------
  搜索            parse\[\'搜索\'\](keyword)   用户输入关键词后触发，返回列表数组

  解析            parse\[\'解析\'\](url)       用户点击选集后触发，返回播放链接字符串

  最新            parse\[\'最新\'\]()          \"最新\"菜单触发，返回列表数组，支持分页

  频道            parse\[\'频道\'\]()          \"频道\"菜单触发，返回频道数据或执行频道函数

  换源            壳子重新加载接口代码         用户切换数据源后，壳子重新执行一级/二级
  -------------------------------------------------------------------------------------------

解析函数返回格式：

> // 视频模式：返回播放链接字符串
>
> return \'video://\' + url + \';{User-Agent@\' + ua + \'&&Referer@\' +
> host + \'/}\';
>
> // 小说模式：返回章节内容数组
>
> return \[{ title: \'第一章\', content: \'正文内容\...\' }\];
>
> // 嗅探兜底
>
> return url + \'#嗅探\';

**29.4 新增铁律（Law 64-66）**

  -----------------------------------------------------------------------------
  **编号**   **铁律名称**       **核心要义**                 **关键口诀**
  ---------- ------------------ ---------------------------- ------------------
  Law 64     静态分类 type 固定 静态分类.type 必须为         type 写错 =
                                \"主页\"，壳子通过 type      壳子不识别
                                识别入口                     

  Law 65     line/list 一一对应 line 与 list                 line.length ===
                                长度必须相等，list           list.length
                                是二维数组                   

  Law 66     页码字段联动       页码对象定义起始页，fypage   页码: { 主页: 1 }
                                由引擎自动递增               
  -----------------------------------------------------------------------------

> 📌
> 本章内容来源：聚阅测试仓（壳子）核心机制分析，适用于所有基于聚阅壳子开发的接口规则。
>
> **第三十章：base64 封面图处理------正确选型与 batchFetch 并发方案**

本章修正第二十六章关于 \$.image()
的部分描述，并总结封面图处理的正确选型原则。核心结论：引擎原生支持
data:image 字符串，无需 Java 解码；\$.image()
只用于需要额外处理（XOR/AES解密）的场景。

**30.1 引擎对 pic_url 的支持格式**

聚阅/海阔视界引擎的 pic_url
字段支持以下三种格式，直接赋值即可渲染，不需要任何额外处理：

  ---------------------------------------------------------------------------------------------------
  **格式**                     **示例**                            **说明**
  ---------------------------- ----------------------------------- ----------------------------------
  普通 HTTP URL                https://cdn.example.com/cover.jpg   最常见，直接请求图片

  data:image 字符串            data:image/webp;base64,UklGR\...    引擎原生支持，直接渲染，无需解码

  data:image（反斜杠分隔符）   data:image\\webp;base64,UklGR\...   引擎同样支持，replace(/\\\\/g,
                                                                   \'/\') 修正后更稳
  ---------------------------------------------------------------------------------------------------

> 📌 站点 txt 文件内容是 data:image\\webp;base64,\... 格式时，直接 fetch
> 取到字符串，replace(/\\\\/g, \'/\') 修正反斜杠后赋给 pic_url
> 即可。引擎会自动解码渲染，不需要 Java Base64 解码。

**30.2 \$.image() 的正确使用场景**

\$.image() 的 func 是图片加载完成后的处理管道，input 是引擎下载该 URL
后得到的
InputStream。只有在需要对图片字节流做额外变换时才用它，直接赋值能解决的一律不用。

  -----------------------------------------------------------------------
  **场景**                **是否需要       **方案**
                          \$.image()**     
  ----------------------- ---------------- ------------------------------
  普通图片 URL（HTTP）    ❌ 不需要        直接赋给 pic_url

  data:image base64       ❌ 不需要        replace(/\\\\/g, \'/\') 后赋给
  字符串                                   pic_url

  图片前 N 字节 XOR 加密  ✅ 需要          \$.image() +
                                           FileUtil.toBytes + 手动 XOR

  图片内容 AES 加密       ✅ 需要          \$.image() +
                                           CryptoUtil.AES.decrypt

  防盗链（需带 Referer）  ❌ 不需要        pic_url + \@Referer=xxx 或
                                           \@headers={} 格式
  -----------------------------------------------------------------------

> ⚠️ 不要把 data:image base64 字符串误当作「需要解密」的场景去用
> \$.image()。\$.image()
> 只处理「图片字节流需要变换」的情况，否则只会让代码复杂化并引入不必要的
> bug。

**30.3 batchFetch 并发方案------解决串行 fetch 资源占用高**

每张封面单独同步
fetch，一页24张卡片就是24次串行网络请求，严重阻塞列表构建。改用
batchFetch
并发拉取，引擎自动分批（最多16个/批），速度大幅提升，不阻塞主线程。

> // ❌ 旧方案：串行 fetch，24张=24次阻塞，资源占用高
>
> for (var i = 0; i \< items.length; i++) {
>
> var b64 = fetch(txtUrl, { headers: { \'Referer\': host + \'/\' } });
> // 阻塞
>
> \_b64Map\[vid\] = b64.replace(/\\\\/g, \'/\').trim();
>
> }
>
> // ✅ 新方案：batchFetch 并发，自动分批，不阻塞
>
> var \_tasks = \[\];
>
> for (var ti = 0; ti \< \_txtUrls.length; ti++) {
>
> \_tasks.push({ url: \_txtUrls\[ti\].url, options: { headers: {
> \'Referer\': host + \'/\' } } });
>
> }
>
> var \_results = batchFetch(\_tasks); // 最多16个/批，超出自动分批串行
>
> for (var ri = 0; ri \< \_results.length; ri++) {
>
> var b64 = \_results\[ri\];
>
> if (typeof b64 === &quot;string&quot; && b64.length \> 50) {
>
> \_b64Map\[\_txtUrls\[ri\].vid\] = b64.replace(/\\\\/g, \'/\').trim();
>
> }
>
> }
>
> // 构建卡片时直接取结果
>
> d.push({
>
> pic_url: \_b64Map\[\_vid\] \|\| \'\',
>
> // \...
>
> });
>
> 📌 batchFetch 并发上限为 16（手册第三章 3.3 节）。传入超过 16 个 URL
> 时自动分批串行执行。对于一页24张封面，实际是第一批16个并发 +
> 第二批8个并发，比串行快约 8-10 倍。

**30.4 完整封面处理流程（batchFetch 方案模板）**

> \_parseList: function(html, cfg) {
>
> var \_host = this.host;
>
> var \_tupath = (cfg && cfg.tupath) ? cfg.tupath : \'\';
>
> // Step1：解析封面路径映射
>
> var \_coverMap = {};
>
> var \_fcm = html.match(/fetchMultipleCover\\(\\\[(\[\^\\\]\]+)\\\]/);
>
> if (\_fcm) {
>
> var \_pairs = \_fcm\[1\].match(/\'(\[\^\'\]+)\'/g) \|\| \[\];
>
> for (var j = 0; j \< \_pairs.length; j++) {
>
> var \_pair = \_pairs\[j\].replace(/\'/g, \'\').split(\'\|\');
>
> if (\_pair.length === 2) {
>
> var \_vm = \_pair\[0\].match(/-(\\d+)\$/);
>
> if (\_vm) \_coverMap\[\_vm\[1\]\] = \_pair\[1\];
>
> }
>
> }
>
> }
>
> // Step2：收集所有 txt URL
>
> var \_txtUrls = \[\];
>
> var \_list = pdfa(html, \'body&&.cell\');
>
> for (var i = 0; i \< \_list.length; i++) {
>
> var \_vidm = \_list\[i\].match(/\\/(\\d+)\\.htm\$/);
>
> var \_vid = \_vidm ? \_vidm\[1\] : \'\';
>
> if (\_vid && \_tupath && \_coverMap\[\_vid\]) {
>
> \_txtUrls.push({ vid: \_vid, url: \_tupath + \'cover_64_2/\' +
> \_coverMap\[\_vid\] + \'.txt\' });
>
> }
>
> }
>
> // Step3：batchFetch 并发拉取
>
> var \_b64Map = {};
>
> if (\_txtUrls.length \> 0) {
>
> var \_tasks = \_txtUrls.map(function(item) {
>
> return { url: item.url, options: { headers: { \'Referer\': \_host +
> \'/\' } } };
>
> });
>
> var \_results = batchFetch(\_tasks);
>
> for (var ri = 0; ri \< \_results.length; ri++) {
>
> var \_b64 = \_results\[ri\];
>
> if (\_b64 && \_b64.length \> 50) {
>
> \_b64Map\[\_txtUrls\[ri\].vid\] = \_b64.replace(/\\\\/g,
> \'/\').trim();
>
> }
>
> }
>
> }
>
> // Step4：构建卡片，pic_url 直接赋 base64 字符串
>
> var d = \[\];
>
> for (var i = 0; i \< \_list.length; i++) {
>
> // \... 标题/时长解析 \...
>
> d.push({
>
> title: \_title,
>
> pic_url: (\_vid && \_b64Map\[\_vid\]) ? \_b64Map\[\_vid\] : \'\', //
> ✅ 直接赋值
>
> url: \$(detailUrl).lazyRule(fn, \_host, \_ua), // ✅ \$(url).lazyRule
>
> col_type: \'movie_3\'
>
> });
>
> }
>
> return d;
>
> },
>
> **第三十一章：新版海阔视界 ES6 兼容性实测报告**

新版海阔视界引擎声称支持约 90% 的 ES6
写法。本章记录实测验证结果（基于动漫站规则开发测试），明确哪些特性可用、哪些仍需回避，供后续规则开发参考。

**31.1 兼容性速查表**

  ---------------------------------------------------------------------------
  **ES6 特性**             **支持状态**   **说明 / 替代方案**
  ------------------------ -------------- -----------------------------------
  const / let              ✅ 支持        可完全替代 var

  箭头函数 () =\> {}       ✅ 支持        回调、map/filter
                                          等链式操作均可用。完整函数体 item
                                          =\> { } 和单参数简写 item =\> expr
                                          均支持，实测验证

  模板字符串 \`\${}\`      ✅ 支持        字符串拼接推荐使用

  .filter().map() 链式     ✅ 支持        数组处理可替代 for 循环

  对象属性简写 { x }       ✅ 支持        { title, url } 等简写正常

  方法简写 fn() {}         ❌ 不支持      必须写成 fn: function() {}

  展开运算符 \...          ❌ 不支持      数组合并用 .concat()，对象合并用
                                          Object.assign()

  参数解构 ({ name, id })  ❌ 不支持      改为 item =\> { const name =
                                          item.name }

  对象解构 const { a } =   ⚠️ 不稳定      建议逐行赋值：const a = obj.a
  obj                                     

  可选链 ?.                ⚠️ 不稳定      用 && 替代：obj && obj.prop

  空值合并 ??              ⚠️ 不稳定      用 \|\| 替代

  async / await            ❌ 不适用      引擎单线程模型，无实际并发效果
  ---------------------------------------------------------------------------

**31.2 推荐写法模板（ES6 安全子集）**

基于实测结果，以下是在新版引擎中既简洁又安全的写法组合：

> // ✅ 变量：const/let
>
> const host = this.host;
>
> let d = \[\];
>
> // ✅ 箭头函数 + 模板字符串
>
> const url = MY_PAGE \> 1 ? \`\${host}/list/\${MY_PAGE}.html\` :
> \`\${host}/list/\`;
>
> // ✅ 数组链式处理
>
> const cards = pdfa(html, \'.item&&li\')
>
> .filter(it =\> pdfh(it, \'a&&Text\') && pd(it, \'a&&href\'))
>
> .map(it =\> ({
>
> title: pdfh(it, \'a&&Text\'),
>
> url: pd(it, \'a&&href\'),
>
> col_type: \'movie_3\'
>
> }));
>
> // ✅ 数组合并：concat 而非 \...
>
> return d.concat(cards);
>
> // ❌ 不用方法简写
>
> // 主页() { } ← 报错
>
> // ✅ 用冒号函数
>
> 主页: function() { }
>
> // ❌ 不用参数解构
>
> // cats.forEach(({ name, id }) =\> { }) ← 报错
>
> // ✅ 先取属性
>
> cats.forEach(cat =\> {
>
> const name = cat.name;
>
> const id = cat.id;
>
> });

**31.3 lazyRule / rule 内部可以用完整 ES6**

lazyRule 和 rule 的 func 参数是独立编译的字符串，引擎对其 ES6
支持更完整。在 func
内部可以放心使用箭头函数、模板字符串、const/let、链式数组方法。

> 📌 parse 对象方法层（外层）受引擎限制较多；lazyRule/rule 的 func
> 内部（内层）ES6 支持更好。外层保守、内层可激进。
>
> ⚠️ 无论内层外层，展开运算符 \... 和方法简写 fn(){}
> 都不支持，一律避免。

**31.4 原 Law 34 更新**

原 Law 34「ES5 全量降级------禁用 let/const/=\>/?./ 全量 var +
function{}」在新版引擎已过时，更新如下：

  ------------------------------------------------------------------------------
  **编号**     **铁律名称**         **新内容**
  ------------ -------------------- --------------------------------------------
  Law          ES6 安全子集         const/let/箭头函数/模板字符串/链式数组方法
  34（更新）                        可用；方法简写/展开运算符/参数解构/?./??
                                    禁用；方法必须写成 key: function() {}
                                    形式。箭头函数完整体 item =\> { } 与简写
                                    item =\> expr 均实测可用。⚠️ window/document
                                    等引擎内置名禁止用 let/const
                                    重声明，必须保持 var。

  ------------------------------------------------------------------------------

> 📌
> 本章结论基于新版海阔视界引擎实测（2026年），旧版引擎或聚阅低版本仍需遵守原
> Law 34 全量 ES5。建议开发时标注目标引擎版本。
>
> **⚡ Law 68（开发规范）：所有规则统一使用 ES6
> 安全子集开发。即：const/let 替代 var，箭头函数替代回调
> function，字符串保持 + 拼接，方法写成 key: function() {}
> 形式，window/document 等引擎内置名保持 var
> 声明。禁止使用方法简写、展开运算符、模板字符串（可选）、可选链 ?.
> 、空值合并 ?? 。**

**31.5 Rhino 引擎 ES6 支持详表（基于 mozilla/rhino 官方仓库）**

来源：github.com/mozilla/rhino，纯 Java 实现的 JavaScript 引擎，最新版本
1.9.0（2025-12）。海阔视界/聚阅内置 Rhino
版本决定实际可用特性，以下为官方各版本演进。

**31.5.1 版本演进路线**

  --------------------------------------------------------------------------------------------------------------
  **版本**   **发布时间**   **ES6 关键变化**
  ---------- -------------- ------------------------------------------------------------------------------------
  1.7.13     2020-09        function\* 生成器、for-of 遍历 Java Iterable

  1.7.14     2022-01        Promise、BigInt、模板字面量、\*\* 指数运算符、shorthand
                            属性、Object.values/entries/fromEntries、globalThis

  1.7.15     2024-05        rest 参数、Symbol.species、属性排序修正

  1.8.0      2025-01        默认语言级别改为 ES6、super、Reflect、Proxy

  1.9.0      2025-12        解构增强、展开增强、正则命名捕获组/后行断言/Unicode模式、Promise.withResolvers/try
  --------------------------------------------------------------------------------------------------------------

**31.5.2 特性支持状态（基于 1.8.0+）**

  -----------------------------------------------------------------------
  ✅ 完整支持（1.8.0+）               ❌ / ⚠️ 不支持或未确认
  ----------------------------------- -----------------------------------
  ✅ let / const 块级作用域           ❌ class 语法（不稳定）

  ✅ 箭头函数（含 lexical this）      ❌ async / await

  ✅ 模板字面量 / 标签模板            ❌ import / export（ES Modules）

  ✅ 解构赋值（数组/对象/参数）       ⚠️ 可选链 ?. （未确认）

  ✅ 默认参数 / rest 参数             ⚠️ 空值合并 ?? （未确认）

  ✅ 展开语法 \... （数组/可迭代）    

  ✅ for\...of 循环                   

  ✅ Promise / BigInt                 

  ✅ Reflect / Proxy / super          

  ✅ 计算属性名 / 简写属性/方法       
  -----------------------------------------------------------------------

**31.5.3 image/lazyRule 回调的箭头函数陷阱**

虽然 Rhino 1.8.0+ 完整支持箭头函数，但在 \$.image() 和 \$.lazyRule()
回调中，框架通过 input 变量注入数据，箭头函数会导致 input
取不到（作用域问题），必须使用 function(){}。

**⚠️ \$.image() 和 \$.lazyRule() 回调必须用
function(){}，不能用箭头函数，否则 input 为 undefined。**

> // ❌ 错误：箭头函数导致 input 取不到
>
> return \$(\'\').image(() =\> { /\* input 是 undefined \*/ });
>
> // ✅ 正确：function 回调
>
> return \$(\'\').image(function() { /\* input 正常注入 \*/ });

**31.5.4 开发建议**

📌 海阔内置 Rhino
版本可能滞后于官方最新版。以下特性已实测可靠可用：const/let、箭头函数（非回调场景）、模板字面量、解构、展开、for\...of、Promise。

**⚠️ class / async-await / import-export / ?. / ??
等特性不确定，规则内禁止使用，统一按不可用处理。**

**Law 68 补充：image/lazyRule 回调固定使用
function(){}；其余位置（forEach、map、filter 等）可自由使用箭头函数。**

> **第三十二章：分类方案选型------rule() 沙箱限制与矩阵方案适用边界**

本章补充第六章（矩阵方案）和第二十七章（独立标签页方案）的适用边界说明。核心结论：rule()
沙箱内的 http 链接永远跳浏览器，有二级的视频站必须用矩阵方案。

**32.1 rule() 沙箱的硬限制**

\$(\'hiker://empty##fypage\').rule(fn)
新开的页面是独立沙箱，引擎对沙箱内卡片 url 的处理规则与主页不同：

  ---------------------------------------------------------------------------------
  **卡片 url 类型**       **沙箱内行为**   **说明**
  ----------------------- ---------------- ----------------------------------------
  hiker:// 协议           ✅ 正常处理      hiker://empty、hiker://page/
                                           等内部协议正常

  pics:// 协议            ✅ 正常处理      图集/漫画直接渲染

  video:// 协议           ✅ 正常处理      直接播放

  http/https URL          ❌ 跳浏览器      引擎不知道走哪个函数，直接打开外部链接

  lazyRule 返回 http      ❌ 跳浏览器      lazyRule 返回 http url 同样跳浏览器
  ---------------------------------------------------------------------------------

> ⚠️ rule() 沙箱内的卡片 url 如果是 http
> 链接，无论如何包装都会跳浏览器。没有任何方法让沙箱内的 http url
> 走规则的二级函数。（Law 67）

**32.2 两种分类方案的适用场景**

  ---------------------------------------------------------------------------
  **方案**        **适用场景**                   **不适用场景**
  --------------- ------------------------------ ----------------------------
  独立标签页      漫画/图集（pics://）           有二级函数的视频站
  \$().rule(fn)   视频直链（video://） Next.js   需要封面/简介/选集的详情页
                  等无二级的站点                 

  矩阵方案        所有视频站（有二级）           基本无限制，通用性最强
  getMyVar +      动漫/影视等需要详情页的站点    
  主页刷新        任何需要走规则二级函数的场景   
  ---------------------------------------------------------------------------

**32.3 有二级的视频站标准分类方案（矩阵）**

有二级函数的站点，分类必须用矩阵方案------分类按钮在主页内渲染，用
getMyVar 记录选中状态，点按钮刷新主页，主页根据选中状态请求对应分类
URL，卡片直接给详情页地址让引擎走二级。

> // 页码必须开启翻页
>
> 页码: { \'主页\': true },
>
> 主页: function() {
>
> const d = \[\];
>
> const KEY = \'site_cindex\'; // 纯字母前缀（Law 51）
>
> const cur = getMyVar(KEY, \'0\');
>
> // 第一页渲染分类按钮行
>
> if (MY_PAGE == 1) {
>
> cats.forEach((cat, i) =\> {
>
> const isSel = (i + \'\' === cur);
>
> d.push({
>
> title: isSel ? \`❆ \${cat.name} ❆\` : cat.name,
>
> col_type: \'scroll_button\',
>
> url: \$(\`#noLoading#\`).lazyRule((k, idx) =\> {
>
> putMyVar(k, idx);
>
> refreshPage(false);
>
> return \'hiker://empty\';
>
> }, KEY, i + \'\')
>
> });
>
> });
>
> d.push({ col_type: \'blank_block\' });
>
> }
>
> // 根据选中分类请求列表
>
> const cat = cats\[parseInt(getMyVar(KEY, \'0\'))\] \|\| cats\[0\];
>
> const url = MY_PAGE \> 1
>
> ? \`\${host}/type\${cat.id}/\${MY_PAGE}.html\`
>
> : \`\${host}/type\${cat.id}\`;
>
> const html = fetch(url, { headers: { \'User-Agent\': ua } });
>
> // 卡片直接给详情页 url，引擎自动走二级函数
>
> return d.concat(parseCards(html));
>
> },
>
> 📌 矩阵方案的卡片 url
> 在主页上下文中，引擎认识当前规则的二级函数，点击会正常进入二级页面渲染封面/简介/选集。这是唯一能让分类列表正确走二级的方案。

**32.4 新增铁律（Law 67）**

  -------------------------------------------------------------------------------------------------
  **编号**   **铁律名称**         **核心要义**                                   **关键口诀**
  ---------- -------------------- ---------------------------------------------- ------------------
  Law 67     沙箱 http 跳浏览器   rule() 沙箱内 http url                         有二级 =
                                  永远跳浏览器，有二级的站点禁用独立标签页方案   矩阵方案；无二级 =
                                                                                 可用沙箱

  -------------------------------------------------------------------------------------------------

> 📌 本章结论来自动漫站（dk95.com）开发实测，经历了 rule() 沙箱 →
> rulePage → 矩阵方案的完整踩坑路径。

**第三十三章：API 接口型规则实战技巧（熊猫APP实战提炼）**

**33.1 POST body 直接传对象**

海阔/聚阅的 fetch 内部会自动处理对象 body，无需手动
JSON.stringify，直接传 JS 对象即可。这是 API 型规则最简洁的 POST 写法。

> // ✅ 直接传对象，无需 JSON.stringify
>
> let json = fetch(url, {
>
> body: { \"command\": \"GET_LIST\", \"pageNumber\": page, \"typeId\":
> fl },
>
> method: \'POST\'
>
> });

**33.2 fyAll 占位符 + \## 多段传参**

fyAll 会把所有筛选维度（class/area/year/sort）统一替换为同一个值，适合
API 只有一个分类 ID 参数的站点。配合 \## 分隔符在 url 字段中同时携带 API
地址和分类 ID，主页函数用 split(\"##\") 分别取出。

> // 静态分类写法
>
> 静态分类: {
>
> type: \"主页\",
>
> url: \"hiker://empty##https://api.example.com/forward##fyAll\",
>
> class_name: \"分类A&分类B\", class_url: \"1&2\",
>
> // 主页函数取值
>
> let apiUrl = MY_URL.split(\"##\")\[1\]; //
> https://api.example.com/forward
>
> let typeId = MY_URL.split(\"##\")\[2\]; // 1 或 2 等分类ID

**33.3 分类 ID 动态 type 判断**

用分类 ID 范围区分内容类型（如视频 vs 图集），一行三元表达式替代多个
if/else。

> let type = fl == 4 \| fl \> 29 ? 2 : 1; // ID=4 或 ID\>29
> 为图集(2)，其余为视频(1)

**33.4 lazyRule 传 API 地址参数**

lazyRule 内部无法访问外部变量，把 API 地址通过参数传入，input
留给列表项的唯一 ID，职责清晰。

> let lazy = \$(\'\').lazyRule((apiUrl) =\> {
>
> // input = 列表项 id，apiUrl = 外部传入的接口地址
>
> let json = fetch(apiUrl, { body: { \"id\": input }, method: \'POST\'
> });
>
> return \"pics://\" + JSON.parse(json).pics.join(\"&&\");
>
> }, apiUrl); // ← 把 apiUrl 作为参数传入

**33.5 封面图推导播放地址**

当封面图与播放地址同域名同路径时，直接替换文件名得到
m3u8，省去二级请求，效率最高。

> url: list\[i\].vod_pic.replace(\"/1.jpg\", \"/playlist.m3u8\")

**33.6 随机颜色函数**

用于 detail1/detail2
等标题的随机颜色渲染，每次调用返回一个随机十六进制颜色值。

> let Color = function() {
>
> return \'#\' + (\'00000\' + (Math.random() \* 0x1000000 \<\<
> 0).toString(16)).substr(-6);
>
> };

📌 本章技巧来自熊猫APP规则（2025年）实战提炼，适用于所有 POST JSON
接口型站点。

> **第三十四章：JS 混淆播放器解析\-\-\-\-\--Dean Edwards Packer
> 解包实战**

本章来自 91短视频类站点实战，总结双层 Packer
混淆播放器的完整解析流程，以及三个高价值开发技巧：多选择器降级容错、toast://
调试返回值、时间戳分段 token。

**34.1 Dean Edwards Packer 通用解包函数**

p,a,c,k,e,d 混淆是前端最常见的 JS
压缩方式，通过字典替换还原变量名。规则里需要手动在 lazyRule
内实现解包，不能用外部库。

> // ✅ 通用 Packer 解包，支持任意 a 进制
>
> function unpack(code) {
>
> let pm =
> code.match(/\\(\'(\[\\s\\S\]\*?)\',(\\d+),(\\d+),\'(\[\\s\\S\]\*?)\'\\.split\\(\'\\\|\'\\)/);
>
> if (!pm) return code;
>
> let p = pm\[1\], a = parseInt(pm\[2\]), c = parseInt(pm\[3\]), k =
> pm\[4\].split(\'\|\');
>
> function e(c) {
>
> return (c \< a ? \'\' : e(parseInt(c/a))) +
>
> ((c = c%a) \> 35 ? String.fromCharCode(c+29) : c.toString(36));
>
> }
>
> let d = {};
>
> while (c\--) { let key = e(c); d\[key\] = k\[c\] \|\| key; }
>
> return p.replace(/\\b\\w+\\b/g, w =\> d\[w\] !== undefined ? d\[w\] :
> w);
>
> }

**⚠️ lazyRule 内不能 require 外部库，解包函数必须内联定义在 lazyRule
内部。**

**34.2 双层 Packer 播放解析流程**

部分站点播放地址经过两层混淆：第一层 embed 页解包拿 token，第二层
embed_play.js 解包拿 m3u8。流程固定，可作为模板复用。

> // Step1：请求视频详情页，提取 videoId
>
> let html = request(input, { headers: { \'User-Agent\': ua } });
>
> let vidMatch = html.match(/videoId\\s\*=\\s\*\[\'\"\](
> \\d+)\[\'\"\]/);
>
> if (!vidMatch) return \'toast://未找到videoId\';
>
> // Step2：请求 embed 页，第一层解包取 token
>
> let embedUrl = home + \'/index/embed?id=\' + vidMatch\[1\];
>
> let embedHtml = request(embedUrl, { headers: { \'Referer\': input }
> });
>
> let layer1 = unpack(embedHtml);
>
> let tkMatch = layer1.match(/encodeURIComponent\\(\"(\[\^\"\]+)\"\\)/);
>
> if (!tkMatch) return \'toast://token失败\';
>
> // Step3：构造时间戳 token（每30分钟变化）
>
> let t = parseInt(Date.now() / 1000 / 1800);
>
> let playUrl = home + \'/index/embed_play.js?u=\' +
> encodeURIComponent(tkMatch\[1\]) + \'&t=\' + t;
>
> // Step4：请求 embed_play.js，第二层解包取 m3u8
>
> let resp = fetch(playUrl, { headers: { \'Referer\': embedUrl } });
>
> let layer2 = unpack(resp);
>
> let m3u8 =
> layer2.match(/https?:\\/\\/\[\^\\s\"\'\<\>\\\\\]+\\.m3u8\[\^\\s\"\'\<\>\\\\\]\*/i);
>
> if (m3u8) return m3u8\[0\].replace(/&amp;/g, \'&\');

**34.3 三个高价值开发技巧**

**技巧1：多选择器降级容错**

站点改版时列表结构可能变化，提前写好备用选择器，按顺序降级尝试，避免全量失效。

> // ✅ 三级降级，任一命中即停
>
> let items = pdfa(html, \'body&&.video-items li\');
>
> if (items.length === 0) items = pdfa(html, \'body&&li.video-item\');
>
> if (items.length === 0) items = pdfa(html, \'body&&.list li\');

**技巧2：toast:// 作为调试返回值**

解析失败时返回 toast://
弹出提示，用户能看到具体失败原因，比黑屏或空播放器更容易定位问题。

> // ✅ 各步骤返回不同提示，快速定位失败节点
>
> if (!vidMatch) return \'toast://未找到videoId\';
>
> if (!tkMatch) return \'toast://token提取失败\';
>
> if (!m3u8) return \'toast://未找到m3u8地址\';

**技巧3：时间戳分段 token（防盗链）**

部分站点用时间分段值作为请求验证参数，每 N 分钟变化一次。用 Date.now()
整除计算当前分段，无需额外请求。

> // 每30分钟变化一次的 t 参数
>
> let t = parseInt(Date.now() / 1000 / 1800); // 1800秒=30分钟
>
> // 每小时变化：/ 1000 / 3600，每10分钟变化：/ 1000 / 600

**34.4 新增铁律**

  ---------------------------------------------------------------------------------------------------------
  编号     名称               正确做法                                    错误现象
  -------- ------------------ ------------------------------------------- ---------------------------------
  Law 69   Packer解包内联     lazyRule内直接定义unpack函数，不能require   require外部库在lazyRule内不可用

  Law 70   toast://调试返回   解析各步骤失败返回toast://具体原因          黑屏无提示，难以定位问题

  Law 71   多选择器降级       按优先级列出3个备用选择器依次尝试           单一选择器，站点改版即全量失效
  ---------------------------------------------------------------------------------------------------------

> **第三十五章：多站点实战总结（禁漫/7mmtv/Pornhub/XVideos/MissAV）**

**35.1 scroll_button 不支持 HTML------Law 78**

**⚠️ Law 78：scroll_button 标题内禁止使用 fontcolor()/bold() 等 HTML
方法，会直接显示原始标签字符串。选中状态只能靠 backgroundColor +
纯文本符号区分。**

> // ❌ 错误：scroll_button 不渲染 HTML
>
> title: i === bigIdx ? \'\"\"\"\"\' +
> big.title.fontcolor(\"#FFFFFF\").bold() : big.title
>
> // ✅ 正确：纯文本符号 + backgroundColor
>
> title: i === bigIdx ? \"▶ \" + big.title : big.title,
>
> extra: { backgroundColor: i === bigIdx ? \"#FF6B6B\" : \"\" }

**35.2 CF 过盾两段式方案------Law 72**

访问 CF 站点先 fetch 带缓存 Cookie 尝试，失败则 fetchCodeByWebView
自动过盾，过盾后保存 Cookie 供下次复用。

**⚠️ Law 72：fetchCodeByWebView 的 UA 必须放在 headers 内；check
用纯文本关键词（不含引号）；javaScriptEnabled: true 必须加。参数名是
check 不是 checkJs。**

> html = fetchCodeByWebView(url, {
>
> headers: { \"User-Agent\": \_ua },
>
> check: \"页面特征文本\", // 纯文本，不含引号，不是 checkJs
>
> timeout: 25000,
>
> javaScriptEnabled: true // CF 验证必须
>
> });

**35.3 Packer 站点正则变量提取------Law 73**

7mmtv 播放列表在 Dean Edwards Packer
混淆块内。解包后用捕获组取数值，不要 eval 拼接变量声明。

> let src = (packedMatch ? unpack(packedMatch\[0\]) : \"\") + html;
>
> let \_hcdeedg252 = src.match(/hcdeedg252=(\\d+)/);
>
> let hcdeedg252 = parseInt(\_hcdeedg252\[1\]); // 直接用数值

**35.4 Pornhub 图片与播放地址------Law 74/75**

**⚠️ Law 74：Pornhub 列表图片优先用 default_thumb（ei.phncdn.com
稳定域名），thumb 字段是 pix-cdn 带鉴权时效参数会过期变空白。**

**⚠️ Law 75：Pornhub m3u8 地址 em-h.phncdn.com 需替换为
im-h.phncdn.com，截掉 ? 后面鉴权参数才能播放。**

> let img = v.default_thumb \|\| v.thumb \|\| \"\";
>
> url = url.replace(\"em-h.phncdn.com\", \"im-h.phncdn.com\");
>
> let q = url.indexOf(\"?\"); if (q \> -1) url = url.substring(0, q);

**35.5 MissAV surrit UUID 直取------Law 76**

**⚠️ Law 76：MissAV 原规则用 checkJs 参数名错误（应为 check），且两次
WebView 请求容易超时。正确做法：surrit UUID 直接写在 missav 页面 HTML
里，正则取出无需 WebView。**

> let uuidM =
> html.match(/surrit\\.com\\/(\[a-f0-9\]{8}-\[a-f0-9\]{4}-\[a-f0-9\]{4}-\[a-f0-9\]{4}-\[a-f0-9\]{12})/);
>
> let baseUrl = \"https://surrit.com/\" + uuidM\[1\] + \"/\";
>
> let m3u8 = request(baseUrl + \"video.m3u8\", { headers: { \"Referer\":
> baseUrl } });

**35.6 搜索函数禁用 MY_PAGE------Law 77**

**⚠️ Law 77：搜索函数内禁止使用 MY_PAGE，否则报错\"MY_PAGE
未定义\"。搜索直接拼 URL，不加翻页参数。**

> 搜索: function(key) {
>
> // ❌ let url = this.\_buildUrl(\...) // \_buildUrl 内用了 MY_PAGE
>
> // ✅
>
> let url = this.host + \"/search?q=\" + encodeURIComponent(key);
>
> return this.\_parse(fetch(url));
>
> },
>
> **第三十七章：多标签 categoryList
> 写法------与矩阵方案、静态分类的全面对比**

本章基于 Pornhub 规则（作者 R）的 categoryList
架构，深入分析其多标签写法，并与手册第六章（矩阵方案）、第二章（静态分类）做系统对比，帮助开发者在三种方案中准确选型。

**37.1 categoryList 核心架构解析**

**categoryList 是一个 JSON
数组，每项代表一个顶级标签页，包含四个字段：**

> // 每项的完整结构
>
> {
>
> \"title\": \"视频\", // 显示在顶部 scroll_button 的文字
>
> \"path\": \"\", // 无 sub 时直接用此路径；有 sub 时忽略
>
> \"type\": \"video\", // 内容类型标识，驱动 switch(type) 分发
>
> \"sub\": \[ // 子分类数组，空数组表示无子分类
>
> { \"title\": \"色情片\", \"path\": \"/video\" },
>
> { \"title\": \"🇨🇳中文\", \"path\": \"/language/chinese\" }
>
> \]
>
> }

六种内容类型（type）及其对应的渲染函数：

-   video → videoType(url, html) 普通视频列表

-   category → categoryType(html) 分类入口（大类 + 小类网格）

-   pornstar → pornstarType(url, html) 明星/演员卡片

-   channel → channelType(url, html) 频道列表

-   playlist → playlistType(url, html) 片单列表

-   login → loginType(html) 登录/我的页面，再按 url 特征二次分流

**37.2 两级标签的渲染逻辑**

第一级（顶栏）：遍历整个 categoryList，每项渲染一个
scroll_button，点击记录 Apo.category 并清空子分类状态，刷新主页。

第二级（子栏）：只有当前选中的 category 有 sub 时才渲染，遍历
currentCate.sub，记录 Apo.subCate，同样刷新主页。

> // 一级标签渲染
>
> categoryList.forEach((cate, index) =\> {
>
> parse.d.push({
>
> title: parseInt(parse.data.category) === index
>
> ? \'\"\"\"\"\' + cate.title.fontcolor(\"#FFFFFF\").bold()
>
> : cate.title,
>
> url: \$(parse.empty + \'#noLoading#\').lazyRule((index) =\> {
>
> putMyVar(\"Apo.category\", index.toString())
>
> putMyVar(\"Apo.subCate\", \"0\") // ✅ 切换一级时重置二级
>
> clearMyVar(\"url\")
>
> refreshPage(true)
>
> return \"hiker://empty\"
>
> }, index),
>
> col_type: \'scroll_button\',
>
> extra: {
>
> backgroundColor: parseInt(parse.data.category) === index
>
> ? parse.getRangeColors() : \'\'
>
> }
>
> })
>
> })
>
> // 二级标签：仅当 currentCate.sub.length \> 0 时渲染
>
> if (currentCate.sub.length \> 0) {
>
> parse.d.push({ col_type: \'blank_block\' })
>
> currentCate.sub.forEach((cate, index) =\> {
>
> parse.d.push({
>
> title: parseInt(parse.data.subCate) === index
>
> ? \'\"\"\"\"\' + cate.title.fontcolor(\"#FFFFFF\").bold()
>
> : cate.title,
>
> // \... 同一级，记录 Apo.subCate
>
> })
>
> })
>
> }

**37.3 switch(type) 分发------URL 判断二次路由**

switch 的第一层按 type 分流，但 login 类型内部还有 url
特征判断，实现了三层路由：

> switch (type) {
>
> case \'video\': parse.videoType(url, html); break
>
> case \'pornstar\':
>
> // url 含 search 时走视频列表，否则走明星卡片
>
> url.includes(\'search\') ? parse.videoType(url, html) :
> parse.pornstarType(url, html)
>
> break
>
> case \'login\':
>
> // url 特征二次分流：片单/登录/订阅 走不同函数
>
> if (url.includes(\'playlists\')) parse.playlistType(url, html)
>
> else if (url.includes(\'login\')) parse.loginType(html)
>
> else if (url.includes(\'subscriptions\'))parse.pornstarType(url, html)
>
> else parse.videoType(url, html)
>
> break
>
> // \...
>
> }
>
> setResult(parse.d) // ✅ switch 后统一调用（Law 80）
>
> 📌 login 类型的二次分流是关键：同一个「我的」入口，根据当前 url
> 的路径特征决定走登录页、片单页还是订阅页，一个 type
> 字段覆盖了多种实际内容类型。

**37.4 \$.require 模块化调用机制**

该规则大量使用 \$.require(\"jiekou\") 跨规则调用，共 8 处：

> // 视频卡片点击 → 调用 videoParse
>
> url: vurl + \$(\'#noHistory#\').rule(() =\> {
>
> const parse = \$.require(\"jiekou\").parse()
>
> parse.videoParse(MY_URL)
>
> setResult(parse.d)
>
> }),
>
> // 明星/频道/片单卡片点击 → 调用 yijiParse
>
> url: \"hiker://empty#\" + vurl + \$(\"/videos##fypage\").rule(() =\> {
>
> const parse = \$.require(\"jiekou\").parse()
>
> parse.yijiParse(MY_URL.split(\'##\')\[0\])
>
> setResult(parse.d)
>
> }),
>
> ⚠️ \$.require(\"jiekou\") 中的 \"jiekou\"
> 是规则在聚阅里保存的接口名称，不是文件名。如果规则没有保存为对应接口名，会报\"Module
> not found\"错误。
>
> ⚠️ 解决方案：把规则保存成接口名 \"jiekou\"，或者把调用改成
> \$.require(MY_RULE.find_rule)（但 MY_RULE.find_rule 在 rule()
> 沙箱里会返回整个代码字符串而非名称，同样失效）。最可靠的方案是把
> videoParse 逻辑内联进 rule() 里，完全不依赖 \$.require。

**37.5 三种方案全面对比**

三种方案对比：

**对比维度 \| 静态分类（第二章） \| 矩阵方案（第六章） \| categoryList
多标签（本章）**

-   【分类数据位置】 静态: 静态分类字段（class_name/url） \| 矩阵:
    主页函数内 dataClass 数组 \| categoryList: 主页函数内 categoryList
    数组

-   【分类维度】 静态: 1-4 维（fyclass/area/year/sort） \| 矩阵: 最多 9
    行独立筛选行 \| categoryList: 1级主标签 + 1级子标签，共2层

-   【内容类型分流】 静态: 无，所有分类走同一解析逻辑 \| 矩阵:
    无，所有分类走同一解析逻辑 \| categoryList: ✅ type 字段 + switch
    分发，不同标签走完全不同的解析函数

-   【URL 构建】 静态: 引擎自动替换占位符 \| 矩阵: 主页函数自己读
    cindex/curl 拼接 \| categoryList: 主页函数读 category/subCate 取
    path 拼接

-   【翻页】 静态: fypage 引擎自动管理 \| 矩阵: 主页函数自己处理 MY_PAGE
    \| categoryList: putMyVar(\'nextPage\') 手动追踪下一页 URL

-   【引擎依赖】 静态: 强依赖（占位符替换） \| 矩阵: 弱依赖（只用
    MY_PAGE） \| categoryList: 弱依赖（只用 MY_PAGE）

-   【代码复杂度】 静态: 低，配置式 \| 矩阵: 中，循环+渲染函数 \|
    categoryList: 高，多函数分工，需要 \$.require 或内联

-   【适用场景】 静态: URL 规律固定、分类维度少的普通视频站 \| 矩阵:
    分类极多、多维筛选、URL 不规律的复杂站 \| categoryList:
    内容类型多样（视频/明星/频道/片单/登录）的大型站

-   【主要优点】 静态: 零代码、配置即用、引擎全自动 \| 矩阵:
    灵活、支持多维叠加筛选 \| categoryList:
    类型分发清晰、各类型独立渲染、扩展性强

-   【主要缺点】 静态: 只能走同一解析函数，不能按分类切换内容类型 \|
    矩阵: 各分类仍走同一解析函数，不能按分类切换内容类型 \|
    categoryList: 代码量大，\$.require 跨规则调用有坑

**37.6 categoryList 方案的核心优势**

**相比静态分类和矩阵方案，categoryList 的最大价值在于：**

-   内容类型完全隔离：视频/明星/频道/片单各走独立的渲染函数，互不干扰。静态分类和矩阵方案所有分类都走同一个解析逻辑，无法做到这一点

-   登录态感知：login type 内部根据 url
    特征二次分流，登录前显示登录按钮，登录后显示个人内容（点赞/收藏/片单/订阅），其他方案无法实现

-   子分类动态展开：只有选中有 sub
    的主分类时才渲染子分类行，其他主分类不显示子分类，比矩阵方案的固定多行更灵活

-   可扩展性强：增加新内容类型只需在 categoryList 加一项、在 switch
    加一个 case、新增一个渲染函数，结构清晰

-   search 自动兼容：各 type 分支都有 url.includes(\'search\')
    判断，搜索结果统一走 videoType，无需单独处理

**37.7 categoryList 方案的局限与注意事项**

-   \$.require 依赖陷阱：rule() 沙箱内用 \$.require 调用
    videoParse/yijiParse，必须把规则保存为对应接口名才能找到。单规则直接跑会报
    Module not found（Law 67 延伸）

-   翻页非标准：用 putMyVar(\'nextPage\') 手动追踪下一页 URL，不依赖
    fypage。MY_PAGE \> 1 时直接用 nextPage，没有 nextPage
    则显示\'没有下一页\'，逻辑简单但不支持跳页

-   parse.data 每次刷新重新取值：data 对象在规则加载时执行
    getMyVar，主页刷新时规则重新加载，所以 category/subCate
    始终是最新值，这是正确的

-   parse.d 必须在入口清空：d 定义在顶层，主页/搜索入口必须先执行
    parse.d = \[\]（Law 79），否则刷新时数据累积

-   不适合有二级函数的普通视频站：如果站点只有一种内容类型（视频列表），用
    categoryList 是过度设计，静态分类或矩阵方案更简洁

**37.8 选型决策树**

> 问题1：站点有多种内容类型（视频/明星/频道/片单/登录）吗？
>
> → 是：用 categoryList + switch(type) 方案（本章）
>
> → 否：继续问题2
>
> 问题2：分类超过4维，或 URL 结构不规律，无法用占位符表达吗？
>
> → 是：用矩阵方案（第六章）
>
> → 否：继续问题3
>
> 问题3：分类数量少（每行 \< 20 个），URL 有固定规律吗？
>
> → 是：用静态分类（第二章）--- 最简单，零代码
>
> → 否：用矩阵方案（第六章）

**37.9 \$.require 内联方案------彻底解决 Module not found**

**当无法使用 \$.require 时，把 videoParse 逻辑直接内联进 rule()
是最可靠的方案：**

> // ✅ 完全内联，不依赖 \$.require，不依赖接口名
>
> url: vurl + \$(\'#noHistory#\').rule(() =\> {
>
> var ck =
> fetchPC(\'hiker://files/rules/parse/Cookie/pornhub_cookie.txt\') \|\|
> \'\';
>
> var ua = \'Mozilla/5.0 \...\';
>
> var html = fetch(MY_URL, { headers: { cookie: ck, \'User-Agent\': ua }
> });
>
> var d = \[\];
>
> // 标题
>
> d.push({ title: \'\"\"\"\"\' +
> pdfh(html,\'body&&h1&&Text\').fontcolor(\'\').small(),
>
> url: MY_URL, col_type: \'text_1\', extra: { lineVisible: false } });
>
> // flashvars 解析
>
> var playerObjList = {}, flashvars;
>
> try {
>
> var js = parseDomForHtml(html,
> \'\[id=mobileContainer\]&&script&&Html\');
>
> eval(js.replace(/var flashvars\_.\*?=/, \'var flashvars =\'));
>
> } catch(e) {}
>
> var urls = \[\], names = \[\];
>
> if (typeof flashvars !== \'undefined\' && flashvars &&
> flashvars.mediaDefinitions) {
>
> flashvars.mediaDefinitions
>
> .sort((a,b) =\> b.quality - a.quality)
>
> .forEach(item =\> {
>
> if (typeof item.quality === \'string\' && item.videoUrl) {
>
> urls.push(item.videoUrl.replace(/&amp;/g,\'&\'));
>
> names.push(item.quality + \'P\');
>
> }
>
> });
>
> }
>
> // 封面 + playlist
>
> d.push({ img: parseDom(html,\'#videoPlayerPlaceholder&&img&&src\')
> \|\| \'\',
>
> url: JSON.stringify({urls,names}), col_type: \'pic_1_full\' });
>
> setResult(d);
>
> }),
>
> ⚠️ Law 83（新增）：\$.require 在 rule()/lazyRule()
> 沙箱内调用时，接口名必须和规则在聚阅里保存的名称完全一致，不是文件名。无法确认接口名时，优先把逻辑内联进
> rule() 里，彻底消除依赖。

**37.10 本章新增铁律（Law 83）**

**Law 83 \$.require 接口名必须与聚阅保存名一致**

> ✅ 正确做法：规则保存为接口名 \'jiekou\'，代码里写
> \$.require(\'jiekou\')；或把逻辑内联进 rule() 彻底不用 \$.require
>
> ❌ 错误后果：接口名不匹配 → Module not found；用文件名（如
> 1754149556905）→ 同样找不到

**37.11 三种卡片跳转模式------不走规则二级函数**

categoryList 规则里所有卡片都不走规则的「二级函数」，而是用 \$().rule()
新开沙箱页面，在页面内部自己
setResult。这是该架构与手册标准骨架的最大差异之一。

**三种卡片跳转模式对比：**

> // ── 模式1：视频卡片 → 直接调 videoParse 渲染播放页 ──────────
>
> // videoType() 里的卡片
>
> url: vurl + \$(\'#noHistory#\').rule(() =\> {
>
> const parse = \$.require(\'jiekou\').parse()
>
> parse.videoParse(MY_URL) // 渲染标题+封面+清晰度选择+相关视频
>
> setResult(parse.d)
>
> }),
>
> // ── 模式2：明星/频道卡片 → 新开列表页，带 fypage 翻页 ────────
>
> // pornstarType / channelType 里的卡片
>
> url: \"hiker://empty#\" + vurl + \$(\"/videos##fypage\").rule(() =\> {
>
> const parse = \$.require(\'jiekou\').parse()
>
> parse.yijiParse(MY_URL.split(\'##\')\[0\]) // 渲染视频列表
>
> setResult(parse.d)
>
> }),
>
> // ── 模式3：分类/片单卡片 → 新开列表页，带 fypage 翻页 ─────────
>
> // categoryType / playlistType 里的卡片
>
> url: \"hiker://empty#\" + vurl + \$(\'##fypage\').rule(() =\> {
>
> const parse = \$.require(\'jiekou\').parse()
>
> parse.yijiParse(MY_URL.split(\'##\')\[0\])
>
> setResult(parse.d)
>
> }),

**三种模式的关键区别：**

-   模式1（videoParse）：\$(\'#noHistory#\').rule() --- #noHistory#
    不记录历史；无 hiker://empty# 前缀，直接在当前页覆盖渲染；调
    videoParse 展示播放页内容

-   模式2（yijiParse + /videos##fypage）：hiker://empty#
    前缀新开独立页面；/videos##fypage 表示路径补充 + 支持 fypage
    翻页；MY_URL.split(\'##\')\[0\] 取出真实 URL

-   模式3（yijiParse + ##fypage）：和模式2 相同，区别是 \##
    前无路径补充，直接用 vurl 本身作为列表 URL

> 📌 为什么不走规则二级函数？因为「二级函数」返回 {line, list}
> 格式，只能渲染选集列表。这里的明星/频道/分类点进去是「另一个视频列表页」而不是「选集页」，必须用
> \$().rule() + setResult 自己渲染，不能用二级函数。
>
> 📌 hiker://empty# 前缀的作用：告诉引擎新开一个独立页面，页面 URL 是 \#
> 后面的内容。没有这个前缀时（如模式1），rule()
> 的内容直接在触发页面覆盖渲染。
>
> ⚠️ 这三种模式都依赖 \$.require(\'jiekou\')，如果规则没有保存为接口名
> \'jiekou\'，全部报错。解决方案：把 videoParse 和 yijiParse
> 逻辑分别内联进对应的 rule() 里，或把规则保存为接口名（Law 83）。

**37.12 yijiParse 的翻页机制------手动追踪 nextPage**

yijiParse 没有用 fypage 占位符翻页，而是手动追踪下一页 URL：

> yijiParse: (url) =\> {
>
> // 翻页：从 getMyVar 取上次保存的下一页 URL
>
> let nextPage = getMyVar(\'YnextPage\', \'\')
>
> if (nextPage && MY_PAGE \> 1) {
>
> url = url.includes(\'random\') ? url : nextPage
>
> } else if (MY_PAGE \> 1) {
>
> url = \'没有下一页哦😑\' // 没有 nextPage 时硬终止
>
> }
>
> var html = fetch(url, { headers: { \... } })
>
> // 请求完成后，从页面提取下一页 URL 存起来
>
> try {
>
> var next = parse.url + pdfh(html,
> \'body&&.pagination&&.page_next&&a&&href\')
>
> putMyVar(\'YnextPage\', next)
>
> } catch {
>
> clearMyVar(\'YnextPage\') // 没有下一页按钮，清空
>
> }
>
> }

**与 fypage 方案的对比：**

-   fypage 方案：引擎用 MY_PAGE 自动计算页码，URL 模板固定（如
    /page/N/），适合 URL 规律的站点

-   nextPage 方案：从页面 HTML 里提取「下一页」按钮的 href，适合 URL
    不规律、每页下一页 URL 动态变化的站点（如 Pornhub 的 ?page=N
    带参数翻页）

-   nextPage 的缺点：不能跳页，只能顺序翻；第一次进入必须
    MY_PAGE=1，否则 nextPage 为空直接报没有下一页

> **第三十六章：分类方案选型------rule() 沙箱限制与矩阵方案适用边界**

本章补充第六章（矩阵方案）和第二十七章（独立标签页方案）的适用边界说明。核心结论：rule()
沙箱内的 http 链接永远跳浏览器，有二级的视频站必须用矩阵方案。

**32.1 rule() 沙箱的硬限制**

\$(\'hiker://empty##fypage\').rule(fn)
新开的页面是独立沙箱，引擎对沙箱内卡片 url 的处理规则与主页不同：

  ---------------------------------------------------------------------------------
  **卡片 url 类型**       **沙箱内行为**   **说明**
  ----------------------- ---------------- ----------------------------------------
  hiker:// 协议           ✅ 正常处理      hiker://empty、hiker://page/
                                           等内部协议正常

  pics:// 协议            ✅ 正常处理      图集/漫画直接渲染

  video:// 协议           ✅ 正常处理      直接播放

  http/https URL          ❌ 跳浏览器      引擎不知道走哪个函数，直接打开外部链接

  lazyRule 返回 http      ❌ 跳浏览器      lazyRule 返回 http url 同样跳浏览器
  ---------------------------------------------------------------------------------

> ⚠️ rule() 沙箱内的卡片 url 如果是 http
> 链接，无论如何包装都会跳浏览器。没有任何方法让沙箱内的 http url
> 走规则的二级函数。（Law 67）

**32.2 两种分类方案的适用场景**

  ---------------------------------------------------------------------------
  **方案**        **适用场景**                   **不适用场景**
  --------------- ------------------------------ ----------------------------
  独立标签页      漫画/图集（pics://）           有二级函数的视频站
  \$().rule(fn)   视频直链（video://） Next.js   需要封面/简介/选集的详情页
                  等无二级的站点                 

  矩阵方案        所有视频站（有二级）           基本无限制，通用性最强
  getMyVar +      动漫/影视等需要详情页的站点    
  主页刷新        任何需要走规则二级函数的场景   
  ---------------------------------------------------------------------------

**32.3 有二级的视频站标准分类方案（矩阵）**

有二级函数的站点，分类必须用矩阵方案------分类按钮在主页内渲染，用
getMyVar 记录选中状态，点按钮刷新主页，主页根据选中状态请求对应分类
URL，卡片直接给详情页地址让引擎走二级。

> // 页码必须开启翻页
>
> 页码: { \'主页\': true },
>
> 主页: function() {
>
> const d = \[\];
>
> const KEY = \'site_cindex\'; // 纯字母前缀（Law 51）
>
> const cur = getMyVar(KEY, \'0\');
>
> // 第一页渲染分类按钮行
>
> if (MY_PAGE == 1) {
>
> cats.forEach((cat, i) =\> {
>
> const isSel = (i + \'\' === cur);
>
> d.push({
>
> title: isSel ? \`❆ \${cat.name} ❆\` : cat.name,
>
> col_type: \'scroll_button\',
>
> url: \$(\`#noLoading#\`).lazyRule((k, idx) =\> {
>
> putMyVar(k, idx);
>
> refreshPage(false);
>
> return \'hiker://empty\';
>
> }, KEY, i + \'\')
>
> });
>
> });
>
> d.push({ col_type: \'blank_block\' });
>
> }
>
> // 根据选中分类请求列表
>
> const cat = cats\[parseInt(getMyVar(KEY, \'0\'))\] \|\| cats\[0\];
>
> const url = MY_PAGE \> 1
>
> ? \`\${host}/type\${cat.id}/\${MY_PAGE}.html\`
>
> : \`\${host}/type\${cat.id}\`;
>
> const html = fetch(url, { headers: { \'User-Agent\': ua } });
>
> // 卡片直接给详情页 url，引擎自动走二级函数
>
> return d.concat(parseCards(html));
>
> },
>
> 📌 矩阵方案的卡片 url
> 在主页上下文中，引擎认识当前规则的二级函数，点击会正常进入二级页面渲染封面/简介/选集。这是唯一能让分类列表正确走二级的方案。

**32.4 新增铁律（Law 67）**

  -------------------------------------------------------------------------------------------------
  **编号**   **铁律名称**         **核心要义**                                   **关键口诀**
  ---------- -------------------- ---------------------------------------------- ------------------
  Law 67     沙箱 http 跳浏览器   rule() 沙箱内 http url                         有二级 =
                                  永远跳浏览器，有二级的站点禁用独立标签页方案   矩阵方案；无二级 =
                                                                                 可用沙箱

  -------------------------------------------------------------------------------------------------

> 📌 本章结论来自动漫站（dk95.com）开发实测，经历了 rule() 沙箱 →
> rulePage → 矩阵方案的完整踩坑路径。

**学习追加：随机颜色函数的调整------对黑色标签说"不！"**

有很多书源，或者主题都使用了随机颜色的标签，但这个随机函数非常简陋。以下是对这个函数的修正，大家可以将函数替换你原本的小程序或主题里面的函数。注意，保持函数名一致。

**代码说明：**排除了饱和度过低，以及亮度过低的颜色。可自行根据注释调整范围，获取不同风格的标签。

例如 30%～100% 饱和度 + 30%～100% 亮度（图一）

例如 50%～100% 饱和度 + 50%～70% 亮度（图二）

例如 50%～100% 饱和度 + 30%～100% 亮度（图三）

**代码如下（参数都取 50%～100%）：**

> // 随机标签颜色，排除亮度低的背景
>
> var getRangeColors = function() {
>
> var h = Math.floor(Math.random() \* 360), //色相
>
> s = Math.floor(Math.random() \* 50 + 50) / 100, //饱和度，x\*a+b 范围
> b%～(b+a)%
>
> l = Math.floor(Math.random() \* 50 + 50) / 100, //亮度，x\*a+b 范围
> b%～(b+a)%
>
> c = (1 - Math.abs(2 \* l - 1)) \* s,
>
> x = c \* (1 - Math.abs((h / 60) % 2 - 1)),
>
> m = l - c/2,
>
> \[r1,g1,b1\] =
> h\<60?\[c,x,0\]:h\<120?\[x,c,0\]:h\<180?\[0,c,x\]:h\<240?\[0,x,c\]:h\<300?\[x,0,c\]:\[c,0,x\],
>
> r=Math.floor((r1+m)\*255), g=Math.floor((g1+m)\*255),
> b=Math.floor((b1+m)\*255);
>
> return \'#\' + ((1\<\<24) + (r\<\<16) + (g\<\<8) +
> b).toString(16).slice(1);
>
> }
>
> 📌 调整范围说明：公式 x\*a+b 中，a 控制范围宽度，b
> 控制最小下限。如需获取更暗或更亮的标签，直接修改 s 和 l 的 a、b
> 参数即可。

**学习追加：选择器降级模式------if(!\_p) try-catch 链式写法**

在聚阅规则里，if else if
主要用于《选择器降级》------优先试精确选择器，失败了换备选。

**基本模式**

> var \_p = \"\";
>
> try { \_p = pdfh(\_h, \".entry-content img&&src\"); } catch(e) {}
>
> if (!\_p) try { \_p = pdfh(\_h, \".entry-content img&&data-src\"); }
> catch(e) {}
>
> if (!\_p) try { \_p = pdfh(\_h, \"img.lazyload&&data-src\"); }
> catch(e) {}

**执行流程：**

> 1\. 先试 .entry-content img&&src → 取到了就停
>
> 2\. \_p 还是空 → 试 data-src 懒加载属性
>
> 3\. 还是空 → 最后兆底试全局 img.lazyload

**等价展开写法**

写成一行是简写，逻辑完全一样：

> var \_p = \"\";
>
> try {
>
> \_p = pdfh(\_h, \".entry-content img&&src\");
>
> } catch(e) {}
>
> if (!\_p) {
>
> try {
>
> \_p = pdfh(\_h, \".entry-content img&&data-src\");
>
> } catch(e) {}
>
> }
>
> if (!\_p) {
>
> try {
>
> \_p = pdfh(\_h, \"img.lazyload&&data-src\");
>
> } catch(e) {}
>
> }

**为什么要这样写**

聚阅的 pdfh 底层是 JSoup，选择器匹配不到元素时直接抛 Java NPE，不是返回
null。所以每个调用必须 try-catch：

> // 会崩：NPE，\|\| 根本执行不到
>
> var \_p = pdfh(\_h, \".不存在的元素&&src\") \|\| \"\";
>
> // 安全：崩了也只是空串，后续逻辑正常走
>
> var \_p = \"\";
>
> try { \_p = pdfh(\_h, \".不存在的元素&&src\"); } catch(e) {} //
> catch住，\_p保持\"\"

**实际场景举例**

> // 提取码：先试id选择器，再试class
>
> var \_co = \"\";
>
> try { \_co = pdfh(\_h, \"#refurl&&data-clipboard-text\"); } catch(e)
> {}
>
> if (!\_co) try { \_co = pdfh(\_h, \".copypaw&&data-clipboard-text\");
> } catch(e) {}
>
> // 标题：先试精确class，再试通用h1
>
> var \_t = \"\";
>
> try { \_t = pdfh(\_h, \"h1.entry-title&&Text\"); } catch(e) {}
>
> if (!\_t) try { \_t = pdfh(\_h, \"h1&&Text\"); } catch(e) {}
>
> // 有条件才继续取（省一次无意义的try）
>
> var \_ml = \"\";
>
> try { \_ml = pdfh(\_h, \".btn\--danger&&href\"); } catch(e) {}
>
> var \_mt = \"\";
>
> if (\_ml) try { \_mt = pdfh(\_h, \".btn\--danger&&Text\"); } catch(e)
> {}
>
> // ↑ 链接都没有就不用取按钮文字了

**这样用有什么好处**

在聚阅规则开发里，这个模式解决三个实际问题：

**1. 防崩溃**

pdfh 匹配不到元素时不是返回 null，而是直接 Java 层 NPE
崩溃，整个页面白屏。try-catch 是唯一的防护手段。

> // 没有 try-catch → 页面上没这个元素就整个规则崩了
>
> var \_p = pdfh(\_h, \".entry-content img&&src\");
>
> // 有 try-catch → 崩了也只是空串，后续逻辑正常走
>
> var \_p = \"\";
>
> try { \_p = pdfh(\_h, \".entry-content img&&src\"); } catch(e) {}

**2. 适配不同页面结构**

同一个站不同页面 HTML
结构可能不一样。链式降级保证不管哪种结构都能取到值：

> // 情况A：PC游戏页封面用 src
>
> try { \_p = pdfh(\_h, \".entry-content img&&src\"); } catch(e) {}
>
> // 情况B：Switch页封面用 data-src 懒加载
>
> if (!\_p) try { \_p = pdfh(\_h, \".entry-content img&&data-src\"); }
> catch(e) {}
>
> // 情况C：老帖子图片挂了，光剩 img.lazyload
>
> if (!\_p) try { \_p = pdfh(\_h, \"img.lazyload&&data-src\"); }
> catch(e) {}

**3. 取到即停，省性能**

if (!\_p) 判断确保第一个成功就不再往下跳。对比每次都跳三个的写法：

> // 差：每次都跳三个，浪费
>
> var \_p1=\"\", \_p2=\"\", \_p3=\"\";
>
> try { \_p1 = pdfh(\_h, \"选择器1\"); } catch(e) {}
>
> try { \_p2 = pdfh(\_h, \"选择器2\"); } catch(e) {}
>
> try { \_p3 = pdfh(\_h, \"选择器3\"); } catch(e) {}
>
> var \_p = \_p1 \|\| \_p2 \|\| \_p3;
>
> // 好：第一个成功后面就跳过了
>
> var \_p = \"\";
>
> try { \_p = pdfh(\_h, \"选择器1\"); } catch(e) {}
>
> if (!\_p) try { \_p = pdfh(\_h, \"选择器2\"); } catch(e) {}
>
> if (!\_p) try { \_p = pdfh(\_h, \"选择器3\"); } catch(e) {}
>
> **总结：**防崩 + 兼容 + 高效，三合一。在聚阅里写 pdfh
> 基本都应该套这个模式。核心原则：精确 → 宽泛 → 兆底，每步
> try-catch，取到即停。
>
> **第三十八章：if/else if 分流模式深度总结**

通过实际开发，总结出 if/else if 在规则开发中的四种核心应用场景。

**1. 分类入口分流（最常用）**

根据 MY_URL 的值决定走哪个渲染函数：

> 主页: function() {
>
> var cls = MY_URL;
>
> if (cls === \'tags\') {
>
> return this.\_renderTagCloud(); // 热门标签页
>
> } else if (cls === \'index\') {
>
> return this.\_renderList(\'/\'); // 最新精选
>
> } else if (cls === \'hots\') {
>
> return this.\_renderList(\'/hots.html\'); // 美图精选
>
> } else if (cls.indexOf(\'/tag/\') === 0) {
>
> return this.\_renderList(this.host + cls); // 标签列表页
>
> } else {
>
> return this.\_renderList(this.host + \'/group/\' + cls + \'.html\');
> // 普通分类
>
> }
>
> }

**2. URL 类型识别**

根据 URL 特征决定请求方式和参数：

> function getRequestUrl(url) {
>
> if (url.indexOf(\'/all-tags\') !== -1) {
>
> return url; // 全部分类页，直接请求
>
> } else if (url.indexOf(\'/tags/\') !== -1) {
>
> return url + (MY_PAGE \> 1 ? \'/\' + MY_PAGE : \'\'); //
> 标签列表页，加页码
>
> } else if (url.indexOf(\'/file/\') !== -1) {
>
> return url; // 详情页，直接请求
>
> } else {
>
> return host + \'/group/\' + url + \'.html\'; // 普通分类，补全路径
>
> }
>
> }

**3. 内容类型分流（320影视模式）**

同一入口下，根据内容类型走不同解析逻辑：

> if (/vod\|label/.test(cid)) {
>
> // 视频版块：用 lazyRule 解析
>
> return this.\_parseVideo(html);
>
> } else if (/art-type-id-2/.test(cid)) {
>
> // 小说版块：用 rule 唤起阅读器
>
> return this.\_parseNovel(html);
>
> } else if (/art-type-id-1/.test(cid)) {
>
> // 图片版块：返回 pics://
>
> return this.\_parsePics(html);
>
> }

**4. 选择器降级**

多个选择器依次尝试，取第一个成功的：

> var items = pdfa(html, \'ul.update_area_lists&&li\');
>
> if (items.length === 0) items = pdfa(html,
> \'#content&&article.node-teaser\');
>
> if (items.length === 0) items = pdfa(html,
> \'li:has(a\[href\*=\"/pic/\"\])\');
>
> if (items.length === 0) items = pdfa(html, \'a\');

**为什么 if/else if 如此重要？**

> 1\. 引擎调用机制：点击分类时，引擎把 class_url 的值赋给
> MY_URL，然后调用主页函数。必须在主页函数里用 if/else if 判断 MY_URL
> 来分流。
>
> 2\. URL 多样性：一个站点可能有多种 URL
> 格式（/newest、/tags/xxx、/group/xxx、/all-tags），只有用 if/else if
> 才能统一处理。
>
> 3\. 内容类型混合：像
> 320影视那样，同一入口下有视频、小说、图片三种内容，必须根据 URL
> 特征分流。
>
> 4\. 降级容错：选择器可能失效，用 if/else if 逐步降级，保证规则稳定性。
>
> **核心口诀：**MY_URL 分入口，if/else if 走不同路 \|
> 选择器降级保底，备选链条不能输 \| URL 特征先识别，请求参数再构造 \|
> 类型分流按 cid，各走各的逻辑树
>
> **核心定位：**if/else if 不是简单的条件判断，而是整个规则的路由系统。
>
> **第三十九章：if/else if 同页面多功能模块拼接------Tab 切换模式**

if/else if 最强大的地方在于在同页面内拼接不同功能模块，实现类似"Tab
切换"的效果。以下是三种典型模式：

**模式1：顶部导航 + 内容区域（320影视模式）**

> 主页: function() {
>
> var d = \[\];
>
> var cid = MY_URL;
>
> // ===== 第一部分：顶部分类按钮 =====
>
> if (MY_PAGE == 1) {
>
> var navs = \[\'热门标签\', \'最新精选\', \'美图精选\'\];
>
> for (var i = 0; i \< navs.length; i++) {
>
> d.push({
>
> title: navs\[i\],
>
> url: \$(\'#noLoading#\').lazyRule(function(mode) {
>
> putMyVar(\'mode\', mode);
>
> refreshPage(false);
>
> }, navs\[i\]),
>
> col_type: \'scroll_button\'
>
> });
>
> }
>
> d.push({ col_type: \'blank_block\' });
>
> }
>
> // ===== 第二部分：根据当前模式显示不同内容 =====
>
> var mode = getMyVar(\'mode\', \'index\');
>
> if (mode === \'tags\') {
>
> // 功能A：显示标签云
>
> var tags = this.\_getTags();
>
> for (var i = 0; i \< tags.length; i++) {
>
> d.push({ title: tags\[i\], col_type: \'text_1\' });
>
> }
>
> } else if (mode === \'taglist\') {
>
> // 功能B：显示标签下的图片列表
>
> var tag = getMyVar(\'tag\', \'\');
>
> d.push({ title: \'🔖 \' + tag, col_type: \'text_center_1\' });
>
> // \... 加载图片列表
>
> } else {
>
> // 功能C：显示普通分类图片列表
>
> // \... 加载图片列表
>
> }
>
> return d;
>
> }

**模式2：搜索框 + 结果区域**

> 主页: function() {
>
> var d = \[\];
>
> // ===== 第一部分：搜索框（始终显示）=====
>
> if (MY_PAGE == 1) {
>
> d.push({ title: \"🔍\", col_type: \"input\", desc: \"搜你想要\...\",
>
> url: \$(\'hiker://search?s=\' + input + \'&rule=\' +
> MY_RULE.title).rule()
>
> });
>
> }
>
> // ===== 第二部分：搜索结果或默认内容 =====
>
> var keyword = getVar(\'keyword\', \'\');
>
> if (keyword) {
>
> // 显示搜索结果
>
> var html = fetch(this.host + \'/search?q=\' + keyword);
>
> var items = pdfa(html, \'.result-item\');
>
> for (var i = 0; i \< items.length; i++) {
>
> d.push(this.\_parseCard(items\[i\]));
>
> }
>
> } else {
>
> // 显示默认推荐列表
>
> var html = fetch(this.host + \'/\');
>
> var items = pdfa(html, \'.recommend-item\');
>
> for (var i = 0; i \< items.length; i++) {
>
> d.push(this.\_parseCard(items\[i\]));
>
> }
>
> }
>
> return d;
>
> }

**模式3：多级筛选（分类 + 地区 + 年份）**

> 主页: function() {
>
> var d = \[\];
>
> var cat = getMyVar(\'cat\', \'0\');
>
> var area = getMyVar(\'area\', \'0\');
>
> var year = getMyVar(\'year\', \'0\');
>
> // ===== 第一行：分类筛选 =====
>
> if (MY_PAGE == 1) {
>
> var cats = \[\'全部\', \'电影\', \'电视剧\', \'动漫\'\];
>
> for (var i = 0; i \< cats.length; i++) {
>
> d.push({
>
> title: i == cat ? \'❆ \' + cats\[i\] + \' ❆\' : cats\[i\],
>
> url: \$(\'#noLoading#\').lazyRule(function(idx) {
>
> putMyVar(\'cat\', idx); refreshPage(false);
>
> }, i),
>
> col_type: \'scroll_button\'
>
> });
>
> }
>
> d.push({ col_type: \'blank_block\' });
>
> // ===== 第二行：地区筛选 =====
>
> var areas = \[\'全部\', \'大陆\', \'香港\', \'台湾\', \'日本\',
> \'韩国\'\];
>
> for (var i = 0; i \< areas.length; i++) {
>
> d.push({
>
> title: i == area ? \'❆ \' + areas\[i\] + \' ❆\' : areas\[i\],
>
> url: \$(\'#noLoading#\').lazyRule(function(idx) {
>
> putMyVar(\'area\', idx); refreshPage(false);
>
> }, i),
>
> col_type: \'scroll_button\'
>
> });
>
> }
>
> d.push({ col_type: \'blank_block\' });
>
> // ===== 第三行：年份筛选 =====
>
> var years = \[\'全部\', \'2026\', \'2025\', \'2024\'\];
>
> for (var i = 0; i \< years.length; i++) {
>
> d.push({
>
> title: i == year ? \'❆ \' + years\[i\] + \' ❆\' : years\[i\],
>
> url: \$(\'#noLoading#\').lazyRule(function(idx) {
>
> putMyVar(\'year\', idx); refreshPage(false);
>
> }, i),
>
> col_type: \'scroll_button\'
>
> });
>
> }
>
> d.push({ col_type: \'blank_block\' });
>
> }
>
> // ===== 第四部分：根据三个筛选条件请求内容 =====
>
> var url = this.host + \'/list\';
>
> if (cat != \'0\') url += \'/cat/\' + cat;
>
> if (area != \'0\') url += \'/area/\' + area;
>
> if (year != \'0\') url += \'/year/\' + year;
>
> if (MY_PAGE \> 1) url += \'/page/\' + MY_PAGE;
>
> var html = fetch(url);
>
> var items = pdfa(html, \'.list-item\');
>
> for (var i = 0; i \< items.length; i++) {
>
> d.push(this.\_parseCard(items\[i\]));
>
> }
>
> return d;
>
> }

**核心原理**

> 1\. 状态存储：用 putMyVar/getMyVar 保存当前处于哪个"功能模块"
>
> 2\. 刷新机制：点击按钮时更新状态，调用 refreshPage(false) 刷新页面
>
> 3\. 条件渲染：页面刷新后，根据状态变量决定显示哪个功能模块的内容
>
> 4\. 模块拼接：在同一个主页函数里，用 if/else if
> 把不同功能模块的渲染逻辑拼接在一起

**优势总结**

> · 无跳转：所有切换都在当前页面完成，用户感知不到页面刷新
>
> · 状态持久：putMyVar 保存的状态在翻页时依然有效
>
> · 模块化：每个功能模块的代码独立，易于维护
>
> · 灵活组合：可以自由组合搜索框、筛选器、内容列表等多个模块
>
> **总结：**if/else if
> 在同页面拼接不同功能的强大之处------无跳转、状态持久、模块化、灵活组合，是规则开发中实现复杂交互的核心武器。
>
> **第四十章：tlazy 内联图集规则------替换 lazyRule 的轻量方案（tuzac
> 实战）**

本章记录将 \_getPicsLazyRule（外部 lazyRule 函数）切换为内联 tlazy
写法的实战过程，以及两种方案的核心差异。

**背景：两种图集加载方案**

> · 方案A（\_getPicsLazyRule）：封装为独立函数，支持多页分页 +
> batchFetch 并发，逻辑完整但代码量大
>
> · 方案B（tlazy 内联）：用 \$(\'\').rule() 直接内联在 \_renderList
> 里，轻量、调试方便

切换原因：tlazy 方案在 \_renderList
里更直观，避免闭包参数传递的复杂性；\_getPicsLazyRule 仍保留在
\_renderAllTags 中使用。

**切换后的 \_renderList 核心写法**

> // tlazy：内联图集规则，直接拼接到 url
>
> var tlazy = \$(\'\').rule(() =\> {
>
> var d = \[\]
>
> var html = fetch(MY_URL)
>
> var htmlpage = pdfh(html, \'#pager&&a,-1&&href\').split(\'=\')\[1\]
>
> var surl = pdfh(html, \'#auto-play&&data\').replace(/\\D/g, \'\');
>
> var url = MY_URL.replace(/\\/\[\^/\]+\$/, \'/\' + surl);
>
> var map = (html, arr) =\> {
>
> parseDomForArray(html, \'.file-detail&&img\').map(item =\> {
>
> arr.push(parseDom(item, \'img&&src\'));
>
> });
>
> }
>
> var htmlUrl = \[\]
>
> for (var i = 2; i \<= htmlpage; i++) {
>
> htmlUrl.push({ url: url + \'/?at=\' + i })
>
> }
>
> var htmlArr = batchFetch(htmlUrl);
>
> var pics = \[\];
>
> map(html, pics);
>
> htmlArr.map(item =\> map(item, pics));
>
> for (let k = 0; k \< pics.length; k++) {
>
> d.push({ pic_url: pics\[k\], url: \'pics://\' + pics\[k\],
>
> title: \'\', col_type: \'pic_3\' });
>
> }
>
> setResult(d)
>
> })
>
> // 列表项 url 直接拼接 tlazy
>
> \_d.push({
>
> title: \_title,
>
> pic_url: \_pic ? this.\_addReferer(\_pic) : \'\',
>
> url: \_link + tlazy, // ← 切换点：原为 this.\_getPicsLazyRule(\_link)
>
> col_type: \'movie_3\'
>
> });

**两种方案对比**

\_getPicsLazyRule vs tlazy 核心差异：

> · 作用域：\_getPicsLazyRule 通过闭包参数传入 host/UA；tlazy 用 MY_URL
> 直接在规则内部 fetch，无需外部参数
>
> · 分页处理：\_getPicsLazyRule
> 解析末页链接提取最大页码，容错更强；tlazy 从 #pager&&a,-1&&href
> 提取总页数，更简洁
>
> · 图片选择器：\_getPicsLazyRule 用 .image-loading-box 降级到
> .file-detail&&img；tlazy 直接用 .file-detail&&img
>
> · 去重：\_getPicsLazyRule 有 seen 对象去重；tlazy
> 无去重（站点图片一般不重复）
>
> · 保留位置：\_getPicsLazyRule 仍在 \_renderAllTags
> 中使用（标签页图集）；tlazy 用于 \_renderList 主列表

**注意：保留 \_getPicsLazyRule 的原因**

> **重要：**\_renderAllTags 里的标签页列表仍调用
> \_getPicsLazyRule，因为标签页需要传入 host/UA 参数，且通过 eval
> 重建函数上下文（\_getRuleFn.call(\_ctx,
> \_link)）。两种方案并存，各司其职。

**同时修复的代码问题**

原代码 \_getPicsLazyRule 内有一行乱码需清理：

> // ❌ 原始乱码行（应删除）
>
> } catch (e) {} ;、v，
>
> // ✅ 修复后
>
> } catch (e) {}
>
> **铁律：**rule() 内联写法（tlazy）适合逻辑简单、参数少的场景；lazyRule
> 封装函数适合需要多参数、多分支、跨函数复用的场景。根据复杂度选型，不必强求统一。
# 第四十一章：工具对象封装模式——多媒体类型解析函数复用

## 41.1 背景与问题

规则里有视频、图集、有声三种内容类型，每种解析逻辑不同。如果每个卡片都内联一套完整的 lazyRule 代码，会导致：

- 代码重复量极大
- `renderList` 函数臃肿难以维护
- 修改解析逻辑时需要改多处

**解决方案：** 把各类型的解析逻辑封装到 `工具` 对象里，作为具名函数，在 `lazyRule` 里传函数引用。

## 41.2 工具对象标准结构

```js
var parse = {
    host: 'https://example.com',
    UA:   'Mozilla/5.0 ...',

    工具: {
        // 视频解析：正则提取 m3u8，带 Cookie
        videoLogic: function(host, ua) {
            var ck  = getItem('site_cookie', '');
            var html = request(input, { headers: { 'User-Agent': ua, 'Referer': host + '/', 'Cookie': ck } });
            var m = String(html || '').match(/https?:[^\s"'<>\\]+\.m3u8[^\s"'<>\\]*/i);
            if (m) {
                var sl  = String.fromCharCode(92);
                var url = m[0].split(sl).join('');
                return 'video://' + url + ';{User-Agent@' + ua + '&&Referer@' + host + '/&&Cookie@' + ck + '}';
            }
            return input + '#嗅探';
        },

        // 图集解析：$.image() 带 Cookie 防盗链，返回 pics://
        picLogic: function(host, ua) {
            var ck  = getItem('site_cookie', '');
            var html = fetch(input, { headers: { 'User-Agent': ua, 'Referer': host + '/', 'Cookie': ck } });

            // $.image() 带 Cookie：处理需要登录认证的防盗链图片（Law 28 延伸）
            var imgProxy = $().image(function(ua_s, ck_s) {
                var FU   = com.example.hikerview.utils.FileUtil;
                var real = String(input || '').split('@')[0];
                var h    = { 'User-Agent': ua_s, 'Cookie': ck_s };
                var dm   = real.match(/^(https?:\/\/[^\/]+)/);
                if (dm) h['Referer'] = dm[1] + '/';
                try { return FU.toInputStream(FU.readBytes(real, h)); } catch(e) { return null; }
            }, ua, ck);

            var pics = [];
            var items = pdfa(html, 'body&&.photo-item');
            for (var i = 0; i < items.length; i++) {
                var m = (pdfh(items[i], '.img&&style') || '').match(/url\('([^']+)'\)/);
                if (m) {
                    var iurl = m[1];
                    var dm   = iurl.match(/^(https?:\/\/[^\/]+)/);
                    pics.push(iurl + (dm ? '@Referer=' + dm[1] + '/' : '') + imgProxy);
                }
            }
            return pics.length > 0 ? 'pics://' + pics.join('&&') : 'toast://未提取到图片';
        },

        // 有声解析：提取 audio src，返回音频协议
        audioLogic: function(host, ua) {
            var ck   = getItem('site_cookie', '');
            var html = request(input, { headers: { 'User-Agent': ua, 'Referer': host + '/', 'Cookie': ck } });
            var src  = pdfh(html, 'audio source&&src') || '';
            if (!src) {
                var m = String(html || '').match(/https?:[^"'\s]+\.mp3[^"'\s]*/i);
                if (m) src = m[0];
            }
            if (src) return String(src).replace(/\\/g, '') + ';{Referer@' + host + '/}#isMusic=true#';
            return 'toast://未发现音频';
        },

        // 统一列表渲染：按 URL 特征自动分流三种内容类型
        renderList: function(html, host, ua) {
            var res   = [];
            var items = pdfa(html, 'body&&.item');
            for (var j = 0; j < items.length; j++) {
                try {
                    var item = items[j];
                    var link = pd(item, 'a&&href', host);
                    if (!link) continue;

                    var style = pdfh(item, '.img&&style') || '';
                    var pic   = (style.match(/url\('([^']+)'\)/) || [])[1];
                    if (pic) pic += '@Referer=' + host + '/';

                    var title = pdfh(item, '.title&&Text');

                    // ✅ 按 URL 特征分流，传函数引用复用解析逻辑
                    if (String(link).indexOf('/video/') > -1) {
                        var dur = pdfh(item, '.duration&&Text') || '';
                        res.push({
                            title: title, desc: '🎬 ' + dur, pic_url: pic,
                            url: $(link).lazyRule(parse.工具.videoLogic, host, ua),
                            col_type: 'movie_2'
                        });
                    } else if (String(link).indexOf('/photo/') > -1) {
                        var cnt = pdfh(item, '.count&&Text') || '';
                        res.push({
                            title: title, desc: '📸 ' + cnt, pic_url: pic,
                            url: $(link).lazyRule(parse.工具.picLogic, host, ua),
                            col_type: 'movie_3'
                        });
                    } else if (String(link).indexOf('/fiction/') > -1) {
                        res.push({
                            title: '🎧 ' + title, desc: '有声', pic_url: pic,
                            url: $(link).lazyRule(parse.工具.audioLogic, host, ua),
                            col_type: 'movie_2'
                        });
                    }
                } catch(e) {}
            }
            return res;
        }
    }
};
```

## 41.3 调用方式

```js
// ✅ 传函数引用，不是字符串，不是内联代码
url: $(link).lazyRule(parse.工具.videoLogic, host, ua)

// ❌ 不要这样写（内联重复逻辑）
url: $(link).lazyRule(function(host, ua) {
    // 和 videoLogic 里一模一样的代码...
}, host, ua)
```

## 41.4 $.image() 带 Cookie 的防盗链图片（Law 28 延伸）

手册第十章介绍了 `$.image()` 的基本用法。当站点图片需要**登录态（Cookie）** 才能访问时，Cookie 也要通过参数传入 `image()` 的 func：

```js
// ✅ Cookie 作为参数传入，lazyRule 内部读取持久化存储
var imgProxy = $().image(function(ua_s, ck_s) {
    var FU   = com.example.hikerview.utils.FileUtil;
    var real = String(input || '').split('@')[0];  // 去掉 @Referer= 后缀
    var h    = { 'User-Agent': ua_s, 'Cookie': ck_s };
    var dm   = real.match(/^(https?:\/\/[^\/]+)/);
    if (dm) h['Referer'] = dm[1] + '/';           // 自动从图片 URL 提取域名作为 Referer
    try {
        return FU.toInputStream(FU.readBytes(real, h));
    } catch(e) { return null; }
}, ua, ck);

// 图片 URL 后拼接 imgProxy 即可
pics.push(imgUrl + '@Referer=' + host + '/' + imgProxy);
```

> ⚠️ **注意**：`$().image()` 里的 `input` 是图片 URL（引擎下载前传入的 URL），不是详情页 URL。`FU.readBytes(real, h)` 用指定 headers 下载图片字节，`toInputStream` 转为流返回引擎渲染。

---

# 第四十二章：cate:// 虚拟协议——静态分类与矩阵子分类二级联动

## 42.1 问题背景

静态分类（第二章）：顶部 tab 零代码，但所有分类走同一解析逻辑，无法在进入某个大类后动态渲染**该大类专属的子分类按钮行**。

矩阵方案（第六章）：支持子分类，但全部逻辑都在主页函数里，分类按钮要手动维护。

**`cate://` 方案** 融合两者优势：静态分类做顶部大类 tab，进入后主页函数按 `cate://` 标识动态渲染子分类。

## 42.2 静态分类配置

```js
静态分类: {
    type: '主页',
    url: 'fyclass',
    class_name: '全部🎬&中文AV&日本AV&📸套图&💃模特',
    // ✅ 真实路径 和 cate:// 虚拟协议 混用
    class_url: '/videos/1.html&cate://中文AV&cate://日本AV&/photos/1.html&/models.html'
},
```

- 有实际路径的分类（如 `/videos/1.html`）：主页直接请求该路径
- `cate://大类名`：主页识别后查询子分类数据库，渲染子分类按钮行

## 42.3 子分类数据库

```js
_subCategories: {
    '中文AV': [
        { name: '麻豆传媒', url: '/videos/series-xxx1.html' },
        { name: '糖心Vlog', url: '/videos/series-xxx2.html' },
        { name: '蜜桃传媒', url: '/videos/series-xxx3.html' },
        // ...
    ],
    '日本AV': [
        { name: '有码AV', url: '/videos/series-yyy1.html' },
        { name: '无码AV', url: '/videos/series-yyy2.html' },
    ],
    // ...
},

// 大类默认列表页（点进大类时默认展示全部）
_mainCateUrlMap: {
    '中文AV': '/videos/series-all-cn.html',
    '日本AV': '/videos/series-all-jp.html',
},
```

## 42.4 主页函数核心逻辑

```js
主页: function() {
    var d      = [];
    var host   = this.host;
    var ua     = this.UA;
    var curUrl = typeof MY_URL !== 'undefined' ? MY_URL : '';

    // ── Step1：切换大类时清空子分类状态（状态互斥清除）──────────────
    var lastClass = getMyVar('site_last_class', '');
    if (lastClass !== curUrl) {
        putMyVar('site_sub_path', '');   // 清空子分类路径
        putMyVar('site_sub_idx', '0');   // 重置子分类选中
        putMyVar('site_last_class', curUrl);
    }

    // ── Step2：解析当前路径 ──────────────────────────────────────────
    var subPath    = getMyVar('site_sub_path', '');
    var isMainCate = false;
    var mainName   = '';
    var path       = subPath || curUrl;

    if (String(path).indexOf('cate://') === 0) {
        // 初次进入大类（未选子分类）
        isMainCate = true;
        mainName   = path.split('//')[1];
        path       = this._mainCateUrlMap[mainName] || path;
    } else if (String(curUrl).indexOf('cate://') === 0 && subPath !== '') {
        // 已选子分类，curUrl 仍是 cate://xxx，path 已是子分类路径
        isMainCate = true;
        mainName   = curUrl.split('//')[1];
    }

    // ── Step3：请求列表页 ────────────────────────────────────────────
    var fetchUrl = this._buildUrl(path, MY_PAGE);
    var html     = fetch(fetchUrl, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });

    // ── Step4：第一页渲染子分类按钮行 ───────────────────────────────
    if (MY_PAGE == 1 && isMainCate && this._subCategories[mainName]) {
        var subs    = [{ name: '全部', url: this._mainCateUrlMap[mainName] }]
                      .concat(this._subCategories[mainName]);
        var curIdx  = parseInt(getMyVar('site_sub_idx', '0'));

        for (var s = 0; s < subs.length; s++) {
            var isSel = (s === curIdx);
            d.push({
                title:    isSel ? '❆ ' + subs[s].name + ' ❆' : subs[s].name,
                col_type: 'scroll_button',
                url: $('#noLoading#').lazyRule(function(idx, subUrl) {
                    putMyVar('site_sub_idx',  idx + '');
                    putMyVar('site_sub_path', subUrl);
                    refreshPage(false);
                    return 'hiker://empty';
                }, s, subs[s].url)
            });
        }
        d.push({ col_type: 'blank_block' });
    }

    // ── Step5：渲染内容列表 ──────────────────────────────────────────
    return d.concat(this.工具.renderList(html, host, ua));
},
```

## 42.5 状态互斥清除——防止子分类残留（Law 84）

```js
// ❌ 不清除时的 bug：
// 用户在"中文AV"里选了"麻豆传媒"，putMyVar('site_sub_path', '/videos/series-xxx')
// 切到"日本AV"后，site_sub_path 仍是麻豆的路径 → 日本AV页面显示麻豆内容

// ✅ 切换大类时强制清除子分类状态
var lastClass = getMyVar('site_last_class', '');
if (lastClass !== curUrl) {
    putMyVar('site_sub_path', '');
    putMyVar('site_sub_idx', '0');
    putMyVar('site_last_class', curUrl);
}
```

> 📌 **Law 84（新增）**：静态分类 + 矩阵子分类联动时，必须用 `lastClass !== curUrl` 检测大类切换，在切换瞬间清空所有子分类状态变量，防止跨分类残留。

## 42.6 选型决策

| 场景 | 推荐方案 |
|---|---|
| 分类少、URL 规律、单内容类型 | 静态分类（第二章）|
| 分类多、多维筛选、单内容类型 | 矩阵方案（第六章）|
| 有大类+子分类两级结构 | **cate:// 虚拟协议融合方案（本章）**|
| 内容类型多样（视频/图集/有声/模特）| categoryList（第三十七章）|

---

# 第四十三章：模特/演员二级页——noShow + extenditems 完全自定义

## 43.1 场景说明

模特/演员详情页不是普通的「选集列表」页，它包含：

- 头像大图
- 姓名 + 简介
- AI 评价文字
- 「查看全部视频」入口（带翻页）
- 「查看全部套图」入口（带翻页）
- 收录作品缩略列表

用引擎默认的二级格式（`line + list`）无法满足。需要用 `noShow` 隐藏默认组件，用 `extenditems` 完全自定义页面内容。

## 43.2 完全自定义二级页模板

```js
二级: function(url) {
    var d    = [];
    var host = this.host;
    var ua   = this.UA;
    var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });

    if (url.indexOf('/model/') > -1 || url.indexOf('/actor/') > -1) {
        // 头像
        var avatar = pdfh(html, '.avatar&&img&&src') || '';
        if (avatar) d.push({ pic_url: avatar + '@Referer=' + host + '/', url: avatar, col_type: 'pic_1_full' });

        // 姓名
        var name = pdfh(html, '.info&&.name&&Text') || '';
        if (name) d.push({ title: '💎 ' + name, col_type: 'text_center_1', extra: { textSize: 22, isBold: true } });

        // 简介
        var bio = pdfh(html, '.info&&.bio&&Text') || '';
        if (bio && bio.length > 5) {
            d.push({ title: bio, col_type: 'rich_text', extra: { textSize: 15 } });
            d.push({ col_type: 'line' });
        }

        // 「查看全部视频」入口——用 $().rule() 新开列表页，带翻页
        var vLink = pd(html, 'a:contains(全部视频)&&href', host) || '';
        if (vLink) {
            d.push({
                title:    '🎬 查看全部视频 >>',
                col_type: 'text_center_1',
                url: $(vLink + '##fypage').rule(function(lnk, h, u) {
                    var pg  = typeof MY_PAGE !== 'undefined' ? MY_PAGE : 1;
                    var ck  = getItem('site_cookie', '');
                    var base = lnk.replace(/\/\d+\.html$/, '').replace('.html', '');
                    var pageUrl = pg <= 1 ? lnk : base + '/' + pg + '.html';
                    var rHtml = fetch(pageUrl, { headers: { 'User-Agent': u, 'Referer': h + '/', 'Cookie': ck } });
                    // 解析视频列表...
                    var rs = [];
                    var its = pdfa(rHtml, 'body&&.item');
                    for (var i = 0; i < its.length; i++) {
                        var lk = pd(its[i], 'a&&href', h);
                        if (!lk || lk.indexOf('/video/') === -1) continue;
                        rs.push({
                            title:    pdfh(its[i], '.title&&Text'),
                            pic_url:  pd(its[i], 'img&&src', h) + '@Referer=' + h + '/',
                            url:      $(lk).lazyRule(parse.工具.videoLogic, h, u),
                            col_type: 'movie_2'
                        });
                    }
                    setResult(rs);
                }, vLink, host, ua)
            });
        }

        d.push({ col_type: 'line' });
        d.push({ title: '📚 收录作品', col_type: 'text_center_1', extra: { isBold: true } });
        d = d.concat(this.工具.renderList(html, host, ua));

        // ✅ noShow 隐藏引擎默认组件，extenditems 完全自定义
        return {
            vod_name: name,
            vod_pic:  avatar,
            noShow:   { 简介: true, 选集: true, 排序: true },
            list:     [[]],       // 必须给空二维数组，否则引擎报错
            extenditems: d
        };
    }

    return { detail1: '详情加载中', list: [[]] };
},
```

## 43.3 noShow 字段速查

```js
noShow: {
    封面: false,   // 是否隐藏封面图区域
    简介: true,    // 隐藏引擎默认简介区域（用 extenditems 自定义）
    排序: true,    // 隐藏选集排序按钮
    选集: true     // 隐藏选集列表区域（用 extenditems 代替）
}
```

> ⚠️ 使用 `extenditems` 时，`list` 字段必须给 `[[]]`（空的二维数组），否则引擎会报结构错误。

## 43.4 三种二级页结构对比

| 结构 | 适用场景 | 关键字段 |
|---|---|---|
| 标准结构 | 有选集的视频/动漫 | `line + list` |
| 图文二级 | 图集、文章 | `extenditems` |
| 完全自定义 | 模特/演员/作者主页 | `noShow + extenditems + list:[[]]` |

---

# 附录：本次新增铁律

| 编号 | 铁律名称 | 核心要义 | 口诀 |
|---|---|---|---|
| Law 84 | 子分类状态互斥清除 | 切换顶部大类时必须清空所有子分类 getMyVar，防止跨分类残留 | lastClass !== curUrl → 清空 sub 相关变量 |
| Law 85 | $.image() Cookie 传参 | 需要登录态的防盗链图片，Cookie 要作为参数传入 $.image() 的 func | $.image() 不能访问外部变量，Cookie 必须参数传入 |
| Law 86 | noShow + extenditems 配套 | 使用 extenditems 完全自定义二级页时，list 必须给 `[[]]`，noShow 按需隐藏默认组件 | extenditems 自定义，list:[[]] 保底，noShow 隐组件 |
| Law 87 | typeof 防御注入变量 | MY_URL / MY_PAGE 等引擎注入变量在某些环境可能未定义，用 typeof 判断兜底 | `typeof MY_URL !== 'undefined' ? MY_URL : ''` |

---

# 第四十四章：1024吃瓜网规则开发难点总结

## 44.1 章节背景

1024吃瓜网是典型的高对抗性成人内容站点，具有域名频繁轮换、图片/视频双重加密、API 结构多变三大核心难点。本章系统梳理开发过程中遇到的问题与解决方案，作为同类型站点的参考范本。

---

## 44.2 域名动态发现与自愈机制

### 难点描述

网站使用"单词池 + 随机后缀"生成大量随机域名，固定写死域名会快速失效。需要实现**自动发现 → 测试 → 缓存 → 失效重发现**的完整自愈闭环。

### 核心策略

| 环节 | 方案 |
|---|---|
| 域名发现 | 遍历候选域名列表，逐一发起探测请求 |
| 有效性测试 | 检测页面特征内容（如 `.post-card`），**不能**仅用 `html.length` 判断 |
| 缓存策略 | 成功域名缓存 1 小时（`setItem` + 时间戳比较），避免每次请求重新发现 |
| 失效自愈 | 缓存命中后若请求仍失败，清除缓存重新走发现流程 |

### 为什么不能用 `html.length` 判断？

CDN 返回的 4xx/5xx 错误页、CF 挑战页、空白跳转页都可能有相当长度的 HTML，但不含业务内容。必须用**页面特征选择器**（如 `pdfh(html, '.post-card&&Text')`）来确认真正获取到了列表数据。

---

## 44.3 双重加密体系

站点对**图片**和**视频**分别采用不同的 AES 加密策略。

### 图片加密（固定 key/iv）

```javascript
// 使用 crypto-java.js，key/iv 在规则内固定写死
var IMG_KEY = 'xxxxxxxxxxxxxxxx';
var IMG_IV  = 'xxxxxxxxxxxxxxxx';
// CryptoUtil.decrypt(encData, key, iv) → 明文 URL
```

### 视频加密（动态 key/iv）

```javascript
// key/iv 从页面 data-config 属性中动态提取，每个视频不同
var config = JSON.parse(pdfh(html, '.dplayer&&data-config').replace(/&quot;/g, '"'));
var api  = config['data-api'];
var _key = config['data-key'];   // 动态，不可硬编码
var _iv  = config['data-iv'];    // 动态，不可硬编码
```

> ⚠️ **key/iv 的动态/静态属性决定了能否复用解密逻辑**。开发前必须先抓包确认，切勿假设二者相同。

---

## 44.4 播放器配置提取

### data-config 解析要点

| 难点 | 解决方案 |
|---|---|
| `data-config` 嵌套在 DPlayer 容器中 | 正则或 `pdfh` 匹配 `.dplayer&&data-config` |
| JSON 中含 HTML 转义字符 | 先 `.replace(/&quot;/g, '"')` 再 `JSON.parse()` |
| 一篇文章含多个播放器 | `pdfa(html, "body&&.dplayer")` 遍历，循环处理 |

### 多视频容器遍历模板

```javascript
var players = pdfa(html, 'body&&.dplayer');
for (var i = 0; i < players.length; i++) {
    var rawConfig = pdfh(players[i], 'self&&data-config') || '';
    if (!rawConfig) continue;
    var config = JSON.parse(rawConfig.replace(/&quot;/g, '"'));
    // 生成各自独立的 lazyRule...
}
```

---

## 44.5 API 响应结构多变

同一站点的视频 API 可能返回三种结构，需按优先级降级处理：

```javascript
// 优先级：video_url → url → 解密 data
var obj  = JSON.parse(resp);
var mUrl = '';

if (obj.video_url && obj.video_url.indexOf('http') === 0) {
    mUrl = obj.video_url;               // 情况1：直接可用
} else if (obj.url && obj.url.indexOf('http') === 0) {
    mUrl = obj.url;                     // 情况2：url 字段
} else if (obj.data) {
    mUrl = decrypt(obj.data, key, iv);  // 情况3：需解密
}
```

> 📌 **兜底机制**：解密仍失败时，尝试从原始页面 HTML 正则匹配 `.m3u8` 链接，确保播放成功率。

---

## 44.6 聚阅/海阔引擎限制——与本站的交叉影响

| 引擎限制 | 对本站的影响 | 解决方案 |
|---|---|---|
| 同步请求阻塞 | 多个视频 API 串行请求导致页面卡顿 | 懒加载（点击时才请求 API） |
| `lazyRule` 参数顺序严格 | 顺序错乱 → `enc invalid` 报错 | 严格按 `(apiUrl, key, iv, host, ua)` 顺序传参 |
| `this` 在 `lazyRule` 内丢失 | 无法访问 `this.UA` 等实例属性 | 提前 `var self = this`，通过参数传入 |
| 无 `atob`/`btoa` | Base64 解码报错 | 用 `android.util.Base64`（Law 62） |
| `CryptoUtil` 非标准 API | 与 CryptoJS 用法不同 | 用 `Data.parseUTF8() + .toText()` |

---

## 44.7 懒加载实现——正确的参数传递方式

参数顺序错乱是本站开发中最高频的坑点，必须严格对照。

```javascript
// ✅ 正确写法：lazyRule 函数的形参顺序 = 传入参数顺序
var lazyUrl = $(apiUrl).lazyRule(function(apiUrl, key, iv, host, ua) {
    // 在此发起 API 请求并解密
    var resp = fetch(apiUrl, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });
    // ... 解密返回 m3u8 URL
}, apiUrl, _key, _iv, host, self.UA);
//  ↑第1实参→第1形参  ↑第2  ↑第3  ↑第4  ↑第5

// ❌ 错误示例：实参与形参顺序不一致
var lazyUrl = $(apiUrl).lazyRule(function(key, apiUrl, iv, ua, host) {
    // key 收到的实际是 apiUrl 的值 → 解密必然失败
}, apiUrl, _key, _iv, host, self.UA);
```

> 📌 **调试技巧**：在 lazyRule 内 `log('key=' + key + ' iv=' + iv)` 打印收到的参数，与预期值对比，快速定位顺序问题。

---

## 44.8 多视频支持

| 难点 | 解决方案 |
|---|---|
| 一篇文章多个 DPlayer | `pdfa(html, "body&&.dplayer")` 遍历 |
| 每个视频有独立 API + key/iv | 循环内各自提取 config，生成独立 lazyRule |
| 线路名称动态化 | 单视频显示"播放"，多视频显示"视频列表 (N个)" |

---

## 44.9 Cookie 与防盗链处理

| 场景 | 方案 |
|---|---|
| CF 验证 Cookie | `fetchCookie` 获取后 `setItem` 持久化 |
| 图片防盗链 | 图片 URL 拼接 `@Referer=host/` |
| 视频 Headers | `video://url;{User-Agent@xxx&&Referer@xxx}` |

---

## 44.10 静态分类矩阵方案

| 难点 | 解决方案 |
|---|---|
| 12 个分类需记忆选中状态 | `getMyVar`/`putMyVar` 存储 `cindex` |
| 翻页时保持分类 | `MY_PAGE` 自动管理；分类按钮只在第一页渲染 |
| 分类 URL 动态拼接 | 根据 `cindex` 从数组中取路径拼接完整 URL |

---

## 44.11 性能优化要点

| 问题 | 优化方案 | 备注 |
|---|---|---|
| 进入详情页慢 | 懒加载：点击视频才请求 API | 已实现 |
| 多个 API 串行 | 懒加载天然解决（只请求点击的那个） | 已实现 |
| 重复请求 | 增加 `getMyVar` 结果缓存 | 后续可加 |

---

## 44.12 核心开发经验总结

1. **先跑通基础流程，再优化性能**——能播放的版本永远比完美的方案更重要。
2. **保留能正常播放的版本**，在其基础上逐步迭代，避免"破坏式重构"。
3. **`lazyRule` 参数顺序是最大坑点**，必须严格对照函数签名，上线前逐参打 log 验证。
4. **日志是调试利器**，关键节点（参数收到的值、API 响应结构、解密结果）务必打出来，上线时可保留非敏感日志。
5. **兜底机制很重要**：解密失败 → 正则抓页面 m3u8 → 最终兜底显示错误提示，三层保障播放成功率。
6. **页面特征测试优于长度测试**：任何涉及"请求是否成功"的判断，必须用业务内容特征而非 `html.length`。

---

> 📌 本章所涉及的铁律：Law 62（无 atob/btoa）、Law 63（多字节 UTF-8 字节级解密）、Law 67（sandbox HTTP URL 开浏览器）均在前章已详细说明，本章不重复展开，以交叉引用为准。


---

# 第四十五章：壳子内置工具函数完全参考（dy2020 框架）

本章基于实战规则代码（007吃瓜、51暗网、海角等站点）提炼，记录聚阅壳子（dy2020框架）提供的内置工具函数。这些函数由壳子通过 `rc()` 远程加载的公共库提供，**不属于海阔/聚阅引擎原生 API**。

> ⚠️ **Law 88（新增）：禁止在规则中使用 `rc()` / `gfd()` / `fc()` 远程加载调用。**
> 这三个函数用于从 Gitee/GitHub 远程拉取公共库代码，每次执行都会发起网络请求，存在以下致命问题：
> - **网络依赖**：Gitee/GitHub 被墙或访问慢时，规则直接卡死
> - **不可控**：远程库随时可能变更或失效，导致规则无法运行
> - **性能差**：每次主页调用都额外消耗网络请求
>
> **正确做法**：将所需工具函数直接内联到规则代码中，或使用手册前述章节的标准 API 实现相同功能。

---

## 45.1 rc() / gfd() / fc() ——远程库加载（禁止使用）

```javascript
// ❌ 禁止：远程拉取公共库
rc((rc('https://gitee.com/mistywater/hiker_info/raw/master/ghproxy.js'), gfd()) 
   + 'https://raw.githubusercontent.com/mistywater/hiker/main/f', 24);

// rc(url, cacheHours)  → 加载远程 JS 并 eval，cacheHours 为缓存小时数
// gfd()               → 返回 GitHub 代理前缀（来自 ghproxy.js）
// fc(url)             → 获取 JSON 配置文件中的代理前缀
```

**替代方案**：把所需功能（`getHtml`、`classTop` 等）直接实现在规则内，或使用手册各章的标准写法。

---

## 45.2 getHtml() ——带 CF 缓存的 fetch 封装

壳子提供的 `getHtml` 是对 `fetch`/`fetchCodeByWebView` 的封装，集成了 Cookie 管理和 CF 过盾：

```javascript
// 壳子写法（需要 rc 加载公共库）
var html = getHtml(MY_URL);
var html = getHtml(MY_URL, '', '', 1);  // 第4参数=1 表示需要 WebView 渲染

// ✅ 手册标准替代写法（第十二章）
var ck = getItem('site_cookie', '');
var html = fetch(MY_URL, { headers: { 'User-Agent': parse.UA, 'Cookie': ck } });
if (html.indexOf('视频卡片class') === -1) {
    // CF 过盾逻辑（详见第十二章）
}
```

---

## 45.3 cpage() ——分类页码解析

`cpage()` 从 `MY_URL` 里解析当前分类路径 `_c` 和页码 `page`：

```javascript
// 壳子写法
eval(cpage(''));
// 执行后注入：_c（分类路径）、page（当前页码）

// ✅ 手册标准替代写法
var _c = '';
var page = MY_PAGE;
if (MY_URL && MY_URL !== 'fyclass') {
    _c = MY_URL;  // MY_URL 就是 class_url 里的值（Law 60）
}
```

---

## 45.4 classTop() ——分类标签行渲染

`classTop()` 渲染顶部 `scroll_button` 分类行，对应手册第六章的 `renderRow`：

```javascript
// 壳子写法
var dataClass = [{
    title: '首页&最新热瓜&每日精选',
    id: '&htzx&mrjx'
}];
dataClass.forEach((item, index) => {
    classTop(index, item, host, d);
});

// ✅ 手册标准替代写法（第六章 renderRow）
var renderRow = function(rowIdx, titleStr, idStr, d, h) {
    var tArr = titleStr.split('&');
    var iArr = idStr.split('&');
    var curIdx = getMyVar(h + 'cindex' + rowIdx, '0');
    for (var i = 0; i < tArr.length; i++) {
        var isSel = (i == curIdx);
        d.push({
            title: isSel ? '❆ ' + tArr[i] + ' ❆' : tArr[i],
            url: $('#noLoading#').lazyRule(function(rIdx, idx, val, h) {
                putMyVar(h + 'cindex' + rIdx, idx + '');
                putMyVar(h + 'curl' + rIdx, val);
                if (rIdx == 0) {
                    for (var n = 1; n < 10; n++) {
                        putMyVar(h + 'cindex' + n, '0');
                        putMyVar(h + 'curl' + n, '');
                    }
                }
                refreshPage(false);
                return 'hiker://empty';
            }, rowIdx, i, iArr[i], h),
            col_type: 'scroll_button'
        });
    }
    d.push({ col_type: 'blank_block' });
};
```

---

## 45.5 pageAdd() / pageMoveto() ——翻页与记忆定位

```javascript
// pageAdd(page, host)  → 记录当前页码，供 pageMoveto 恢复用
// pageMoveto(host, page, type, pages, chchePath) → 生成 extra 对象，点击卡片后跳回上次位置

// ✅ 手册标准替代写法（简化版，不用位置记忆）
// 直接依赖 MY_PAGE 翻页即可，无需手动 pageAdd
```

---

## 45.6 getdTemp() / writeFile() ——页面缓存机制

壳子用 `hiker://files/_cache/` 缓存页面列表数据，避免重复网络请求：

```javascript
// 壳子写法
let _chchePath = `hiker://files/_cache/juyue/${jkdata.name}_${safePath(MY_URL)}.txt`;
dTemp = getdTemp(dTemp, d, _chchePath);
if (dTemp.length != 0) return dTemp;
// ... 构建 d ...
if (d.length != 0) writeFile(_chchePath, JSON.stringify(d));
return dTemp.concat(d);

// ✅ 手册标准替代写法（用 setItem 简化缓存）
var _cacheKey = 'page_cache_' + MY_PAGE;
var _cached = getItem(_cacheKey, '');
if (_cached) {
    try { return JSON.parse(_cached); } catch(e) {}
}
// ... 构建 d ...
setItem(_cacheKey, JSON.stringify(d));
return d;
```

> ⚠️ `hiker://files/` 路径写文件需要引擎版本支持，不如 `setItem` 通用。缓存策略应根据内容更新频率决定是否启用。

---

## 45.7 safePath() ——URL 转安全文件名

`safePath()` 把 URL 中的特殊字符替换为下划线，生成合法文件名：

```javascript
// 壳子写法
safePath('https://example.com/page/2/') 
// → 'https___example_com_page_2_'

// ✅ 手册标准替代（无需此函数，直接用 setItem，key 用 URL 的 hash）
var _key = 'cache_' + MY_PAGE + '_' + (_c || 'home');
```

---

## 45.8 getFastestDomain() ——多域名竞速选优

`getFastestDomain()` 并发探测多个备用域名，返回响应最快的：

```javascript
// 壳子写法（配合 fetchCodeByWebView 获取域名列表）
let urls = pdfa(html, '#list-wrap&&.btnLink').map(item => pdfh(item, 'Text'));
host = getFastestDomain(urls);
host = host ? 'https://' + host.slice(0, -1) : 'https://backup.example.com';

// ✅ 手册标准替代写法（第十三章域名自愈 + batchFetch 测速）
var candidates = ['https://site1.com', 'https://site2.com', 'https://site3.com'];
var tasks = candidates.map(function(u) { return { url: u + '/', options: { timeout: 3000 } }; });
var results = batchFetch(tasks);
var host = candidates[0];  // 默认备用
for (var ri = 0; ri < results.length; ri++) {
    if (results[ri] && results[ri].indexOf('特征关键词') > -1) {
        host = candidates[ri];
        break;
    }
}
setItem('current_host', host);
```

---

## 45.9 imgDec() ——AES 图片解密 lazyRule 生成

`imgDec(key, iv, 'AES')` 生成一段 `@js=` 图片解密字符串，拼接到图片 URL 后触发 AES 解密：

```javascript
// 壳子写法
img: 'https://cdn.example.com/path' + '@js=' + imgDec(key, iv, 'AES')

// ✅ 手册标准替代写法（第十章 $.image() + CryptoUtil，Law 28）
var tlazy = $('').image(function(k, iv_s) {
    try {
        var CryptoUtil = $.require('hiker://assets/crypto-java.js');
        var keyBytes = CryptoUtil.Data.parseUTF8(k);
        var ivBytes  = CryptoUtil.Data.parseUTF8(iv_s);
        var rawData  = CryptoUtil.Data.parseInputStream(input);
        var dec = CryptoUtil.AES.decrypt(rawData, keyBytes, {
            mode: 'AES/CBC/PKCS7Padding', iv: ivBytes
        });
        return dec.toInputStream();
    } catch(e) { log('imgDec fail: ' + e); return input; }
}, key, iv);

// 使用（提前调用一次，循环内复用）
d.push({ pic_url: imgUrl + tlazy });
```

---

## 45.10 $.toString() ——函数字符串化存储模式

壳子规则用 `$.toString(() => { ... })` 把代码块序列化成字符串存在 `parse` 属性中，后续用 `eval()` 执行：

```javascript
// 壳子写法
parse.list = $.toString(() => {
    list = pdfa(html, 'article');
    list.forEach((item) => { d.push({...}); });
});

// 在主页函数中执行
eval(parse.list);

// ✅ 手册标准替代：直接定义为函数方法
_parseList: function(html, host, ua) {
    var d = [];
    var items = pdfa(html, 'article');
    for (var i = 0; i < items.length; i++) {
        d.push({ ... });
    }
    return d;
},

// 在主页函数中调用
var d = this._parseList(html, this.host, this.UA);
```

> 📌 `$.toString()` + `eval()` 模式的问题：作用域不透明，依赖外部变量（`html`、`d`、`host` 等）隐式注入，调试困难，且 `eval` 在引擎沙箱里性能差。**推荐改用明确参数的函数调用**。

---

## 45.11 String 扩展方法——壳子注入的链式 API

壳子公共库给 String 原型注入了一批链式方法，用于快速格式化标题：

| 方法 | 说明 | 等价写法 |
|---|---|---|
| `.color('fff')` | 文字着色（fontcolor 别名）| `str.fontcolor('#fff')` |
| `.colorR(333)` | 随机色 + 文字（333=透明度变体）| 接近 `str.fontcolor(randomColor())` |
| `.sbR(n)` | 加粗 + 随机大小（n 控制字号范围）| `'<big>' + str + '</big>'` |
| `.big(n)` | HTML big 标签包裹 | `'<big>' + str + '</big>'` |
| `strong(str, color)` | 加粗着色 | `str.bold().fontcolor('#' + color)` |
| `strongR(str, color)` | 富文本格式化（含 blockquote 处理）| 复杂 HTML 格式化 |

> ⚠️ 这些方法来自壳子公共库，不是引擎原生 API，在不使用 `rc()` 的规则里无法使用。替代方案：直接用 `fontcolor()`、`bold()`，或用 `rich_text` col_type 自定义 HTML。

---

## 45.12 getRandomColor() ——壳子版随机颜色

壳子版 `getRandomColor(type)` 参数控制颜色风格：

```javascript
// 壳子写法（type=2 表示高饱和度色彩）
extra: { backgroundColor: getRandomColor(2) }

// ✅ 手册标准替代（第三十七章"学习追加"节的改进版）
var getRangeColors = function() {
    var h = Math.floor(Math.random() * 360),
        s = Math.floor(Math.random() * 50 + 50) / 100,
        l = Math.floor(Math.random() * 50 + 50) / 100,
        c = (1 - Math.abs(2 * l - 1)) * s,
        x = c * (1 - Math.abs((h / 60) % 2 - 1)),
        m = l - c/2,
        [r1,g1,b1] = h<60?[c,x,0]:h<120?[x,c,0]:h<180?[0,c,x]:h<240?[0,x,c]:h<300?[x,0,c]:[c,0,x],
        r=Math.floor((r1+m)*255), g=Math.floor((g1+m)*255), b=Math.floor((b1+m)*255);
    return '#' + ((1<<24) + (r<<16) + (g<<8) + b).toString(16).slice(1);
};
```

---

# 第四十六章：WordPress 主题型博客站规则开发（吃瓜/暗网站型）

本章基于 007吃瓜（WordPress + Mirages主题）和 51暗网同类型站点规则提炼，总结此类站点的完整开发模式。

**站点特征**：WordPress + 自定义主题、图文混合内容（视频 + 图片 + 文字 + 评论）、图片 AES 加密、域名频繁更换。

---

## 46.1 整体架构模式

```
parse 对象
├── host()         → 域名自愈函数（读缓存 → fetchCodeByWebView 发现 → 写缓存）
├── 主页()         → 分类渲染 + 列表加载，图片 AES 解密
├── list           → $.toString 字符串化的列表解析逻辑（改为函数后见下）
├── listTags       → 标签云渲染
├── listArchives   → 归档列表渲染
├── jiexi          → 详情页解析（视频 + 图文 + 评论）
└── 搜索()         → 搜索功能
```

---

## 46.2 域名发现与缓存（WordPress 型）

WordPress 型站点通常在跳转页 `#list-wrap&&.btnLink` 里列出所有备用域名：

```javascript
host: function(pathHost) {
    // Step1：读取持久化缓存
    var cached = getItem('site_host', '');
    if (cached) return cached;
    
    // Step2：从跳转页获取域名列表
    var landingHtml = fetchCodeByWebView('https://备用跳转页.com/', { timeout: 8000 });
    var host = '';
    if (landingHtml) {
        var urls = pdfa(landingHtml, '#list-wrap&&.btnLink').map(function(item) {
            return pdfh(item, 'Text');
        });
        // Step3：batchFetch 并发测速（替代 getFastestDomain）
        var tasks = urls.map(function(u) {
            return { url: 'https://' + u, options: { timeout: 3000 } };
        });
        var results = batchFetch(tasks);
        for (var i = 0; i < results.length; i++) {
            if (results[i] && results[i].indexOf('post-card') > -1) {
                host = 'https://' + urls[i].replace(/\/$/, '');
                break;
            }
        }
    }
    if (!host) host = 'https://备用兜底域名.com';
    setItem('site_host', host);
    // 设置1小时缓存有效期
    setItem('site_host_time', Date.now() + '');
    return host;
},
```

> ⚠️ **域名缓存必须加时间戳校验**：`setItem` 存的域名不会自动过期，需手动比较时间戳（1小时 = 3600000ms），否则域名更换后仍用旧缓存。

---

## 46.3 列表解析——图片 AES 解密标准写法

WordPress 站点图片 URL 通常经过 AES 加密存在 `data-*` 属性中：

```javascript
// 实战中的两种图片 URL 提取方式
// 方式A：从 z-image-loader-url 或 data-bg 属性取
var bgUrl = pd(item, 'img&&z-image-loader-url', host) || pd(item, '.lazy-bg&&data-bg', host);

// 方式B：从 script 标签 JS 内容中提取（51暗网）
var bgUrl = pdfh(item, 'script&&Html').split("'")[1];

// AES 解密图片（替代壳子的 imgDec，见第十章）
var key = 'f5d965df75336270';
var iv  = '97b60394abc2fbe1';
var tlazy = $('').image(function(k, iv_s) {
    var CryptoUtil = $.require('hiker://assets/crypto-java.js');
    var dec = CryptoUtil.AES.decrypt(
        CryptoUtil.Data.parseInputStream(input),
        CryptoUtil.Data.parseUTF8(k),
        { mode: 'AES/CBC/PKCS7Padding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
    );
    return dec.toInputStream();
}, key, iv);

// 列表卡片
d.push({
    title: title,
    desc: desc,
    pic_url: bgUrl + tlazy,  // 直接拼接解密 lazyRule
    col_type: 'pic_1_card',
    url: $(url).rule(parse._jiexi, host, key, iv)  // 进入详情页
});
```

---

## 46.4 详情页解析——视频 + 图文 + 评论混合

WordPress 详情页包含三类内容，需要分别提取：

```javascript
_jiexi: function(host, key, iv) {
    var html = getResCode();  // 或 fetch(MY_URL)
    var d = [];
    
    // ── 标题 ──────────────────────────────────────────
    d.push({
        title: pdfh(html, 'h1&&Text'),
        desc: pdfh(html, '.post-meta&&Text'),
        url: MY_URL, col_type: 'text_1',
        extra: { lineVisible: false }
    });
    
    // ── 视频提取（DPlayer 格式）────────────────────────
    // 方式A：直接从 .dplayer&&data-config 提取（无加密）
    var videoNodes = pdfa(html, 'body&&article&&.post-content>.dplayer');
    for (var vi = 0; vi < videoNodes.length; vi++) {
        try {
            var cfg = JSON.parse(pdfh(videoNodes[vi], '.dplayer&&data-config'));
            var vUrl = cfg.video ? cfg.video.url : cfg.url;
            if (vUrl) {
                d.push({
                    title: '点击播放视频',
                    url: vUrl,
                    col_type: 'text_center_1',
                    extra: { lineVisible: false }
                });
            }
        } catch(e) { log('视频解析失败: ' + e); }
    }
    
    // 方式B：动态 API 解密（1024吃瓜模式，见第四十四章）
    // var apiCfg = JSON.parse(pdfh(html, '.dplayer&&data-config').replace(/&quot;/g, '"'));
    // 从 apiCfg['data-api'] + apiCfg['data-key'] + apiCfg['data-iv'] 解密
    
    // ── 图文内容 ──────────────────────────────────────
    // 过滤掉无关内容（广告、备用地址等）
    var contentNodes = pdfa(
        pdfh(html, 'body&&Html').replace(/更多热门大瓜[\s\S]*/, ''),
        'body&&article&&.post-content>*'
    ).filter(function(x) {
        return /^<(p|hr|h2|h3|blockquote)>|dplayer/.test(x) && !/备用地址/.test(x);
    });
    
    for (var ci = 0; ci < contentNodes.length; ci++) {
        var it = contentNodes[ci];
        
        if (/dplayer/.test(it)) {
            // 跳过，视频已在上方单独处理
            continue;
        }
        
        if (it.indexOf('<hr>') > -1) {
            d.push({ col_type: 'line' });
            continue;
        }
        
        // 图片节点
        var imgNodes = pdfa(it, 'body&&img');
        var tlazy = $('').image(function(k, iv_s) {
            var CryptoUtil = $.require('hiker://assets/crypto-java.js');
            var dec = CryptoUtil.AES.decrypt(
                CryptoUtil.Data.parseInputStream(input),
                CryptoUtil.Data.parseUTF8(k),
                { mode: 'AES/CBC/PKCS7Padding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
            );
            return dec.toInputStream();
        }, key, iv);
        
        for (var ii = 0; ii < imgNodes.length; ii++) {
            var imgSrc = pdfh(imgNodes[ii], 'img&&data-src') || pdfh(imgNodes[ii], 'img&&src');
            if (imgSrc) {
                d.push({
                    title: '',
                    pic_url: imgSrc + tlazy,
                    url: 'pics://' + imgSrc,
                    col_type: 'pic_1_full'
                });
            }
        }
        
        // 站内链接跳转
        var linkHref = pdfh(it, 'a&&href');
        if (linkHref && /\d+$/.test(linkHref)) {
            d.push({
                title: pdfh(it, 'a&&Text'),
                url: $(linkHref.replace(/https?:\/\/.*?\//, host + '/')).rule(parse._jiexi, host, key, iv),
                col_type: 'text_1'
            });
            continue;
        }
        
        // 富文本段落
        var textContent = pdfh(it, 'body&&Text');
        if (textContent && textContent.trim().length > 3) {
            var cleaned = it
                .replace(/<\/?(br|p)>/g, '\n')
                .replace(/<h2>/g, '<big>')
                .replace(/<img[\s\S]*?>/g, '')
                .replace(/^[\n]+|[\n]+$/g, '');
            d.push({
                title: cleaned,
                col_type: 'rich_text'
            });
        }
    }
    
    // ── 评论区 ──────────────────────────────────────
    parse._renderComments(html, d, host);
    
    setResult(d);
},
```

---

## 46.5 评论区渲染——标准模板

WordPress 博客评论区结构固定，包含父评论和子回复两层：

```javascript
_renderComments: function(html, d, host) {
    var list = pdfa(html, 'body&&.comment-separator+ol>li');
    
    d.push({
        title: '已有 ' + list.length + ' 条评论',
        url: 'hiker://empty',
        col_type: 'text_icon'
    });
    
    for (var i = 0; i < list.length; i++) {
        var item = list[i];
        var isNested = /comment-children/.test(item);
        
        // 父评论
        d.push({
            title: pdfh(item, '.comment-author&&Text'),
            desc:  pdfh(item, '.comment-meta&&Text'),
            url:   'hiker://empty',
            col_type: 'avatar'
        });
        d.push({
            title: pdfh(item, '.comment-content&&Text'),
            url:   'hiker://empty',
            col_type: 'text_1',
            extra: { lineVisible: false }
        });
        
        // 子回复（嵌套评论）
        if (isNested) {
            var replies = pdfa(item, 'body&&.comment-list>li');
            for (var ri = 0; ri < replies.length; ri++) {
                d.push({
                    title: '　　' + pdfh(replies[ri], '.comment-reply-author&&Text'),
                    desc:  pdfh(item, '.comment-meta&&Text'),
                    url:   'hiker://empty',
                    col_type: 'avatar'
                });
                d.push({
                    title: '　　' + (
                        pdfh(replies[ri], '.comment-reply-author&&Text') + 
                        '　' + pdfh(replies[ri], 'p--span&&Text')
                    ),
                    url:   'hiker://empty',
                    col_type: 'text_1'
                });
            }
            d.push({ col_type: 'line' });
        }
    }
},
```

---

## 46.6 标签云渲染——flex_button 彩色标签

WordPress 标签页通常在 `#archives-tags` 容器中，用 `flex_button` 渲染彩色标签云：

```javascript
_renderTagCloud: function(html, d, host) {
    var tags = pdfa(html, '#archives-tags&&.itags');
    for (var i = 0; i < tags.length; i++) {
        var tagName = pdfh(tags[i], 'a&&Text');
        var tagHref = pd(tags[i], 'a&&href');
        d.push({
            title: tagName,
            col_type: 'flex_button',
            // 标签点击 → 新开独立列表页（带翻页）
            url: $('hiker://empty##' + tagHref + '##fypage').rule(function(host, key, iv) {
                var d = [];
                setPageTitle(getPageTitle().split(' ')[0] + ' - ' + MY_PAGE);
                var tagUrl = MY_URL.split('##')[1] + MY_PAGE + '/';
                var html = fetch(tagUrl, { headers: { 'User-Agent': 'pc' } });
                // 解析列表...
                setResult(d);
            }, host, key, iv),
            extra: { backgroundColor: getRangeColors() }
        });
    }
    d.push({ col_type: 'line' });
},
```

---

## 46.7 归档列表渲染——archive 时间线

```javascript
_renderArchives: function(html, d, host, key, iv) {
    // 归档标题（如"2024年12月"）
    d.push({
        title: pdfh(html, '.archive-title&&h2&&Text'),
        col_type: 'rich_text',
        url: 'hiker://empty'
    });
    
    var items = pdfa(html, '.archive-title&&.brick');
    for (var i = 0; i < items.length; i++) {
        var it = items[i];
        d.push({
            title: pdfh(it, '.time&&Text') + '  ' + pdfh(it, 'a--span&&Text'),
            col_type: 'text_1',
            url: $(pd(it, 'a&&href')).rule(parse._jiexi, host, key, iv)
        });
    }
},
```

---

## 46.8 完整规则骨架（WordPress 型，无远程调用）

```javascript
var parse = {
    作者: 'dev',
    版本: '20260408.V1',
    
    // ── 配置区 ──────────────────────────────────────────
    _defaultHost: 'https://备用域名.com',
    _key: 'f5d965df75336270',
    _iv:  '97b60394abc2fbe1',
    UA: 'Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36',
    
    页码: { '主页': 1 },
    
    静态分类: {
        type: '主页',
        url: 'fyclass',
        class_name: '首页&分类A&分类B&热门标签&往期内容',
        class_url: '&cata&catb&tags&archives'
    },
    
    // ── 域名自愈 ─────────────────────────────────────────
    _getHost: function() {
        var cached = getItem('wp_host', '');
        var cachedTime = parseInt(getItem('wp_host_time', '0'));
        if (cached && (Date.now() - cachedTime) < 3600000) return cached;
        
        // 从公告页发现域名列表
        try {
            var html = fetchCodeByWebView('https://公告页.com/', {
                timeout: 8000, javaScriptEnabled: true
            });
            if (html) {
                var urls = pdfa(html, '#list-wrap&&.btnLink').map(function(item) {
                    return pdfh(item, 'Text');
                });
                var tasks = urls.map(function(u) {
                    return { url: 'https://' + u, options: { timeout: 3000 } };
                });
                var results = batchFetch(tasks);
                for (var i = 0; i < results.length; i++) {
                    if (results[i] && results[i].indexOf('post-card') > -1) {
                        var host = 'https://' + urls[i].replace(/\/$/, '');
                        setItem('wp_host', host);
                        setItem('wp_host_time', Date.now() + '');
                        return host;
                    }
                }
            }
        } catch(e) { log('域名发现失败: ' + e); }
        
        setItem('wp_host', this._defaultHost);
        setItem('wp_host_time', Date.now() + '');
        return this._defaultHost;
    },
    
    // ── 主页 ─────────────────────────────────────────────
    主页: function() {
        var d = [];
        var host = this._getHost();
        var key  = this._key;
        var iv   = this._iv;
        var ua   = this.UA;
        var cls  = MY_URL && MY_URL !== 'fyclass' ? MY_URL : '';
        
        // 构建请求 URL
        var url;
        if (!cls) {
            url = host + '/page/' + MY_PAGE + '/';
        } else if (cls === 'tags') {
            url = host + '/tags/' + MY_PAGE + '/';
        } else if (cls === 'archives') {
            url = host + '/archives/page/' + MY_PAGE + '/';
        } else {
            url = host + '/category/' + cls + '/' + MY_PAGE + '/';
        }
        
        var html = fetch(url, { headers: { 'User-Agent': ua } });
        if (!html || html.indexOf('post-card') === -1) {
            // 域名失效，清除缓存重试
            setItem('wp_host', '');
            toast('域名可能已更换，请重新刷新~');
            return d;
        }
        
        // 按分类分流
        if (cls === 'tags') {
            this._renderTagCloud(html, d, host);
        } else if (cls === 'archives') {
            this._renderArchives(html, d, host, key, iv);
        } else {
            d = d.concat(this._parseList(html, host, key, iv, ua));
        }
        
        return d;
    },
    
    // ── 列表解析 ─────────────────────────────────────────
    _parseList: function(html, host, key, iv, ua) {
        var d = [];
        var tlazy = $('').image(function(k, iv_s) {
            var CryptoUtil = $.require('hiker://assets/crypto-java.js');
            var dec = CryptoUtil.AES.decrypt(
                CryptoUtil.Data.parseInputStream(input),
                CryptoUtil.Data.parseUTF8(k),
                { mode: 'AES/CBC/PKCS7Padding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
            );
            return dec.toInputStream();
        }, key, iv);
        
        var items = pdfa(html, 'div[role="main"]&&article:has(h2)').filter(function(x) {
            var t = pdfh(x, 'h2&&Text');
            return t !== '' && !/更新通知|^高端约炮$/.test(t);
        });
        
        for (var i = 0; i < items.length; i++) {
            var it = items[i];
            // 图片 URL 提取（两种站点格式兼容）
            var bgUrl = pd(it, 'img&&z-image-loader-url', host) 
                     || pd(it, '.lazy-bg&&data-bg', host)
                     || pdfh(it, 'script&&Html').split("'")[1]
                     || '';
            var title = pdfh(it, 'h2&&Text');
            var url   = pd(it, 'a&&href', host);
            var desc  = pdfh(it, '.post-card-info&&Text');
            
            if (!url) continue;
            
            d.push({
                title: title,
                desc:  desc,
                pic_url: bgUrl ? bgUrl + tlazy : '',
                col_type: 'pic_1_card',
                url: $(url).rule(parse._jiexi, host, key, iv)
            });
        }
        return d;
    },
    
    // ── 详情页（rule 内调用，见 46.4）────────────────────
    _jiexi: function(host, key, iv) {
        var d = [];
        var html = fetch(MY_URL, { headers: { 'User-Agent': 'pc' } });
        // ... 见 46.4 完整实现
        setResult(d);
    },
    
    // ── 评论区（见 46.5）────────────────────────────────
    _renderComments: function(html, d, host) { /* 见 46.5 */ },
    
    // ── 标签云（见 46.6）────────────────────────────────
    _renderTagCloud: function(html, d, host) { /* 见 46.6 */ },
    
    // ── 归档列表（见 46.7）──────────────────────────────
    _renderArchives: function(html, d, host, key, iv) { /* 见 46.7 */ },
    
    // ── 搜索 ─────────────────────────────────────────────
    搜索: function(name) {
        var d = [];
        var host = this._getHost();
        var html = fetch(host + '/search/' + encodeURIComponent(name) + '/' + page + '/');
        return this._parseList(html, host, this._key, this._iv, this.UA);
    }
};
```

---

## 46.9 本章新增铁律

| 编号 | 铁律名称 | 核心要义 | 关键口诀 |
|---|---|---|---|
| Law 88 | 禁止远程调用 | 禁止 `rc()`/`gfd()`/`fc()` 远程加载，所有逻辑内联 | 远程库 = 不可控，内联 = 稳定 |
| Law 89 | 域名缓存加时间戳 | `setItem` 缓存域名必须同时存时间戳，1小时后重新发现 | 无时间戳 = 域名换了还用旧缓存 |
| Law 90 | $.toString+eval禁用 | 禁止 `$.toString()` + `eval()` 模式，改为明确参数的函数 | eval 调试难，函数调用清晰 |
| Law 91 | 内容过滤必须显式 | 过滤广告/无关内容用具体正则（如 `/更多热门大瓜/`），不依赖壳子 | 正则 = 可维护，壳子函数 = 不可移植 |

---

# 第四十七章：WordPress Elementor 图集站规则开发（海角爱 型）

本章基于 haijiaolove.xyz（WordPress + Elementor 页面生成器）规则提炼，总结此类用 Elementor 构建的图集展示站的开发模式。

**站点特征**：Elementor widget 布局、多层嵌套 section、图集 + 视频混合、谷歌搜索引流。

---

## 47.1 Elementor 站点 HTML 结构特点

Elementor 生成的 HTML 有固定的嵌套模式：

```html
<article>
  <section>  <!-- 第一个 section 通常是导航/TOC -->
    <div class="uael-toc-heading">...</div>
    <ul class="uael-toc-list">...</ul>
  </section>
  <section>  <!-- 后续 section 是内容区块 -->
    <div class="elementor-widget-container">
      <h3>...</h3>
      <div class="e-gallery-image" data-thumbnail="..."></div>
      <iframe src="..."></iframe>
    </div>
  </section>
</article>
```

---

## 47.2 图集列表页解析

```javascript
主页: function() {
    var d = [];
    var host = this.host;
    var ua   = this.UA;
    var html = fetch(host, { headers: { 'User-Agent': ua } });
    
    // Elementor 图集列表
    var items = pdfa(html, 'body&&.e-gallery-item');
    for (var i = 0; i < items.length; i++) {
        var item = items[i];
        var thumb = pd(item, 'body&&.e-gallery-image&&data-thumbnail');
        // 分类 URL 藏在 elementor-gallery-item__description 的 Text 里
        var categoryUrl = pdfh(item, 'body&&.elementor-gallery-item__description&&Text');
        
        d.push({
            title: '',
            desc:  '0',
            pic_url: thumb,
            col_type: 'card_pic_2',
            // 点击进入该分类的文章列表（带翻页）
            url: $('hiker://empty##fypage').rule(function(catUrl, host, ua) {
                var d = [];
                var pageUrl = catUrl + '?_page=' + MY_PAGE;
                var html = fetch(pageUrl, { headers: { 'User-Agent': ua } });
                
                // 文章卡片
                var cards = pdfa(html, 'body&&.pt-cv-content-item');
                for (var j = 0; j < cards.length; j++) {
                    var card = cards[j];
                    d.push({
                        title: pdfh(card, 'body&&h4&&Text'),
                        desc:  pdfh(card, 'body&&.pt-cv-ctf-value&&Text'),
                        pic_url: pd(card, 'body&&.pt-cv-thumb-wrapper&&img&&data-cvpsrc'),
                        col_type: 'movie_2',
                        url: $('hiker://empty##' + pd(card, 'body&&a&&href')).rule(function() {
                            // 进入 Elementor 详情页
                            var d = [];
                            var pageUrl = MY_URL.split('##')[1];
                            var html = fetch(pageUrl, { headers: { 'User-Agent': 'pc' } });
                            parse._parseElementorDetail(html, d);
                            setResult(d);
                        })
                    });
                }
                setResult(d);
            }, categoryUrl, host, ua)
        });
    }
    return d;
},
```

---

## 47.3 Elementor 详情页解析

```javascript
_parseElementorDetail: function(html, d) {
    var sections = pdfa(html, 'article&&section');
    for (var si = 0; si < sections.length; si++) {
        var section = sections[si];
        
        // 第一个 section：TOC 导航
        if (si === 0) {
            d.push({
                title: pdfh(section, '.uael-toc-heading&&Text'),
                col_type: 'rich_text'
            });
            var tocItems = pdfa(section, '.uael-toc-list&&li');
            for (var ti = 0; ti < tocItems.length; ti++) {
                d.push({
                    title: pdfh(tocItems[ti], 'a&&Text'),
                    url: 'hiker://empty',
                    col_type: 'text_icon'
                });
            }
            continue;
        }
        
        // 后续 section：内容区块
        var widgets = pdfa(section, 'body&&.elementor-widget-container');
        for (var wi = 0; wi < widgets.length; wi++) {
            var widget = widgets[wi];
            
            // h3 标题
            var h3s = pdfa(widget, 'body&&h3');
            for (var hi = 0; hi < h3s.length; hi++) {
                d.push({ title: pdfh(h3s[hi], 'body&&Text'), col_type: 'rich_text' });
            }
            
            // h4 小标题
            var h4s = pdfa(widget, 'body&&h4');
            for (var h4i = 0; h4i < h4s.length; h4i++) {
                d.push({ title: pdfh(h4s[h4i], 'body&&Text'), col_type: 'rich_text' });
            }
            
            // 图片（e-gallery）
            var galleryImgs = pdfa(widget, 'body&&.e-gallery-image');
            for (var gi = 0; gi < galleryImgs.length; gi++) {
                d.push({
                    pic_url: pd(galleryImgs[gi], '.e-gallery-image&&data-thumbnail'),
                    col_type: 'pic_2_card'
                });
            }
            
            // 视频 iframe
            var iframes = pdfa(widget, 'body&&iframe');
            if (iframes.length > 0) {
                d.push({ col_type: 'blank_block' });
                for (var ii = 0; ii < iframes.length; ii++) {
                    var iframeSrc = pdfh(iframes[ii], 'iframe&&src');
                    d.push({
                        title: '点击播放视频',
                        url: $(iframeSrc).lazyRule(function() {
                            return 'video://' + input;
                        }),
                        col_type: 'text_2'
                    });
                }
                d.push({ col_type: 'blank_block' });
            }
        }
    }
},
```

---

## 47.4 谷歌搜索引流——搜索函数

部分无自带搜索的站点借用 Google `site:` 搜索：

```javascript
搜索: function(name) {
    var d = [];
    // ⚠️ 谷歌对爬取有限制，频繁调用会被封，仅作参考
    var url = 'https://www.google.com/search?q=' + encodeURIComponent(name) 
            + '+site:' + 'example.com' 
            + '&start=' + (page - 1) * 10;
    var html = fetchPC(url);  // 必须用 PC UA
    
    if (/errors|captcha/.test(html)) {
        toast('搜索被限制，稍后再试！');
        return d;
    }
    
    var items = pdfa(html, 'body&&.MjjYud');
    for (var i = 0; i < items.length; i++) {
        var it = items[i];
        var link = pdfh(it, 'a&&href');
        if (!link) continue;
        d.push({
            title:   pdfh(it, 'h3&&Text'),
            desc:    pdfh(it, 'div[style="-webkit-line-clamp:2"]&&Text'),
            col_type: 'text_1',
            url: $(link).lazyRule(function() { return input; })
        });
    }
    return d;
},
```

> ⚠️ **谷歌搜索限制说明**：Google 会对频繁的非浏览器请求返回 CAPTCHA 页面（特征：`/errors/` 路径或含 `captcha` 字样），必须检测并提示用户。此方案不稳定，优先考虑站点自带搜索。

---

# 第四十八章：综合工具函数内联库——完全脱离壳子的独立实现

本章将各章散落的工具函数整合为一份可直接内联到任何规则的"本地工具库"，彻底消除对 `rc()` 壳子的依赖。

---

## 48.1 完整内联工具库（复制到 parse 对象即可使用）

```javascript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 聚阅/海阔视界 内联工具库 V1.0（无远程依赖）
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// ── 随机颜色（排除低亮度/低饱和度）────────────────────────
var _getRangeColors = function() {
    var h = Math.floor(Math.random() * 360),
        s = Math.floor(Math.random() * 50 + 50) / 100,
        l = Math.floor(Math.random() * 50 + 50) / 100,
        c = (1 - Math.abs(2 * l - 1)) * s,
        x = c * (1 - Math.abs((h / 60) % 2 - 1)),
        m = l - c/2,
        r1, g1, b1;
    if      (h < 60)  { r1=c; g1=x; b1=0; }
    else if (h < 120) { r1=x; g1=c; b1=0; }
    else if (h < 180) { r1=0; g1=c; b1=x; }
    else if (h < 240) { r1=0; g1=x; b1=c; }
    else if (h < 300) { r1=x; g1=0; b1=c; }
    else              { r1=c; g1=0; b1=x; }
    var r=Math.floor((r1+m)*255), g=Math.floor((g1+m)*255), b=Math.floor((b1+m)*255);
    return '#' + ((1<<24)+(r<<16)+(g<<8)+b).toString(16).slice(1);
};

// ── Base64 解码（双保险，Law 37）──────────────────────────
var _base64Decode = function(str) {
    if (!str) return '';
    str = str.replace(/-/g, '+').replace(/_/g, '/');
    while (str.length % 4) str += '=';
    try {
        var b = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
        return new java.lang.String(b, 'UTF-8') + '';
    } catch(e) {
        try { return decodeURIComponent(escape(atob(str))); }
        catch(e2) { log('Base64 decode fail'); return ''; }
    }
};

// ── 自动脱壳（最多3层，Law 39）───────────────────────────
var _autoDeshell = function(html) {
    if (!html) return html;
    for (var i = 0; i < 3; i++) {
        if (html.indexOf('decodeURIComponent("') > -1) {
            var m = html.match(/decodeURIComponent\("([^"]+)"\)/);
            if (m) html = decodeURIComponent(m[1]);
        } else break;
    }
    return html;
};

// ── AES 图片解密 lazyRule 生成（Law 28，复用版）─────────
var _getAesImageLazy = function(key, iv) {
    return $('').image(function(k, iv_s) {
        try {
            var CryptoUtil = $.require('hiker://assets/crypto-java.js');
            var dec = CryptoUtil.AES.decrypt(
                CryptoUtil.Data.parseInputStream(input),
                CryptoUtil.Data.parseUTF8(k),
                { mode: 'AES/CBC/PKCS7Padding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
            );
            return dec.toInputStream();
        } catch(e) { log('AES img fail: ' + e); return input; }
    }, key, iv);
};

// ── URL 翻页构建（兼容 ? 和路径翻页）────────────────────
var _buildPageUrl = function(base, page) {
    if (page <= 1) return base;
    if (base.indexOf('?') > -1) return base + '&page=' + page;
    return base.replace(/\/?$/, '') + '/page/' + page + '/';
};

// ── 中文路径安全编码（Law 17）────────────────────────────
var _safeEncodeUrl = function(url) {
    // 只对非 ASCII 字符编码，保留 ://?=&#
    return url.replace(/[^\x00-\x7F]/g, function(c) {
        return encodeURIComponent(c);
    });
};

// ── 多域名竞速（替代 getFastestDomain）──────────────────
var _getFastestHost = function(candidates, keyword, timeout) {
    var tasks = candidates.map(function(u) {
        return { url: u, options: { timeout: timeout || 3000 } };
    });
    var results = batchFetch(tasks);
    for (var i = 0; i < results.length; i++) {
        if (results[i] && (keyword ? results[i].indexOf(keyword) > -1 : results[i].length > 100)) {
            return candidates[i];
        }
    }
    return candidates[0];  // 兜底返回第一个
};

// ── 滚动按钮选中样式（纯文本符号，Law 42）────────────────
var _selTitle = function(title, isSel) {
    return isSel ? '❆ ' + title + ' ❆' : title;
};

// ── Dean Edwards Packer 解包（第三十四章）────────────────
var _unpack = function(code) {
    var pm = code.match(/\('([\s\S]*?)',(\d+),(\d+),'([\s\S]*?)'\.split\('\|'\)/);
    if (!pm) return code;
    var p = pm[1], a = parseInt(pm[2]), c = parseInt(pm[3]), k = pm[4].split('|');
    function e(c) {
        return (c < a ? '' : e(parseInt(c/a))) +
               ((c = c%a) > 35 ? String.fromCharCode(c+29) : c.toString(36));
    }
    var dict = {};
    while (c--) { var key = e(c); dict[key] = k[c] || key; }
    return p.replace(/\b\w+\b/g, function(w) {
        return dict[w] !== undefined ? dict[w] : w;
    });
};
```

---

## 48.2 壳子 API → 手册标准 API 对照速查

| 壳子函数 | 说明 | 手册标准替代 |
|---|---|---|
| `rc(url, hours)` | 远程加载 JS 库 | 禁止，内联逻辑（Law 88）|
| `gfd()` | 获取 GitHub 代理 | 禁止使用 |
| `fc(url)` | 获取代理 JSON | 禁止使用 |
| `getHtml(url, ...)` | 带 CF 处理的 fetch | `fetch()` + 第十二章 CF 过盾 |
| `classTop(idx, item, host, d)` | 渲染分类按钮行 | `renderRow()`（第六章）|
| `cpage('')` | 解析分类和页码 | `MY_URL` 直接使用（Law 60）|
| `pageAdd(page, host)` | 记录翻页信息 | `putMyVar` 手动管理 |
| `pageMoveto(...)` | 生成位置记忆 extra | 不推荐，用标准翻页 |
| `getdTemp(...)` | 读取页面缓存 | `getItem` + `JSON.parse` |
| `writeFile(path, content)` | 写文件缓存 | `setItem` 持久化 |
| `getFastestDomain(urls)` | 多域名测速 | `batchFetch` + 特征检测（见 45.8）|
| `imgDec(key, iv, 'AES')` | AES 图片解密 lazyRule | `$('').image()` + CryptoUtil（第十章）|
| `safePath(url)` | URL 转文件名 | 不需要，改用 `setItem` key |
| `getRandomColor(type)` | 随机颜色 | `_getRangeColors()`（第三十七章）|
| `.color('fff')` | 字符串着色 | `.fontcolor('#fff')` |
| `.colorR(333)` | 随机着色 | `.fontcolor(randomColor())` |
| `.sbR(n)` | 加粗放大 | `'<big>' + str + '</big>'` |
| `strong(str, color)` | 加粗着色 | `str.bold().fontcolor('#' + color)` |
| `$.toString(() => {...})` | 函数字符串化 | 改为函数方法，显式传参 |
| `eval(parse.list)` | 执行字符串化代码 | 改为 `this._parseList(html, ...)` |

---

# 附录 B：铁律总表更新（Law 88-91）

| 编号 | 铁律名称 | 核心要义 | 关键口诀 |
|---|---|---|---|
| Law 88 | 禁止远程调用 | 禁止 `rc()`/`gfd()`/`fc()`，所有逻辑必须内联 | 远程库 = 不可控，内联 = 稳定 |
| Law 89 | 域名缓存时间戳 | 域名 `setItem` 缓存必须同时记时间戳，超时重新发现 | 无时间戳 = 域名换了还用旧缓存 |
| Law 90 | 禁用 $.toString+eval | 禁止 `$.toString()`+`eval()` 代码字符串化，改为函数方法 | eval 调试难，函数调用清晰 |
| Law 91 | 内容过滤显式化 | 过滤广告/无关内容用具体正则，不依赖壳子内置过滤 | 正则可维护，壳子函数不可移植 |

---

*文档版本 V3.46 · 2026年4月 · 新增第四十五-四十八章（壳子工具函数参考、WordPress站型、Elementor站型、内联工具库）· 去除 rc()/gfd()/fc() 远程调用，统一内联实现 · 基于 007吃瓜/51暗网/海角爱实战规则提炼*

---

# 第四十九章：手册勘误与补充——基于实战验证的修正

本章记录手册中经实战验证发现的错误、不足和遗漏，作为对前述章节的正式修订。

---

## 49.1 严重错误修正

### 错误1：第十章 $.image() 的 input 参数说明错误

**手册原文描述（错误）**：`$.image()` 的 func 中，input 是图片 URL，需要先 fetch 获取加密数据。

**实际验证结论**：`$.image()` 回调中的 `input` 是引擎**已经下载好的 InputStream**，不是 URL 字符串，不需要也不能再手动 `fetch`。手动 fetch 会触发类型错误：`fetch argument 1 has type java.lang.String, got okio.RealBufferedSource`。

**错误写法**：
```javascript
// ❌ 错误：input 已经是 InputStream，不能当 URL 传给 fetch
var tlazy = $('').image(function(k, iv_s) {
    var raw = fetch(input, { headers: {...} });  // 报错！
    var CryptoUtil = $.require('hiker://assets/crypto-java.js');
    var dec = CryptoUtil.AES.decrypt(CryptoUtil.Data.parseInputStream(raw), ...);
});
```

**正确写法**：
```javascript
// ✅ 正确：input 直接就是 InputStream，可以直接传给 parseInputStream
var tlazy = $('').image(function(k, iv_s) {
    var CryptoUtil = $.require('hiker://assets/crypto-java.js');
    // input 直接可用，它是引擎下载图片后传入的 InputStream
    var textData  = CryptoUtil.Data.parseInputStream(input);
    var decrypted = CryptoUtil.AES.decrypt(textData,
        CryptoUtil.Data.parseUTF8(k),
        { mode: 'AES/CBC/NoPadding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
    );
    return decrypted.toInputStream();
}, key, iv);
```

> **⚠️ Law 92（新增）：`$.image()` 回调中的 `input` 是引擎下载完成后传入的 `InputStream`，不是 URL 字符串，禁止对 `input` 调用 `fetch()`。直接将 `input` 传给 `CryptoUtil.Data.parseInputStream()` 即可。**

---

### 错误2：padding 模式不能默认 PKCS7Padding

**手册原文描述（不完整）**：示例统一使用 `AES/CBC/PKCS7Padding`，给读者暗示这是通用选择。

**实际验证结论**：不同站点的 padding 模式不同。padding 模式写错会导致解密结果乱码或解密失败，且不会抛出明显错误，极难调试。必须从站点 JS 源码中确认。

| padding 模式 | 特征 | 常见于 |
|---|---|---|
| `PKCS7Padding` | 明文长度不是 16 的倍数时自动补齐 | 多数标准实现 |
| `NoPadding` | 明文长度必须是 16 的整数倍 | 部分自定义实现 |
| `PKCS5Padding` | 与 PKCS7 等价，Java 环境常见别名 | Java 后端 |

**如何从站点源码确认 padding 模式**：
```javascript
// 在站点 JS bundle 中搜索以下关键词：
// - "NoPadding"
// - "PKCS7"  / "PKCS7Padding"
// - "PKCS5"  / "PKCS5Padding"
// - CryptoJS.AES.decrypt(...)  → 查看第三个参数的 padding 字段

// 示例：站点源码含
// CryptoJS.AES.decrypt(data, key, { iv: iv, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.NoPadding })
// → 对应规则写法：mode: 'AES/CBC/NoPadding'
```

> **⚠️ Law 93（新增）：padding 模式必须从站点 JS 源码中确认，不能假设为 `PKCS7Padding`。常见站点使用 `NoPadding`，写错会导致解密失败且无明显报错。**

---

## 49.2 重要不足补充

### 补充1：$.image() input 参数的完整说明

`$.image()` 的执行流程如下，手册第十章未做完整说明：

```
1. 引擎遇到 pic_url 字段中拼接了 $.image() 生成的 lazyRule 字符串
2. 引擎自动下载该图片 URL 对应的字节流
3. 将下载结果封装为 InputStream，赋值给 input 变量
4. 执行 $.image() 的 func 回调，func 内可直接使用 input
5. func 必须返回 InputStream，引擎用于渲染图片
```

**关键约束总结**：
- `input` 类型：`InputStream`（`okio.RealBufferedSource`）
- `input` 来源：引擎自动下载，开发者无需手动 fetch
- `func` 返回值：必须是 `InputStream`（通过 `toInputStream()` 获得），返回 `null` 则图片不显示
- **`$.image()` 回调不能使用箭头函数**（Law 31.5.3），否则 `input` 为 `undefined`

**完整可用的图片防盗链 + AES 解密模板**：
```javascript
// ✅ 完整正确写法（无 AES 加密，仅防盗链）
var tlazy = $('').image(function(referer) {
    var FU = com.example.hikerview.utils.FileUtil;
    // 防盗链：用 FileUtil.readBytes 自定义 headers 下载
    // 注意：这里的 input 是已下载的 InputStream，
    // 如果需要自定义 headers，要用另一种方式（见下）
    return FU.toInputStream(FU.readBytes(
        // 此处不能传 input！input 已经是流了
        // 这种模式用于 pic_url 本身就是 URL 的情况
        // 需要在 pic_url 后拼 @Referer= 来处理防盗链
    ));
    return input;  // 无需解密时直接返回
}, referer);

// ✅ 图片 AES 解密（input 已是流）
var tlazy = $('').image(function(k, iv_s) {
    var CryptoUtil = $.require('hiker://assets/crypto-java.js');
    var dec = CryptoUtil.AES.decrypt(
        CryptoUtil.Data.parseInputStream(input),  // ✅ input 直接用
        CryptoUtil.Data.parseUTF8(k),
        { mode: 'AES/CBC/NoPadding', iv: CryptoUtil.Data.parseUTF8(iv_s) }
    );
    return dec.toInputStream();
}, key, iv);
```

> 📌 **防盗链（无解密）直接用 `pic_url + '@Referer=xxx'` 或 `'@headers={...}'` 格式处理，不需要 `$.image()`。`$.image()` 只用于需要字节流变换（解密/XOR）的场景。**

---

### 补充2：eval 字符串代码模式——复用解密逻辑的正确姿势

当需要在 `lazyRule`/`rule` 回调内部复用解密代码时，由于 lazyRule 的作用域隔离，无法直接访问外部函数。正确做法是把代码存为字符串，内部用 `eval` 执行：

```javascript
var parse = {
    // 把 tlazy 生成代码存为字符串属性
    _imageLazyCode: 'var tlazy = $(\'\').image(function(k, iv_s) {\
        var CryptoUtil = $.require(\'hiker://assets/crypto-java.js\');\
        var dec = CryptoUtil.AES.decrypt(\
            CryptoUtil.Data.parseInputStream(input),\
            CryptoUtil.Data.parseUTF8(k),\
            { mode: \'AES/CBC/NoPadding\', iv: CryptoUtil.Data.parseUTF8(iv_s) }\
        );\
        return dec.toInputStream();\
    }, \'' + KEY + '\', \'' + IV + '\');',

    // 在需要 tlazy 的地方 eval 执行，得到 tlazy 变量
    _getPageRule: function() {
        return $('').lazyRule(function(lazyCode) {
            eval(lazyCode);  // 执行后 tlazy 变量可用
            var d = [];
            var pics = [];
            // ... 解析图片 URL ...
            for (var i = 0; i < pics.length; i++) {
                d.push({ pic_url: pics[i] + tlazy, col_type: 'pic_1_full' });
            }
            return 'pics://' + pics.map(function(p) { return p; }).join('&&');
        }, this._imageLazyCode);
    }
};
```

> ⚠️ **eval 字符串模式的注意事项**：字符串内的引号需要转义；字符串较长时可读性差；调试困难。**优先用参数传递 key/iv，在 lazyRule 内部直接定义解密逻辑**；只有当解密代码非常长且需要在多个 lazyRule 中复用时，才考虑此模式（Law 90 精神：优先函数方法，eval 作为最后手段）。

---

### 补充3：二级函数完整返回结构与 noShow 用法（补充第七章）

手册第七章给出了基础结构，以下补充完整字段说明和 `noShow + extenditems` 的实战模板。

**noShow 字段完整说明**：

```javascript
noShow: {
    封面: false,   // true = 隐藏封面图区域（默认显示）
    简介: true,    // true = 隐藏默认简介区域
    排序: true,    // true = 隐藏选集排序按钮
    选集: true     // true = 隐藏默认选集列表区域
}
// 注意：key 是中文字符串，不是英文
```

**extenditems 完全自定义二级页面模板**：

```javascript
二级: function(url) {
    var html  = fetch(url, { headers: { 'User-Agent': this.UA } });
    var host  = this.host;
    var extra = [];

    // ── 自定义内容区（图文混排示例）──────────────────────
    var nodes = pdfa(html, 'body&&.text-content&&*');
    for (var i = 0; i < nodes.length; i++) {
        var node = nodes[i];
        // 图片节点
        var imgUrl = '';
        try { imgUrl = pdfh(node, 'img&&z-image-loader-url') || pdfh(node, 'img&&src'); } catch(e) {}
        if (imgUrl) {
            imgUrl = imgUrl.replace(/`/g, '').trim();
            extra.push({
                title:    '',
                pic_url:  imgUrl + '@Referer=' + host + '/',
                url:      'pics://' + imgUrl,
                col_type: 'pic_1_full'
            });
            continue;
        }
        // 视频节点（DPlayer）
        var cfg = '';
        try { cfg = pdfh(node, 'self&&data-config'); } catch(e) {}
        if (cfg) {
            try {
                var cfgObj = JSON.parse(cfg.replace(/&quot;/g, '"'));
                var vUrl   = cfgObj.video && cfgObj.video.url ? cfgObj.video.url : '';
                if (vUrl) {
                    var sl = String.fromCharCode(92);
                    extra.push({
                        title:    '▶ 点击播放',
                        url:      'video://' + vUrl.split(sl).join('') + ';{Referer@' + host + '/}',
                        col_type: 'text_center_1'
                    });
                }
            } catch(e) {}
            continue;
        }
        // 文字节点
        var txt = '';
        try { txt = pdfh(node, 'Text'); } catch(e) {}
        if (txt && txt.trim().length > 3) {
            extra.push({ title: txt.trim(), col_type: 'rich_text' });
        }
    }

    return {
        vod_name:    pdfh(html, 'h1&&Text'),
        vod_pic:     '',
        // ⚠️ 使用 extenditems 时：
        // 1. list 必须给 [[]]（空二维数组），否则引擎报结构错误
        // 2. noShow 按需隐藏默认组件
        // 3. extenditems 放自定义内容，在选集区域下方渲染
        list:        [['']],
        noShow:      { 封面: true, 简介: true, 排序: true, 选集: true },
        extenditems: extra
    };
},
```

> **⚠️ Law 94（新增）：使用 `extenditems` 完全自定义二级页面时，`list` 字段必须给 `[['']]`（含一个空字符串的二维数组），否则引擎抛结构错误。`noShow` 按需隐藏不需要的默认组件，`extenditems` 中的内容在选集区域下方渲染。**

---

### 补充4：图文按 HTML 顺序穿插显示

手册只说了分别提取图片和文字，但实际上内容页通常图文穿插，需要按原始 HTML 中元素的出现顺序渲染，而不是先渲染所有图片再渲染所有文字。

**错误做法（先图后文，顺序错乱）**：
```javascript
// ❌ 先单独提取图片，再单独提取文字，顺序完全乱掉
var imgs  = pdfa(html, 'body&&.text-content&&img');
var paras = pdfa(html, 'body&&.text-content&&p');
for (var i = 0; i < imgs.length; i++) { d.push({...img...}); }
for (var j = 0; j < paras.length; j++) { d.push({...text...}); }
```

**正确做法（按 HTML 元素顺序遍历）**：
```javascript
// ✅ 遍历直接子节点，按出现顺序处理每个元素
var nodes = pdfa(html, 'body&&.text-content&&*');  // 取所有直接子节点
for (var i = 0; i < nodes.length; i++) {
    var node = nodes[i];
    // 先判断是否含图片
    var imgUrl = '';
    try { imgUrl = pdfh(node, 'img&&z-image-loader-url') || pdfh(node, 'img&&src'); } catch(e) {}
    if (imgUrl && imgUrl.indexOf('http') === 0) {
        d.push({ pic_url: imgUrl + '@Referer=' + host + '/', col_type: 'pic_1_full', url: 'pics://' + imgUrl });
        continue;
    }
    // 再判断是否有文字
    var txt = '';
    try { txt = pdfh(node, 'Text'); } catch(e) {}
    if (txt && txt.trim().length > 3) {
        d.push({ title: txt.trim(), col_type: 'rich_text' });
    }
}
```

> **⚠️ Law 95（新增）：详情页图文穿插渲染，必须用 `pdfa(html, '内容容器&&*')` 遍历所有直接子节点，在循环内按节点类型分流处理，保持原 HTML 顺序。不能分别提取图片和文字再拼接，否则顺序错乱。**

---

### 补充5：getResCode() vs fetch() 区别说明

手册中两个函数都出现但未做区分说明，实际使用场景不同：

| | `getResCode()` | `fetch(url, opts)` |
|---|---|---|
| **可用位置** | 仅限 `$().rule()` 沙箱内部 | 任意位置（主页、二级、lazyRule 等）|
| **获取内容** | 当前沙箱页面的 HTML 源码 | 指定 URL 的响应内容 |
| **URL 来源** | 自动从 `$('url').rule()` 传入的 url 取 | 必须手动指定 |
| **典型场景** | `rule()` 回调内获取详情页 HTML | 所有主动发起请求的场景 |

```javascript
// getResCode() 用法：只在 $().rule() 内部使用
url: $(detailUrl).rule(function() {
    var html = getResCode();  // ✅ 获取 detailUrl 对应的页面源码
    var d = [];
    // ... 解析 html ...
    setResult(d);
});

// fetch() 用法：主动请求任意 URL
var html = fetch(url, { headers: { 'User-Agent': ua } });
```

> 📌 在 `$().rule()` 内部，`getResCode()` 和 `fetch(MY_URL)` 效果相同，但 `getResCode()` 更简洁。在 `$().lazyRule()` 内部只能用 `fetch()`，没有 `getResCode()`。

---

### 补充6：二级函数分步调试最佳实践

手册缺少完整的调试流程说明，以下是实战验证的推荐步骤：

**第一步：单独验证详情页 URL 可访问**
```javascript
// 在主页函数中临时添加日志
log('详情页 URL: ' + detailUrl);
log('请求结果长度: ' + html.length);
log('页面特征: ' + (html.indexOf('特征关键词') > -1 ? '✅' : '❌'));
```

**第二步：验证图片 URL 提取正确**
```javascript
// log 出前3张图片的 URL
var imgs = pdfa(html, '图片选择器&&img');
for (var i = 0; i < Math.min(3, imgs.length); i++) {
    log('图片' + i + ': ' + pdfh(imgs[i], 'img&&z-image-loader-url'));
}
```

**第三步：验证解密后的 URL 可播放/可显示**
```javascript
// 在 $.image() 回调内 log 解密结果的前100字节
log('解密结果类型: ' + (dec ? dec.getClass().getName() : 'null'));
```

**第四步：整合测试**
建议先做不含解密的基础版本，确认列表和跳转正常后，再加入图片解密逻辑。

---

### 补充7：站点 JS 源码分析方法（找 key/iv/padding）

手册缺少如何定位加密配置的具体说明：

**方法1：搜索全局配置对象**
```javascript
// 常见位置：页面 HTML 的 <script> 标签内，或加载的 JS 文件中
// 搜索关键词：
// - __APP_CONFIG__
// - window.config
// - crypto / encrypt / decrypt
// - CryptoJS.AES
// - key: / iv: / secret:

// 示例站点源码（可能藏在混淆的 JS bundle 中）：
var __APP_CONFIG__ = {
    crypto: {
        key: 'abcdef1234567890',
        iv:  '1234567890abcdef',
        mode: 'NoPadding'     // ← 关键！
    }
};
```

**方法2：浏览器 DevTools 断点调试**
1. 打开 DevTools → Sources → 搜索 `AES.decrypt` 或 `CryptoJS`
2. 在解密函数处打断点
3. 刷新页面，断点触发时查看 key/iv 的实际值

**方法3：正则搜索 JS 文件中的密钥**
```javascript
// 在规则调试时，先抓一个 JS 文件并 log 搜索结果
var jsContent = fetch('https://site.com/js/app.bundle.js');
var keyMatch  = jsContent.match(/key\s*[:=]\s*['"]([a-zA-Z0-9]{16,32})['"]/);
var ivMatch   = jsContent.match(/iv\s*[:=]\s*['"]([a-zA-Z0-9]{16,32})['"]/);
log('key: ' + (keyMatch ? keyMatch[1] : '未找到'));
log('iv:  ' + (ivMatch  ? ivMatch[1]  : '未找到'));
```

> 📌 密钥长度参考：AES-128 = 16字节，AES-192 = 24字节，AES-256 = 32字节。找到的 key 字符串长度是加密强度的直接判断依据。

---

## 49.3 表述澄清

### 澄清1：MY_PAGE 注入条件的完整描述

手册多处提到 `type:0` 不注入 `MY_PAGE`，但未给出正向的完整说明。补充如下：

**MY_PAGE 注入的充要条件**：
```javascript
静态分类: {
    type: '主页',   // ✅ 必须是字符串 '主页'，不能是数字 0 或 1
    url: 'fypage'   // ✅ url 必须含 fypage（可以是纯 'fypage' 或含占位符的完整 URL）
}
// 同时满足以上两个条件，MY_PAGE 才会被正确注入
```

**MY_PAGE 的类型**：注入后是数字（`number`），可以直接用于算术运算和比较，无需 `parseInt()`。

### 澄清2：lazyRule 参数传递的完整规则

手册说"参数只能传基础类型"，补充具体说明：

```javascript
// ✅ 可以传：string、number、boolean
url: $(link).lazyRule(function(str, num, bool) { }, 'abc', 123, true)

// ❌ 不能传：object、array、function
// 传对象时会序列化为 '[object Object]'，传数组变为 '1,2,3' 字符串
var obj = { key: 'val' };
url: $(link).lazyRule(function(o) {
    log(typeof o);   // 'string'
    log(o);          // '[object Object]'
}, obj)              // ❌ obj 变成字符串

// ✅ 对象拆解成多个参数传递
url: $(link).lazyRule(function(k, v) { }, obj.key, obj.val)

// ✅ 或者 JSON.stringify 后传，内部 JSON.parse
url: $(link).lazyRule(function(s) {
    var obj = JSON.parse(s);
}, JSON.stringify(obj))
```

### 澄清3：parse 对象在 lazyRule/rule 中的可访问性

```javascript
// ❌ lazyRule 内部无法访问外层 parse 变量
url: $(link).lazyRule(function() {
    var host = parse.host;  // ❌ ReferenceError: parse is not defined
})

// ✅ 必须通过参数传入
var _host = this.host;
url: $(link).lazyRule(function(host) {
    // host 正常可用
}, _host)

// ⚠️ 特殊情况：$.toString() 字符串化的代码可以访问 parse，
// 因为 eval 执行时处于包含 parse 声明的同一作用域内
// 但这是 eval 特性，不是 lazyRule 特性，不可混淆
```

---

## 49.4 本章新增铁律汇总

| 编号 | 铁律名称 | 核心要义 | 关键口诀 |
|---|---|---|---|
| Law 92 | $.image() input 直接可用 | 回调中的 `input` 是引擎已下载的 `InputStream`，禁止对 `input` 调用 `fetch()` | `input` 是流不是 URL |
| Law 93 | padding 从源码确认 | 不能默认 `PKCS7Padding`，必须从站点 JS 中确认，写错无明显报错 | padding 写错 = 解密失败但无报错 |
| Law 94 | extenditems 配套 list | 使用 `extenditems` 时 `list` 必须给 `[['']]`，`noShow` 按需隐藏组件 | `extenditems + list:[['']] + noShow` 三件套 |
| Law 95 | 图文穿插按节点顺序 | 详情页图文混排须遍历所有子节点按序处理，不能分别提取再拼接 | `pdfa('内容&&*')` 遍历，循环内分类 |
| Law 96 | eval 字符串复用谨慎 | `eval` 字符串模式可用于复用解密代码，但优先用参数传递，`eval` 是最后手段 | 优先传参，eval 兜底 |

---

*手册勘误版本 V3.47 · 2026年4月 · 第四十九章：基于私房KTV规则开发实战的修正与补充*

---

# 第五十章：图集类站点开发总结——JunMeiTu 实战经验

本章基于 JunMeiTu（写真图集站）完整规则开发过程提炼，记录图集类站点的核心模式与手册通用规范的差异点。

---

## 50.1 图集类站点的核心架构

图集站与视频站的最大区别：内容的终点是 `pics://` 协议，而不是 `video://`。这个差异导致整个规则结构都有所不同：

| 维度 | 视频站 | 图集站 |
|---|---|---|
| 列表卡片 url | 详情页地址 → 进二级 | 直接 `$(url).lazyRule` 返回 `pics://` |
| 二级函数 | 必须，返回 `line/list` | 可选，仅资讯/街拍类内容需要 |
| 解析函数 | 必须，返回 `video://` | 不需要 |
| 翻页数据 | 引擎通过 `fypage` 处理 | 图集内部翻页需手动 `batchFetch` |

---

## 50.2 图集列表直接返回 pics://（不进二级）

纯图集类内容（写真集），点击卡片后直接看图，无需二级函数。列表项的 `url` 字段直接用 `lazyRule` 返回 `pics://`：

```javascript
// ✅ 图集列表标准写法：列表项直接 lazyRule 返回 pics://
var items = pdfa(html, 'body&&.gallery-item');
for (var i = 0; i < items.length; i++) {
    var it     = items[i];
    var link   = pd(it, 'a&&href', host);
    var title  = pdfh(it, '.title&&Text');
    var cover  = pd(it, 'img&&src', host);

    d.push({
        title:    title,
        pic_url:  cover + '@Referer=' + host + '/',
        col_type: 'movie_3',
        // ✅ 直接 lazyRule，不用二级函数
        url: $(link).lazyRule(function(h, ua) {
            // 获取图集详情页，批量抓取所有分页图片
            var html = fetch(input, { headers: { 'User-Agent': ua, 'Referer': h + '/' } });
            if (!html || html.length < 100) return 'toast://加载失败';

            var pics = [];
            // 提取图片 URL（见 50.3）
            // ...
            if (pics.length === 0) return 'toast://未找到图片';
            return 'pics://' + pics.join('&&');
        }, host, ua)
    });
}
```

---

## 50.3 pdfh 选择器不可靠时用正则——稳健图片提取

**场景**：部分站点图片容器的 HTML 结构较复杂，或 JSoup 对某些 class 名处理异常，`pdfh(html, '.pictures&&img&&src')` 会报错或返回空。

**原则**：`pdfh`/`pdfa` 优先，报错或空时降级到正则。两种方式都需要 `try...catch` 包裹。

```javascript
// ✅ 方式A：pdfa 选择器（优先）
var pics = [];
try {
    var imgNodes = pdfa(html, '.pictures&&img');
    for (var i = 0; i < imgNodes.length; i++) {
        var src = '';
        try { src = pdfh(imgNodes[i], 'img&&src'); } catch(e) {}
        if (!src) try { src = pdfh(imgNodes[i], 'img&&data-src'); } catch(e) {}
        if (src && src.indexOf('http') === 0) pics.push(src);
    }
} catch(e) { log('pdfa 图片提取失败，降级正则: ' + e); }

// ✅ 方式B：正则降级（pdfa 失败或返回空时）
if (pics.length === 0) {
    var re = /<img[^>]+src="(https?:\/\/[^"]+\.(jpg|jpeg|png|webp)[^"]*)"[^>]*>/gi;
    var m;
    while ((m = re.exec(html)) !== null) {
        pics.push(m[1]);
    }
}
```

> 📌 正则提取图片的通用模式：`/<img[^>]+src="(https?:\/\/[^"]+\.(jpg|jpeg|png|webp)[^"]*)"[^>]*>/gi`，用 `while + exec` 遍历所有匹配。注意正则对象不能复用（每次循环前重新赋值），否则 `lastIndex` 导致漏匹配。

> **⚠️ Law 97（新增）：`pdfh`/`pdfa` 是优先选择，但在图片提取场景下选择器可能因站点 HTML 结构异常而失败，必须有正则兜底。选择器和正则都要包在 `try...catch` 里。**

---

## 50.4 图集内部分页——batchFetch 批量抓取

图集站通常每页只显示部分图片，需要抓取所有分页才能得到完整图集。翻页 URL 规律通常是 `index-2.html`、`index-3.html` 这类非标准格式。

```javascript
// 标准图集分页批量抓取模式
url: $(link).lazyRule(function(h, ua) {
    var baseUrl = input;  // 如 https://site.com/set/001/index.html
    var html1 = fetch(baseUrl, { headers: { 'User-Agent': ua, 'Referer': h + '/' } });
    if (!html1 || html1.length < 100) return 'toast://加载失败';

    var pics = [];

    // Step1：提取第一页图片
    var addPics = function(html) {
        var re = /<img[^>]+src="(https?:\/\/[^"]+\.(jpg|jpeg|png|webp)[^"]*)"[^>]*>/gi;
        var m;
        while ((m = re.exec(html)) !== null) {
            var src = m[1];
            // 过滤广告图/Logo（根据实际站点路径特征）
            if (src.indexOf('/thumb/') > -1 || src.indexOf('/ad/') > -1) continue;
            pics.push(src + '@Referer=' + h + '/');
        }
    };
    addPics(html1);

    // Step2：从第一页提取总页数
    var totalPages = 1;
    var pageM = html1.match(/共\s*(\d+)\s*页/) || html1.match(/\/(\d+)\s*页/);
    if (pageM) totalPages = parseInt(pageM[1]);
    // 也可以从分页链接推断：找最大页码
    var maxPageM = html1.match(/index-(\d+)\.html/g);
    if (maxPageM) {
        var maxPage = 1;
        for (var mi = 0; mi < maxPageM.length; mi++) {
            var n = parseInt(maxPageM[mi].match(/\d+/)[0]);
            if (n > maxPage) maxPage = n;
        }
        totalPages = Math.max(totalPages, maxPage);
    }

    // Step3：batchFetch 并发抓取剩余页（最多16个/批，Law 3.3）
    if (totalPages > 1) {
        var tasks = [];
        // 翻页 URL 规律：index.html → index-2.html → index-3.html
        for (var p = 2; p <= Math.min(totalPages, 20); p++) {
            var pageUrl = baseUrl.replace(/index(\.html)?$/, 'index-' + p + '.html');
            tasks.push({ url: pageUrl, options: { headers: { 'User-Agent': ua, 'Referer': h + '/' } } });
        }
        var results = batchFetch(tasks);
        for (var ri = 0; ri < results.length; ri++) {
            if (results[ri] && results[ri].length > 100) addPics(results[ri]);
        }
    }

    if (pics.length === 0) return 'toast://未找到图片';
    return 'pics://' + pics.join('&&');
}, host, ua)
```

> 📌 `batchFetch` 最多16个并发（Law 3.3），图集页数超过16时自动分批串行，不需要额外处理。但抓取超过20页的图集时，考虑只抓前N页以控制等待时间。

---

## 50.5 资讯/街拍类内容——二级函数 + extenditems

资讯文章和街拍帖子是图文混排内容，需要用二级函数返回 `extenditems`，按 HTML 顺序渲染（Law 95）：

```javascript
二级: function(url) {
    var host = this.host;
    var ua   = this.UA;
    var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });
    if (!html || html.length < 100) {
        return { list: [['']], noShow: { 封面: true, 简介: true, 排序: true, 选集: true },
                 extenditems: [{ title: '加载失败', col_type: 'text_1', url: 'hiker://empty' }] };
    }

    var extra = [];

    // 标题
    var title = '';
    try { title = pdfh(html, 'h1&&Text') || pdfh(html, '.article-title&&Text'); } catch(e) {}
    if (title) extra.push({ title: title, col_type: 'text_1', url: url, extra: { lineVisible: false } });

    extra.push({ col_type: 'line' });

    // 按顺序遍历正文节点（Law 95）
    var nodes = [];
    try { nodes = pdfa(html, 'body&&.article-content&&*'); } catch(e) {}
    // 备用选择器
    if (nodes.length === 0) try { nodes = pdfa(html, 'body&&.content&&*'); } catch(e) {}

    for (var i = 0; i < nodes.length; i++) {
        var node = nodes[i];

        // 图片
        var imgSrc = '';
        try { imgSrc = pdfh(node, 'img&&src') || pdfh(node, 'img&&data-src'); } catch(e) {}
        if (imgSrc && imgSrc.indexOf('http') === 0) {
            extra.push({
                title:    '',
                pic_url:  imgSrc + '@Referer=' + host + '/',
                url:      'pics://' + imgSrc + '@Referer=' + host + '/',
                col_type: 'pic_1_full'
            });
            continue;
        }

        // 文字
        var txt = '';
        try { txt = pdfh(node, 'Text'); } catch(e) {}
        if (txt && txt.trim().length > 3) {
            extra.push({ title: txt.trim(), col_type: 'rich_text' });
        }
    }

    return {
        vod_name:    title,
        list:        [['']], // ✅ Law 94：必须给空二维数组
        noShow:      { 封面: true, 简介: true, 排序: true, 选集: true },
        extenditems: extra
    };
},
```

---

## 50.6 标签云 / 主题分类——独立新开页面

当标签云或主题分类需要独立的渲染逻辑（如彩色 `flex_button` + 翻页列表），适合用 `$().rule()` 新开沙箱页面，而不是依赖静态分类：

```javascript
// 标签云入口（在主页列表中）
d.push({
    title:    '🏷️ 所有标签',
    col_type: 'text_center_1',
    url: $('hiker://empty##fypage').rule(function(h, ua) {
        var d = [];
        var url = h + '/tags/' + (MY_PAGE > 1 ? 'page/' + MY_PAGE + '/' : '');
        var html = fetch(url, { headers: { 'User-Agent': ua } });
        if (!html || html.length < 100) { setResult(d); return; }

        var tags = pdfa(html, 'body&&.tag-item');
        for (var i = 0; i < tags.length; i++) {
            var tagLink  = pd(tags[i], 'a&&href', h);
            var tagName  = pdfh(tags[i], 'a&&Text');
            var tagCount = pdfh(tags[i], '.count&&Text') || '';
            d.push({
                title:    tagName + (tagCount ? ' (' + tagCount + ')' : ''),
                col_type: 'flex_button',
                // 点击标签 → 新开该标签的图集列表页（带翻页）
                url: $(tagLink + '##fypage').rule(function(th, tua) {
                    var d = [];
                    var tUrl = MY_URL.split('##')[0];
                    if (MY_PAGE > 1) tUrl = tUrl.replace(/\/?$/, '') + '/page/' + MY_PAGE + '/';
                    var tHtml = fetch(tUrl, { headers: { 'User-Agent': tua } });
                    // ... 解析图集列表 ...
                    setResult(d);
                }, h, ua),
                extra: { backgroundColor: getRangeColors() }
            });
        }
        setResult(d);
    }, host, ua)
});
```

> ⚠️ **标签云走 `rule()` 沙箱的限制（Law 67 延伸）**：沙箱内的卡片如果需要进二级（有 `二级` 函数的视频站），URL 会跳浏览器。图集站的卡片直接返回 `pics://` 或 `video://` 协议，不受此限制，所以图集站标签云适合用 `rule()` 方案。

---

## 50.7 空值防御——fetch 后的标准检查

图集站的图片和页面请求失败时，不抛出异常而是返回空字符串或极短内容，必须在每次 `fetch` 后做检查：

```javascript
// ✅ 标准空值检查模板
var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });

// 两个条件都要检查：
// 1. !html：fetch 返回 null 或 undefined
// 2. html.length < 100：返回了空页面或极短的错误页
if (!html || html.length < 100) {
    log('页面加载失败: ' + url);
    // 列表场景：返回空数组
    return d;
    // lazyRule 场景：返回 toast 提示
    // return 'toast://加载失败，请重试';
}

// ✅ 额外检查：验证页面含预期特征（比长度检查更准确）
if (html.indexOf('预期的CSS类名或关键词') === -1) {
    log('页面内容异常，可能被 CF 拦截或域名变更');
    return d;
}
```

> **⚠️ Law 98（新增）：每次 `fetch` 后必须做双重检查：`!html || html.length < 100`。对于关键页面（如详情页），进一步用页面特征关键词验证内容有效性，不能仅依赖长度。**

---

## 50.8 防盗链——图片 URL 后缀的完整写法

图集站几乎100%有防盗链，漏加 `Referer` 会导致图片显示为403或空白：

```javascript
// ✅ 三种防盗链写法（任选一种）

// 写法1：简短格式（推荐，适合单一 Referer）
var picUrl = imgSrc + '@Referer=' + host + '/';

// 写法2：完整 headers 格式（需要多个请求头时）
var picUrl = imgSrc + '@headers={"Referer":"' + host + '/","User-Agent":"' + ua + '"}';

// 写法3：在 $.image() 中用 FileUtil 带 headers 下载（需要解密时）
// 见第十章 / 第四十一章

// ⚠️ 注意：Referer 填图片所在页面的域名，不是图片 CDN 的域名
// 错误：@Referer=https://pic.cdn.example.com/
// 正确：@Referer=https://www.example.com/
```

---

## 50.9 图集站完整规则骨架

```javascript
var parse = {
    作者: 'dev',
    版本: '20260408.V1',
    host: 'https://www.example.com',
    UA:   'Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36',

    页码: { '主页': 1, '分类': 1, '搜索': 1 },

    静态分类: {
        type: '主页',
        url: 'fyclass',
        class_name: '推荐&写真&街拍&资讯',
        class_url:  '/&/photos/&/street/&/news/'
    },

    主页: function() {
        var d    = [];
        var host = this.host;
        var ua   = this.UA;
        var cls  = (MY_URL && MY_URL !== 'fyclass') ? MY_URL : '/';
        var url  = host + cls;
        if (MY_PAGE > 1) url = url.replace(/\/?$/, '') + '/page/' + MY_PAGE + '/';

        var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });
        if (!html || html.length < 100) return d;

        // 按路径分流（Law 38 精神：if/else if 路由）
        if (cls.indexOf('/news/') > -1 || cls.indexOf('/street/') > -1) {
            // 资讯/街拍：进二级
            return this._parseNewsListToSecond(html, host, ua);
        } else {
            // 写真/推荐：直接 lazyRule 返回 pics://
            return this._parsePhotoList(html, host, ua);
        }
    },

    // 写真列表解析（卡片直接返回 pics://）
    _parsePhotoList: function(html, host, ua) {
        var d     = [];
        var items = [];
        try { items = pdfa(html, 'body&&.photo-item'); } catch(e) {}
        for (var i = 0; i < items.length; i++) {
            var it    = items[i];
            var link  = pd(it, 'a&&href', host);
            var title = pdfh(it, '.title&&Text') || pdfh(it, 'img&&alt') || '';
            var cover = pd(it, 'img&&src', host) || pd(it, 'img&&data-src', host);
            if (!link) continue;
            d.push({
                title:    title,
                pic_url:  cover ? cover + '@Referer=' + host + '/' : '',
                col_type: 'movie_3',
                url: $(link).lazyRule(function(h, ua) {
                    var html = fetch(input, { headers: { 'User-Agent': ua, 'Referer': h + '/' } });
                    if (!html || html.length < 100) return 'toast://加载失败';
                    var pics = [];
                    // 选择器方式
                    try {
                        var imgs = pdfa(html, '.pictures&&img');
                        for (var i = 0; i < imgs.length; i++) {
                            var src = '';
                            try { src = pdfh(imgs[i], 'img&&src'); } catch(e) {}
                            if (!src) try { src = pdfh(imgs[i], 'img&&data-src'); } catch(e) {}
                            if (src && src.indexOf('http') === 0) pics.push(src + '@Referer=' + h + '/');
                        }
                    } catch(e) {}
                    // 正则降级（Law 97）
                    if (pics.length === 0) {
                        var re = /<img[^>]+src="(https?:\/\/[^"]+\.(jpg|jpeg|png|webp)[^"]*)"[^>]*>/gi;
                        var m;
                        while ((m = re.exec(html)) !== null) pics.push(m[1] + '@Referer=' + h + '/');
                    }
                    if (pics.length === 0) return 'toast://未找到图片';
                    return 'pics://' + pics.join('&&');
                }, host, ua)
            });
        }
        return d;
    },

    // 资讯/街拍列表解析（卡片进二级）
    _parseNewsListToSecond: function(html, host, ua) {
        var d = [];
        var items = [];
        try { items = pdfa(html, 'body&&.news-item'); } catch(e) {}
        for (var i = 0; i < items.length; i++) {
            var it    = items[i];
            var link  = pd(it, 'a&&href', host);
            var title = pdfh(it, '.title&&Text') || pdfh(it, 'h3&&Text') || '';
            var cover = pd(it, 'img&&src', host);
            if (!link) continue;
            d.push({
                title:    title,
                pic_url:  cover ? cover + '@Referer=' + host + '/' : '',
                col_type: 'movie_3',
                url:      link   // 直接给链接，引擎走二级函数
            });
        }
        return d;
    },

    // 二级（资讯/街拍详情）
    二级: function(url) {
        var host = this.host;
        var ua   = this.UA;
        var html = fetch(url, { headers: { 'User-Agent': ua, 'Referer': host + '/' } });
        if (!html || html.length < 100) {
            return { list: [['']], noShow: { 封面: true, 简介: true, 排序: true, 选集: true },
                     extenditems: [{ title: '加载失败', col_type: 'text_1', url: 'hiker://empty' }] };
        }
        var extra = [];
        var title = '';
        try { title = pdfh(html, 'h1&&Text'); } catch(e) {}
        if (title) extra.push({ title: title, col_type: 'text_1', url: url, extra: { lineVisible: false } });
        extra.push({ col_type: 'line' });

        // 按 HTML 顺序遍历正文（Law 95）
        var nodes = [];
        try { nodes = pdfa(html, 'body&&.content&&*'); } catch(e) {}
        for (var i = 0; i < nodes.length; i++) {
            var imgSrc = '';
            try { imgSrc = pdfh(nodes[i], 'img&&src') || pdfh(nodes[i], 'img&&data-src'); } catch(e) {}
            if (imgSrc && imgSrc.indexOf('http') === 0) {
                extra.push({ title: '', pic_url: imgSrc + '@Referer=' + host + '/',
                             url: 'pics://' + imgSrc, col_type: 'pic_1_full' });
                continue;
            }
            var txt = '';
            try { txt = pdfh(nodes[i], 'Text'); } catch(e) {}
            if (txt && txt.trim().length > 3) extra.push({ title: txt.trim(), col_type: 'rich_text' });
        }
        return { vod_name: title, list: [['']], // ✅ Law 94
                 noShow: { 封面: true, 简介: true, 排序: true, 选集: true }, extenditems: extra };
    },

    搜索: function(kw) {
        var host = this.host;
        var ua   = this.UA;
        var url  = host + '/search/?q=' + encodeURIComponent(kw);
        if (MY_PAGE > 1) url += '&page=' + MY_PAGE;
        var html = fetch(url, { headers: { 'User-Agent': ua } });
        if (!html || html.length < 100) return [];
        return this._parsePhotoList(html, host, ua);
    },

    _base64Decode: function(str) {
        if (!str) return '';
        str = str.replace(/-/g, '+').replace(/_/g, '/');
        while (str.length % 4) str += '=';
        try {
            var b = android.util.Base64.decode(str, android.util.Base64.DEFAULT);
            return new java.lang.String(b, 'UTF-8') + '';
        } catch(e) { return ''; }
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

## 50.10 本章新增铁律

| 编号 | 铁律名称 | 核心要义 | 关键口诀 |
|---|---|---|---|
| Law 97 | 图片选择器降级正则 | `pdfa`/`pdfh` 提取图片失败时，用正则 `/<img[^>]+src="(https?:\/\/[^"]+)"[^>]*>/gi` 兜底 | 选择器失败 → 正则接手 |
| Law 98 | fetch 双重空值检查 | 每次 `fetch` 后检查 `!html \|\| html.length < 100`，关键页面再验证内容特征 | 空串 + 短串 + 特征，三重保险 |

---

## 50.11 图集站 vs 视频站 vs 资讯站——三种模式对比

| 维度 | 图集站（写真） | 视频站 | 资讯站 |
|---|---|---|---|
| 列表卡片 url | `$(link).lazyRule` → `pics://` | `link`（进二级）或 `$(link).lazyRule` → `video://` | `link`（进二级）|
| 二级函数 | 不需要 | 必须（返回 `line/list`）| 必须（返回 `extenditems`）|
| 解析函数 | 不需要 | 按需（m3u8 解密等）| 不需要 |
| 内容终点 | `pics://` 协议 | `video://` 协议 | `extenditems` 富文本 |
| 翻页策略 | 图集内部 `batchFetch` 分页 | 引擎 `fypage` + 请求列表 | 引擎 `fypage` + 请求列表 |
| 核心 Law | Law 97/98 + `pics://` | Law 3/4/16/39 | Law 94/95 |

---

*手册补充版本 V3.47 · 第五十章：图集类站点开发总结（JunMeiTu 实战）*
