---
name: chinesexiv
description: 把 arXiv 论文自动翻译为中文 PDF 的 ChineseXiv skill。触发后按本 skill 四步顺序直接执行，勿长篇规划。用户提供论文标题或 arXiv ID、说「翻译论文」「我想读中文版」等时立即使用。支持本地 xelatex 与在线编译双引擎、英文原文 _zh 副本对照、多篇并行处理。无需用户手动操作。
---

# ChineseXiv：arXiv 论文中文翻译

**目标：** 将指定论文的 LaTeX 源码译为中文，并编译得到 PDF。

**流程：** 须严格按下文「第一步」至「第四步」顺序执行，不得擅自省略、合并或调换步骤。

**交互：** 仅在论文 ID 无法确定、检索结果存在多个需用户择一才可向用户提问；其余情况一律无中断的执行得到最终翻译后的PDF。

**翻译：** 翻译全部由当前对话模型自身完成，严禁使用外部翻译工具以及下载已有的翻译版本。

---

## 第一步：确定论文 ID

- arXiv URL/ID → 直接提取 ID
- 论文标题 → 搜索 arXiv / 网页查找 ID；找不到时给出候选让用户确认

---

## 第二步：获取源码

```bash
python3 {SKILL_DIR}/scripts/download.py "{PAPER_ID}" "$OUTPUT_DIR/source"
```

`download.py` 一步完成：下载源码 → 解压 → 递归查找 `.tex` → 定位主文件 → 提取标题 → 把主文件以及它通过 `\input{}` / `\include{}` 引用的子 `.tex` 文件都复制为 `_zh.tex` 副本，并把 zh 主文件里的 input 引用一并改写到 `_zh` 版本。

`OUTPUT_DIR` 为用户指定的保存路径，未指定则为当前目录；源码统一落在 `$OUTPUT_DIR/source/`（不再使用 `.tmp_arxiv` 之类的临时名）。

无源码（仅 PDF）则告知用户跳过。

脚本向 stdout 输出四行，格式如下：
```
WORK_DIR=<源码目录绝对路径，即 $OUTPUT_DIR/source>
MAIN_TEX=<英文原始主文件相对路径，只读快照>
MAIN_TEX_ZH=<供翻译的中文主文件相对路径（download.py 已自动复制好）>
PDF_NAME=<论文英文标题（仅供生成备用名）>
```

`MAIN_TEX` 永远是英文原文，**永远不要修改**；翻译只动 `MAIN_TEX_ZH` 及其 `_zh` 子文件。两者并列存放，方便对照。

---

## 第三步：翻译

由当前**对话模型**直接对 `$WORK_DIR/$MAIN_TEX_ZH`（以及它 input 进来的 `_zh` 子文件）进行翻译修改。`$MAIN_TEX` 是英文只读快照，**永远不要修改**。按以下规则翻译：

- **翻译范围：** 默认翻译全文，包括 `\appendix` 之后的附录内容。用户明确要求「只翻正文」「不翻附录」时，才在 `\appendix` 之前停止翻译。
- **必须翻译：** 正文叙述、摘要、图表标题、列表项、脚注中的描述文本、代码块中的注释；**机构名**（如「斯坦福大学（Stanford University）」）与**描述性方法名/模块名**（如「自适应检索模块（Adaptive Retrieval Module, ARM）」）也要中译，按下方「术语首次出现规则」给出中英对照。
- **保留不翻：** 数学环境、LaTeX 命令、`\cite{}`/`\ref{}`/`\label{}`、图片路径、URL、代码本体、`.bib`、**人名**、**专有模型名**（GPT-4、LLaMA、Qwen、DeepSeek 等已成符号的型号）、**专有数据集/基准名**（ImageNet、MMLU、GSM8K 等）。
- **不要新增字体修饰：** 严禁额外添加 `\textit{}`、`\textbf{}`、`\emph{}` 等格式命令——只在原文已存在时按位置保留。括号里的英文机构名、英文术语原文一律用 \textbf{正体（plain）}，不要套 `\textit{}`；括号内的英文只是为方便对照，不属于「需要强调」的内容。
- **专有名词：** Transformer、Softmax、Token、Attention、Self-Attention、Multi-Head Attention、Scaled Dot-Product Attention 等已通用化的学术术语**严格保留英文**，不要硬译为「转换器 / 注意力 / 自注意力 / 多头注意力」等。复合词同样保留：`Multi-Head Self-Attention`、`Restricted Self-Attention`、`Encoder Self-Attention` 等都不要拆译。其它领域的同类约定俗成术语（CNN、RNN、LSTM、Embedding、Token、Logits、Softmax、Dropout、BLEU 等）同理。
- **术语首次出现规则（重要）：** 全文术语、符号、代号需统一；非通用新名词、新术语、新概念在**首次出现**时给出清晰说明。
  - 反复出现（≥2 次）的较长词组，在**首次出现**时写作「中文全称（English Full Name, ABBR）」，例如「灾害行动响应智能体（Disaster Operational Response Agent, DORA）」，之后全文统一用缩写 `ABBR` 或中文简称；
  - 全文仅出现一次的，直接用中文全称即可，必要时在括号中附英文全称（不加缩写）；
  - **摘要与正文各自独立统计**：同一术语若在摘要和正文中都是首次出现，两处都要分别注明完整定义，确保摘要可以独立阅读。
  - 仅有缩写而无英文全称的写法（例如「MCP 库」「SAR 影像」）必须替换为定义形式。
