# 用 Grok Bot 指挥 Claude Code 与 DeepSeek Harness：一个结果文件省掉大半 token



我用一堆 Grok Bot 管几个项目，游戏、SEO 小站、自己的 MCP 工具都有。它们负责想事情和排期，真正写代码、跑测试、点浏览器的活丢给本机 Mac 上的两个 agent，dsh 跑在沙箱里，Claude Code 用 `claude -p` 非交互地改文件。

重活在本机，协调在云端，中间那条缝一直不好处理。Bot 自己有额度，上下文一长就贵，也容易犯迷糊。最早的版本是人肉粘贴，我自己开 dsh 或 Claude Code，跑完把终端输出整段贴回对话。慢还能忍，难受的是那段输出会一直压在后面的每一轮里，bot 一边读日志一边想下一步，读着读着连自己前面说过什么都不太确定。

后来定了一件事，执行方干完活写一个结果文件，人不用再搬日志。好处是文件待在磁盘上，bot 想知道什么就读什么，没读的那部分不占上下文，执行方也不用等人来问。

{{< image src="/images/agent-handoff-loop.svg" alt="Grok Bot 与 Mac 执行方之间只传结果文件的交接流程" width="480" caption="任务下发、写结果文件、`dsh-wait` 打印三行，回到 bot 的就只有 `ok=` 和 `summary=`。" >}}

## 先约定结果文件

字段顺序固定：`ok`、`summary`、`task`、`finished_at`、`changed`、`next`、`detail`。失败也要写，`ok` 放 false，`summary` 说清楚卡在哪。密钥不进这个文件。

dsh 的结果写在工作区的 `.dsh-results/<task-id>.json`。我一开始想写 `~/.dsh-results`，试了才发现沙箱写不出工作区以外，这个念头当场作废。Claude Code 写在 `.claude-results/<task-id>.json`，整个目录丢进 `.gitignore`。两边都先写临时文件再 `mv` 到正式名字，免得协调方读到写了一半的 JSON。

有了文件，下一个念头自然是让 bot 等着。

## 我以为后台跑完会叫醒我

本机放了个 `dsh-wait`，每 15 秒看一眼结果文件在不在，出现了就只打印 `ok=`、`summary=`、`file=` 三行，超时退出。想法很顺：bot 把脚本丢到后台，结束本轮，Mac 跑完自然会把它叫醒。我信这个方案，是因为云端的后台命令确实会回调，我把同一个印象直接搬到了本机 Shell 上，没去验证 Mac 这边有没有这种机制。

不会。Mac 上的后台 Shell 跑完不叫醒 bot，只有云端自己的后台命令才有那个回调。这一点我踩过两次。竞品调研那次，结果文件早就躺在那了，bot 一声不响睡到我主动去问；试玩测试又来一遍。任务没丢，bot 半天没汇报，省下来的那点 token 早就亏回去了。

所以流程按这台机器的规则写：本机 Shell 跑完不回调，任何一步都不许等回调。

## 不结束本轮，挂住等

改法是别结束本轮。`dsh-wait` 启动之后，bot 用 AwaitShell 原地等，匹配 `^ok=` 这一行，一轮大约 13 分钟，没等到就再等一轮，等到了只报 `ok` 和 `summary`。等待期间没有新东西进上下文。

接着撞上第二个限制，Grok Bot 单轮大约 50 分钟会被系统掐掉。任务长一点，bot 就在等待里被杀，那一轮等的 13 分钟白等。

## 四十分钟交一次班

绕过去靠接龙。等到差不多 40 分钟还没结果，让 bot 给自己定一个 5 分钟后触发的一次性提醒，安静结束本轮；闹钟响了先删掉提醒，再接着 `dsh-wait`。40 分钟一次微唤醒，离 50 分钟硬限还差一截，代价只是每轮多醒一次。

顺带一个参数：`dsh-wait` 默认超时 30 分钟，对长任务太短，后来长活一律设 2 小时。

## 最贵的用法

中间有段时间，别的 bot 走了另一条路，自己 `nohup claude -p`，每半小时去读一次进度文件，定时把进度和截图发出来。看着挺勤快，其实是最费 token 的用法，每隔半小时就把一份大文件重新拉进上下文。

后来做了一次审计，同一套技能下，样板 bot（006）整条链路按规矩走，另一只（005）几乎每个环节都错，nohup 轮询进度文件就是它。规则于是写成硬话：禁止 nohup 之后轮询进度文件，每个任务只起一个 `dsh-wait`，禁止定时读进度。

## 只读 ok 和 summary

上下文纪律最后收成一句话，`detail` 永远留在磁盘上。bot 之间交接、向人汇报、执行方回传，都只带 `ok`、`summary` 和结果文件路径，`changed` 短的话可以捎上。要细节就用 `plutil` 加 `rg` 从 JSON 里抽那一段，不许 `cat` 整份文件，更不许把整份日志贴进对话。

`summary` 后来加了 200 字硬上限。起因是有个任务往里塞了五百多字，读完发现那就是一份报告，进上下文的量和粘贴日志差不多。200 字大概是既能说清做没做成、又不会让人滚屏的长度，多出来的内容一律落到 `detail`。

## Notion 那本账

活多了得有个地方看进度。开工前在 Notion 建一行或复用一行，状态设进行中，带上来源（哪个 bot）、编号（项目号）和完整 prompt。成功改成已完成，摘要进备注；失败或超时，状态留着不动，备注写失败原因。失败不改成已完成，是为了下次翻表还能看见它没做完，写成完成等于把它埋了。

