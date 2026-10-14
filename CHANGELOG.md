# Changelog
## 1.2.1 (2026-10-14)

### Fixed（install.sh 两处 + 发布闸门，与 qwiki-skill 同款）

- **`--target hermes,claude`（空格形式）从未生效**：解析只认 `--target=`，空格形式把值吞成空串后落进"非交互 = 装全部"分支——而 README / 用法行 / 脚本头部**三处文档全写的空格形式**。现两种形式都支持，且 `--target` 缺参数时直接报错退出（不再静默装全部）。
- **备份落在 skills 扫描根之内**：旧版备份写成 `<skills>/<name>.bak.<时间戳>`，而 agent 按「含 SKILL.md 的目录」发现技能、不看目录名 ⇒ 备份被当成**第二个同名技能**加载（重名歧义下静默跳过其一）。现备份落 `<tool 根>/skills-backup/`（扫描根之外），README 与脚本尾注同步。
- **挂上 pre-push 闸门（脱敏 + CHANGELOG 同步）**：`scripts/publish-check.py`（扫项目代号 / 内网 IP / MAC / 长 hash / 凭据 / 本机路径，自带 `--self-test`）+ `scripts/hooks/pre-push`；启用一次 `git config core.hooksPath scripts/hooks`。理由与脱敏规则同源：**靠人记必然回潮**，检查要做成"推送时自动拦"而不是"记得手动跑"。CHANGELOG 闸门名单 = SKILL.md / install.sh / scripts/ / references/。
- 版本 1.2.0 → 1.2.1

# CHANGELOG

## 1.2.0 (2026-09-30)

### Added（manifest 切换全流程 + 09-30 实测坑）

- **description/triggers 扩到 manifest 切换场景**：`repo init -m X` 换 manifest、整树 sync、切回常规分支、`repo start --all`、`index.lock`
- **新增 Step 0c 切 manifest**：任务期冻结 manifest ↔ 常规 manifest 的整树动作流程（备份 `.repo/manifest.xml` → `repo init -u/-b/-m` 显式带全 → 切前清 dirty + 逐仓记本地领先 commit → `repo sync -c -j8` 后台跑、**不加 `-d`**）+ 四条验收（repo 收尾语判据 / 数项目别数 stub / Change-Id 回查 / 关键仓 status）；**manifest 归属以 `project.conf` 为准**——同名项目 revision 可能一致、项目数却差很多（如 `manifest-smart.xml` 150 vs `default.xml` 54），切到精简集会让几十个项目退出 manifest 管理（目录不自动删，要 `repo prune`）
- **新增 Step 0d 切完 manifest 还要切分支**：`repo start --all <常规分支>` 语义（分支已存在 → 只 `git checkout --`、**不动指针**；不存在 → 从 manifest revision 新建）与三点验收；与 Step 3「冲掉本地 commit 时勿 repo start」的场景区分
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


## v1.1.0 (2026-08-18)

- **ZCode 安装目标**：install.sh 支持 ZCode（探测 `~/.zcode` → 安装到 `~/.zcode/skills/repo-cloud-alignment`），README 补手动复制 ZCode 路径，发布页一键命令即可装到 ZCode

## v1.0.0 (2026-08-18)

- 初版：repo 多仓强制拉齐云端（丢弃本地改动）+ 精确复现 Jenkins 构建（增量对齐）
- 全仓覆盖原则：repo status 首屏截断 / detached 仓带本地 commit（SDK 仓漏拉 → 编译 undefined reference）
- 云端基线确认：逐个 fetch 防 MERGED 误判 LOCAL_ONLY；gerrit query 状态核验
- 删除重建分支（repo start 同名旧分支复活坑）
- 增量对齐构建基线（08-18 新增）：Jenkins artifact 下载锁 revision manifest → 对比 HEAD 只 checkout 差异仓 → changeid 链 cherry-pick；repo download 拒 MERGED change 须手动 cherry-pick；FETCH_HEAD 覆盖坑
- 交叉仓 undefined reference 诊断链（IMP_Buf_Vaddr2Paddr 案例）
- 来源：2026-08-17 SDK 交叉仓对齐 + 2026-08-18 构建 #549 增量对齐实证

## 1.1.1 (2026-08-28)

### Added

- install.sh 新增 pi 安装目标（探测 `~/.pi/agent` → 安装到 `~/.pi/agent/skills/<skill>`，pi 自动发现）
- 版本 1.1.0 → 1.1.1
