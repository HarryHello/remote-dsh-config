# dsh 远程访问配置手册（Tailscale + HTTPS）

> **本文档用途**：指导任意 AI Agent（或人类用户）把一台机器上运行的 DeepSeek Harness（`dsh web`）通过 Tailscale 暴露为 HTTPS 地址，使 Tailnet 内任意设备（Mac / Windows / Linux / iOS）可以用浏览器远程访问。
>
> **工作方式**：Agent 自动执行带 [Agent] 标记的步骤；带 [用户] 标记的步骤必须由用户手动完成（登录、后台开关、提权确认）。每一步都附带验证命令，失败时跳转到「故障排查」章节。
>
> **适用平台**：主机可以是 Windows、macOS 或 Linux（含无 GUI 服务器）；客户端（远程访问方）任意平台，只需要浏览器。
>
> **平台约定**：标注 `macOS / Linux` 的段落表示两平台命令相同（POSIX shell 通用）；仅在确有差异时才分写。

---

## 0. 目标架构

```
远程设备浏览器
     │  https://<machine>.<tailnet>.ts.net
     ▼
tailscaled（监听 443，TLS 终止，Tailscale 签发证书）
     │  本机回环转发（不经防火墙）
     ▼
dsh web（127.0.0.1:3080，DeepSeek Harness）
```

- 只有你的 Tailnet 内设备能访问（tailnet only）
- 全程 HTTPS → 浏览器视为安全上下文（secure context）
- 回环转发 → 无需开放 3080 端口、无需配置防火墙

---

## 1. 环境检查（Agent 自动）

先收集主机环境事实，后续步骤据此分派。

```bash
# 平台判定
uname -a        # macOS / Linux；Windows 用 $env:OS

# dsh 是否可用
dsh --version        # 或
npx @deepseek-ai/dsh --version

# tailscale CLI 是否可用
# Windows:
C:\Program Files\Tailscale\tailscale.exe version
# macOS（GUI 客户端自带 CLI）:
/Applications/Tailscale.app/Contents/MacOS/Tailscale version
# Linux / 已加入 PATH 的任意平台（Linux 安装后通常在 /usr/bin/tailscale）：
tailscale version
```

**判定表**：

| 检查 | 通过 | 失败 → 动作 |
|---|---|---|
| dsh 可用 | 进入第 3 步 | 第 2 步安装 |
| tailscale CLI 可用 | 继续 | 第 2 步安装 Tailscale |
| tailscale 已登录 | `tailscale status` 输出设备列表 | [用户] 登录（第 2 步） |
| MagicDNS 可用 | 能解析 `<设备名>.ts.net` | [用户] 后台开启（第 2 步） |
| HTTPS 证书可用 | `tailscale cert` 能签发 | [用户] 后台开启（第 2 步） |

---

## 2. 安装与初始化

### 2.1 安装 dsh [Agent]

```bash
# 临时使用（推荐先验证）：
npx @deepseek-ai/dsh web --help

# 全局安装（推荐，永久可用，脱离 npx 缓存）：
npm install -g @deepseek-ai/dsh
```

> ⚠️ 全局安装若报 `EPERM`（无法写入 nvm/node 安装目录），说明当前进程无权限，需 [用户] 在普通终端（或管理员终端）执行上面的 `npm install -g`。

### 2.2 安装 Tailscale [Agent 尝试自动安装；登录必须 [用户]]

**Windows**：
```powershell
winget install --id Tailscale.Tailscale -e
# 安装后 CLI 位于 C:\Program Files\Tailscale\tailscale.exe
```

**macOS**（推荐 App Store 或官网，Agent 无法代装 App Store 应用）：
```bash
# 若用 Homebrew（需用户同意）：
brew install --cask tailscale
```

**Linux**（含无 GUI 服务器，官方脚本一条命令装好 CLI + 守护进程）：
```bash
curl -fsSL https://tailscale.com/install.sh | sh
# 启动并设为开机自启：
sudo systemctl enable --now tailscaled
# 安装后 CLI 位于 /usr/bin/tailscale
```

**[用户] 必须手动完成的两件事**（Agent 无法代办）：

