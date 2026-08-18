# repo-cloud-alignment

repo 多仓项目强制拉齐云端 / 精确复现 Jenkins 构建的 skill — **全仓对齐、分支删除重建、交叉仓 API 依赖编译诊断、增量对齐构建基线**。

解决核心痛点：对齐/拉齐操作容易漏仓（SDK 仓 detached 状态带本地 commit 不出现在 repo status 首屏），漏拉 SDK 仓 → 编译 undefined reference；以及"复现某 Jenkins 构建的确切代码"时需要锁 revision 基线而非主干最新。

## 安装

### 方式 1：一键脚本（推荐）

```bash
curl -fsSL https://gitee.com/GreatBigM/repo-cloud-alignment-skill/raw/main/install.sh | bash
```

等价于手动复制，不经过安全扫描。重复执行 = 升级（自动备份旧版到 `.bak.<时间戳>`）。

### 方式 2：手动复制

```bash
git clone --depth 1 https://gitee.com/GreatBigM/repo-cloud-alignment-skill.git /tmp/rca-skill
cp -r /tmp/rca-skill/references /tmp/rca-skill/SKILL.md /tmp/rca-skill/CHANGELOG.md ~/.hermes/skills/repo-cloud-alignment/
```

### 方式 3：GitHub 镜像（海外备选）

```bash
curl -fsSL https://raw.githubusercontent.com/GreatBigM/repo-cloud-alignment-skill/main/install.sh | bash
```

## 依赖

| 依赖 | 必需 | 说明 |
|------|------|------|
| repo + git | ✓ | repo 工作区管理 |
| gerrit ssh 访问 | ✓ | `ssh -p 29418 <user>@<gerrit_host>`（免密配置） |
| Jenkins 访问 | 复现构建时 | Basic Auth，真实 IP 直连（域名被代理劫持时） |

## 使用

对 AI 说"拉齐云端 / 对齐主干 / 复现某构建号 / 丢弃本地改动"，或加载本 skill 按流程执行：

- **强制拉齐云端主干**：全仓扫描（repo forall，防首屏截断）→ 逐个 fetch（防 MERGED 误判 LOCAL_ONLY）→ 删分支重建（勿 repo start 同名旧分支）→ 编译门禁
- **精确复现某 Jenkins 构建**（增量对齐）：从构建 artifact 下载锁 revision 基线 manifest → 对比各仓 HEAD 只 checkout 差异仓 → 逐 change cherry-pick（MERGED change 被 repo download 拒绝时手动 cherry-pick）
- **交叉仓 undefined reference 诊断**：引用点 → 头文件 → SDK 路径真相源（cfg.cmake）→ nm 符号 → 跨仓找提供者

## 边界

- 本 skill 只做**对齐/拉齐**（丢弃本地改动、复现构建），不做推送——推送见 [gerrit-patch-flow](https://gitee.com/GreatBigM/gerrit-patch-flow-skill)（对齐是对齐，推送是推送）。
- 提交格式/gerrit 规则见 gerrit-workflow。
