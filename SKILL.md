---
name: repo-cloud-alignment
description: repo 多仓项目云端对齐与 manifest 切换（-m）— 全仓对齐、分支重建、整树 sync 坑、交叉仓依赖诊断
version: 1.2.0
category: devops
metadata:
  agent:
    triggers: [拉齐云端, 强制拉齐, 丢弃本地改动, 以云端为准, 对齐主干, 主干最新, 删代码重拉, 全仓对齐, 切manifest, 换manifest, manifest回退, repo init -m, 整树sync, 切回常规分支, repo start --all, 分支没切, index.lock]
---

# repo 多仓项目云端对齐

场景：repo 管理的嵌入式多仓工程（如 hm6502_wifi），本地有 commit/改动，目标 = 强制对齐云端主干并丢弃全部本地改动；或需在「主干最新」与「manifest 锁定基线」间抉择。

## 核心原则

1. **全仓覆盖**：对齐必须扫全部项目，不能只看有分支的仓。repo status 首屏会截断；detached/NO BRANCH 仓也能带本地 commit（08-17 实锤：third_party/rootfs 带 4 个本地 commit 未出现在首屏）。
   ⚠️ **`repo forall` 自身也会漏项**：它跳过 checkout 失败（未成功初始化）的项目 —— 整树 sync 报过 `Failing repos: <x>` 之后，拿 `repo forall -c 'git status -sb'` 去数「落后项目数」会得到**假 0**（实测：报 0 而实际该仓 behind 1）。全树状态以 repo 自己的收尾语为准（`repo sync has finished successfully.` ↔ `Unable to fully sync the tree … Failing repos: …`），个别仓一律 `git -C <repo> status -sb` 直查。
2. **SDK 仓最易漏 = 编译必炸**：交叉仓 API 依赖——应用仓引用新 API，提供方在 SDK 仓（third_party/rootfs）的新 commit。漏拉 SDK 仓 → 编译 undefined reference（案例：IMP_Buf_Vaddr2Paddr）。
3. **编译是最终门禁**：git 层面 HEAD 对齐 ≠ 能编译。拉齐后必须全量编译验证。
4. **manifest 决定目标**：锁 revision 的 manifest（如 HM6502-Master-Build-SI-401）= 构建基线（旧）；manifest-smart.xml 不锁 revision = 跟 m/smart 主干最新。要「主干最新」必须换 manifest，删代码重拉（rm -rf + repo init -m manifest-smart.xml + repo sync）最干净，逐仓 git reset 会留下 SDK 漂移。
5. **先钉死意图：保留还是冲掉**：用户说「同步到服务器最新代码」= 冲掉本地改动（默认期待），终态应只剩他指名保留/待推送的那一笔。不要默认保守保留 —— 保留会被判成「同步没做干净」，且让他后续推送一直被历史 commit 连累（见《意图取舍》）。
6. **只动用户点名的仓，且用证据回答「有没有碰过别的」**：同机常并存多个同源工作区（主库 + 专项工作区），用户会明确排除其中一个。写操作只落在点名的那个；被他排除的那个事后被追问时不要口头保证，用实测说话 —— 该工作区 `.repo/project.list` 的 mtime、各仓 `git reflog -1` 时间戳、`pgrep -af 'repo sync|git fetch'` 有无残留，三条一对即可自证。顺带：被排除工作区里冒出的「今日新文件」若在 `out/`、`0_workbench/` 这类非 git 目录，多半是用户自己的并行活动，别认领。

### Step 0b: 精确复现某 Jenkins 构建（增量对齐，08-18 #549 验证）

目标 = 复现某构建号的确切代码（基线 + changeid 链），非主干最新。已有 repo 工作树时无需从零建仓：

