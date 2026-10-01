---
name: video-teardown
description: 视频拉片 / 镜头语言拆解 — 拿到一条片子，自动切分镜头、抽取代表帧、量化剪辑节奏与声音结构，产出逐镜拆解表 + 方法论提炼 + 可复用清单的 HTML 拉片报告。当用户说"拉一下这条片""拆镜头语言""分镜拆解""逐镜分析""这条片子怎么拍的""分析运镜/布光/节奏"，或给出一个视频链接/本地视频说"帮我拆"时使用。也用于竞品内容拆解、爆款方法论逆向。反向于 video-shotcraft（那个是做片，这个是拆片）。
author: "@夏"
agent_created: true
---

# video-teardown：视频拉片（拆片）

> **出处：@夏**
> 原作出自 **@夏**。本版本由 `yoyojay` 整理发布，并附本地实战增补（微信视频号接口现状、`yt-dlp` 403 解法、无法读图时的程序化替代管线）。原方法论版权归 @夏 所有。

把一条成片拆成可复用的方法论。核心不是"描述画面"，而是回答三个问题：
**它用什么手法？为什么有效？我怎么抄？**

## 与相邻技能的边界

| 技能 | 方向 | 什么时候用它 |
|---|---|---|
| `video-downloader` | 拿片子 | 只要有链接还没素材，先走它（见 Step 0） |
| `video-shotcraft` | 做片子 | 用户要"做一条宣传片"，不是你 |
| **本技能** | **拆片子** | 用户要分析已有片子 |

---

## Step 0 · 拿素材（最耗时的一步，先走捷径）

**优先级顺序，不要跳步：**

1. **先试 `video-downloader` 技能**——粘贴页面分享链接即可，覆盖抖音/小红书/快手/视频号/B站/YouTube 等，返回无水印直链。这是最省事的路。
2. **再试 `yt-dlp`**（`~/.local/bin/yt-dlp`）。
   - **报 `HTTP Error 403: Forbidden`（视频数据下载失败）时 = 版本太旧**。老版本（如 2026.03.17）会选 `android_vr` 客户端拿直链被 YouTube 拒。
   - **解法：升级**。若 `yt-dlp -U` 提示 "installed with pip"，用 `python3 -m pip install -U yt-dlp --user` 升（实测 2026.03.17 → 2026.08.19 后立即恢复，新版本会自动做 JS challenge）。
   - 别先折腾 `--extractor-args player_client=web`——换客户端常报 "Requested format is not available"（itag 不同），升级才是根因。
3. **再试平台 API 手搓**（见下方平台坑表）。
4. **最后才考虑抓包/代理**（见"微信视频号"一节，代价极高，先跟用户确认）。

已经有本地文件就直接 `ffprobe` 进 Step 1。

### 平台坑表（实战踩过）

**B站**
- `yt-dlp` 请求网页**必被 412 反爬**（带/不带 cookie、带/不带浏览器 UA 都一样）。不要在这上面耗时间。
- **绕法 = 手动调 API**（curl 全程正常）：
  ```bash
  UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
  # 1) 搜作品（拿到 bvid）
  curl -s -A "$UA" -H "Referer: https://www.bilibili.com" \
    "https://api.bilibili.com/x/web-interface/wbi/search/type?search_type=video&keyword=<关键词>"
  # 2) 取 cid
  curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
    "https://api.bilibili.com/x/web-interface/view?bvid=<BV>"
  # 3) 取 DASH 流（匿名最高 480p、720p 视视频而定）
  curl -s -A "$UA" -H "Referer: https://www.bilibili.com/video/<BV>/" \
    "https://api.bilibili.com/x/player/playurl?bvid=<BV>&cid=<CID>&qn=80&fnval=4048&fourk=1"
  ```
- 下载 DASH 分片时 **必须带 `Referer: https://www.bilibili.com/...`**，否则 403。
- 合成：`ffmpeg -i v.m4s -i a.m4s -c copy -movflags +faststart out.mp4`
- 想要 1080p 需要登录态 SESSDATA；从 Chrome 导 cookie 可以，但**带 cookie 反而触发 412**，别混用。

