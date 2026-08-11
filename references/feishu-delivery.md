# 把写作手册交付到飞书

## 目录

- 先确认当前 Agent 能做什么
- 检查飞书登录和权限
- 准备本地交付文件
- 创建一篇飞书文档
- 把正文和画板放进文档
- 检查文档和每张画板
- 飞书暂时不能用时怎样交付

## 先确认当前 Agent 能做什么

只要用户选择飞书交付，就读取本文件并走完整流程。不同 Agent 使用同一组 `lark-cli` 命令。

当前 Agent 已经安装 `lark-doc`、`lark-whiteboard` 和 `lark-shared` 时，可以调用这些 Skill 完成对应步骤。没有这些 Skill 时，直接按本文件的命令执行。

`lark-cli` 自带与当前版本匹配的操作说明。执行飞书命令前先读取这些说明；如果命令或返回字段有变化，以用户机器上的说明为准：

```bash
lark-cli skills read lark-shared
lark-cli skills read lark-doc references/lark-doc-xml.md
lark-cli skills read lark-doc references/style/lark-doc-style.md
lark-cli skills read lark-doc references/style/lark-doc-create-workflow.md
lark-cli skills read lark-doc references/lark-doc-create.md
lark-cli skills read lark-doc references/lark-doc-fetch.md
lark-cli skills read lark-doc references/style/lark-doc-update-workflow.md
lark-cli skills read lark-doc references/lark-doc-update.md
lark-cli skills read lark-whiteboard
lark-cli skills read lark-whiteboard references/lark-whiteboard-update.md
lark-cli skills read lark-whiteboard references/lark-whiteboard-export.md
```

这些命令只读取说明，不会创建或修改飞书内容。

读取不到其中某个文件时，先运行对应命令的 `--help`，按帮助顶部给出的 `lark-cli skills read ...` 路径重新读取。不要靠旧记忆猜参数。

先检查两个工具：

```bash
lark-cli --version
npx -y @larksuite/whiteboard-cli@^0.2.13 -v
```

- 找不到 `lark-cli`：说明当前 Agent 无法直接写入飞书，转到本文件最后的本地交付流程。不要擅自安装软件。
- 找不到 `npx` 或画板工具：仍可写飞书正文；先告诉用户画板暂时只能交付 SVG 文件，不能宣称画板已经写入或检查完成。
- 两个工具都可用：继续。

## 检查飞书登录和权限

默认使用用户身份（`--as user`），让新文档出现在用户自己的飞书空间里。

```bash
LARKSUITE_CLI_NO_UPDATE_NOTIFIER=1 LARKSUITE_CLI_NO_SKILLS_NOTIFIER=1 \
  lark-cli auth status --json --verify
```

继续前检查这些结果：

- 命令退出码为 0，JSON 中 `ok` 为 `true`；
- `verified` 为 `true`；
- 用户身份状态正常，登录凭证（token）没有过期；
- 后续文档命令统一带 `--as user`。

登录失效或没有用户身份时，按下面的分步授权处理：

```bash
lark-cli auth login --domain docs --domain drive --no-wait --json
```

拿到 `verification_url` 和 `device_code` 后：

1. 保持授权链接原样，不改写参数；
2. 用 `lark-cli auth qrcode "<verification_url>" --output ./feishu-auth.png` 生成二维码；
3. 把链接和二维码都交给用户；
4. 用户回复“已授权”后，由当前 Agent 执行 `lark-cli auth login --device-code <device_code>`；
5. 重新运行登录状态检查。

创建或更新文档时如果返回 `missing_scopes`，使用错误信息给出的具体权限（scope）重新授权：

```bash
lark-cli auth login --scope "<missing_scope>" --no-wait --json
```

错误中有 `console_url` 时，把它原样交给用户。应用身份（bot）缺权限时，按错误里的后台链接处理。不要输出 token、app secret 或其他密钥。

## 准备本地交付文件

先在当前工作目录准备完整的本地产物，再写入飞书：

```text
deliverables/<run-name>/
├── document.xml
├── sections/
│   ├── 01-book-map.xml
│   ├── 02-close-reading.xml
│   ├── 03-use-it.xml
│   ├── 04-practice.xml
│   ├── 05-personal-skill.xml
│   └── 06-full-map.xml
└── boards/
    ├── book-question.svg
    ├── book-structure.svg
    ├── book-methods.svg
    └── group-*.svg
```

所有传给 `lark-cli` 的文件路径都使用当前工作目录下的相对路径。不要传绝对路径。

正文默认使用飞书 XML：

- 文档标题用 `<title>`；
- 大章节用 `<h1>`，卡片组用 `<h2>`，单张卡用 `<h3>`；
- 卡片正文使用 `<table>`；
- 简短提醒使用少量 `<callout>`；
- 练习入口使用 `<checkbox done="false">`；
- 画板放在它负责解释的章节后面。

完整 XML 语法以当前 `lark-cli docs +create --help` 和 `docs +update --help` 为准。当前 Agent 有 `lark-doc` 时，创建前先读取它的 XML、文档样式和新建文档规则。

## 先把画板做成可检查的 SVG

