# 在火山引擎 ECS 上部署 OpenClaw 的完整教程

> 本文档根据真实部署经验修订，修复了多个常见坑点！

## 本文档核心价值

本教程提供在火山引擎 ECS 上部署开源 AI 助手 OpenClaw 的完整、分步指南。通过将 OpenClaw 的"本地优先"隐私优势与火山引擎的云端稳定、高性价比算力及飞书生态无缝对接相结合，您可以在约 1 小时内，以极低的成本（按量计费约 0.1-0.2 元/小时）获得一个 7x24 小时在线、安全可控的私有 AI 助手。

教程涵盖从账号准备、实例创建、环境配置，到火山方舟 Coding Plan AI 模型对接、飞书机器人长连接模式配置，以及安全加固和故障排查的全流程。

---

## 一、OpenClaw 与火山引擎 ECS：为何是绝佳组合？

OpenClaw（原名 Moltbot/Clawdbot）是一个开源、自托管的个人 AI 助手。它不仅仅是一个聊天机器人，而是一个拥有"眼睛和双手"的智能体，能够操作系统文件、运行 Shell 命令、控制浏览器、处理邮件，并通过 WhatsApp、Telegram、Discord 及飞书等主流聊天平台与用户交互。

其"本地优先"的设计理念意味着所有数据存储在您的设备或服务器上，确保了数据的隐私安全和完全的掌控权。

选择在火山引擎 ECS 上部署 OpenClaw，是将其强大能力与云端优势结合的最佳实践。

### OpenClaw 核心特性

- **本地优先：** 数据和对话历史存储在您自己的服务器上，隐私得到最大保护。
- **多平台集成：** 原生支持飞书、企业微信、WhatsApp、Telegram 等 10+ 个聊天平台。
- **系统级访问：** 具备读写文件、执行命令、控制专用浏览器的能力，可完成复杂自动化任务。
- **技能扩展：** 可通过 ClawdHub 安装社区技能，或让 AI 自行编写新技能来扩展功能。

### 火山引擎 ECS 部署优势

- **飞书生态无缝对接：** 作为同一生态体系，配置飞书机器人时天然契合，采用长连接模式无需公网回调，极大简化部署。
- **集成 AI 模型服务：** 火山方舟 Coding Plan 提供稳定、高性价比的 AI 模型 API（如豆包、Kimi、DeepSeek），解决国内开发者使用 OpenAI/Claude 的网络和成本门槛。
- **高性价比与灵活性：** 提供按量计费的 ECS 实例，用多少付多少，最低仅需 2 核 4GB 配置即可流畅运行，成本可控。

---

## 二、部署前准备：账号、认证与资源

在开始创建服务器之前，需要完成三项核心准备工作：火山引擎账号的注册与充值、获取 AI 模型 API 凭证、以及预先创建飞书机器人应用。

> **重要：** 请务必在本章节完成火山引擎账号实名认证和充值（至少 100 元），否则无法创建按量计费的 ECS 实例。同时，提前复制好火山方舟 API Key 和飞书应用的 App ID & App Secret，后续步骤会频繁用到。

### 2.1 火山引擎账号准备

1. **注册账号：** 访问火山引擎官网，点击右上角"免费注册"，使用手机号或邮箱完成注册。
2. **实名认证：** 注册成功后，登录控制台，点击右上角头像进入"账号管理" → "实名认证"。根据指引完成个人或企业信息认证。这是使用付费云服务的法定要求。
3. **账户充值：** 由于本教程推荐使用按量计费的 ECS 实例，需确保账户现金余额和代金券总值不低于 100 元人民币。可在"费用中心"进行充值。

### 2.2 获取 AI 模型 API（火山方舟 Coding Plan）

OpenClaw 的"大脑"需要调用外部大模型。火山方舟 Coding Plan 是国内最佳选择，一个订阅即可接入豆包、Kimi、DeepSeek 等多款顶级模型，且访问稳定。