**微信视频号（`weixin.qq.com/sph/XXXX`）**
- 分享页是 SPA（`channels.weixin.qq.com/finder-preview/pages/sph?id=<短码>`），**页面本身零视频数据**，纯前端壳，<title> 只有"视频号"，无任何标题/作者/封面内嵌，也无法从这里拿到 shortUri 之外的信息。
- 唯一取数接口 `POST https://channels.weixin.qq.com/finder-preview/api/feed/get_feed_info`，body `{"baseReq":{"generalToken":""},"shortUri":"<短码>"}`。
- **实测（2026-10，已验证桌面 UA 与伪装 MicroMessenger UA 两种）：网页端一律 `401 {"errCode":-1,"errMsg":"permission verification failed"}`——连封面/文案/计数都不返回了**（旧经验"非微信环境返回封面+文案"已失效，勿再依赖）。`exportId` 模式报"此内容暂时无法播放"；网页播放端报 `{"desc":"非法请求","errCode":10012}`。`yt-dlp` 实测 `Unsupported URL`，无提取器；公网搜短码也搜不到标题/作者。
- **结论：网页端已拿不到任何素材（含封面），不要在该接口上耗时间。** 要拆只有两条路：① 找**同源跨平台镜像**（同一创作者常同步发 B站/抖音——但需先知道标题/作者才能搜，SPA 不提供，得用户给或自行识别）；② 微信内播放 + MITM 代理捕获（`ltaoo/wx_channels_download`，需装根证书+改系统代理，且会被其他代理软件抢系统代理，代价高，**先问用户**）。

---

## Step 1 · 读参数

```bash
ffprobe -v error -show_entries format=duration,size,bit_rate \
  -show_entries stream=codec_type,codec_name,width,height,r_frame_rate,nb_frames \
  -of default=noprint_wrappers=1 "$V"
```

## Step 2 · 场景检测切分镜

```bash
ffmpeg -hide_banner -i "$V" -filter:v "select='gt(scene,0.22)',showinfo" -an -f null - 2>&1 \
  | grep -oE "pts_time:[0-9.]+" | sed 's/pts_time://' > cuts.txt
```

**必须做去闪频**：同镜头内的闪光/频闪会产生 0.1s 内的密集检测点。合并规则：

```js
const merged=[]; for(const t of cuts){ if(merged.length && t-merged[merged.length-1]<0.7) continue; merged.push(t); }
// 得到镜头边界 [0, ...merged, DUR]
```

阈值调参：`0.22` 适合快剪剧情/广告片；纪录片/长镜头用 `0.30–0.35`；MV/高动态可降到 `0.15` 但要更激进去闪。**输出前打印镜头数与时长分布，人工看一眼是否合理。**

## Step 3 · 抽代表帧

每镜取 `start + dur*0.45` 处（避开转场首尾）：

```bash
i=0; while read -r T; do i=$((i+1)); N=$(printf "%02d" $i)
  ffmpeg -hide_banner -loglevel error -ss "$T" -i "$V" -frames:v 1 -vf "scale=640:360" -y "fr/$N.png"
done < ts.txt
```

## Step 4 · 拼接触印相表

```bash
# tile 只吃"单条流里的连续帧"，所以用 glob 输入，不要用多个 -i
ffmpeg -hide_banner -loglevel error -framerate 1 -pattern_type glob -i 'fr/*.png' \
  -vf "tile=2x3:margin=8:padding=8:color=white" -fps_mode passthrough -y "sheets/sheet_%02d.png"
```

每张 2×3=6 帧；先用 `-q:v 4` 转 JPEG 再内联进 HTML（35 帧约 850KB，可接受）。

## Step 5 · 声音结构（很多人漏掉，但它是节奏的一半）

