---
name: repo-cloud-alignment
description: repo 多仓项目强制拉齐云端（丢弃本地改动）— 全仓对齐、分支删除重建、交叉仓 API 依赖编译诊断
version: 1.0.0
category: devops
metadata:
  hermes:
    triggers: [拉齐云端, 强制拉齐, 丢弃本地改动, 以云端为准, 对齐主干, 主干最新, 删代码重拉, 全仓对齐]
---

# repo 多仓项目云端对齐

场景：repo 管理的嵌入式多仓工程（如 hm6502_wifi），本地有 commit/改动，目标 = 强制对齐云端主干并丢弃全部本地改动；或需在「主干最新」与「manifest 锁定基线」间抉择。

## 核心原则

1. **全仓覆盖**：对齐必须扫全部项目，不能只看有分支的仓。repo status 首屏会截断；detached/NO BRANCH 仓也能带本地 commit（08-17 实锤：third_party/rootfs 带 4 个本地 commit 未出现在首屏）。
2. **SDK 仓最易漏 = 编译必炸**：交叉仓 API 依赖——应用仓引用新 API，提供方在 SDK 仓（third_party/rootfs）的新 commit。漏拉 SDK 仓 → 编译 undefined reference（案例：IMP_Buf_Vaddr2Paddr）。
3. **编译是最终门禁**：git 层面 HEAD 对齐 ≠ 能编译。拉齐后必须全量编译验证。
4. **manifest 决定目标**：锁 revision 的 manifest（如 HM6502-Master-Build-SI-401）= 构建基线（旧）；manifest-smart.xml 不锁 revision = 跟 m/smart 主干最新。要「主干最新」必须换 manifest，删代码重拉（rm -rf + repo init -m manifest-smart.xml + repo sync）最干净，逐仓 git reset 会留下 SDK 漂移。

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

## 工作流

### Step 1: 现状扫描（只读）
```bash
repo status 2>/dev/null | head -60   # 首屏会截断，仅参考
# 全量本地 commit 扫描（关键！覆盖所有仓）：
repo forall -c 'echo "== $REPO_PROJECT"; git log --oneline origin/m/smart..HEAD 2>/dev/null | head -5'
# 分支 vs detached 状态：
repo forall -c 'echo "== $REPO_PROJECT"; git branch -vv | head -2'
```

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
# gerrit 状态确认（MERGED=已合入主干 / NEW=open 未合入）：
ssh -p 29418 <user>@<gerrit_host> gerrit query 'change:<Change-Id>' --format=JSON --current-patch-set | grep -E '"number"|"status"'
```

### Step 3: 删除重建分支（勿 repo start！）
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
| cherry-pick 验证用整树 diff | parent 不同带出全部基线差异（135 files 假象） | 同父链才 git diff；否则 git show --stat 对比文件集+行数+Change-Id |

## 参考
- `references/20260817-sdk-cross-repo-alignment.md` — IMP_Buf_Vaddr2Paddr 案例完整排查链
- 推送侧流程（只推一笔保持 hash）→ gerrit-patch-flow skill
- 提交格式/gerrit 规则 → gerrit-workflow skill
- 编译产物验证 → hm6502-build-flash-test / embedded-build-artifact-verification
