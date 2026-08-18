# CHANGELOG

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