```bash
# 旁白气口
ffmpeg -hide_banner -i "$V" -af "silencedetect=noise=-32dB:d=0.35" -f null - 2>&1 \
  | grep -oE "silence_(start|end): [0-9.]+" | paste - -
# 整体响度
ffmpeg -hide_banner -i "$V" -af loudnorm=print_format=summary -f null - 2>&1 \
  | grep -E "Input (Integrated|LRA|True Peak)"
```

读法：气口密度 → 旁白是"覆盖式"还是"留白式"；LRA 小（<6）说明动态被压平，典型短视频响度处理；真峰值 0.0 dBTP 说明上了限幅器。

## Step 6 · 读图分析（本技能的核心，必须真的看图）

用 Read 工具逐张读接触表，**不要靠猜**。每镜记录：

- **景别/机位**：大远景 / 全景 / 中景 / 近景 / 特写 / 大特写；平视 / 仰 / 俯
- **构图**：中心 / 三分 / 对角 / 留白比例
- **镜头的"贵贱感"来源**：焦段压缩、浅景深、逆光轮廓、材质特写
- **色彩纪律**：主体是否独占色相、环境是否降饱和
- **花字/字幕**：字体、压色块、位置、文案句式
- **这一镜在结构里的功能**：定场 / 立规则 / 制造反差 / 空镜定调 / 收口

### 6.1 看图失效时的替代管线（已验证可用）

**有些会话里 Read 读图会返回「current model does not support images」**——此时绝不能靠猜画面编造分析。改用这条程序化管线，产出同样是「从画面里读出来的真实文字 + 客观特征」，且在报告里注明方法：

```bash
# ① 旁白：whisper.cpp 本地转录（无 API 成本）
ffmpeg -y -i "$V" -ac 1 -ar 16000 -c:a pcm_s16le audio16k.wav
whisper-cli -m ~/.cache/whisper/ggml-small.bin -f audio16k.wav -l zh -otxt -osrt -of transcript -t 8
# 模型缺失时可用：find ~ -maxdepth 6 -iname "ggml*.bin" 找已有模型
# 得到 transcript.srt，按时码与镜头区间做重叠匹配 → 每镜【旁白】

# ② 花字/界面文字：tesseract OCR（中文视频必装 chi_sim）
mkdir -p tessdata && curl -sL -o tessdata/chi_sim.traineddata \
  "https://github.com/tesseract-ocr/tessdata_fast/raw/main/chi_sim.traineddata"
cp /opt/homebrew/share/tessdata/eng.traineddata tessdata/   # 中英混读
TESSDATA_PREFIX=$PWD/tessdata tesseract frame.png stdout -l chi_sim+eng --psm 6
# 关键：OCR 前把帧放大到 ~1000px 宽再转灰度，小字识别率明显提升
```

**③ 客观特征替掉肉眼判断**（Pillow + numpy，逐帧算）：
`亮度 / 饱和度 / 肤色占比 / 边缘密度 / 色彩丰富度` → 自动归类镜头：
- 肤色占比 > 0.10 → **口播人物**；若同时边缘密度高 → **录屏+画中画**
- 文字量大或边缘密度高 → **录屏/界面**
- 色彩丰富度高且文字少 → **标题卡**
- 其余 → **B-roll/空镜**

归类结果直接填进逐镜表的「景别/机位」列，并在报告里标注「（推断）」以区分观察与推断。这条管线对长片（>10 分钟、百镜以上）尤其划算：肉眼逐张读上百帧不可行，程序化反而更全。

## Step 7 · 出报告（HTML，严格对齐交付参考版式）

产出**一份**自包含 HTML（图片 base64 内联，零外部依赖，零 emoji）。下面是已验证的版式规范——**直接照抄这份骨架，把数据填进去即可，不要自创排版**。以《拉片报告 · 人字拖》为交付参考标准。

### 7.0 样式令牌（CSS，原样用，决定整份报告的观感）

