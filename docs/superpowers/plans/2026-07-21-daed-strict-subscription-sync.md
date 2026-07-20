# daed Strict Subscription Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (- [ ]) syntax for tracking.

**Goal:** 让 daed 原生面板、daed 自带 cron、LuCI 手动更新和 LuCI cron 都能严格同步订阅节点，并在代理运行时立即应用运行态。

**Architecture:** 保持现有 updateSubscription GraphQL 签名不变，在 pinned dae-wing 上维护一个 0006 补丁。补丁在同一数据库事务里按节点 Link 对账：保留仍存在的节点、保护 fixed 组节点、删除普通组里的失效节点、导入新增节点；如果系统正在运行，则提交前调用现有 config.Run 应用运行态。UpdateById 用互斥锁串行化，兼容原生 Web 的批量并发请求。

**Tech Stack:** OpenWrt Makefile、Go 1.26、GORM/SQLite、GraphQL、POSIX Shell、dae-wing pinned source patches。

---

## 文件结构

- Create: daed/patches/0006-reconcile-stale-subscription-nodes.patch — dae-wing 严格同步、运行态应用及回归测试。
- Modify: daed/Makefile — PKG_RELEASE 从 2 升到 3。
- Modify: docs/superpowers/specs/2026-07-21-daed-strict-subscription-sync-design.md — 记录保持旧 GraphQL 签名的最终取舍。

### Task 1: 在精确 pinned dae-wing 上建立失败测试

**Files:**
- Create in temporary pinned source: graphql/service/subscription/mutation_utils_test.go
- Reference: ci/pins.env
- Reference: daed/patches/0001 through 0005

- [ ] **Step 1: 准备精确源码并顺序应用现有补丁**

Clone dae-wing，checkout dc503088945812c11235b35362d2bfa1a4c3bdf0，并按编号依次 git apply 0001 到 0005。

Expected: 五个补丁 clean apply。

- [ ] **Step 2: 写入 DB 回归测试**

测试使用 db.InitDatabase(t.TempDir())，直接创建 Subscription、Node、Group 和 group_nodes 关系，不并行执行。加入这些测试：

    func TestReconcileKeepsCurrentReferencedNode(t *testing.T)
    func TestReconcileDeletesStaleUnreferencedNode(t *testing.T)
    func TestReconcileDeletesStaleNormalGroupNode(t *testing.T)
    func TestReconcileDetachesStaleFixedNode(t *testing.T)
    func TestReconcilePreservesNodeReferencedByFixedAndNormalGroups(t *testing.T)
    func TestReconcileAllLinksAlreadyExistSucceeds(t *testing.T)
    func TestReconcileRollsBackWhenNoValidSubscriptionNodesRemain(t *testing.T)

每个测试断言 nodes.subscription_id、group_nodes、节点 ID、最终订阅节点数和需要变更的 Group.Version。

- [ ] **Step 3: 验证测试先失败**

Run on Linux:

    go test -tags dae_stub_ebpf ./graphql/service/subscription -count=1

Expected: FAIL，原因是 reconcileSubscriptionNodes 尚不存在。

### Task 2: 实现事务内严格同步和运行态应用

**Files:**
- Modify in temporary pinned source: graphql/service/subscription/mutation_utils.go
- Modify in temporary pinned source: graphql/root_schema.go

- [ ] **Step 1: 增加串行更新锁和运行态依赖**

在 mutation_utils.go 引入 graphql/service/config，并增加：

    var subscriptionUpdateMu sync.Mutex

UpdateById 从拉取订阅开始持有该锁，避免原生 Web 的 Promise.all 同时触发多个 config.Run。

- [ ] **Step 2: 实现严格节点对账 helper**

增加：

    func reconcileSubscriptionNodes(tx *gorm.DB, subID uint, links []string) error

实现规则：

    incomingLinks = 去重后的新 links
    existing = subscription_id == subID 的旧节点
    current = existing.Link 仍在 incomingLinks：保留 ID 和组关系
    staleFixed = stale 且至少被一个 policy=fixed 的组引用：subscription_id 置 NULL
    staleNormal = 其余 stale：先 node.AutoUpdateVersionByIds，再删除节点及 group_nodes
    newLinks = incomingLinks - current.Link：调用 node.Import
    finalCount = subscription_id == subID 的节点数；为 0 则返回错误

