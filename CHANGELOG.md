# Changelog

## 1.2.4 (2026-10-15)

### Fixed（发布件脱敏收尾 · 第二轮，发布闸门 rc=0）

- **1.2.3 自述「脱敏收尾」但仍有漏网**（教训：**「已脱敏」的提交自述不是证据，只认闸门 rc + 全仓 grep 实跑**）：
  - `SKILL.md` 分支示例里的任务期分支名，代号前缀已占位、**迭代号后缀仍是真值** ⇒ 改为 `<项目A>_<迭代号>`
  - `SKILL.md` 坑表里的**内部构建配置脚本名**（管 `CONFIG_PROJECT_*` 开关那只）仍在 ⇒ 改为「构建配置脚本 `<构建配置脚本>`」
- **references 案例里的厂商 SDK 标识一并占位化**（同批自查发现，不在闸门判定名单内）：厂商内部构建号 → `<厂商> SDK <构建号>`；内部 SDK 版本路径 → `<SDK 版本路径>`（4 处，含 SKILL.md 的 SDK 路径真相源表），保留纯版本号做对照以维持「别查错 SDK」的可读性
- **CHANGELOG 旧节名未随 SKILL 重编号同步**：1.2.0 段的 `Step 0c` / `Step 0d` 加注现名（`Step 0-A` / `Step 0-B`），消掉「引用的节名在正文找不到」的歧义
- 版本 1.2.3 → 1.2.4

## 1.2.3 (2026-10-15)

### Fixed（发布件脱敏收尾 + 文档修复）

- **正文与 references 的项目代号 / 厂商 SDK 名 / 同事姓名 / 内部 change 号全部占位化**（gitee 实测本仓 public=True）：manifest 名、编译脚本名、SDK 路径表里的真实代号 → `<项目A>` / `<项目B>` / `<项目C>`；案例里的提交人姓名 → `<提交人>`；内部 change 号 → `<change 号>`
- **本条 1.2.2 条目自身含真代号**（大写写法与「代号紧贴汉字」示例）→ 改写为不含真名的等价表述（**纪律：CHANGELOG 条目与 commit message 里同样不得写真实标识**）
- **SKILL.md 结构错位**：Step 0b/0c/0d 原挂在《核心原则》之下 → 移入《工作流》的 Step 1 之前，重编号 Step 0 / 0-A / 0-B，同步文内交叉引用
- **死链技能名**：参考段的 gerrit-workflow / embedded-build-artifact-verification 在本机只存在于归档 ⇒ 改为自包含表述（gerrit 规则已内嵌于 gerrit-patch-flow；编译产物验证指向本仓章节）；README 同步删掉死链行
- **triggers 补「复现某 Jenkins 构建 / 增量对齐」场景**（README 早已把它列为入口）
- **README 补「仓库结构」块**（其余仓都有）
- **CHANGELOG 结构**：删重复 H1；1.1.1 段上移到 1.1.2 之后；`## v1.1.0` / `## v1.0.0` 去 `v` 前缀
- **占位符统一**：`<项目>` → `<项目A>`（一套占位符）
- 版本 1.2.2 → 1.2.3

## 1.2.2 (2026-10-15)

### Fixed
- **发布闸门 publish-check.py 三个洞同批修**（脱敏反复回潮的根因之一：闸门对真实标识零防护却一直报「干净」）：
  ① 代号正则大小写不敏感（旧版只认小写写法，大写写法整体漏）；
  ② 白名单改为**按匹配项豁免**（旧版 `if any(a in line for a in ALLOW): continue` 是整行豁免——一行里出现
  `<项目A>` 就给同行的真名背书；且原 7 个词对 12 条 pattern 全部零命中 = 纯死豁免）；
  ③ 词边界改用显式字符类（`\b` 在 str 模式下 `\w` 含汉字 ⇒「代号紧贴汉字」这类中文常见写法漏检，
  加不加空格决定报不报）。另补 PRJ0## / 裸数字代号 / MAC 连字符 / 凭据变体（`token=`、`Bearer`、中文「密码」加冒号写法）/
  邮箱 六类形态。