```css
:root{--red:#b3382c;--ink:#1c1c1c;--gray:#6b6b6b;--line:#e4e0da;--bg:#fdfcfb;}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:"PingFang SC","Helvetica Neue",sans-serif;background:var(--bg);color:var(--ink);line-height:1.75;font-size:15px}
.wrap{max-width:900px;margin:0 auto;padding:56px 24px 96px}
h1,h2,h3,.serif{font-family:"Songti SC","STSong","SimSun",serif}
h1{font-size:30px;font-weight:700;letter-spacing:.02em}
h1 .thin{font-weight:400;color:var(--gray);font-size:19px;display:block;margin-top:8px}
.rule{width:44px;height:3px;background:var(--red);margin:22px 0 34px}
h2{font-size:21px;margin:56px 0 6px;padding-left:14px;border-left:4px solid var(--red)}
h2 .en{font-family:"Courier New",monospace;font-size:12px;color:var(--gray);font-weight:400;margin-left:10px;letter-spacing:.08em}
h3{font-size:16px;margin:34px 0 12px}
h3 .sub{font-size:12.5px;color:var(--gray);font-family:"PingFang SC";font-weight:400;margin-left:10px}
p{margin:10px 0}
.meta{display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:1px;background:var(--line);border:1px solid var(--line);margin:26px 0}
.meta div{background:#fff;padding:12px 14px}
.meta b{display:block;font-size:11px;color:var(--gray);letter-spacing:.12em;font-weight:400;margin-bottom:3px}
.meta span{font-family:"Courier New",monospace;font-size:14px}
table{width:100%;border-collapse:collapse;margin:18px 0;font-size:13px}
th{font-weight:400;color:var(--gray);text-align:left;padding:8px 10px;border-bottom:2px solid var(--ink);font-size:12px;letter-spacing:.06em}
td{padding:9px 10px;border-bottom:1px solid var(--line);vertical-align:top}
td.c{text-align:center}
td.mono{font-family:"Courier New",monospace;white-space:nowrap;font-size:12px}
td.vo{color:var(--red)}
tr:hover td{background:#faf7f4}
.sheet{width:100%;border:1px solid var(--line);margin:14px 0 6px;display:block}
.cap{font-size:12px;color:var(--gray);margin-bottom:8px}
.formula{counter-reset:f;margin:20px 0}
.formula li{list-style:none;position:relative;padding:14px 16px 14px 58px;border:1px solid var(--line);background:#fff;margin-bottom:10px}
.formula li:before{counter-increment:f;content:counter(f,decimal-leading-zero);position:absolute;left:16px;top:14px;font-family:"Courier New",monospace;color:var(--red);font-size:15px}
.formula b{font-family:"Songti SC",serif;font-size:15.5px}
.formula span{display:block;color:var(--gray);font-size:13px;margin-top:2px}
.keyrow{display:grid;grid-template-columns:1fr 1fr 1fr;gap:10px;margin:18px 0}
.keyrow img{width:100%;border:1px solid var(--line);display:block}
.keyrow p{font-size:12px;color:var(--gray);margin-top:4px}
.bars{margin:20px 0}
.note{background:#faf6f2;border-left:3px solid var(--red);padding:12px 16px;font-size:13.5px;margin:16px 0}
.foot{margin-top:64px;padding-top:18px;border-top:1px solid var(--line);font-size:12px;color:var(--gray)}
@media(max-width:720px){.keyrow{grid-template-columns:1fr}td,th{padding:7px 6px;font-size:12px}}
```

### 7.1 报告分节顺序（14 块，顺序别动）