```bash
# 1. 基线真相源 = 构建 artifact 的锁 revision manifest（构建参数 manifest 为空时尤甚）：
curl -u <user>:<pass> "http://<jenkins>/job/<Job>/<n>/artifact/<Job>-<n>_manifest.xml" -o /tmp/jb_manifest.xml
# 2. 解析 manifest 对比各仓 HEAD，只 checkout 差异仓（#549: 145 仓仅 2 仓差异）：
python3 - <<'EOF'
import re, subprocess
man = open('/tmp/jb_manifest.xml').read()
for m in re.finditer(r'<project name="([^"]+)" path="([^"]+)" revision="([^"]+)"', man):
    path, rev = m.group(2), m.group(3)
    head = subprocess.run(['git','-C',path,'rev-parse','HEAD'], capture_output=True, text=True).stdout.strip()
    if head != rev: print(f"{path}: local={head[:10]} base={rev[:10]}")
EOF
# 3. 差异仓 git fetch + checkout 到基线 revision（detached 即可）
# 4. changeid 链逐个 repo download -c（MERGED 的会被拒，见坑速查"repo download 拒 MERGED"）
# 5. 内容级验证：同父链 git diff <gerrit PS> <本地> 应为空；parent 不同用 git show --stat 对比文件集+行数+Change-Id
```

注意：此方式不改 .repo/manifest.xml 声明（仍指向原 manifest）。工作树=构建精确代码，但任何 repo sync 会漂回声明的 manifest 基线并丢 cherry-pick。要焊死基线：把 jb_manifest.xml 拷入 .repo/manifests/ 并改 include（副作用：sync 清 cherry-pick 需重打）。

### Step 0c: 切 manifest（换 `-m`：任务期冻结 manifest → 常规 manifest）

任务期常用 SI 冻结 manifest（逐项目钉死 revision），任务完成后切回常规 manifest（如 `manifest-smart.xml`，不锁 revision）。整树动作，**先出声再动手**（会把全树从旧 revision 搬到新 tip）。

```bash
cd <项目树>
cp -a .repo/manifest.xml <备份目录>/manifest.xml.<旧manifest名>.bak   # 留底
repo init -u ssh://<gerrit>/manifest -b smart -m manifest-smart.xml  # -u/-b 显式带上，别依赖 repo 的记忆
# 切前：清掉会挡 checkout 的 dirty（见坑速查），并记下本地领先 commit
repo forall -c 'git log --oneline origin/m/smart..HEAD'              # 逐仓记账：hash + Change-Id
repo sync -c -j8        # background + notify 跑；**不加 -d**（只有 -d 会丢本地 commit）
```

验收（按序）：
1. **repo 收尾语**：`repo sync has finished successfully.` = 全 checkout 成功；`Unable to fully sync the tree … Failing repos: <x>` = 只差那几仓，其余已同步完，别整个重跑
2. `.repo/manifests/<X.xml>` 或 `repo manifest | grep -c '<project'` 数项目数（**不数 `.repo/manifest.xml`**，它是 stub）
3. 每个本地领先 commit 逐个回查 Change-Id 还在（不带 `-d` 时 repo 对本地分支走 rebase：commit 保留、**hash 会变**）
4. `git -C <关键仓> status -sb` 对齐、工作区干净

**别依赖 rebase 保命**：带本地 commit 的仓在切 manifest 前先 `git format-patch`/bundle 备份 —— rebase 保留是实测行为，不是可依赖的保证。

**manifest 归属以 project.conf 为准，别按名字猜**：同一棵树常并存多份 manifest（`manifest-smart.xml` 全集 vs `default.xml` 精简集），同名项目 revision 可能一致、项目数却差很多（如 150 vs 54）⇒ 切之前数一遍项目数、跟用户点名的 intent 对上再执行；`default.xml` 这类精简集切过去会让**几十个项目退出 manifest 管理**（目录不自动删，要 `repo prune` 才清）。

### Step 0d: 切完 manifest 还要切分支（`repo start --all`）

**切 manifest 不会改各仓所在分支**：sync 只把当前分支 rebase 到新 manifest revision，全树仍停在任务期分支（如 `hm6503_p348`）上，看着像「切好了」，其实常规工作分支（`project.conf` 记的那个，如 `hm6503`）还在原处、落后 m/smart。用户会拿「分支切了吗」验收，别只报 sync 成功。

```bash
repo start --all <常规分支>      # 整树切到常规工作分支
```