- **`--self-test` 样本按已知缺陷类别扩容**（原 15 条只覆盖「正常裸值」一档，上述六类一条样本都没有）；
  自检与扫描共用同一份判定口径（`line_hits`），不再各写一套造成假绿。
- **`scripts/hooks/pre-push` 在推送路径上先跑 `--self-test`**（规则表被改坏时不得静默通过）；
  CHANGELOG 名单统一为超集（SKILL / SCHEMA / install.sh / templates / references / scripts，9 仓文本自此一致）；
  基点改为 `origin/main → gitee/main → 根提交` 三级回落（旧版无 `origin/main` 时回落到 `HEAD~1`，
  多笔提交一起推会只看最后一笔）。
- **新增本机私用名单 `scripts/publish-deny.local`**（gitignore、不进仓）：人名 / 昵称 / 厂商 SDK 名 /
  内部 skill 名这类**无法正则枚举、也不该写进发布仓**的标识，在本机未跟踪文件里逐行列真名，闸门自动一并扫。

### 说明
- 口径全文见 `scripts/publish-check.py` 文件头（已知边界：长 hash 折行、全角句点 IP 不识别；该文件自身在
  `SKIP_FILES` 内，改它时需人工核对）。9 仓共用同一份闸门（md5 一致）——改一处必须批量同步。
- 版本 1.2.1 → 1.2.2

## 1.2.1 (2026-10-14)

### Fixed（install.sh 两处 + 发布闸门，与 qwiki-skill 同款）

- **`--target hermes,claude`（空格形式）从未生效**：解析只认 `--target=`，空格形式把值吞成空串后落进"非交互 = 装全部"分支——而 README / 用法行 / 脚本头部**三处文档全写的空格形式**。现两种形式都支持，且 `--target` 缺参数时直接报错退出（不再静默装全部）。
- **备份落在 skills 扫描根之内**：旧版备份写成 `<skills>/<name>.bak.<时间戳>`，而 agent 按「含 SKILL.md 的目录」发现技能、不看目录名 ⇒ 备份被当成**第二个同名技能**加载（重名歧义下静默跳过其一）。现备份落 `<tool 根>/skills-backup/`（扫描根之外），README 与脚本尾注同步。
- **挂上 pre-push 闸门（脱敏 + CHANGELOG 同步）**：`scripts/publish-check.py`（扫项目代号 / 内网 IP / MAC / 长 hash / 凭据 / 本机路径，自带 `--self-test`）+ `scripts/hooks/pre-push`；启用一次 `git config core.hooksPath scripts/hooks`。理由与脱敏规则同源：**靠人记必然回潮**，检查要做成"推送时自动拦"而不是"记得手动跑"。CHANGELOG 闸门名单 = SKILL.md / install.sh / scripts/ / references/。
- **脱敏：正文里的项目代号与示例内网 IP 换占位符**——`scripts/publish-check.py` 挂上后首扫命中，逐条判定（另 4 处『凭据赋值』为误报：都是串口提示符 `Password:` 字面量与说明，无真实密钥）后替换：正文 / references 里的项目代号 → `<项目A>`（多处依次 A/B/C/D），模板示例里的内网 IP → 文档保留段 `192.0.2.x`。这些标识已随历史提交公开过，本次只保证**往后的版本不再含**；历史不回改——force push 也收不回（已 clone/fork/缓存的副本），脱敏的真实收益是往后不再泄漏。
- 版本 1.2.0 → 1.2.1

## 1.2.0 (2026-09-30)

### Added（manifest 切换全流程 + 09-30 实测坑）