1. **标题区**：`<h1>主标题<span class="thin">拉片报告 · 片名 · 时长 X:XX.X</span></h1>` + `<div class="rule">`
2. **影片档案（.meta 六格卡片）**：固定字段 → 片长 / 镜头数 / 平均镜头 / 规格 / 剪辑点检出（原检 N → 去闪 M）/ 响度。值用等宽字体。
3. **开场论点段（2 段，必须有）**：第 1 句点明这条片「形式上是什么」（如「教学式解构」）；第 2 句拆出「几层可抄的价值」（段子/语法说明书/方法论）。这是参考样例最强的部分。
4. **核心公式 `THE FORMULA`**：`<ol class="formula">`，每条 = `<b>动作名</b><span>一句说明，最好引原文旁白/花字作证据</span>`。6–8 条，每条都要能对应到画面。
5. **结构：五幕与节奏曲线 `STRUCTURE`**：先一段「快慢呼吸」判断 → 内联 SVG 柱图（`.bars`，红 `#d8a08f`=降速段、灰 `#7a7a72`=快切，每格标镜头号+时长）→ 分幕汇总表（幕 / 时码 / 镜头 / 均长 / 功能）。
6. **分幕配接触表图**：每幕一个 `<section class="act"><h3>第X幕 · 名<span class="sub">时码 · Sx–Sy</span></h3><img class="sheet" src="data:image/jpeg;base64,..."></section>`。
7. **三个关键帧 `KEY FRAMES`**：`.keyrow` 三栏，每栏 `<img>` + 灰字说明。
8. **逐镜头拆解表 `SHOT LIST`**：列 = `# / 时码 / 秒 / 景别机位 / 画面 / 旁白花字 / 拉片笔记`。旁白/花字单元格加 `.vo`（砖红）；正文用【旁白】/【花字】标注来源；拉片笔记须「有画面依据、区分观察与推断」、并回答「为什么有效/我怎么抄」。
9. **声音结构 `SOUND`**：覆盖模式（覆盖式/留白式）+ 气口位置 + 响度/LRA/真峰值；末尾加一条 `.note`「可学的点」（从这条片提炼的可迁移剪辑技巧）。
10. **色彩与光 `COLOR & LIGHT`**：配色纪律（主体独占色相、环境降饱和）+ 光（逆光/侧逆光/轮廓光）+ 景深（长焦浅焦杜绝「清晰到廉价」）。
11. **带走就能用的清单 `CHECKLIST`**：表 `# / 动作 / 执行要点`，6–10 条，动作必须可立刻执行。
12. **反向提醒**：`.note` callout，点明这套手法什么时候会翻车（参考样例：反讽成立依赖「自曝」，相信了自己就成土味）。
13. **给你自己的用法 `FOR YOU`**：必须具体到用户真实项目（参考样例把它套到「小红书财经图文视频化」「室内设计项目」），别写泛泛的「可用于短视频」。
14. **foot**：拉片方法说明（ffmpeg 管线）+ 素材来源（链接/平台）+ 「仅供学习研究」。

### 7.2 硬标准（来自交付参考）

- 零 emoji；标题宋体（Songti SC），正文 PingFang SC 15px，最大宽 900px，砖红 `#b3382c` 点缀。
- 接触表/关键帧用 JPEG base64 内联（Step 4 的 tile 2×3 → `-q:v 4` 转 JPEG，35 帧约 850KB 可接受）。
- 镜头时长曲线用**内联 SVG 柱图**，不引外部图表库。
- 旁白/花字必须是从画面读出的原文，标注【旁白】/【花字】，不得编造。
- 每条「拉片笔记」都要落到「为什么有效 / 我怎么抄」，不能停在描述画面。

### 7.3 最小骨架（复制后填数据）