1. **登录**：把本机加入目标 tailnet：
   - Windows：托盘图标 → Log in
   - macOS：菜单栏图标 → Log in
   - **Linux（无 GUI）**：`sudo tailscale up`——终端会打印一条**认证 URL**，把 URL 复制到**任意设备的浏览器**打开即可完成授权（headless 友好，无需在服务器上操作浏览器）
   - 或通用 CLI：`tailscale up`（有 GUI 的桌面端会弹出浏览器）
2. **开启 MagicDNS 与 HTTPS Certificates**：
   - 打开 <https://console.tailscale.com/admin/dns>
   - 勾选 **MagicDNS**
   - 勾选 **HTTPS Certificates**（Enable HTTPS）
   - 若使用了自定义 DNS 名称空间，确保 MagicDNS 在该命名空间内开启

**Agent 验证（[Agent]）**：
```bash
tailscale status
# 期望：本机在列表里，有 100.x.y.z 地址
```

> 若 `tailscale status` 报 `Access is denied` / 无法连接 tailscaled：
> - **Windows**：CLI 需要管理员权限。[用户] 在「以管理员身份运行」的 PowerShell 里执行，或确认 tailscaled 服务运行中。
> - **Linux**：加 `sudo`（tailscaled 以 root 运行，CLI 通过受保护 socket 通信），或把当前用户加入 `tailscale` 组后重新登录：`sudo usermod -aG tailscale $USER`。
> - **macOS**：通常无需额外操作；报权限错误时加 `sudo`。

---

## 3. 启动 dsh web（Agent 自动）

```bash
# 首次：临时运行
npx @deepseek-ai/dsh web

# 全局安装后：直接运行
dsh web
```

**成功标志**（终端输出）：
```
dsh web: http://127.0.0.1:3080 (LAN: http://<本机局域网IP>:3080)
```

**Agent 验证**：
```bash
# 本机直连（应返回 HTTP 200）
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/
```

> 默认监听 `127.0.0.1`（回环）。无需改监听地址——Tailscale serve 的转发目标就是回环。

---

## 4. 用 tailscale serve 暴露 HTTPS（Agent 自动）

### 4.1 执行 serve

**Windows（必须管理员 PowerShell）**：
```powershell
tailscale serve --bg --https=443 http://127.0.0.1:3080
tailscale serve status
```

**macOS / Linux**：
```bash
tailscale serve --bg --https=443 http://127.0.0.1:3080
tailscale serve status
```
（权限不足时加 `sudo`；macOS 也可用 GUI：Tailscale 菜单 → Serve）

> 443 被占用时换端口：`--https=8443`，访问地址相应变为 `https://<machine>.<tailnet>.ts.net:8443`。

### 4.2 获取访问地址 [Agent]

```bash
tailscale serve status
```

输出示例：
```
https://harryhelloo-1.tailf5d13d.ts.net (tailnet only)
|-- / proxy http://127.0.0.1:3080
```

**提取规则**：第一行的 `https://<设备名>.<tailnet名>.ts.net` 即为远程访问地址，记为 `$DSH_URL`。

### 4.3 Agent 自检

```bash
# 1. DNS 解析（应返回本机 tailnet IP）
# Windows:
Resolve-DnsName <设备名>.<tailnet名>.ts.net
# macOS / Linux:
dig +short <设备名>.<tailnet名>.ts.net

# 2. 本机经 serve 访问（应 HTTP 200）
curl -s -o /dev/null -w "%{http_code}" https://<设备名>.<tailnet名>.ts.net/
```

---

## 5. 远程访问验证（[用户] 在另一台设备上）

让用户在远程设备浏览器打开 `$DSH_URL`：

- ✅ 页面加载，选择工作区能列出目录 → **完成**
- ❌ 页面报 `crypto.randomUUID is not a function` → 见故障排查 F1
- ❌ 选择工作区报 `HTTP 403` → 见故障排查 F2
- ❌ 页面打不开 / 意外终止连接 / 超时 → 见故障排查 F3–F6

---

## 6. 持久化 trusted_host（Agent 自动，推荐）

`dsh web` 的 `/api` 接口有一道**浏览器信任栅栏**：只放行 loopback 与显式声明的 `trustedHosts`。远程访问是 HTTPS + `ts.net` 域名，**每次启动都要用 `--trusted-host` 参数太脆弱**，应固化到 profile 补丁层。