每张画板都先生成一个完整、自包含的 SVG 文件，再插入飞书。颜色、字号、布局和单句卡省略规则按 [`whiteboard-spec.md`](whiteboard-spec.md) 执行。

逐张运行：

```bash
npx -y @larksuite/whiteboard-cli@^0.2.13 \
  -i deliverables/<run-name>/boards/<board>.svg \
  -o deliverables/<run-name>/boards/<board>.png -f svg

npx -y @larksuite/whiteboard-cli@^0.2.13 \
  -i deliverables/<run-name>/boards/<board>.svg \
  -f svg --check
```

检查结果必须没有文字溢出、节点重叠和内容遮挡。再打开 PNG 看一次，确认文字完整、箭头方向清楚、没有裁切。

在章节 XML 中按下面的方式插入对应画板：

```xml
<whiteboard type="svg" path="@deliverables/<run-name>/boards/<board>.svg"></whiteboard>
```

这种方式会在写入正文时一起创建画板。SVG 必须有 `<svg>` 根节点和 `viewBox`，文字使用 `<text>`，不引用外部图片、脚本或远程资源。

## 创建一篇飞书文档

这类写作手册通常很长，优先先建文档骨架，再按顺序写入各节。`document.xml` 只放标题、开头说明和各大章节标题：

```bash
lark-cli docs +create --as user \
  --content @deliverables/<run-name>/document.xml
```

从成功结果中保存两项：

- `data.document.document_id`；
- `data.document.url`。

判断成功时看退出码和 `ok == true`。不要用 `code == 0` 判断。

接着获取标题对应的内容块 ID（block ID）：

```bash
lark-cli docs +fetch --as user \
  --doc "<document_id>" \
  --scope outline --max-depth 3 --detail with-ids
```

## 把正文和画板放进文档

按文档顺序逐节写入。每个章节文件包含该节的正文、表格和对应画板：

```bash
lark-cli docs +update --as user \
  --doc "<document_id>" \
  --command block_insert_after \
  --block-id "<section_heading_block_id>" \
  --content @deliverables/<run-name>/sections/01-book-map.xml
```

每次写入后检查：

- 退出码为 0，`ok == true`；
- `data.result` 为 `success`；
- `warnings` 中没有内容丢失、画板被改成其他形式或位置错误；
- `data.document.new_blocks` 中新增的画板都有 `block_token`。

一节过长需要拆成多次写入时，重新获取该节内容和 block ID，把下一段插在刚写入的最后一个 block 后面。不要反复锚在同一个标题上，避免段落顺序颠倒。

正文和画板必须进入同一篇文档。不要为完整章节表、练习或画板再创建第二篇文档。

### 画板没有随正文写入时

先在正确位置插入空白画板：

```bash
lark-cli docs +update --as user \
  --doc "<document_id>" \
  --command block_insert_after \
  --block-id "<target_block_id>" \
  --content '<whiteboard type="blank"></whiteboard>'
```

从 `data.document.new_blocks` 中取得 `block_token`，再把已经检查过的 SVG 写进去：

```bash
lark-cli whiteboard +update --as user \
  --whiteboard-token "<board_token>" \
  --input_format svg \
  --source @deliverables/<run-name>/boards/<board>.svg \
  --overwrite \
  --idempotent-token "<10个字符以上且本次重试保持不变的标识>"
```

同一次逻辑更新重试时复用同一个 `idempotent-token`，避免重复写入。

## 检查文档和每张画板

全部写完后重新读取文档：

```bash
lark-cli docs +fetch --as user \
  --doc "<document_id>" --detail with-ids
```

逐项核对：

- 标题只出现一次；
- 大章节顺序与 [`feishu-course-template.md`](feishu-course-template.md) 一致；
- 章节表或文本地图、拆句卡、练习和个人 Skill 入口都在同一篇文档；
- 卡片编号连续，M 和 C 第一次出现时已经解释；
- 每个需要画板的位置都有 `<whiteboard token="...">`；
- 只有单句、没有步骤可画的组可以没有画板。

逐张导出飞书里的实际画板预览：

```bash
lark-cli whiteboard +export --as user \
  --whiteboard-token "<board_token>" \
  --output-type preview \
  --output ./deliverables/<run-name>/boards/feishu-<board>
```

打开导出的图片检查内容是否完整。飞书缩略图可能补成方形，重点检查文字、节点和连线有没有被裁掉。

全部通过后，只给用户一个 `data.document.url`。同时简短说明：文档包含多少张拆句卡、多少张画板，以及完整章节表或文本地图在文末。

## 飞书暂时不能用时怎样交付

保留同一套内容，交付下面的本地目录：

```text
deliverables/<run-name>/
├── handbook.md
└── boards/
    ├── *.svg
    └── *.png
```

Markdown 使用相对路径插入画板，例如：

```markdown
![全书结构](boards/book-structure.png)
```

飞书待办改成 Markdown 待办：

```markdown
- [ ] 写完练习后，把答案发给我批改
```

交付时给出 `handbook.md` 的完整本地路径，并说明飞书失败发生在哪一步。文件已经生成但客户端暂时无法写入飞书时，不宣称飞书文档已经创建。