- **description/triggers 扩到 manifest 切换场景**：`repo init -m X` 换 manifest、整树 sync、切回常规分支、`repo start --all`、`index.lock`
- **新增 Step 0c 切 manifest**：任务期冻结 manifest ↔ 常规 manifest 的整树动作流程（备份 `.repo/manifest.xml` → `repo init -u/-b/-m` 显式带全 → 切前清 dirty + 逐仓记本地领先 commit → `repo sync -c -j8` 后台跑、**不加 `-d`**）+ 四条验收（repo 收尾语判据 / 数项目别数 stub / Change-Id 回查 / 关键仓 status）；**manifest 归属以 `project.conf` 为准**——同名项目 revision 可能一致、项目数却差很多（如 `manifest-smart.xml` 150 vs `default.xml` 54），切到精简集会让几十个项目退出 manifest 管理（目录不自动删，要 `repo prune`）（节名后重编号为 Step 0-A）
- **新增 Step 0d 切完 manifest 还要切分支**：`repo start --all <常规分支>` 语义（分支已存在 → 只 `git checkout --`、**不动指针**；不存在 → 从 manifest revision 新建）与三点验收；与 Step 3「冲掉本地 commit 时勿 repo start」的场景区分（节名后重编号为 Step 0-B）
- **坑表新增六条（09-30 实测）**：单个 dirty 文件挡住整仓 checkout（先判是否脚本可再生的 `CONFIG_PROJECT_*`）/ `index.lock` 僵尸锁判定与清理（`fuser`/`lsof` 无持有者才 rm）/ 大仓 checkout 须 `background + notify` / **`repo forall` 漏掉 checkout 失败的仓**（拿它数「落后项目数」得假 0）/ `.repo/manifest.xml` 是 `<include>` stub（`grep -c '<project'` 得 0 属正常，数 `.repo/manifests/<X.xml>`）/ 整树 sync 被单次工具调用上限中断后呈半同步（幂等，直接重跑，勿从零重建）
- **全树分支分布秒级查法**：直读 `.repo/projects/*/HEAD`（**必须 `-not -path '*/logs/*'`**，否则 reflog 混入导致分支计数失真）
- 版本 1.1.3 → 1.2.0

## 1.1.3 (2026-09-29)

### Fixed

- 修正「`repo sync` 不删本地 commit」的表述：全仓 sync 会把跟踪**移动分支**（如 `m/smart`）的仓 checkout 走，本地提交从分支引用上被摘掉、只能靠 `git reflog` 找回（09-29 三十余仓事故实证：rootfs 8 / factory_test 2 / c_mi_ipc 1 / toolchain 3 / config 1 笔）
- 新增纪律：要更新单仓改用该仓 `git fetch`，不要跑全仓 `repo sync`；抢救用 `git branch <name> <sha>` 而非 `reset --hard`
- 版本 1.1.2 → 1.1.3


## 1.1.2 (2026-09-28)

### Changed

- 回灌本机 Hermes 副本的实测修正：意图取舍（冲掉 vs 保留，默认冲掉）+ 只动点名仓 + 新增 references/push-readiness-and-change-id-repair.md
- 版本 1.1.1 → 1.1.2


## 1.1.1 (2026-08-28)

### Added

- install.sh 新增 pi 安装目标（探测 `~/.pi/agent` → 安装到 `~/.pi/agent/skills/<skill>`，pi 自动发现）
- 版本 1.1.0 → 1.1.1

## 1.1.0 (2026-08-18)

- **ZCode 安装目标**：install.sh 支持 ZCode（探测 `~/.zcode` → 安装到 `~/.zcode/skills/repo-cloud-alignment`），README 补手动复制 ZCode 路径，发布页一键命令即可装到 ZCode

## 1.0.0 (2026-08-18)

- 初版：repo 多仓强制拉齐云端（丢弃本地改动）+ 精确复现 Jenkins 构建（增量对齐）
- 全仓覆盖原则：repo status 首屏截断 / detached 仓带本地 commit（SDK 仓漏拉 → 编译 undefined reference）
- 云端基线确认：逐个 fetch 防 MERGED 误判 LOCAL_ONLY；gerrit query 状态核验
- 删除重建分支（repo start 同名旧分支复活坑）
- 增量对齐构建基线（08-18 新增）：Jenkins artifact 下载锁 revision manifest → 对比 HEAD 只 checkout 差异仓 → changeid 链 cherry-pick；repo download 拒 MERGED change 须手动 cherry-pick；FETCH_HEAD 覆盖坑
- 交叉仓 undefined reference 诊断链（IMP_Buf_Vaddr2Paddr 案例）
- 来源：2026-08-17 SDK 交叉仓对齐 + 2026-08-18 构建 #549 增量对齐实证