### 6.1 找到 profile 补丁文件

```
# Windows:
C:\Users\<用户名>\.dsh\profiles\web\cordis.patch.yml
# macOS / Linux（路径相同）:
~/.dsh/profiles/web/cordis.patch.yml
```

> 首次运行 `dsh web` 时 profile 会自动初始化。若文件不存在（或内容为空 `[]`），直接创建/覆盖为下面的内容（保留原有的 `webserver` 行）。

### 6.2 写入补丁

```yaml
# Your patch layer for this dsh profile
# ── Network exposure ──────────────────────────────
# (若已存在 webserver 行则保留；不存在则添加)
- id: webserver
  config:
    host: 0.0.0.0
    port: 3080

# ── Tailscale HTTPS trust ─────────────────────────
- id: connection
  config:
    trustedHosts: !!js "['<设备名>.<tailnet名>.ts.net', ...ctx.webRuntime.trustedHosts]"
```

**⚠️ 语法红线**：`!!js` 后**必须跟字符串标量**（用引号包裹整个表达式），不能直接跟数组字面量——`!!js ['a', ...]` 会报 `unknown tag !<tag:yaml.org,2002:js>`。正确的就是上面带引号的写法。

### 6.3 验证补丁（Agent 自动）

```bash
# 组合配置应能解析，且 connection 行的 trustedHosts 含 ts.net 域名
dsh --profile web --dump-config | grep -A5 "id: connection"
# 期望输出类似：
#   trustedHosts: !!js '[''<设备名>.<tailnet名>.ts.net'', ...ctx.webRuntime.trustedHosts]'
```

### 6.4 重启并验证 [Agent]

```bash
# 停掉旧 dsh web，重新启动（不再需要 --trusted-host 参数）
dsh web
```

重启后重新执行第 5 步验证。**此后每次 `dsh web` 都自动信任该域名。**

### 6.5 扩展：第三方插件的 `/sidebar` 路由信任（dsh-better-sidebar 等）

**问题背景**：6.2 的 `connection` 补丁只覆盖 dsh 自身的 `/api` 网关。浏览器插件（如 `dsh-better-sidebar`）的 `/sidebar/*` 路由（懒加载 chunk `/sidebar/bundle/*.js`、文件 API、终端 WebSocket）走**另一道信任栅栏**——它实时读取 `ctx.webRuntime.trustedHosts`（`web-runtime` 行启动时采样：LAN IP 字面量 + `--trusted-host` 参数），与 connection 的配置是**两套独立名单**。因此远程经 ts.net 访问时页面能打开、`/api` 正常，但侧边栏报：

```
[dsh-better-sidebar] chunk script /sidebar/bundle/terminal.js failed to load
```

**修复**：给 `web-runtime` 行追加同样的 trustedHosts 配置。⚠️ 注意两点：该行的 config 原本是 `{printUrl, surfaceContext, trustedHosts: !!js ctx.webStartup.trustedHosts}`，**补丁替换整行 config，三个字段都要写全**；且必须引用 `ctx.webStartup.trustedHosts` 而非 `ctx.webRuntime.trustedHosts`——后者由本行自己提供，引用会成环。

```yaml
# ── 插件 /sidebar 路由信任（web-runtime）────────────
# dsh-better-sidebar 等插件的 /sidebar/* 栅栏实时读 ctx.webRuntime.trustedHosts，
# connection 补丁只覆盖 /api。此处并入 ts.net 域名，远程访问时插件路由不再 403。
- id: web-runtime
  config:
    printUrl: true
    surfaceContext: true
    trustedHosts: !!js "['<设备名>.<tailnet名>.ts.net', ...ctx.webStartup.trustedHosts]"
```

**验证**：

```bash
# ① 配置合成（无需重启）
dsh --profile web --dump-config | grep -A6 "id: web-runtime"
# 期望：trustedHosts 行含 ts.net 域名

# ② 运行时验证（重启 dsh web 后；200 = 放行，403 = 补丁未生效）
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: <设备名>.<tailnet名>.ts.net" http://127.0.0.1:3080/sidebar/bundle/terminal.js
# 交叉验证：用 127.0.0.1 作 Host 头应恒为 200
```

