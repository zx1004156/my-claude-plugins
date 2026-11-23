---
name: jp-epaper-pdf
description: "Roy 給日文內容（字幕、文章、翻譯練習 HTML）要做成電子書閱讀器用的日中對照 PDF 時使用：斷句、翻譯、挑難字讀音，產出 mooInk Nana 7 吋 PDF。"
---

# 日中對照 電子書閱讀器 PDF

Roy 用 Readmoo mooInk Nana（7 吋黑白電子紙）學日文。他給內容，你產出一份可直接上傳 Readmoo 的 PDF。

## 版面（已定案，不要改）
- 奇數頁：日文句子表格（編號＋句子），明朝體
- 偶數頁：同編號中文翻譯；該句難字讀音以小字列在翻譯下方，格式「漢字（よみ）」，中間用細短虛線（0.5pt dashed）分隔
- 中文頁頁首黑底白字；日文頁不放任何讀音
- 頁面 105×140mm，依句長自動分頁（最多 7 句），每個 Part 從新頁開始
- 字型必須以 TrueType 嵌入（不可有 Type 3，否則 Readmoo 轉檔失敗）— 腳本已處理

## 流程
1. **取得內容**
   - 純日文：自己翻成自然的台灣繁體中文
   - 已有日中對照（例如他的翻譯練習 HTML 裡的 `const DATA`）：用 Python 解析，沿用原翻譯
   - 有章節/Part 的話保留為 `# 標題` 行（標題格式建議 `Part 1｜主題 〔時間〕`）
2. **斷句**：超過約 50 字的句子在逗號、轉折處切開，中文翻譯對應切開。用程式檢查：切開後日文串接起來必須等於原句。
3. **挑讀音**：Roy 已過 N2。挑 N2 以上或少見的漢字詞（如 凝り固まる、開き直る、捉え方），不要標太簡單的字。同一詞在同一 Part 只在第一次出現時標。key 必須是句中實際出現的字串（可用語幹，如 `喋=しゃべ`）。
4. **寫 TSV**：每行 `日文<TAB>中文<TAB>漢字=よみ;漢字=よみ`
5. **產生 PDF**：把下方腳本寫成檔案執行 `python3 jp_epaper_pdf.py input.tsv 檔名_日中對照.pdf`
   - 確認輸出 `type3=False`，且頁數等於 expected
   - 抽一兩頁用 `pdftoppm -r 90 -png -f N -l N` 看一下
6. **交付**：用 SendUserFile 傳 PDF。回覆簡短：句數、頁數、斷了幾句、標了多少讀音，請他告訴你要增減哪些讀音。

## 注意
- 回覆要短，只講重點（Roy 不喜歡長文）
- 檔名用內容標題，例如 `內向的な自分との付き合い方_日中對照.pdf`
- 需要 fontTools、google-chrome、poppler（pdffonts/pdfinfo/pdftoppm）；缺的話先安裝（pip 要加 --break-system-packages）

## 腳本 jp_epaper_pdf.py