语义（repo v20241130 `project.py` `StartBranch`，读源码核过）：HEAD 已在该分支 → 直接返回；**分支已存在 → 只做 `git checkout -q <name> --`，不动分支指针**（已有分支安全，不会被重置）；分支不存在 → 从 manifest revision 新建（detached / NO BRANCH 的仓会被带上并建分支）。

⚠️ 与 Step 3 的「勿 repo start」不冲突：Step 3 说的是**要冲掉本地 commit** 时 repo start 会检出同名旧分支、让回退白做；这里是要**回到常规工作分支**，本就保留分支原状，正是 repo start 的用途。

验收三点：① 全树分支分布只剩 `<常规分支>`（用 Step 1 的直读法）；② 任务分支上那笔本地 commit 还在且**没被带走**：`git -C <repo> branch --contains <sha>` 只命中任务分支；③ 切完树**不再与 m/smart 齐平**（落在常规分支位置）——要追平再跑一次 `repo sync`（不带 `-d`，走 rebase 前进）。

## 意图取舍：冲掉本地改动 vs 保留（动手前先钉死）

用户的默认期待是**冲掉**：终态 = 工作树是云端主干最新，本地只剩他明确要保留的那一笔。这里最贵的错误是「以为自己在保护用户工作」而默默保留 —— 用户会认为同步没生效，且他后续推送上不去时会被历史 commit 连累。

机制（必须向用户讲清，否则他会以为同步无效）：

- **`repo sync` 不删本地 commit，但会「切走」**（2026-09-29 三十余仓事故修正本行旧说法）：清单里跟踪**移动分支**（如 `m/smart`）的仓，全仓 sync 会直接 checkout 到远端、把本地提交从分支引用上摘掉——commit 对象还在（`git reflog` 可逐仓找回），但工作树看着就像「丢了」。**纪律：不要对这些仓跑全仓 `repo sync`；要更新单仓就对该仓 `git fetch`。** 抢救法：`git reflog` 定位 commit → `git branch <name> <sha>` 指回去（`reset --hard` 可能被权限拦、且会丢当前态）。
- 所以 `git checkout --detach` 只是绕过保护让 sync 能跑；真冲掉要显式做：
  ```bash
  git branch -D <本地分支>          # 单仓冲掉（或 git update-ref 重置到 origin/m/smart）
  repo sync --force-sync           # 整仓强冲：连 .git 一起重建，最彻底
  ```
- 冲掉**前**必备份（bundle + 逐 commit patch + 全仓 HEAD 快照），冲掉后给出回滚路径。
- 终态验收：各仓 `git log --oneline origin/m/smart..HEAD` 只剩用户指名保留的那几笔，其余为空。

若用户要「冲掉其余、只留最新一笔」：删分支 → 基于 `origin/m/smart` 新建分支 → cherry-pick 那一笔 → 本地唯一待推 = 1 笔，不会被历史 commit 连坐。

## 工作流

### Step 1: 现状扫描（只读）
```bash
repo status 2>/dev/null | head -60   # 首屏会截断，仅参考
# 全量本地 commit 扫描（关键！覆盖所有仓）：
repo forall -c 'echo "== $REPO_PROJECT"; git log --oneline origin/m/smart..HEAD 2>/dev/null | head -5'
# 分支 vs detached 状态：
repo forall -c 'echo "== $REPO_PROJECT"; git branch -vv | head -2'
# 全树分支分布 / detached 计数 —— 秒级（不逐仓起 git；repo forall 在大仓 + HDD 上要几分钟）：
find .repo/projects -name HEAD -not -path '*/logs/*' | while read -r f; do
  p=${f#./.repo/projects/}; p=${p%.git/HEAD}; read -r a b < "$f"
  if [ "$a" = "ref:" ]; then echo "${b#refs/heads/}|$p"; else echo "DETACHED|$p"; fi
done | awk -F'|' '{print $1}' | sort | uniq -c | sort -rn
```

⚠️ 上面这条必须带 `-not -path '*/logs/*'`：`-name HEAD` 会把每个仓的 `logs/HEAD`（reflog）一起捞进来，输出混入 `<sha> <author> <email> <ts>` 垃圾行、分支计数直接失真。