**等价替代**：启动参数 `dsh web --trusted-host <设备名>.<tailnet名>.ts.net` 同样有效（会并入 `webRuntime.trustedHosts`），但每次启动都要带，不推荐。

---

## 7. 可选加固

- **开机自启**：
  - Windows：计划任务启动 `dsh web`；
  - macOS：launchd（`launchctl`）启动 `dsh web`；
  - Linux：systemd 服务（示例，`ExecStart` 路径按实际 `which dsh` 调整）：
    ```ini
    # /etc/systemd/system/dsh-web.service
    [Unit]
    Description=dsh web
    After=network-online.target tailscaled.service
    Wants=network-online.target

    [Service]
    ExecStart=/usr/bin/dsh web
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now dsh-web
    ```
  - `tailscale serve` 配置由 tailscaled 持久化，重启后一般自动恢复（异常时重跑第 4.1 步命令即可）。
- **全局安装 dsh**：见 2.1，使 `dsh web` 命令永久可用。

---

## 8. 故障排查

### F1 `crypto.randomUUID is not a function`

**原因**：浏览器只在「安全上下文」提供 `crypto.randomUUID()`。`http://<IP>:3080`（含 Tailscale IP、内网 IP）不是安全上下文；只有 `https://` 或 `localhost` 才是。

**修复**：放弃 `http://IP:端口` 直连，一律走第 4 步生成的 `https://<设备名>.<tailnet名>.ts.net`。这是唯一正路——浏览器对「IP + 明文 http」的判定是硬编码的，无法配置绕过。

### F2 `/api/host.listDirectory: HTTP 403`（或任意 `/api/*` 403）

**原因**：dsh 的浏览器信任栅栏（`dsh-client-connection` 的 `isTrustedApiRequest`）只放行 loopback 和 `trustedHosts` 中声明的地址。经 `ts.net` 域名访问时 Host 头不在信任名单 → 403。

**修复**：第 6 步持久化（推荐），或临时启动参数：
```bash
dsh web --trusted-host <设备名>.<tailnet名>.ts.net
```

**Agent 诊断技巧**：用不同 Host 头请求本机接口可快速定位：
```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: 127.0.0.1:3080" -X POST http://127.0.0.1:3080/api/sessions.list
# 404 = 放行（到达路由层）；403 = 被信任栅栏拦截
```

> 第三方插件（如 dsh-better-sidebar）的 `/sidebar/*` 路由 403 是**另一道栅栏**（实时读 `webRuntime.trustedHosts`），`connection` 补丁不覆盖它，见 6.5。

### F2b `/api/settings.describe: HTTP 403`（或任何 settings/credentials 方法 403）

**原因**：这是**设计如此，不是故障**。dsh 把「配置平面」整体锁死在 loopback：`settings.*`、`credentials.*`、`agentPreset.*`、`host.pickDirectory`、`llm.discoverModels` 属于 `PRIVILEGED_METHODS`，这些方法**即使配置了 `trustedHosts` 也强制用空信任列表校验**——`trustedHosts` 只是 DNS-rebinding 防护栅栏，不是认证，所以配置和密钥域永远只信任 `127.0.0.1`。远程（ts.net）访问时设置服务商必然 403，无法通过加 `--trusted-host` 或补丁解决。

**判断特征**：工作区/对话正常，唯独「设置/服务商/凭据」类页面报 403。

**修复（三选一）**：

1. **SSH 端口转发（推荐）**——在本地机器执行，把 loopback 隧道到服务器：
   ```bash
   ssh -L 3080:127.0.0.1:3080 user@linux服务器
   ```
   然后本地浏览器打开 `http://127.0.0.1:3080` 完成设置（对 dsh 而言请求来自 loopback，全部放行）。设置完关掉隧道，日常继续用 ts.net。
2. **直接编辑配置文件**——服务商与凭据在 `$DSH_HOME`（Linux: `~/.dsh/`）下：
   - `settings.yaml`：服务商/模型配置（如 `llm-pi-ai: providers: ...`）
   - `.credentials.yaml`：API Key 等
   改完备份并重启 `dsh web`。
