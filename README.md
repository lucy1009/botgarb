# Botkeep Grab

在 GitHub Actions 上自动轮询 [Botkeep](https://botkeep.cloud) 的免费服务器容量,有名额就立即创建 Node.js 服务器,并通过 Telegram 通知结果。

## 功能

- 登录 Botkeep,轮询 `creation-availability`,有容量立即创建服务器
- 节点白名单 / 黑名单:抢到不想要的节点自动删除并冷却后重抢
- 抢到后自动冷却 24 小时,期间只检查机器是否还在
- 登录 Cookie 用 AES-256-GCM 加密后缓存,避免每次重新登录
- 可选:通过 `NODE_LINK`(vless / vmess / trojan / ss / socks5 / http)走代理出口,由 sing-box 转成本地 HTTP 代理
- 日志脱敏,可安全用于公开仓库

## 文件结构

```
.
├── grab.mjs                 # 主脚本 (Node.js 20+)
├── package.json             # 依赖 undici(代理模式使用)
├── scripts/
│   ├── start_proxy.py       # 启动 sing-box 本地代理,写入 LOCAL_PROXY / PROXY_IP
│   └── node_proxy.py        # 节点链接解析 + sing-box 配置生成(与上一个同目录)
├── grab.yml                 # 工作流,需移动到 .github/workflows/grab.yml
└── README.md
```

## 部署步骤

### 1. 创建仓库并上传文件

1. 在 GitHub 新建仓库(公开仓库的 Actions 分钟数免费;私有仓库有月度额度,每 5 分钟一次会很快用完)。
2. 解压压缩包,把里面的 `grab.mjs`、`package.json`、`README.md`、`grab.yml` 和 `scripts` 文件夹拖到仓库的 **Add file → Upload files** 页面,提交到 `main` 分支。
3. 点开仓库里的 `grab.yml` → 右上角铅笔图标(Edit)→ 把顶部文件名改成 `.github/workflows/grab.yml`(输入 `/` 会自动建文件夹)→ **Commit changes**。GitHub 网页上传不接受以点开头的路径,所以必须这样移动。
4. 到 Actions 页确认左侧出现 `grab`。

### 2. 配置 Secrets

仓库 **Settings → Secrets and variables → Actions → Secrets**:

| 名称 | 必填 | 说明 |
|---|---|---|
| `COOKIE` | 二选一 | Botkeep 登录后的 Cookie,配置后可不填账号密码,见下方「获取 COOKIE」 |
| `BOTKEEP_EMAIL` | 二选一 | Botkeep 账号邮箱(与 `COOKIE` 二选一,或同时配置作为失效后的备用) |
| `BOTKEEP_PASSWORD` | 二选一 | Botkeep 账号密码 |
| `STATE_KEY` | 建议 | 任意随机长字符串,用于加密缓存里的登录 Cookie;不设则由账号密码派生 |
| `TG_BOT_TOKEN` | 否 | Telegram Bot Token(@BotFather 获取) |
| `TG_CHAT_ID` | 否 | 接收通知的 chat id;不填则只打印日志 |
| `NODE_LINK` | 否 | 代理节点链接;不填则 GitHub 机房直连 |
| `SING_BOX_SHA256` | 否 | sing-box 压缩包的 sha256,设置后下载会强校验 |

#### 获取 COOKIE

1. 浏览器登录 [botkeep.cloud](https://botkeep.cloud),打开任意控制台页面。
2. 按 `F12` 打开开发者工具 → **Network(网络)** → 刷新页面。
3. 点一个发往 `api.botkeep.cloud` 的请求(如 `session` 或 `dashboard`),在 **Request Headers** 里找到 `Cookie:` 一行,复制整行的值(形如 `name1=value1; name2=value2`)。
4. 新建 Secret,名称 `COOKIE`,粘贴该值。

登录顺序:缓存会话 → `COOKIE` → 账号密码。登录态会加密缓存并随服务端刷新而更新。`COOKIE` 过期后,如果配置了账号密码会自动改用密码登录;没配的话会发一条 Telegram 提醒(同一份失效 COOKIE 只提醒一次),你更新 `COOKIE` 后自动恢复。

> 请用 **Secret** 存放 `COOKIE`:它等同于登录凭据,不要放进 Variables 或提交到仓库。

### 3. 配置 Variables(可选)

**Settings → Secrets and variables → Actions → Variables**:

| 名称 | 默认值 | 说明 |
|---|---|---|
| `SERVER_NAME` | `my-project` | 服务器名称 |
| `RAM_MB` | `512` | 内存 |
| `CPU_PERCENT` | `50` | CPU 百分比 |
| `STORAGE_MB` | `1024` | 磁盘 |
| `INCLUDE_NODES` | `node10.botkeep.cloud` | 节点白名单(逗号分隔);非空时只接受这些节点,并忽略黑名单。请写完整域名 |
| `EXCLUDE_NODES` | `node2-it` | 节点黑名单(逗号分隔);要关闭黑名单可填一个不存在的名字,如 `none` |
| `RUN_SECONDS` | `240` | 单次运行轮询总时长 |
| `INTERVAL_SECONDS` | `20` | 轮询间隔,不要太小以免触发风控 |

节点匹配规则:`node1` 能匹配 `node1-it`、`node1-de`,但不会匹配 `node10-xx`。默认只接受 `node10.botkeep.cloud`,抢到其他节点会自动删除并冷却后重抢;如面板里显示的 host 格式不同,请在 Variables 里覆盖 `INCLUDE_NODES`。

### 4. 首次手动运行

1. 打开 **Actions** 页,如提示则点击启用工作流。
2. 选择 `grab` → **Run workflow**。
3. 查看日志,确认出现 `登录成功`;使用代理时还应看到 `代理就绪,出口 IP`。
4. Telegram 收到「🚀 Botkeep 抢机器服务已开启」即部署成功。

### 5. 触发方式(纯 GitHub,无需任何外部服务)

工作流由 `schedule`(每 5 分钟)触发,另支持手动 Run workflow。注意:

- GitHub 的定时任务**不保证准点**,高峰期可能延迟 5~30 分钟,偶尔会跳过一次,这是平台限制。
- 定时触发的运行视为 `worker` 来源,Telegram 启动通知每小时最多一条;手动触发每次都通知。
- 工作流末尾有保活步骤,每次定时运行都会重新启用该 workflow,避免公开仓库 60 天无活动后被 GitHub 自动停用定时任务。
- 并发限制为 1:上一次还没跑完时,新触发会排队,不会同时抢。
- 想提高命中率,可适当调大 `RUN_SECONDS`(如 `270`),但不要超过 `timeout-minutes` 的余量。

## 运行逻辑

```
触发 → 冷却中? ─是→ 跳过
        │否
        ▼
   (可选)启动 sing-box 代理
        ▼
   登录(优先复用加密 Cookie)
        ▼
   账号已有服务器? ─节点不合格→ 删除并冷却 30 分钟
        │              节点合格→ 冷却 24 小时,结束
        │无
        ▼
   每 20 秒查询容量,持续 240 秒
        │有容量 → 创建 → 检查节点 → 合格:冷却 24 小时并通知 / 不合格:删除并冷却
        ▼
   本次未抢到,等待下次触发
```

状态(冷却时间、会话、通知计数)保存在 `.cooldown/`,通过 `actions/cache` 在多次运行间传递。

## 安全说明

- 公开仓库中,脚本会对节点地址、UUID、密码等调用 `::add-mask::` 并自行脱敏日志。
- 缓存里的登录 Cookie 经 AES-256-GCM 加密,务必设置 `STATE_KEY`。
- 建议在 **Settings → Actions → General** 中限制 fork PR 的工作流运行,防止被滥用。
- 不要把任何凭据写进代码或提交到仓库。

## 关于 Fork

- 工作流里限定了 `github.repository`,只会在原仓库运行;fork 出去的仓库即使启用了 Actions,任务也会被跳过。
- Fork 不会复制 Secrets,也读不到原仓库的缓存,因此不会影响原作者的账号和状态。
- 想自己使用的人,请新建仓库并把工作流里的仓库名改成自己的,再配置自己的 Secrets。

## 常见问题

**日志提示"冷却中,本次跳过"** 
正常,说明刚抢到或刚删除过机器。冷却记录在缓存里,想强制重抢可到 Actions → Caches 删除 `grab-state-*`。

**代理自检失败** 
检查 `NODE_LINK` 是否可用、协议及传输方式是否在支持范围内(`tcp / ws / grpc / httpupgrade`;ss 不支持 plugin)。

**定时运行间隔不是 5 分钟** 
GitHub 定时任务会延迟,属于正常现象;可到 Actions 页确认最近的运行记录。

**频繁 429** 
调大 `INTERVAL_SECONDS`。脚本遇到 429 会自动加倍等待。

**收到"读不到节点"** 
脚本在无法判断节点时会保守退出,避免误删或误建,稍后会自动重试。

## 免责声明

本项目仅供个人学习使用。高频轮询可能违反 Botkeep 的服务条款,账号存在被限制的风险,请自行评估并控制频率。