1. 登录 [火山方舟控制台](https://console.volcengine.com/ark)。
2. 进入"Coding Plan"页面，根据需求选择 Lite（尝鲜）或 Pro（重度使用）套餐完成订阅。
3. 在"API Key 管理"页面，创建一个新的 API Key，备注"OpenClaw专用"并立即复制保存（仅显示一次）。

> **安全提示：** 不要把 API Key 分享给他人，也不要提交到公开的 GitHub 仓库！

### 2.3 创建飞书机器人应用（预先操作）

1. 登录 [飞书开放平台](https://open.feishu.cn/)。
2. 点击"创建企业自建应用"，填写应用名称（如 MyOpenClawBot）和描述，点击创建。
3. 进入应用管理页，在"添加应用能力"中勾选"机器人"。
4. 在"凭证与基础信息"页面，记录下 **App ID** 和 **App Secret**。事件订阅配置留待第 6 章进行。

---

## 三、创建与配置火山引擎 ECS 实例

### 3.1 进入创建向导

1. 登录火山引擎控制台。
2. 在左侧导航栏选择"产品" → "计算" → "弹性计算 ECS"。
3. 在 ECS 实例列表页面，点击"创建实例"按钮，选择"自定义购买"。

### 3.2 基础配置详解

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| 计费类型 | 按量计费 | 适合测试和临时部署，用完可随时释放，避免资源浪费。 |
| 地域及可用区 | 华北 2（北京）或其他离您近的地域 | 选择距离业务更近的地域以降低网络延迟。实例创建后无法更改地域。 |
| 实例规格族 | 通用型 g1ie | 提供均衡的 vCPU、内存和网络能力，满足大多数场景需求。 |
| 具体实例规格 | ecs.g1ie.large (2vCPU, 4GiB内存) | OpenClaw 运行的最低推荐配置，可流畅执行大多数任务。 |
| 镜像类型 | 公共镜像 | 稳定、更新快，由火山引擎官方提供。 |
| 镜像 | Ubuntu 22.04 LTS | Docker 安装方便，社区支持好，是本教程的操作环境。 |
| 系统盘类型 | 极速型 SSD 云盘 | 性价比高，IOPS 和吞吐量足够 OpenClaw 和 Docker 使用。 |
| 系统盘容量 | 40 GiB（默认） | 足够容纳系统、Docker 及 OpenClaw 数据，后续可扩容。 |

### 3.3 网络与安全配置

**网络配置：**
- 私有网络：使用默认 VPC 或新建一个。
- 子网：选择与 VPC 对应的子网。
- 公网 IP：必须勾选"分配弹性公网 IP"，以便从外部 SSH 连接。带宽计费方式可选"按固定带宽"，设置 1 或 5 Mbps 用于测试。

**安全组配置（关键）：**
- 新建一个安全组，或修改默认安全组规则。
- 必须添加一条入方向规则：允许 SSH (22) 端口，源地址可暂时设置为 0.0.0.0/0（测试用），生产环境应限制为您的 IP。

> **注意：** 使用飞书长连接模式时，OpenClaw Gateway 无需对外开放 18789 端口！

### 3.4 系统配置与创建

1. **登录方式：** 推荐使用"SSH 密钥"方式，安全性更高。若没有密钥，可点击"创建密钥对"，生成后私钥文件会下载到本地，请妥善保管。
2. **实例名称：** 可自定义，如 OpenClaw-Server。
3. 确认所有配置无误后，点击"下一步：确认订单"，阅读并同意服务条款，点击"立即购买"。
4. 等待 1-3 分钟，实例状态变为"运行中"。在实例列表中找到该实例，记录下其公网 IP 地址。

---

## 四、连接 ECS 并安装基础环境（Node.js 与 Docker）

### 4.1 SSH 连接 ECS 实例

使用您创建实例时设置的登录方式（密钥或密码）连接服务器。

**Windows 用户（推荐使用 MobaXterm 或 PowerShell）：**
```powershell
ssh root@<你的公网IP>
```

**macOS/Linux 用户：**
```bash
ssh root@<你的公网IP>
```

连接成功后，终端提示符将变为类似 `root@i-xxxxxxx:~#`。

### 4.2 安装 Node.js 22

OpenClaw 要求 Node.js 版本必须 ≥ 22。

```bash
# 1. 更新系统包列表
apt update && apt upgrade -y

# 2. 使用 NodeSource 安装 Node.js 22.x LTS 版本
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs

# 3. 验证安装
node --version  # 应输出 v22.x.x
npm --version   # 应输出对应的 npm 版本
```

### 4.3 安装 Docker 与 Docker Compose

为简化部署和避免环境冲突，推荐使用 Docker。

```bash
# 执行轩辕一键安装脚本，自动安装 Docker 和 Docker Compose
bash <(wget -qO- https://xuanyuan.cloud/docker.sh)
```

脚本运行约 2-5 分钟，完成后会自动启动 Docker 服务并设置开机自启。

```bash
# 验证安装
docker --version
docker compose version

# 检查 Docker 服务状态
systemctl status docker
```

---

## 五、OpenClaw 的安装、初始化与火山方舟对接

### 5.1 安装 OpenClaw

推荐使用官方一键安装脚本，它会自动处理依赖和配置。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装完成后，验证版本：

```bash
openclaw --version
```

> **注意：** 在接下来的 Onboarding 向导中，AI 提供商和聊天渠道建议先选择跳过。我们将采用更可控的手动配置方式分别对接火山方舟和飞书，避免自动配置可能带来的问题。

### 5.2 运行初始化向导 (Onboarding)

执行以下命令启动交互式配置向导：

```bash
openclaw onboard
```

在向导中，请按以下指引选择：
- **Gateway Port：** 按回车使用默认 18789。
- **Gateway Bind：** 选择 loopback (127.0.0.1)。因为我们将使用飞书长连接模式，Gateway 无需对外网暴露。
- **AI Provider：** 选择 Skip for now。
- **Chat Channel：** 选择 Skip for now。
- **Skills：** 新手建议选择 Skip for now。

### 5.3 手动配置火山方舟 Coding Plan（关键！）

现在，使用 OpenClaw 内置的火山方舟支持，这是最简单和稳定的方式！

> **⚠️ 关键提示：** 不要用旧的 `models.providers` 方式！直接用 `volcengine-plan` provider！

#### 步骤 1：配置火山方舟 API Key

```bash
# 设置 API Key（替换为你的真实 Key）
export VOLCANO_ENGINE_API_KEY="你的火山方舟API_Key"
```

或者你可以把这行添加到 `~/.bashrc` 中永久生效。

#### 步骤 2：设置默认模型

使用 CLI 快速设置：

```bash
# 查看可用的火山方舟模型
openclaw models list | grep volcengine

# 设置默认模型（推荐用这个！）
openclaw models set volcengine-plan/ark-code-latest
```

就这么简单！OpenClaw 内置了对 `volcengine-plan` 的完整支持，不需要手动配置 baseUrl、api 类型等复杂参数！

---

## 六、配置飞书机器人长连接模式（无需公网回调）

这是本教程简化部署的关键。OpenClaw 内置了飞书的 WebSocket 长连接支持，无需服务器拥有公网 IP 或配置 HTTPS 回调地址。

### 长连接模式的优势

- **无需公网 IP/回调 URL：** 服务器可位于内网。
- **配置极简：** 只需 App ID 和 Secret。
- **连接稳定：** 具备自动重连机制。
- **安全性高：** 端到端加密，无需开放额外公网端口。

### 6.1 完善飞书应用配置

回到飞书开放平台，进入您在第 2.3 节创建的应用。

**1. 添加权限：**
进入"权限管理"，添加以下权限并申请开通：
- `im:message`（接收与发送消息）
- `im:message:send_as_bot`（以机器人身份发送消息）
- `im:chat:readonly`（获取群列表）
- `contact:user.id:readonly`（获取用户 ID）

**2. 配置事件订阅（关键步骤）：**
- 进入"事件与回调"。
- 在"回调配置"中，订阅方式必须选择"**使用长连接接收回调**"，然后保存。
- 在"添加事件"中，选择"消息与群组" → "接收消息 (im.message.receive_v1)"。

**3. 发布应用：**
点击顶部"版本管理与发布" → "创建版本"，填写版本说明后提交发布。等待企业管理员审核（如为自建企业，通常自动通过）。

### 6.2 配置 OpenClaw 连接飞书（关键！）

在 ECS 服务器上执行以下命令，配置飞书连接信息。

> **⚠️ 避坑提示：** 一定要配置 `domain`！如果是飞书用户用 "feishu"，Lark 用户用 "lark"！我们昨天刚踩过这个坑！

```bash
# 配置 App ID（替换为你的）
openclaw config set channels.feishu.appId "cli_xxxxxxxxx"

# 配置 App Secret（替换为你的）
openclaw config set channels.feishu.appSecret "你的AppSecret"

# ⚠️ 关键！设置 domain 为 "feishu"（如果是飞书，不是 Lark）
openclaw config set channels.feishu.domain "feishu"

# 启用飞书
openclaw config set channels.feishu.enabled true

# 如果希望机器人在群聊中可用
openclaw config set channels.feishu.groupPolicy "open"
```

### 6.3 重启服务并测试

**1. 重启 Gateway 服务：**
```bash
openclaw gateway restart
```

**2. 查看连接日志，确认飞书连接成功：**
```bash
# 查看 Gateway 日志
tail -f ~/.openclaw/logs/gateway.log

# 或者只看飞书相关日志
tail -f ~/.openclaw/logs/*.log | grep -i feishu
```

看到类似 `feishu connected` 的日志即表示连接成功！

**3. 最终测试：**
在飞书客户端中搜索您的机器人应用名称，向其发送一条消息（如"你好"）。如果一切正常，几秒内将收到 AI 助手的回复。

---

## 七、安全加固、监控与性能优化

部署完成后，进行适当的安全加固和稳定性优化是保证服务长期可靠运行的关键。

### 7.1 配置防火墙（安全组）最小化原则

回顾并收紧 ECS 安全组规则，遵循最小权限原则：

- **SSH (22 端口)：** 将源地址从 `0.0.0.0/0` 修改为您个人的固定公网 IP 或 IP 段。
- **OpenClaw Gateway (18789 端口)：** 由于我们使用飞书长连接，Gateway 绑定在 `127.0.0.1`，此端口无需对公网开放。可以删除或修改为仅允许本地 `127.0.0.1` 访问。

### 7.2 常用调试命令

```bash
# 重启 Gateway
openclaw gateway restart

# 查看 Gateway 状态
openclaw gateway status

# 查看渠道状态
openclaw channels status

# 查看飞书配置
openclaw config get channels.feishu

# 查看 Gateway 日志
tail -f ~/.openclaw/logs/gateway.log
```

---

## 八、常见问题排查（QA）

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| 安装后输入 `openclaw` 提示找不到命令 | npm 全局 bin 目录未加入系统 PATH | 执行：`export PATH="$(npm prefix -g)/bin:$PATH"`，并将此行添加到 `~/.bashrc` 或 `~/.zshrc` 中。 |
| Gateway 启动失败，提示端口占用 | 端口 18789 被占用或旧 Gateway 进程残留 | 执行：`pkill -9 -f openclaw`，等待 2 秒后 `openclaw gateway start`。 |
| 飞书收不到消息，或发送后无回复 | 1. 飞书应用未选择"长连接"模式；<br>2. 权限未开通；<br>3. OpenClaw 配置的 App ID/Secret 错误；<br>4. **domain 配置错误**（关键！） | 1. 检查飞书后台"事件与回调"配置；<br>2. 检查"权限管理"是否已开通；<br>3. 核对 `openclaw config` 设置，并查看飞书连接日志；<br>4. **确保 domain 设置为 "feishu"（不是 "lark"）！** |
| AI 响应速度特别慢（>100秒） | 使用了不合适的模型或网络问题 | 在火山方舟控制台将默认模型切换为 kimi-k2.5，其在日常对话和指令执行上响应更均衡。 |
| Agent Error 或配置不生效 | 配置文件格式错误，或让 OpenClaw 自动安装了损坏的 Skills | 1. 检查 `~/.openclaw/openclaw.json` 的 JSON 语法；<br>2. 手动清理配置文件中无效或缺少认证信息的 skills 配置段。 |
| 飞书连接日志显示 "domain=lark" 但无法连接 | **domain 配置错误！** 飞书用户应该用 "feishu" | 执行：`openclaw config set channels.feishu.domain "feishu"`，然后重启 Gateway。 |

---

## 六、安装完整浏览器（可选但推荐）

OpenClaw 支持浏览器自动化功能，包括网页截图、点击、表单填写、JavaScript 执行等。为了在无 GUI 的服务器环境中使用这些功能，需要安装 Chromium 浏览器和虚拟显示框架 xvfb。

> **注意：** 此步骤不是必须的，但强烈推荐安装以获得完整的浏览器自动化能力。如果您只需要基本的网页内容抓取，可以使用已有的 `web_fetch` 工具。

### 6.1 安装 Chromium 和 xvfb

在 Ubuntu/Debian 系统上，执行以下命令：

```bash
# 更新包列表
apt-get update

# 安装 Chromium（通过 snap 包系统）和 xvfb（虚拟显示）
apt-get install -y chromium-browser xvfb
```

**⚠️ 重要提醒：**
- 安装过程可能较慢（约 1 小时），因为需要下载 snap 包（core22, core24, chromium）
- 网络速度慢时会卡在下载阶段，请耐心等待
- 安装完成后，Chromium 会以 snap 形式安装，路径为 `/usr/bin/chromium-browser`

### 6.2 配置 OpenClaw 使用 Chromium

编辑 OpenClaw 配置，添加浏览器设置：

```bash
# 设置浏览器可执行路径
openclaw config set browser.executablePath "/usr/bin/chromium-browser"

# 启用无头模式（无 GUI）
openclaw config set browser.headless true

# 添加 noSandbox 参数（snap 版 Chromium 需要）
openclaw config set browser.noSandbox true
```

### 6.3 重启 OpenClaw 网关应用配置

```bash
openclaw gateway restart
```

### 6.4 验证浏览器功能

测试浏览器是否正常工作：

```bash
# 直接测试 Chromium
timeout 10 xvfb-run --auto-servernum chromium-browser --headless --no-sandbox --dump-dom https://example.com

# 或通过 OpenClaw 测试（需要等网关完全启动）
# 使用 browser 工具测试
```

### 6.5 常见问题与解决方案

#### **问题 1：安装卡在 snap 下载**
- **症状：** 长时间卡在 `core22`、`core24` 下载
- **原因：** 网络连接海外 snap 服务器慢
- **解决方案：** 耐心等待，或更换网络环境

#### **问题 2：OpenClaw 浏览器服务连接失败**
- **症状：** `Can't reach the OpenClaw browser control service`
- **原因：** 浏览器服务未正确启动或配置问题
- **解决方案：**
  1. 检查配置：`openclaw config get browser`
  2. 确保 `noSandbox: true` 已设置
  3. 重启网关：`openclaw gateway restart`
  4. 等待 2-3 分钟让服务稳定

#### **问题 3：权限错误**
- **症状：** `AppArmor policy prevents this sender`
- **原因：** snap 版 Chromium 的沙盒限制
- **解决方案：** 这些错误通常不影响基础功能，可以忽略

#### **问题 4：Chromium 能运行但 OpenClaw 无法集成**
- **症状：** 手动启动 Chromium 成功，但 OpenClaw 集成失败
- **解决方案：**
  1. 使用替代方案：先用 `web_fetch` 工具
  2. 手动编写脚本结合 Chromium 和 xvfb
  3. 等待 OpenClaw 版本更新修复集成问题

#### **问题 5：snap Chromium 与 OpenClaw 不兼容（Ubuntu 22.04 常见问题）**
- **症状：** `Can't reach the OpenClaw browser control service` 即使手动启动 Chromium 成功
- **根本原因：** Ubuntu 22.04 默认 Chromium 是 snap 包，其 AppArmor 安全限制干扰 OpenClaw 启动和监控浏览器进程
- **错误信息示例：**
  ```
  AppArmor policy prevents this sender from sending this message to this recipient
  Failed to call method: org.freedesktop.DBus.ListActivatableNames
  ```
- **解决方案（attachOnly 模式）：**
  1. 启用 attachOnly 模式（不自动启动浏览器）：
     ```bash
     openclaw config set browser.attachOnly true
     ```
  2. 创建 systemd 用户服务自动启动浏览器（见下方 6.7 节）
  3. 设置默认 profile 为 openclaw：
     ```bash
     openclaw config set browser.defaultProfile "openclaw"
     ```
  4. 重启 OpenClaw 网关应用配置：
     ```bash
     openclaw gateway restart
     ```

### 6.7 使用 attachOnly 模式（snap Chromium 专用方案）

如果遇到问题 5（snap Chromium 不兼容），请按照以下步骤配置 attachOnly 模式：

#### 6.7.1 更新 OpenClaw 配置
```bash
# 启用 attachOnly 模式（不自动启动浏览器）
openclaw config set browser.attachOnly true

# 设置默认 profile 为 openclaw
openclaw config set browser.defaultProfile "openclaw"

# 确保其他配置正确
openclaw config set browser.executablePath "/usr/bin/chromium-browser"
openclaw config set browser.headless true
openclaw config set browser.noSandbox true
```

#### 6.7.2 创建 systemd 用户服务自动启动浏览器
```bash
# 创建服务目录
mkdir -p ~/.config/systemd/user

# 创建服务文件
cat > ~/.config/systemd/user/openclaw-browser.service << 'EOF'
[Unit]
Description=OpenClaw Browser (Chromium CDP)
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/xvfb-run --auto-servernum chromium-browser --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --remote-debugging-address=0.0.0.0 --disable-dev-shm-usage about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

# 重新加载 systemd 配置
systemctl --user daemon-reload

# 启动服务
systemctl --user start openclaw-browser.service

# 设置开机自启（可选）
systemctl --user enable openclaw-browser.service
```

#### 6.7.3 验证浏览器服务
```bash
# 检查 systemd 服务状态
systemctl --user status openclaw-browser.service

# 检查 CDP 端口是否可访问（等待 5-10 秒让浏览器启动）
sleep 5
curl -s http://127.0.0.1:18800/json/version | head -3

# 检查 OpenClaw 浏览器工具状态
openclaw browser status
```

#### 6.7.4 attachOnly 模式限制
- ✅ **功能完整**：所有浏览器工具（截图、点击、导航等）都可用
- ⚠️ **需要手动管理**：浏览器崩溃需要 systemd 自动重启
- ⚠️ **端口固定**：使用固定端口 18800，不能动态分配
- ✅ **稳定可靠**：systemd 管理生命周期，保证服务持续运行

#### 6.7.5 替代方案：安装 Google Chrome
如果不想使用 attachOnly 模式，可以安装 Google Chrome 替代 snap Chromium：
```bash
# 下载 Google Chrome .deb 包
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb

# 安装
dpkg -i google-chrome-stable_current_amd64.deb
apt --fix-broken install -y  # 修复依赖

# 更新 OpenClaw 配置
openclaw config set browser.executablePath "/usr/bin/google-chrome-stable"
openclaw config set browser.attachOnly false  # 改回自动启动模式
```

### 6.8 浏览器功能对比

| 工具 | 功能 | 是否需要安装 | 推荐场景 |
|------|------|--------------|----------|
| **web_fetch** | 轻量网页内容抓取 | 否（内置） | 快速获取已知 URL 的文本内容 |
| **browser** | 完整浏览器自动化 | 是（需 Chromium+xvfb） | 截图、点击、JavaScript 处理、表单填写 |

**建议安装浏览器**，即使暂时使用 attachOnly 模式，因为：
1. 未来 OpenClaw 更新可能修复 snap 兼容性问题
2. attachOnly 模式下所有功能都可用
3. 为其他应用提供浏览器环境
4. 可以随时切换到 Google Chrome 方案

---

## 恭喜！

如果您已按照教程完成所有步骤，并成功在飞书中与您的 OpenClaw 助手对话，那么您已经拥有了一个 7x24 小时运行在云端的安全、私有的 AI 助手！

### 下一步探索

- **探索技能库（ClawdHub）：** 通过 `openclaw skills search` 命令探索社区技能，或访问 ClawdHub 网站。
- **配置其他平台：** 参考 OpenClaw 官方文档，尝试配置 Telegram、WhatsApp 等其他聊天平台。
- **成本管理：** 测试完成后，如果暂时不用，可以在火山引擎控制台停止 ECS 实例以停止计费（仅停止，不释放），或直接释放实例以彻底删除。定期检查费用中心，设置预算报警。

---

## 修订日志

- **v2.1 (2026-02-26):** 添加浏览器安装详细经验
  - ✅ 添加完整的浏览器安装配置章节（第6章）
  - ✅ 记录 snap Chromium 安装缓慢问题及解决方案
  - ✅ 添加 attachOnly 模式配置（解决 snap 不兼容问题）
  - ✅ 添加 systemd 服务创建步骤
  - ✅ 记录所有常见错误及解决方法
  - ✅ 提供 Google Chrome 替代方案

- **v2.0 (2026-02-26):** 根据真实部署经验修订
  - ✅ 移除所有明文 API Key 和 App Secret
  - ✅ 改用 `volcengine-plan` 内置支持，弃用旧的 `models.providers` 方式
  - ✅ 添加关键的 `domain` 配置说明（飞书 vs Lark 的坑）
  - ✅ 简化配置步骤，使用 CLI 而不是手动编辑 JSON
  - ✅ 更新 QA 部分，添加 domain 相关问题排查