3. **服务器本机 curl**（应急）：在服务器上直接调用 `http://127.0.0.1:3080/api/...`（loopback 放行），仅适合熟悉 API 结构的场景。

> 这不是 bug，是安全设计：配置平面在真正的认证层出现前保持 loopback-only。

### F3 浏览器「意外终止了连接」/ 连接被重置 / 超时

**分层定位法**（Agent 按序执行，锁定在哪一层）：

```bash
# ① DNS 解析
Resolve-DnsName <设备名>.<tailnet名>.ts.net     # Windows
dig +short <设备名>.<tailnet名>.ts.net           # macOS / Linux

# ② TCP 443
Test-NetConnection <设备名>.<tailnet名>.ts.net -Port 443   # Windows
nc -zv <设备名>.<tailnet名>.ts.net 443                      # macOS / Linux

# ③ TLS 握手（用 Node 的独立 TLS 栈，绕开 schannel/沙箱干扰）
node -e "require('tls').connect({host:'<tailnet IP>',port:443,servername:'<设备名>.<tailnet名>.ts.net',rejectUnauthorized:false,timeout:6000},()=>{console.log('TLS OK');process.exit(0)}).on('error',e=>{console.log('TLS FAIL',e.message);process.exit(1)})"

# ④ HTTP
curl -s -o /dev/null -w "%{http_code}" https://<设备名>.<tailnet名>.ts.net/
```

| ①失败 | ②失败 | ③失败 | ④非 200 | 处理 |
|---|---|---|---|---|
| ✅ | ✅ | ✅ | ✅ | 无问题，让用户清浏览器缓存/无痕重试 |
| ❌ | — | — | — | 设备不在线 / tailnet 不对 / MagicDNS 未开 → F4 |
| ✅ | ❌ | — | — | 目标 serve 未配置或离线 → 重跑 4.1；仍不行查防火墙 F6 |
| ✅ | ✅ | ❌ | — | 代理软件劫持 / TLS 层问题 → F5 |
| ✅ | ✅ | ✅ | ❌ | serve 后端没起来 → 确认 dsh web 在跑（第 3 步） |

> ⚠️ **沙箱/受限环境陷阱**：某些隔离 shell 里 `curl`（schannel）对**任何** HTTPS 都报 `SEC_E_NO_CREDENTIALS`，连百度都失败——这是环境限制，不代表网络问题。遇到这种情况用 Node 的 `tls.connect`（第③步）交叉验证。

### F4 设备不在线 / MagicDNS 不通

- 主机端：`tailscale status`，确认目标设备显示在线（无 `offline`）；
- [用户] 检查 <https://console.tailscale.com/admin/dns> 的 MagicDNS 开关；
- 确认两端登录的是**同一个 tailnet**（对比 `tailscale status` 里的 tailnet 名）。

### F5 代理软件（Clash / V2Ray / sing-box 等）劫持 tailnet 流量

**症状特征**：
- 浏览器打不开，但 Agent 用 Node 直连能通（`TLS OK` + `HTTP 200`）；
- 代理软件日志出现 `dial DIRECT ... 100.x.x.x:443: i/o timeout` 或 `dns resolve failed: couldn't find ip`。

**原理**：tailnet 私网段是 `100.64.0.0/10`（CGNAT），域名是 `*.ts.net`。代理软件若不识别它们，会把 tailnet 流量丢进代理节点或自己的虚拟网卡（TUN 环路），导致连接被掐断或解析失败。注意：代理只劫持**本机出站**——所以「A 机能连 B 机的 dsh、B 机却连不上 A 机」完全正常，不用怀疑对称性。

**修复（按优先级）**：

1. **关闭 TUN 模式**（若开启）：代理设置 → TUN 模式关闭 → **重启代理**（务必重启，让 runtime 配置生效）。验证：`route print -4 | findstr 198.18`（Windows）应无输出——`198.18.x.x` 路由消失代表 TUN 已关闭。
2. **加直连规则**（TUN 保留时）：在代理规则最前面加：
   ```
   DOMAIN-SUFFIX,ts.net,DIRECT
   IP-CIDR,100.64.0.0/10,DIRECT,no-resolve
   ```
   ⚠️ 仅加 `IP-CIDR` 不够：浏览器请求是**域名**形式，`no-resolve` 的 IP 规则匹配不到域名流量，必须配 `DOMAIN-SUFFIX,ts.net,DIRECT`。