- **标题要求：** `\title{}` 须改为自然中文题名，不保留英文原题或中英并列；用户也将以此中文题名作为 PDF 文件名。
- **多篇处理：** 多篇论文可以分别处理；只有在用户**明确要求**并行委派时，才开启多个 subagent，否则直接顺序完成。

译后必须做自检（默认 full 模式扫描全文，扫描的是中文翻译目标文件）：

```bash
python3 {SKILL_DIR}/scripts/inspect_tex.py scan "$WORK_DIR" "$MAIN_TEX_ZH" full
```

若用户明确要求「只翻正文」，则改为：

```bash
python3 {SKILL_DIR}/scripts/inspect_tex.py scan "$WORK_DIR" "$MAIN_TEX_ZH" body
```

脚本会输出 `SUSPECT_COUNT=<数字>` 以及若干 `SUSPECT=<文件>:<行号>:<片段>`。
- 只要 `SUSPECT_COUNT` 非 0，就必须逐条回到对应位置进行翻译；
- 只有 `SUSPECT_COUNT=0`，或剩余项明确属于「保留不翻」范围时，才可进入第四步。

---

## 第四步：编译与清理

确定中文 PDF 文件名（基于翻译后的 `\title{}` 自然命名，与 `\title{}` 一致或更精炼），赋值给 `PDF_NAME_ZH`。最终 PDF 直接落在 `$OUTPUT_DIR/$PDF_NAME_ZH.pdf`，与 `source/` 同级。

```bash
python3 {SKILL_DIR}/scripts/compile.py "$WORK_DIR" "$MAIN_TEX_ZH" "$OUTPUT_DIR/$PDF_NAME_ZH.pdf" --engine auto
```

**`--engine` 选项**（默认 `auto`）：
- `auto`：先检测本地是否装有 `xelatex`，装了就用本地编译；未装则回落到在线服务。**首选**。
- `local`：强制本地 `xelatex` 编译；未装 `xelatex` 直接报错。适合环境齐全 + 想完全离线的场景。
- `online`：强制走 `latex.ytotech.com` 在线编译。适合本地 TeX 环境缺包、或者想保留旧行为。

用户可以在调用时通过自然语言指定（例如「用在线编译」「强制本地编译」），否则一律传 `--engine auto`。

`compile.py` 会统一完成以下编译前处理（两种引擎共用）：
- 默认使用 `xelatex` + `xeCJK` 编译栈；若检测到中文且主文件尚无 CJK 支持，自动注入 `fontspec` / `xeCJK` / `\setCJKmainfont{Noto Serif CJK SC}`；
- 自动注释掉与 Unicode 编译栈冲突的 `fontenc` / `inputenc`；
- 自动识别 `bibtex` / `biber` / 已内置 `.bbl` 的情况；
- 自动忽略常见编译中间文件、`build/` 子目录以及未被源码引用的游离 PDF（在线引擎下避免把无关产物上传；本地引擎下避免污染源码目录）。

编译失败时：读取 stderr 中的错误日志，参考 `references/compile-errors.md` 修复源码，重新编译（最多重试 2 次）。

**本地手动重编（应急）：** `source/build/` 子目录是本地 xelatex 中间产物的约定输出位置。手动重编命令：

```bash
cd "$OUTPUT_DIR/source"
xelatex -output-directory=build "$MAIN_TEX_ZH"
BIBINPUTS=".:$BIBINPUTS" bibtex "build/${MAIN_TEX_ZH%.tex}"
xelatex -output-directory=build "$MAIN_TEX_ZH"
xelatex -output-directory=build "$MAIN_TEX_ZH"
cp "build/${MAIN_TEX_ZH%.tex}.pdf" "../$PDF_NAME_ZH.pdf"
```

编译成功后清理（删除 `source/build/` 与 inspect 临时文件，保留英文原文与翻译稿源码以便对照与手动重编）：

```bash
python3 {SKILL_DIR}/scripts/cleanup.py "$OUTPUT_DIR"
```

多篇论文时，所有论文都完成 PDF 编译并保存后再进行中间文件清理。

最后告知用户：
- PDF 路径：`$OUTPUT_DIR/$PDF_NAME_ZH.pdf`
- 源码目录：`$OUTPUT_DIR/source/`（含英文原文 + `_zh.tex` 中文译稿，可手动重编）

---

## 参考文件
- `references/compile-errors.md`：编译常见错误及修复方法
- `references/author-block.md`：作者/机构区的排版与译名规范（短机构内联、长机构换行、`\textit` 禁用规则、「全国/国家重点实验室」译法）。翻译机构/作者块前先读这个。
