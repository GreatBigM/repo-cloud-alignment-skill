# 拉齐后的推送就绪体检 + Change-Id 修复

同步完成后本地只剩「要推的那一笔」时，推送仍可能被拒。这里的规则都属于「推不上去」的诊断与修复。

## 一次 push = 原子校验

gerrit 对一次 push 里的**所有 commit 一起校验**：任何一笔不合规整批 rejected，而报错只点名那一笔。所以「我只想推 X 却被拒」时，先查链上**其它** commit，别怀疑自己要推的那笔。

最常见真凶：链上某笔的 Change-Id 是**手写的非法值** —— 必须 `I` + 40 位 hex（共 41 字符）；典型特征为 `I` 后只有 39 位、且字节像人编的规律序列，而非 hook 生成的随机 SHA-1。另一类：同链重复 Change-Id（gerrit 拒 duplicate）。

```bash
# 1) 逐 commit 查合法性（有输出即非法）
for c in $(git log --format=%H origin/m/smart..HEAD); do
  cid=$(git show -s --format=%B $c | grep '^Change-Id:' | awk '{print $2}')
  echo "$cid" | grep -qE '^I[0-9a-f]{40}$' || echo "BAD $(git rev-parse --short $c) '$cid'"
done
# 2) 链上重复
git log origin/m/smart..HEAD --format='%B' | grep '^Change-Id:' | awk '{print $2}' | sort | uniq -d
# 3) merge commit（refs/for 通常不收）
git log origin/m/smart..HEAD --merges --oneline
# 4) 确认 hook 没被绕：ls -l .git/hooks/commit-msg（应为软链到 repo hooks）
```

## 「这链推没推过」不能只查主干历史

`git log origin/m/smart --grep <Change-Id>` 查不到**只证明未合入主干**；OPEN / ABANDONED 的 change 只在 gerrit 上，主干历史里根本没有。据此判定「从未推送」并决定丢弃/重做，会丢错东西。

```bash
# 批量体检整条链（一次查询远快于逐笔 ssh；注意部分 gerrit 版本没有 --limit 选项，省略即返回全量）
CIDS=$(git log origin/m/smart..HEAD --format='%B' | grep '^Change-Id:' | awk '{print $2}' | sed 's/^/change:/' | paste -sd' OR ' -)
ssh -p 29418 <user>@<gerrit_host> gerrit query --format=JSON "$CIDS"        # 看 stats 行 rowCount
# 看某人历史上推成功过什么（含 NEW/MERGED/ABANDONED）
ssh -p 29418 <user>@<gerrit_host> gerrit query --format=JSON 'project:<proj> owner:<邮箱>'
```

## 修那一笔：只改 message 一行，不动树

```bash
# 1) 先留退路（filter-branch 会重算 hash）
git update-ref refs/backup/<分支>-pre-cidfix $(git rev-parse <分支>)
# 2) 用 gerrit hook 生成合法值，别手编
TMPMSG=$(mktemp); printf 'probe\n' > $TMPMSG; .git/hooks/commit-msg $TMPMSG
NEWCID=$(grep -m1 '^Change-Id:' $TMPMSG | awk '{print $2}')
[ ${#NEWCID} -eq 41 ] || NEWCID="I$(openssl rand -hex 20)"     # hook 不生效时的兜底
# 3) 只替换坏的那一行
NEWCID=$NEWCID git filter-branch -f --msg-filter \
  'sed "s/^Change-Id: <坏值>$/Change-Id: $NEWCID/"' -- origin/m/smart..<分支>
```

## 修完必须自证零副作用（不证明就别报「已修复」）

```bash
git diff --name-only <旧ref> <新ref> | wc -l                 # 0 = 代码零变化
git rev-parse <旧ref>^{tree} <新ref>^{tree}                  # 顶层 tree hash 必须相同
diff <(git log --format='%s|%an|%ae|%ad' --date=iso <旧ref>) \
     <(git log --format='%s|%an|%ae|%ad' --date=iso <新ref>)  # 空 = 元数据逐条一致
# 逐笔 hash 映射：只比目标范围，别把全历史 paste 进来（会错位，得出满屏 CHANGED 的假象）
paste <(git log --format='%h' --reverse <范围>..<旧ref>) <(git log --format='%h' --reverse <范围>..<新ref>)
```

## 必须向用户交代的代价

改一笔的 message 会牵扯它**之后**所有 commit 重算 hash（父子 hash 绑定）：前段不变、后段全变，merge-base 不变。改前若这些 commit 已推过，要提醒 gerrit 侧会变新 patchset。

## 改法之外的更优路径

若用户其实只需要推**一笔**，别去修整条历史：按 gerrit-patch-flow 的方案 C（基于云端最后一个 patch 的 revision 建分支 + cherry-pick 那一笔），历史 commit 一律不带。