3. **修复代理的 DNS 盲区**（报 `dns resolve failed: couldn't find ip` 时）：`*.ts.net` 的记录只存在于 Tailscale MagicDNS（`100.100.100.100`），公网 DNS（腾讯/阿里/Cloudflare DoH）解析不到。在代理的 DNS 配置里加 nameserver-policy（以 Clash Verge 的 Merge.yaml 为例）：
   ```yaml
   dns:
     enable: true
     listen: 0.0.0.0:8853
     ipv6: false
     default-nameserver:
       - 223.5.5.5
       - 119.29.29.29
       - 100.100.100.100
     nameserver:
       - https://119.29.29.29/dns-query
       - https://223.5.5.5/dns-query
     nameserver-policy:
       "+.ts.net":
         - 100.100.100.100
   ```
   改后重启代理。
4. **或绕过代理**：代理的「绕过列表」/系统代理绕过加入 `*.ts.net` 和 `100.64.0.0/10`，让 tailnet 流量完全不进代理。

### F6 防火墙

- **不需要**为 `tailscale serve` 开放任何端口：serve 转发目标是 `127.0.0.1` 回环，防火墙不管回环；443 由 tailscaled 监听（Tailscale 已自授权）。
- 若确认是防火墙拦截（serve 正常但远端 TCP 都连不上），检查系统防火墙是否阻止了 `tailscaled` 的入站；Tailscale 安装时自带的允许规则一般已覆盖。
- **Linux 补充**：官方安装脚本/包会自动为 ufw/firewalld 添加 tailscaled 放行规则，一般无需手动配置。若自定义过策略，确认放行 `tailscale0` 接口的入站（`sudo ufw allow in on tailscale0`）以及 UDP 出站（Tailscale 隧道端口，默认 41641）。

### F7 tailscale CLI 权限问题

- **Windows**：`Access is denied` / 无法连接 tailscaled 命名管道 → CLI 需要管理员权限。[用户] 在「以管理员身份运行」的 PowerShell 里执行，或确认 `tailscaled` 服务（Tailscale 服务）正在运行。
- **Linux**：CLI 通过受保护 socket（`/var/run/tailscale/tailscaled.sock`）与 root 运行的 tailscaled 通信 → 命令前加 `sudo`，或把当前用户加入 `tailscale` 组后重新登录：`sudo usermod -aG tailscale $USER`。
- **macOS**：通常无需额外操作；报权限错误时加 `sudo`。

---

## 9. 最终验收清单

- [ ] `tailscale status`：主机在线，远端设备同 tailnet
- [ ] `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/` → `200`
- [ ] `tailscale serve status` 显示 `https://<设备名>.<tailnet名>.ts.net → http://127.0.0.1:3080`
- [ ] 远端浏览器打开 `$DSH_URL`：页面加载、无 `crypto.randomUUID` 报错
- [ ] 选择工作区：目录正常列出、无 HTTP 403
- [ ] `profiles/web/cordis.patch.yml` 含 `connection.trustedHosts` 的 ts.net 条目（macOS/Linux 路径 `~/.dsh/profiles/web/cordis.patch.yml`）
- [ ] 使用第三方插件时，`profiles/web/cordis.patch.yml` 含 `web-runtime.trustedHosts` 的 ts.net 条目（见 6.5），且 `/sidebar/bundle/terminal.js` 经 ts.net Host 头返回 200
- [ ] 不带 `--trusted-host` 参数重启 `dsh web` 后，远端仍可正常访问

---

## 附：常用命令速查

```bash
# 启动 / 停止
dsh web                                  # 启动（Ctrl+C 停止）
tailscale serve --bg --https=443 http://127.0.0.1:3080   # 暴露
tailscale serve --https=443 off          # 取消暴露
tailscale serve status                   # 查看当前映射

# 诊断
tailscale status                         # 设备在线状态
tailscale ping <设备名>                  # 连通性（tailnet 内 ping）
tailscale cert <设备名>.<tailnet名>.ts.net   # 手动测试证书签发
dsh --profile web --dump-config          # 查看组合后的完整配置
```