同一个 stale 节点同时属于 fixed 和普通组时按 staleFixed 处理，保留该节点和全部显式组关系。

- [ ] **Step 3: 把更新、严格同步和应用放进一个事务**

UpdateById 的顺序改为：

    links, err := fetchLinks(m.Link)
    tx := db.BeginTx(ctx)
    err = reconcileSubscriptionNodes(tx, subID, links)
    err = tx.Model(&m).Update("updated_at", time.Now()).Error
    err = AutoUpdateVersionByIds(tx, []uint{subID})
    if system.Running {
        _, err = config.Run(tx, false)
    }

任一步失败都回滚。config.Run 失败时 GraphQL 返回原错误，不显示更新成功。

- [ ] **Step 4: 更新 schema 注释**

改为：

    # updateSubscription re-fetches and strictly reconciles subscription nodes. Stale nodes pinned by fixed groups become independent nodes.

- [ ] **Step 5: 格式化并运行回归测试**

Run:

    gofmt -w graphql/service/subscription/mutation_utils.go graphql/service/subscription/mutation_utils_test.go
    go test -tags dae_stub_ebpf ./graphql/service/subscription -count=1
    go test -tags dae_stub_ebpf ./graphql/service/group ./graphql/service/config -count=1
    go test -tags dae_stub_ebpf ./graphql/... -count=1

Expected: 所有命令 exit 0。

### Task 3: 生成并接入 OpenWrt 补丁

**Files:**
- Create: daed/patches/0006-reconcile-stale-subscription-nodes.patch
- Modify: daed/Makefile:9

- [ ] **Step 1: 从临时源码生成统一 diff 补丁**

补丁只包含：

    graphql/service/subscription/mutation_utils.go
    graphql/service/subscription/mutation_utils_test.go
    graphql/root_schema.go

Subject: [PATCH] daed: strictly reconcile subscription updates

- [ ] **Step 2: 验证补丁顺序**

在全新的 pinned dae-wing checkout 上按 0001 到 0006 逐个运行 git apply --check 和 git apply。

Expected: 六个补丁都能按顺序 clean apply。

- [ ] **Step 3: bump daed 包版本**

修改 daed/Makefile：

    PKG_RELEASE:=3

不修改 PKG_VERSION、PKG_SOURCE 或 PKG_HASH，因为只增加 Build/Prepare 阶段应用的本地补丁。

- [ ] **Step 4: 本地静态验证**

Run:

    git diff --check
    sh -n luci-app-daede/root/usr/share/luci-app-daede/daed-sub-update.sh

Expected: exit 0。

### Task 4: 构建并在 252 验证真实链路

**Files:**
- Deploy package artifact only; do not edit unrelated router files.

- [ ] **Step 1: 完成 Linux/SDK 构建门禁**

在 OpenWrt SDK 中运行：

    make package/daed/clean
    make package/daed/compile V=s

Expected: 生成 daed 2026.07.17-r3 对应架构包，补丁全部应用且 Go/eBPF 编译成功。

- [ ] **Step 2: 读取路由器实验清单并做只读基线**

确认 252 的架构、包管理器、当前 daed/LuCI 版本、服务状态、订阅数、节点数和更新日志。输出中隐藏订阅 URL、节点链接、密码和 token。

- [ ] **Step 3: 临时安装 r3 并验证原生面板**

备份当前包信息和配置数据库，上传单个 daed 测试包，安装后只重启 daed。构造或选择可安全测试的订阅：普通组单独引用若干节点、fixed 组固定一个节点，然后在 daed 原生面板点击更新。

Expected:

    新节点立即进入运行态
    普通组引用的 stale 节点删除
    fixed stale 节点转为独立节点且组仍有效
    订阅节点数不再累积
    更新失败时页面收到明确错误

- [ ] **Step 4: 验证 daed cron、LuCI 手动按钮和 LuCI cron**

三条入口分别执行一次，检查数据库节点数、策略组关系、进程状态和 /tmp/luci-app-daede.daed-sub-update.log。不得输出订阅或节点链接。

- [ ] **Step 5: 回归代理访问并检查工作树**

确认代理访问正常、daed 没有重启循环或 reload timeout；运行 git status --short 和 git diff --check。

Expected: 只有计划内文件变更，diff 无空白错误。验证通过后再由用户决定 commit、push 和发布。