⚠️ 全树 repo 命令（sync / status / forall）在**大仓 + HDD 上每条都是分钟级**：一次工具调用只跑一条，别串两条（会撞单次调用上限被硬中断，输出全丢）。

### Step 2: 云端基线确认
```bash
# manifest 锁定的 revision（当前工作树应达成的状态）：
grep -E 'kernel/linux|c_mi_ipc|third_party/rootfs' .repo/manifests/<当前manifest>.xml
# 每个有本地 commit 的仓 fetch 主干（必须逐个 fetch，SDK 仓常被漏）：
cd <repo> && git fetch origin m/smart
# 本地 commit 是否已在云端（Change-Id 比对，fetch 前比对会失真）：
for c in $(git log --format=%H origin/m/smart..HEAD); do
  cid=$(git show -s --format=%B $c | grep '^Change-Id:' | awk '{print $2}')
  found=$(git log origin/m/smart --format=%H --grep="$cid" | head -1)
  [ -n "$found" ] && echo "IN_CLOUD $cid" || echo "LOCAL_ONLY $cid"
done
# ⚠️ 上面的 git log --grep 只覆盖已合入（MERGED）的 commit。判「从未推送过」必须再问 gerrit ——
#    OPEN 状态的 change 在主干历史里查不到，只凭 git log 会把「已推但未合入」误判成「从未推送」。
#    批量体检整条链（一次查询，远快于逐笔 ssh）：
CIDS=$(git log origin/m/smart..HEAD --format='%B' | grep '^Change-Id:' | awk '{print $2}' | sed 's/^/change:/' | paste -sd' OR ' -)
ssh -p 29418 <user>@<gerrit_host> gerrit query --format=JSON "$CIDS"      # 看 stats 行 rowCount
# 查某人/某项目全部 change（含 NEW/MERGED/ABANDONED，用来看他历史上推成功过什么）：
ssh -p 29418 <user>@<gerrit_host> gerrit query --format=JSON 'project:<proj> owner:<邮箱>'
# 注：部分 gerrit 版本没有 --limit 选项（报 fatal: "--limit" is not a valid option）—— 省略即返回全量，本地过滤
# 顺带体检 Change-Id 合法性：一笔非法（如 I+39hex 共 40 字符的手写值）会让整批 push 被拒，
#   把真正想推的那笔连坐：grep '^Change-Id:' | awk '{print $2}' | grep -vE '^I[0-9a-f]{40}$'
# gerrit 状态确认（MERGED=已合入主干 / NEW=open 未合入）：
ssh -p 29418 <user>@<gerrit_host> gerrit query 'change:<Change-Id>' --format=JSON --current-patch-set | grep -E '"number"|"status"'
```

### Step 3: 删除重建分支（冲掉本地 commit 时勿 repo start！）

（「勿 repo start」只针对**冲掉本地 commit** 这一场景；切 manifest 后回常规工作分支的正解就是 `repo start --all <分支>`，见 Step 0d。）
```bash
# ⚠️ repo start <旧分支名> 会检出同名旧分支 = 旧 commit 复活。必须删了重建：
git checkout -q --detach origin/m/smart
git branch -D <分支名>                # 会触发 approval，正常
git checkout -q -b <分支名> origin/m/smart
# 验证：
[ "$(git rev-parse HEAD)" = "$(git rev-parse origin/m/smart)" ] && echo OK
# 全仓无 tracked 改动验证：
repo forall -c 'm=$(git status --porcelain | grep -v "^??" | head -1); [ -n "$m" ] && echo "MODIFIED: $REPO_PROJECT"'
```

### Step 4: 编译门禁
全量编译（build_HM6502_debug.sh 或等价）→ 过不了查交叉仓依赖（下节）。

## 交叉仓 undefined reference 诊断