两个坑。来源和编号是 select 列，填表里没有的选项会直接写不进去，得先改表结构。还有一列叫 note，那是我自己的进度日志，bot 一个字都不能碰，它只能用备注。

## 权限开小一点

Claude Code 默认给 `--permission-mode acceptEdits`，能改文件，不能跑任意命令。要跑构建、测试、git、ego-browser 了，再上 `--allowedTools` 的最小白名单。`--dangerously-skip-permissions` 和 bypassPermissions 除非我在对话里明说，否则不用。

启动时把完整命令行写进 `.claude-results/<task-id>.log` 第一行，权限模式和 allowedTools 都在里面。事后想查当时开了哪些权限，翻这一行就够。prompt 另外存成 `.prompt.md`，免得命令行长得吓人。

## 浏览器只认 ego lite

本机的浏览器测试固定在 ego lite，应用在 `/Applications/ego lite.app`，命令行是 `~/.local/bin/ego-browser`。prompt 里写死步骤：先 `ego-browser skill` 读用法，会话名用 task id，优先连本地预览，做完关页。Playwright、Puppeteer、无头 Chromium 都不许用，失败了也不许退回去用。Claude 那边给 `Bash(ego-browser:*)` 一条就够。

## 连不上本机的时候

翻墙不稳的时候 bot 连不上 Mac。有只 bot 自作主张设成每 15 分钟查一次，到晚上能醒几十次，多数是白醒，纯烧额度。现在约定，连不上就定时查，最多 2 小时一次；人等不及自己戳一下；连上了立刻回到每个任务一个 `dsh-wait`。

## 两个技能

这些规矩散在对话里会走样，最后落成两个技能文件：`dsh result json` 和 `claude result json`。后者开头写明结果文件、等待、Notion 都照前一份来，只有启动那段不同。

两个文件里最常被复制出去的是三段模板。结果文件那段，每个任务 prompt 末尾都贴一遍：

```text
When done, write `.dsh-results/<task-id>.json` at the
workspace root with exactly these top-level keys, in this
order:
- `ok`: true if the task fully succeeded, false otherwise
- `summary`: one sentence, hard limit 200 characters, what
happened or why it failed
- `task`: the task id
- `finished_at`: ISO 8601 time with offset
- `changed`: list of files, commits, or resources changed
- `next`: what a human or caller should do next, or empty
- `detail`: the full output, logs, diffs, and reasoning
Write it to a temp file in `.dsh-results/` first, then `mv`
it into place. Write the file even on failure. Never put
secrets, tokens, or keys in it.
```

交给 Claude Code 时，路径换成 `.claude-results/<task-id>.json`。

任务碰到浏览器，再贴一段限制：

```text
Browser testing: use only ego lite through the `ego-browser`
CLI.
First run `ego-browser skill` and follow the guide it
prints.
Do not install, import, or launch Playwright, Puppeteer,
headless Chromium, or Chrome.
Use one TaskSpace named after the task id, open only the
URLs the task needs, close the pages you opened and call
`finish()` at the end.
If ego-browser fails, report ok false with the error; do not
fall back to another browser.
```

Claude Code 的启动命令，`$FLAGS` 里是 `--permission-mode acceptEdits` 和这次的工具白名单，先把整串写进 log 第一行，事后查它当时拿到了什么权限：

```bash
{
  echo "# cmd: claude -p <prompt.md> $FLAGS"
  claude -p "$(cat <prompt.md>)" $FLAGS
} > .claude-results/<task-id>.log 2>&1
```

## 现在的约定

短版本，如果还要这样接：

- `dsh result json` 和 `claude result json` 两个技能，所有 bot 共用
- 结果文件就七个字段，`summary` 不超过 200 字
- 等结果用 `~/.local/bin/dsh-wait <结果文件> [间隔秒=15] [超时秒]` 配 AwaitShell，40 分钟交一次班
- 读结果先看 `ok` 和 `summary`，`detail` 按需切片
- Notion 进行中，成功改已完成，失败留在进行中
- Claude 用 acceptEdits 加最小 allowedTools，命令行进日志首行
- 浏览器只认 ego lite
- 不做的四件事：结束本轮空等 Mac 叫醒，nohup 后轮询进度，把 detail 贴进聊天，15 分钟空转检查

## 后来我把重活收了回来

这套东西能跑，但它有成本。省下的是上下文，省不掉的是编排方自己的额度和稳定性：每 40 分钟接一次龙，连不上要退避，结果回来还得判读，全都压在那个 bot 上。做到后面我算了一笔很朴素的账，写代码、跑测试这些活，我在 dsh 里自己动手，上下文在我眼前，看得见也改得快；让云端 bot 隔着云端和本机两头指挥，光对齐状态就要来回好几轮。

于是分工改了。写代码、跑 dsh 和 Claude Code，我自己来；Grok Bot 以后主要做调研和 SEO 推广。那类活它确实比我划算，信息散在网页上，它能并着翻，还不用我在旁边盯着。

结果文件那套东西我没删，还留着。两个技能文档也留着，`dsh result json` 和 `claude result json`，谁要接就照着做。真正贵的地方在于把大结果反复读进编排方的上下文，结果文件加一句 200 字的 summary，是在编排方和执行方之间立的一道闸。闸还在，只是现在过闸的活儿少了。