```html
<div class="wrap">
<h1>主标题<span class="thin">拉片报告 · 片名 · 时长 X:XX.X</span></h1>
<div class="rule"></div>
<div class="meta">
<div><b>片长</b><span>__s</span></div><div><b>镜头数</b><span>__N</span></div>
<div><b>平均镜头</b><span>__s</span></div><div><b>规格</b><span>__WxH / __fps</span></div>
<div><b>剪辑点检出</b><span>__原 → __去闪</span></div><div><b>响度</b><span>__ LUFS</span></div>
</div>
<p>__开场论点段1：这条片形式上是什么__</p>
<p>__开场论点段2：几层可抄的价值__</p>

<h2>核心公式<span class="en">THE FORMULA</span></h2>
<ol class="formula">
<li><b>__动作名__</b><span>__说明，引原文旁白/花字作证据__</span></li>
</ol>

<h2>结构：五幕与节奏曲线<span class="en">STRUCTURE</span></h2>
<p>__快慢呼吸判断__</p>
<div class="bars"><svg viewBox="0 0 852 455" width="100%">__每镜一根柱，红=降速段__</svg></div>
<p class="cap">每格一个镜头，长度即时长（秒）。红色=结尾降速段，浅色=1–2 秒的快切。</p>
<table><tr><th>幕</th><th>时码</th><th>镜头</th><th>均长</th><th>功能</th></tr>
<tr><td>一 · __</td><td class="mono">__</td><td class="c">S1–S__</td><td class="mono">__s</td><td>__</td></tr></table>

<section class="act"><h3>第一幕 · __<span class="sub">__ · S__–S__</span></h3><img class="sheet" src="data:image/jpeg;base64,__"></section>

<h2>三个关键帧<span class="en">KEY FRAMES</span></h2>
<div class="keyrow"><div><img src="data:image/jpeg;base64,__"><p>__</p></div>×3</div>

<h2>逐镜头拆解表<span class="en">SHOT LIST</span></h2>
<table><tr><th>#</th><th>时码</th><th>秒</th><th>景别 / 机位</th><th>画面</th><th>旁白 / 花字</th><th>拉片笔记</th></tr>
<tr><td class="c">1</td><td class="c mono">__</td><td class="c mono">__</td><td>__</td><td>__</td><td class="vo">【__】__</td><td>__</td></tr></table>

<h2>声音结构<span class="en">SOUND</span></h2>
<p>__覆盖模式/气口/响度/LRA/真峰值__</p>
<div class="note">可学的点：__</div>

<h2>色彩与光<span class="en">COLOR &amp; LIGHT</span></h2>
<p>__配色纪律 + 光 + 景深__</p>

<h2>带走就能用的清单<span class="en">CHECKLIST</span></h2>
<table><tr><th>#</th><th>动作</th><th>执行要点</th></tr>
<tr><td class="c">1</td><td>__</td><td>__</td></tr></table>

<div class="note">反向提醒：__</div>

<h2>给你自己的用法<span class="en">FOR YOU</span></h2>
<p>__具体到用户真实项目__</p>

<div class="foot">拉片方法：ffmpeg 场景检测（阈值 0.22，0.7s 内合并去闪频）→ 逐镜抽帧目验 → 静音检测定位旁白气口 → 响度归一化测量。素材来源：__。仅供学习研究。</div>
</div>
```

---

## 校验清单（交付前自检）

- [ ] 镜头数与时长分布打印出来了，无 0.1s 级碎片（去闪频做对了）
- [ ] 抽帧时间点在镜头中段，没有抽到黑场/转场
- [ ] 接触表图片读过了，每条拉片笔记都有画面依据，不是套话
- [ ] 旁白/花字是**从画面里读出来的原文**，不是编的
- [ ] 报告里区分了「观察」与「推断」
- [ ] 交付路径 + 注意事项（版权/仅供学习）写清楚了
- [ ] 报告开头有**开场论点段**（这条片形式上是什么 + 几层可抄价值），不是一上来就列表
- [ ] 影片档案是**六格 `.meta` 卡片**（片长/镜头数/平均镜头/规格/剪辑点检出/响度），不是纯文字
- [ ] 逐镜表的旁白/花字单元格标了【旁白】/【花字】并加 `.vo` 砖红；镜头时长曲线是**内联 SVG 柱图**
- [ ] 结尾 `foot` 写了方法 + 素材来源 + 「仅供学习研究」
- [ ] 若会话读图失效，走的是 Step 6.1 程序化管线（whisper 旁白 + OCR 花字 + 帧特征归类），且 foot 已注明方法——**没有靠猜画面**

## 注意事项

- 拉片成果用于学习研究；报告尾部注明素材来源，不外传原片。
- 报告别停在"描述画面"——**没有"为什么有效"和"我怎么抄"，就不算拉片**。