症状：链接报 `undefined reference to X`，代码和头文件看着都「合理」。
排查顺序：
1. 引用点确认：grep -rn "X" 定位调用处
2. 头文件声明：grep -rn "X" sdk/**/include/ —— 无声明 = API 不在本 SDK 版本
3. **SDK 路径真相源**：查 toolchain/<项目>_cfg.cmake 的 SMART_SDK_LIB_PATH（hm6502=5.4.0_mxu2cve2，Anona=7.2.0，Anona_hm1009=7.2.0_t41_zm_zg——别查错 SDK！）
4. 符号存在性：nm 该路径 libimp.a / nm -D libimp.so
   - 工作区文件：`nm sdk/.../libimp.a | grep X`
   - 云端某 revision：`git show <rev>:sdk/.../libimp.a > /tmp/x.a && nm /tmp/x.a | grep X`
5. **跨仓找提供者**：gerrit query 'message:<API名>'（注意：query message 只搜 commit message，二进制内的符号名搜不到，需要靠人工定位 SDK 更新 commit 或查引用方 change 的配套 commit）
6. 结论：SDK 仓没拉齐 → 回 Step 2 fetch SDK 仓并对齐，再编译

## 坑速查

| 坑 | 现象 | 解法 |
|----|------|------|
| 只对齐有分支的仓 | 编译 undefined reference | 全仓扫描+对齐，SDK 仓必查 |
| repo status 首屏截断 | 漏看带本地 commit 的仓 | repo forall 全量扫 |
| manifest 锁旧 revision | 对齐回旧版本非主干 | 要主干最新用 manifest-smart.xml（不锁 revision），删仓重拉最干净 |
| repo start 同名分支 | 旧 commit 复活，回退白做 | git branch -D 再建 |
| 部分仓没 fetch | 云端比对失真（MERGED 判成 LOCAL_ONLY） | 每个仓都 git fetch origin m/smart |
| 查错 SDK 路径 | 误判「SDK 无此 API」 | cfg.cmake SMART_SDK_LIB_PATH 为准 |
| 云端 Verified+1 但本地失败 | 误以为本地代码问题 | 先查 SDK 仓是否拉齐（云端构建环境含新 SDK） |
| repo download 拒 MERGED change | 报 "has already been merged" 跳过，但基线锁在合入点之前仍缺该改动 | gerrit query --current-patch-set 拿 revision hash 手动 git cherry-pick（#549/213901 案例） |
| 多次 fetch refs/changes 后 FETCH_HEAD 只留最后一次 | 验证时对比错位（210988 对比成 210412） | 逐个 fetch 后立即 git show FETCH_HEAD |
| repo 拒签 `branch X is published (but not merged)` | sync 报 Cannot checkout，卡住有本地 commit 的仓（含 upstream 指向主干的本地分支） | `git checkout --detach origin/m/smart` 让 repo 放行 —— 但这**只是过路**，不是冲掉。分支去留按用户意图定：默认「同步」= 冲掉，之后必须 `git branch -D`（见《意图取舍》）|
| `prior sync failed; rebase still in progress` | sync 报该仓无法 checkout | 先 `git rebase --abort`；rebase 中夹着的 commit 先用 `git format-patch -1 <sha> --stdout` 抢救成 patch 再 abort |
| 把本地 dirty 改动当垃圾清掉 | 同步后编译炸（项目开关/交叉编译补丁被丢） | 清理前逐个 `git show origin/m/smart:<file>` 核主干是否有该改动：主干**没有** = 本机编译必需品，同步后 `git apply` 重打 |
| 单个 dirty 文件挡住整仓 checkout | sync 报 `error: <repo>/: contains uncommitted changes` + `Cannot checkout m/smart/<repo>` + `Failing repos: <repo>`（**其余仓照常走完**，只卡这一格） | 先判该 dirty 是不是**脚本可再生**的产物（典型：`config/set_common_config.sh` 管的 `CONFIG_PROJECT_<项目> := true/false`）——可再生就 `git -C <repo> checkout -- <file>` 放行 sync，下次构建前跑一次脚本即回来；不可再生走《本机适配改动》存 patch 再重打。**别把可再生当不可再生，也别反过来自作主张丢手写改动** |
| `index.lock` 已存在，checkout / 切分支被拒 | `fatal: Unable to create '…/.git/index.lock': File exists` + `error: <repo>/: cannot start <分支>`（**其余仓照常走完**，只卡这一格） | 先判僵尸还是活锁：`fuser -v <lock>` / `lsof <lock>` 无持有者 + 锁 mtime 远早于当前 + `pgrep -af git` 无该仓相关进程 = 崩溃残留 ⇒ `rm -f <repo>/.git/index.lock` 后重跑 `repo start <分支> <repo>`；有持有者就等，别抢 |
| 大仓 checkout 被前台 timeout 掐断 | 单仓切换（device/soc/ 下的大 SDK 仓尤甚）跑几分钟没完，被掐断后 HEAD 仍 detached、看着像失败 | 大仓 + HDD 上 checkout 要重写大量文件，几分钟属正常 ⇒ **单仓操作也走 `background + notify`**；重试前先排除真因：`git -C <repo> ls-files -m`（tracked 改动数，0 = 不是脏）、目录属主（root 写的构建产物会挡）、有无锁 |
| `repo forall` 漏掉 checkout 失败的仓 | 用它数「落后项目数」得 0，实际该仓 behind N | 别拿它下结论：看 repo 收尾语 + `git -C <repo> status -sb` 直查 |
| 切 manifest 后数 `.repo/manifest.xml` 里没有项目 | 误判 `repo init -m X` 失败（其实成功） | `repo init -m X.xml` 会把 `.repo/manifest.xml` 写成 `<include name="X.xml"/>` 的 **stub（~500B）**，不是内容副本 ⇒ `grep -c '<project'` 得 0 属正常。数项目看 `.repo/manifests/<X.xml>` 或 `repo manifest` |
| 整树 sync 被单次工具调用中断 | 树呈**半同步**（部分仓已到新 tip、其余仍在旧 revision），看着像「同步坏了」 | 整树 sync 远超单次前台工具调用上限（会被硬中断，传更大 timeout 无效）⇒ 一律 `background + notify` 跑；被打断后直接重跑（sync 幂等、增量），不要从零重建工作树 |
| cherry-pick 验证用整树 diff | parent 不同带出全部基线差异（135 files 假象） | 同父链才 git diff；否则 git show --stat 对比文件集+行数+Change-Id |