```python
"""日中對照 電子書閱讀器 PDF（mooInk Nana 7 吋）
用法: python3 jp_epaper_pdf.py input.tsv output.pdf
input.tsv:
  # Part 1｜標題        ← 新 Part，從新頁開始（可省略）
  日文<TAB>中文<TAB>漢字=よみ;漢字=よみ
奇數頁=日文；偶數頁=中文＋難字讀音（細短虛線分隔）。
字型轉成 TrueType 嵌入（避免 Type 3，Readmoo 才能轉檔）。
"""
import html, math, os, subprocess, sys, tempfile
from fontTools.ttLib import TTCollection, newTable
from fontTools import subset
from fontTools.pens.cu2quPen import Cu2QuPen
from fontTools.pens.ttGlyphPen import TTGlyphPen

src, out = sys.argv[1], os.path.abspath(sys.argv[2])
MAX_PER_PAGE = 7
E = html.escape

# ---------- 讀取 ----------
parts = [["", []]]
raw = open(src, encoding="utf-8").read()
for line in raw.splitlines():
    if not line.strip():
        continue
    if line.startswith("# "):
        parts.append([line[2:].strip(), []]); continue
    jp, zh, rd = (line.split("\t") + ["", ""])[:3]
    reads = [tuple(x.split("=", 1)) for x in rd.split(";") if "=" in x]
    parts[-1][1].append((jp.strip(), zh.strip(), reads))
parts = [p for p in parts if p[1]]

# ---------- 分頁 ----------
def w(t): return sum(0.5 if ord(c) < 0x2000 else 1 for c in t)
def row_h(t, sub=""):
    h = math.ceil(w(t) / 18.5) * 6.6 + 3.6
    if sub: h += math.ceil(w(sub) / 27) * 4.6 + 1.6
    return h
def paginate(items, budget=114):
    pages, cur, hj, hz = [], [], 0, 0
    for it in items:
        jp, zh, rd = it
        a = row_h(jp); b = row_h(zh, "　".join(f"{k}（{v}）" for k, v in rd))
        if cur and (hj + a > budget or hz + b > budget or len(cur) >= MAX_PER_PAGE):
            pages.append(cur); cur, hj, hz = [], 0, 0
        cur.append(it); hj += a; hz += b
    if cur: pages.append(cur)
    return pages

def reads_html(rd):
    return "　".join(f"<span class='w'>{E(k)}<span class='y'>{E(v)}</span></span>" for k, v in rd)
def section(cls, title, rows):
    trs = "".join(f'<tr><td class="n">{n}</td><td class="t">{t}</td></tr>' for n, t in rows)
    return f'<section class="{cls}"><div class="hd">{title}</div><table>{trs}</table></section>'

pages, num = [], 0
for ptitle, items in parts:
    short = ptitle.split("｜")[0] + "｜" if ptitle else ""
    chunks = paginate(items); i = 0
    for p, ch in enumerate(chunks, start=1):
        rows = list(enumerate(ch, start=num + i + 1)); i += len(ch)
        tag = f"｜ {short}{rows[0][0]}–{rows[-1][0]}　　{p}/{len(chunks)}"
        pages.append(section("jp", f"日本語 {tag}", [(n, E(jp)) for n, (jp, _, _) in rows]))
        pages.append(section("zh", f"中文翻譯 {tag}",
            [(n, E(zh) + (f"<div class='sub'>{reads_html(rd)}</div>" if rd else ""))
             for n, (_, zh, rd) in rows]))
    num += len(items)

# ---------- 字型：子集化 + 轉 TrueType ----------
work = tempfile.mkdtemp()
chars = "".join(sorted(set(raw + open(__file__, encoding="utf-8").read()
                           + "0123456789（）｜–/ 　")))
FONTS = [("/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc", "Noto Sans CJK TC", "sans-tc.ttf"),
         ("/usr/share/fonts/opentype/noto/NotoSansCJK-Bold.ttc", "Noto Sans CJK TC", "sans-tc-bold.ttf"),
         ("/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc", "Noto Sans CJK JP", "sans-jp.ttf"),
         ("/usr/share/fonts/opentype/noto/NotoSerifCJK-Medium.ttc", "Noto Serif CJK JP", "serif-jp.ttf")]
for path, fam, name in FONTS:
    font = next(f for f in TTCollection(path).fonts if f["name"].getDebugName(1).startswith(fam))
    o = subset.Options(); o.layout_features = ["*"]; o.name_IDs = ["*"]
    s = subset.Subsetter(o); s.populate(text=chars); s.subset(font)
    gs, order = font.getGlyphSet(), font.getGlyphOrder()
    glyf = newTable("glyf"); glyf.glyphOrder = order; glyf.glyphs = {}
    for g in order:
        pen = TTGlyphPen(gs); gs[g].draw(Cu2QuPen(pen, 1.0, reverse_direction=True))
        glyf.glyphs[g] = pen.glyph()
    font["glyf"] = glyf; font["loca"] = newTable("loca"); del font["CFF "]
    if "VORG" in font: del font["VORG"]
    m = font["maxp"]; m.tableVersion = 0x00010000
    for a in ("maxTwilightPoints","maxStorage","maxFunctionDefs","maxInstructionDefs","maxStackElements",
              "maxSizeOfInstructions","maxComponentElements","maxPoints","maxContours",
              "maxCompositePoints","maxCompositeContours","maxComponentDepth"):
        setattr(m, a, 0)
    m.maxZones = 1
    font["head"].indexToLocFormat = 0; font["head"].glyphDataFormat = 0
    font["post"].formatType = 3.0; font.sfntVersion = "\x00\x01\x00\x00"
    font.save(os.path.join(work, name))

# ---------- HTML → PDF ----------
css = f"""
@font-face {{ font-family:'R-TC'; src:url('file://{work}/sans-tc.ttf'); font-weight:400; }}
@font-face {{ font-family:'R-TC'; src:url('file://{work}/sans-tc-bold.ttf'); font-weight:700; }}
@font-face {{ font-family:'R-JP'; src:url('file://{work}/sans-jp.ttf'); }}
@font-face {{ font-family:'R-SERIF'; src:url('file://{work}/serif-jp.ttf'); }}
""" + """
@page { size: 105mm 140mm; margin: 6mm; }
* { box-sizing: border-box; }
body { margin: 0; color: #000; font-family: 'R-TC'; }
section { page-break-after: always; height: 128mm; display: flex; flex-direction: column; }
section:last-child { page-break-after: auto; }
.hd { font-size: 8.5pt; font-weight: 700; border-bottom: 1.5pt solid #000; padding-bottom: 1.5mm; margin-bottom: 1mm; }
.zh .hd { background: #000; color: #fff; padding: 1mm 2mm; border: none; }
table { width: 100%; border-collapse: collapse; flex: 1; }
tr { border-bottom: 0.6pt solid #000; }
td { vertical-align: middle; padding: 1.6mm 0; }
td.n { width: 7mm; font-weight: 700; font-size: 10pt; }
td.t { font-size: 12.5pt; line-height: 1.5; }
.jp td.t { font-family: 'R-SERIF'; }
.w { white-space: nowrap; }
.y { font-size: 0.85em; margin-left: 1.2mm; }
.y::before { content: "（"; } .y::after { content: "）"; }
.sub { font-family: 'R-JP'; font-size: 9pt; margin-top: 1mm; padding-top: 0.8mm; line-height: 1.5;
       border-top: 0.5pt dashed #000; }
"""
doc = f'<!doctype html><html lang="ja"><head><meta charset="utf-8"><style>{css}</style></head><body>{"".join(pages)}</body></html>'
h = os.path.join(work, "a.html"); open(h, "w", encoding="utf-8").write(doc)
subprocess.run(["google-chrome", "--headless", "--no-sandbox", "--disable-gpu", "--no-pdf-header-footer",
                "--allow-file-access-from-files", f"--print-to-pdf={out}", "file://" + h],
               check=True, capture_output=True)

# ---------- 檢查 ----------
fonts = subprocess.run(["pdffonts", out], capture_output=True, text=True).stdout
info = subprocess.run(["pdfinfo", out], capture_output=True, text=True).stdout
n_pages = int(next(l.split()[-1] for l in info.splitlines() if l.startswith("Pages")))
print(f"ok {out}  sentences={num}  pages={n_pages} (expected {len(pages)})  "
      f"type3={'Type 3' in fonts}")
if n_pages != len(pages): print("WARNING: 頁數不符，可能有頁面溢出")
if "Type 3" in fonts: print("WARNING: 有 Type 3 字型，Readmoo 可能轉檔失敗")
```
