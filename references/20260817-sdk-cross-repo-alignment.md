# 案例：交叉仓 SDK 依赖导致编译失败（2026-08-17）

## 场景

hm6502_wifi 云端仓强制拉齐主干（kernel/c_mi_ipc 分支删除重建）后首次全量编译失败：

```
ai_detection/libai_detection.a(ai_detection_manager.c.o): In function `run_ai_capture_loop':
/workspace/applications/c_mi_ipc/ai_detection/src/ai_detection_manager.c:483: undefined reference to `IMP_Buf_Vaddr2Paddr'
collect2: error: ld returned 1 exit status
make: *** [Makefile:156: all] Error 2
```

## 排查链（实测顺序）

1. **引用点确认**：ai_detection_manager.c:483 `.phy_addr = (uint32_t)IMP_Buf_Vaddr2Paddr(yuv_buf)`（yuv 抓帧 buffer 物理地址转换）
2. **头文件声明**：grep 全 SDK include 无声明 → API 不在本地 SDK 版本
3. **SDK 路径真相源**：`toolchain/hm6502_cfg.cmake` → SMART_SDK_LIB_PATH = `third_party/rootfs/sdk/5.4.0_mxu2cve2/lib/uclibc`（**不是 7.2.0**！Anona=7.2.0，Anona_hm1009=7.2.0_t41_zm_zg——查错 SDK 会误判）
4. **符号存在性**：nm 本地全部 libimp.a/.so = 0 匹配 → 本地确实没有
5. **引入 commit**：`git log -S "IMP_Buf_Vaddr2Paddr"` → c8f298e（change 212583，MERGED，wulei，「yuv抓帧buffer调整内存分配到rmem」）——只改了 c_mi_ipc 一个仓，未同步 libimp
6. **gerrit 搜 API 名**：`gerrit query 'message:IMP_Buf_Vaddr2Paddr'` 无结果 —— query message 只搜 commit message，**二进制内符号名搜不到**
7. **关键转折**：rootfs 仓（third_party/rootfs）origin/m/smart **从未 fetch**（停在 80d05c86），fetch 后 → ec85593d，发现 SDK 更新 commit：
   `37bf1620 [hm6502][hm6503][hm6801][hm6802] 更新SDK: 新增Ingenic-SDK-06beda2e-rmem物理地址虚拟地址转化接口`（含 5.4.0_mxu2cve2 libimp.a 二进制更新）
8. **验证**：`git show origin/m/smart:sdk/5.4.0_mxu2cve2/lib/uclibc/libimp.a > /tmp/x.a && nm /tmp/x.a | grep IMP_Buf_Vaddr2Paddr` → `00001f50 T IMP_Buf_Vaddr2Paddr` ✓（云端主干 SDK 有该符号）

## 结论

- 根因**不是**本地残留代码，是 **SDK 仓（third_party/rootfs）没拉齐**：只 fetch/对齐了 kernel + c_mi_ipc，漏了 rootfs
- 云端 Verified+1 与本地失败不矛盾：云端构建环境含 37bf1620 的新 SDK；本地工作树 SDK 是旧版
- 附带发现：rootfs 仓本地 HEAD 比 origin/m/smart 多 4 个本地 commit（TCP 参数固化/telnet 前移/注释 tcp_rmem 等）—— **repo status 首屏（head -60）没显示它**，需 repo forall 全量扫才可见
- 用户决策：删代码重拉（<workspace> + repo init -m manifest-smart.xml + repo sync）—— 因为当前 manifest 是 SI-401（锁旧 revision），要主干最新必须换 manifest-smart.xml（不锁 revision），逐仓 git reset 会留下 SDK 漂移

## 教训

1. **交叉仓 API 依赖**：应用仓的新代码可能引用 SDK 仓新 commit 提供的 API；git 层面两仓各自 HEAD 对齐 ≠ 整体可编译
2. **对齐/拉齐必须全仓**（含 detached 仓），编译门禁不能省
3. **gerrit query message 搜不到二进制符号名**——要靠人工定位 SDK 更新 commit（git log -S 引用方 + 看 SDK 仓 log）或查引用方 change 的配套 commit
4. **云端构建能过 ≠ 本地能过**：先怀疑 SDK 仓版本漂移，再怀疑本地代码