## 本机适配改动：同步前核查 → 同步后重打

本地仓里那 1~N 处 tracked 改动，未必是垃圾。多数是让本机编译出目标项目所必需的项目开关/工具链补丁（如 `config/common_config.mk` 的 `CONFIG_PROJECT_<项目> := true`、`toolchain/CMakeLists.txt` 注释掉编不过的子目录、e2fsprogs/libnl 的 configure 与 shebang 修正）—— 主干长期不带这些。

```bash
# 1) 同步前：逐文件核主干版本（git show，不切工作树）
git -C <repo> show origin/m/smart:<file> | grep -n '<关键字>'
# 2) 存 patch（这一步就是备份）
git -C <repo> diff HEAD > $B/patches/<repo>_dirty.patch
# 3) 同步后重打（apply --check 先验，冲突再手动）
git -C <repo> apply --check $B/patches/<repo>_dirty.patch && git -C <repo> apply $B/patches/<repo>_dirty.patch
```

判定规则：主干无该改动 → 本机必需品；主干已有同效改动 → 以主干为准。

**但「核查」不等于「自作主张保留」**：核查的目的是**能告诉用户他会丢什么、以后怎么恢复**，而不是替他决定留。用户明确说冲掉时，终态就是纯云端（这些 patch 只存备份目录），并在交付时点明「它们是编本项目的必需品，下次编译前这样重打一行」—— 把选择权交回用户，把恢复命令递到他手上。

## 参考
- `references/20260817-sdk-cross-repo-alignment.md` — IMP_Buf_Vaddr2Paddr 案例完整排查链
- `references/push-readiness-and-change-id-repair.md` — 拉齐后推送就绪体检、Change-Id 非法整批被拒的修复与自证
- 推送侧流程（只推一笔保持 hash）→ gerrit-patch-flow skill
- 提交格式/gerrit 规则 → gerrit-workflow skill
- 编译产物验证 → hm6502-build-flash-test / embedded-build-artifact-verification
