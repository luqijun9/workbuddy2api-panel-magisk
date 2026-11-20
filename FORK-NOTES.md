# 自用部署：这两个 fork 跟上游差在哪

本文件同时说明**两个**仓库：

| 仓库 | 作用 | 上游 |
|---|---|---|
| [`luqijun9/workbuddy2api-panel`](https://github.com/luqijun9/workbuddy2api-panel) | 网关本体（Go） | `misakano7545/workbuddy2api-panel` |
| [`luqijun9/workbuddy2api-panel-magisk`](https://github.com/luqijun9/workbuddy2api-panel-magisk)（本仓库） | 模块壳：安装脚本 + WebUI + CI 打包 | `huxc573/workbuddy2api-panel-magisk` |

## 为什么 fork

**症状**：KernelSU / Magisk 模块形态下，面板「添加账号」必报

```
auth state (upstream 0): ... lookup copilot.tencent.com on [::1]:53: connect: connection refused
```

**原因**：面板登录那条链用裸 `net.Dialer` → 走 Go 默认解析器 → Android 没有 `/etc/resolv.conf`
（`/etc` 是 `/system/etc` 的软链，`/system` 是 erofs 只读，建不出来）→ Go 退回 `127.0.0.1:53` /
`[::1]:53` → 没有任何 DNS 在听 → 所有域名解析直接失败。

模块本来会注入 `WB2A_DNS=223.5.5.5,119.29.29.29`，但只接进了**上游客户端**和**代理池**；
登录、更新检查这几条路用的是系统解析器，没接上。Termux 里能用是因为 Termux 的 Go 读
`$PREFIX/etc/resolv.conf`，而模块形态没有这个文件。

**修法**：把 `NewDNSDialer` 造出来的 resolver 同时赋给 `net.DefaultResolver`
（`net/dial.go` 里裸 Dialer 最终 `return DefaultResolver`），一处覆盖所有裸 Dialer 客户端。

**没走**另一条路（CGO 交叉编译自定义解析器，见上游 PR #6）：会破坏 `binaries.yml` 的
`CGO_ENABLED=0` 六平台矩阵，静态二进制也不能再直接扔进任何 Android 上跑。

---

## 一、网关本体 fork（`workbuddy2api-panel`）

状态：**跟随上游**。`sync.yml` 每小时把 `upstream/main` 合并进来，补丁靠 merge 保留。

### 1. `cmd/server/main.go`（+9 / -2）—— 唯一的代码改动

**a) DNS 修复**

```go
if len(cfg.DNSServers) > 0 {
    nd := upstream.NewDNSDialer(cfg.DNSServers)
    up.SetDNSDialer(nd)
    // 同一套解析器接管进程默认解析器：面板登录 loginHTTP、更新检查等
    // 用裸 net.Dialer / http.DefaultTransport 的客户端都走 net.DefaultResolver
    if nd.Resolver != nil {
        net.DefaultResolver = nd.Resolver
    }
}
```

**b) 内置更新指向本 fork**

```go
var appRepository = "luqijun9/workbuddy2api-panel" // 面板「检查更新」看我们自己的 Release
```

### 2. `.github/workflows/sync.yml`（新增，+62）

每小时 `:15`（cron `15 * * * *`）+ 手动触发：

1. `git fetch upstream main` → 没有就合并 → push
2. 读上游最新 release tag
3. 我们已经有了该 tag 的 Release → 退出
4. 没有 → `git tag -f <上游tag>` + push + `gh workflow run binaries.yml --repo <本仓库> --ref <tag>`

**两个踩过的坑**（都已修）：

- **`gh workflow run` 必须带 `--repo`**。不带时 `gh` 在有两个 remote 的仓库里会解析到
  upstream，报 `403 Resource not accessible by integration`。
- **`GITHUB_TOKEN` 推的 tag 不会触发 workflow**（GitHub 的递归保护），所以「打 tag 让
  `binaries.yml` 自己跑」这条路是死的。`workflow_dispatch` 是唯一例外，必须显式 dispatch。

### 3. 仓库设置（不是代码）

关闭了 `autobot.yml`（上游的 AI 自动改进机器人，网关依赖已挂）和 `docker-ghcr.yml`（我们不发镜像）。

### 说明：我们的 Release 说明是空的

`binaries.yml` 用 `gh release create --generate-notes`，而自动生成靠的是 **PR 记录**。
我们用 `git merge` + `git push` 跟进上游，**没有 PR**，所以生成出来的说明只有一行
`**Full Changelog**: ...`。

**这不影响任何东西**：模块壳的更新日志读的是**上游仓库**的说明，不是我们的。
详见「五、两个已知的不对应场景」。

---

## 二、模块壳 fork（本仓库）

状态：**不跟随上游**，自己维护（理由见第三节）。

### 1. 运行体来源（3 个文件，各 1 行）

把「去哪儿取 Go 运行体」从上游仓库改成本 fork：

| 文件 | 改动 |
|---|---|
| `scripts/ci/upstream.sh` | `REPO="${WB2A_UPSTREAM_REPO:-luqijun9/workbuddy2api-panel}"` |
| `scripts/ci/fetch-runtime.sh` | 同上 |
| `.github/workflows/build.yml` | `UP_REPO:` 两处 → 同上 |

### 2. 设备侧地址（2 个文件）

| 文件 | 改动 |
|---|---|
| `src/updater.sh` | `REPO="${WB2A_UPDATE_REPO:-luqijun9/workbuddy2api-panel}"`（运行体自动更新器，默认关闭） |
| `src/customize.sh` | 生成的 `update.conf` 里注释中的仓库地址 |

### 3. `src/webroot/index.html`（+72）—— 监听开关

管理器里点「打开」看到的那页。新增：

- 状态区一行「监听」：局域网时显示 `http://<手机IP>:<端口>/panel/`，仅本机时显示 `127.0.0.1:<端口>`
- 一个切换按钮：写 `/data/adb/wb2a/listen`（`local`/`lan`）后 `service.sh restart`

与 `sh action.sh config` 里的「监听范围 1/2」改的是同一个文件，两边永远一致。

> ⚠️ 源文件里是占位符 `__WB2A_REPO__` / `__WB2A_PAGES__`，由 `build_module.py` 在打包时替换。
> 直接手改手机上的文件时**必须用包内已替换的版本**，否则页面里的「查看 Releases」会失效。

### 4. `src/refresh.sh`（+8 / -2）—— 卡片地址跟随监听范围

上游逻辑是 `IP=$(wlan_ip)`，只要有 WiFi IP 就显示它，**完全不看监听模式**。
结果：切成仅本机后，管理器卡片仍写 `192.168.1.209:7863`，而那个地址此时连不上。

改成读 `wb2a_listen_addr`：仅本机显 `127.0.0.1`，局域网照旧显 WiFi IP。

### 没改的（保持上游原样）

`service.sh`、`lib.sh`、`action.sh`、`watchdog.sh`、`build_module.py`、`customize.sh` 的安装逻辑、
`uninstall.sh`、`binaries` 那套打包流程。

---

## 三、维护约定：模块壳**不**跟上游

模块壳上游 `huxc573/workbuddy2api-panel-magisk` **不再同步**。它只是一个壳：
安装脚本 + 一个静态 HTML + 打包 CI，逻辑简单、稳定，出问题我们自己改更快。

fork 时它就落后 13 个提交，构成很清楚：

| 类型 | 数量 | 价值 |
|---|---|---|
| `chore: 发布 v26.11.xx_x` | 11 | 他自己的发版记录（只改 `update.json`/`CHANGELOG-current.md`/`UPSTREAM`）。我们每次发版也会生成，各记各的账 |
| `feat(ci)` 改 `changelog-current.sh` | 2 | 「上游 release 没写说明时改从 compare 取」。边际收益很小，且改的正是我们已有多处改动的文件 |

合并过来只会跟我们对 `webroot/index.html`、`refresh.sh` 的改动冲突，不划算。

---

## 四、更新链条

```
① 上游 misakano7545 发新版
      ↓  每小时 :15（sync.yml）
② 合并 upstream main → 打同名 tag → dispatch binaries.yml
      ↓
③ binaries.yml：go test → 六平台交叉编译 → 建 Release（带产物）
      ↓  每小时 :30（build.yml）
④ 取 ③ 的 linux-arm64 运行体 → 打 Magisk 包 → 生成 update.json / CHANGELOG-current.md → 推 Pages
      ↓
⑤ 手机管理器读 updateJson → 出现「有更新」角标
      ↓
⑥ 你手动点安装（自动更新始终关闭：WB2A_UPDATE_INTERVAL=0，面板 auto_enabled=false）
```

手动补触发的方式：

```bash
# 立刻跟进上游（不用等 :15）
gh workflow run sync.yml --repo luqijun9/workbuddy2api-panel

# 立刻重打模块包（上游没变时需 force=true）
gh workflow run build.yml --repo luqijun9/workbuddy2api-panel-magisk -f force=true
```

---

## 五、两个已知的「不对应」场景

模块壳的更新日志两段来源：

1. **本模块这版** ← 本仓库 `CHANGELOG.md`
2. **上游这版** ← `gh release view <UPSTREAM 内容> -R misakano7545/workbuddy2api-panel`

第二段要求 `UPSTREAM` 文件里记的 tag **在上游仓库真实存在**，否则查到 404 → 静默空白 →
退化成「本版仅更新运行体」。

| 场景 | 结果 |
|---|---|
| tag 名 = 上游原名（`sync.yml` 打的） | ✅ 日志完整 |
| tag 名自造（`v2026.11.20.1312` 就是教训） | ❌ 上游段空白 |
| 我们自己改了 DNS 补丁等内容 | ⚠️ 日志永远不体现（只记上游 + 模块 CHANGELOG） |

**结论：永远不要手动打 tag**，只让 `sync.yml` 打，它取的就是上游 release 原名。

---

## 六、常用命令

```bash
# 当前监听范围
su -c 'cat /data/adb/wb2a/listen'          # local / lan

# 切换（等价于 WebUI 那个按钮 / action.sh config 里的第 2 问）
su -c 'echo local > /data/adb/wb2a/listen && sh /data/adb/modules/wb2a/service.sh restart'

# 刷新管理器卡片上那行字
su -c 'sh /data/adb/modules/wb2a/refresh.sh'

# 看网关日志
su -c 'tail -50 /data/adb/wb2a/run.log'
```
