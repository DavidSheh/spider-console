# console —— 手机阅读器（无后端、无依赖）

`index.html` 是 inline HTML / CSS / JavaScript 单页应用。浏览器用用户自己的令牌直接读写 Gitee / GitHub 私有仓；公开托管的页面与 Service Worker **只含空壳**，不附带正文、关注名单或凭据。

## 阅读与决策

- 默认进入**干货**，可切换**水贴 / 收藏**，过滤为**默认（全部）/ 已读 / 未读**。
- 正文、引用、转发链和评论始终展开；18px 字号、舒适行距、最大 760px 阅读宽度。顶部是紧凑 sticky 日期 / 视图控制，帖子操作在帖尾，没有固定底部操作栏。
- **干货 → 删除到水贴**；**水贴 → 捞回干货**。收藏只在干货帖尾提供，可取消，只标记，不等于归档。水贴仅提供捞回 / 销毁（及已读标记）。
- 收藏视图只展示 **bucket=good 且 favorite=true** 的帖子；移入水贴保留收藏标记，但从收藏视图隐藏，捞回后重新出现。
- 水贴的**永久销毁**必须确认：该全局 id 在所有日期、所有视图中永不展示，不可恢复，也绝不勾选或新增投递。此操作存墓碑，不删除原始业务文件。后端渲染与手机推送也会剔除销毁正文；本机旧缓存可能仍含正文，但不会展示。
- 默认桶取 feed 的 `it.bucket`（`good` / `water`）；旧 feed 没有该字段时默认 `good`。旧 picks 只有**显式 `[ ]`** 会回落到水贴，缺失 id 或 `[x]` 不会把新 feed 的水贴变成干货。阅读状态中的显式 bucket 优先。

**历史 feed 只含过线干货，页面无法凭空补出水贴正文。需要桌面重新渲染该日期的 feed（以及 GitHub 数据面的推送），才可查看与回捞历史水贴。** 渲染不会重建或覆盖已有 picks。

### 已读

只有**帖尾 sentinel 完整进入视口并连续停留至少 1.5 秒，且 document 可见**才自动标记已读。首屏看到长文开头不会误标整篇已读；切入后台中断计时。帖尾也可手工标为已读 / 未读，不支持 IntersectionObserver 的浏览器使用手工标记。

在未读视图中变为已读时，**当前卡片留在原处**，仅更新帖尾标记，不因过滤瞬移。明确切换视图 / 过滤、切日期或刷新后，重新应用过滤。

### 明确保存与投递（顶部「更多」）

1. 阅读 / 收藏 / 桶变化立即暂存本机，**不自动写 `confirmed`**。
2. **同步阅读状态**：仅合并保存 `data/state/reading_state.json`，不会确认 picks。
3. **确认并保存决策**：点击后再次明确确认，先同步阅读状态，再把当前日期 picks 写回仓库，写入固定 UTC+8 的 `# confirmed:`。即使当天没有改动，也可明确确认默认选择。
4. **投递清洗**：需再次确认，并联网核对最新 picks 已保存、已确认，且与显式桶决策一致。仅将当前 feed 中已勾选干货写入 `data/state/clean_queue.json`，不投递水贴 / 销毁帖；重试按最新队列求并集，不覆盖他端新增项。

投递只是意图：桌面消费队列、生成归档骨架，后续仍需 Agent / 人做高保真清洗。**原有 shadow_mode 与未确认保护不变**；页面不会关闭影子模式或调用 `--force`，收藏不绕过这些保护。

## 阅读状态与冲突协议

状态按 `wb:<id>` / `xq:<id>` 跨日期生效，写入所选私有仓：

```json
{
  "version": 1,
  "items": {
    "wb:123": {
      "bucket": "water",
      "bucket_updated_at": "2026-10-25T12:00:00.000Z",
      "favorite": true,
      "favorite_updated_at": "2026-10-25T11:00:00.000Z",
      "read_at": null,
      "read_updated_at": "2026-10-25T12:05:00.000Z"
    }
  }
}
```

每条只存在被改变的字段及其时间戳，不补默认值。独立合并 bucket、favorite、read_at，时间戳均为 ISO UTC；较新字段获胜，同时间戳取后者（与 `reading_state.py` 一致）。**destroyed 永远优先，已有墓碑不能恢复**。标为未读使用 `read_at: null` 与新的 `read_updated_at`，不能靠删除字段表示。

每次写入 / 冲突重试重新读取远端，用最新远端与本机状态按字段合并后再 PUT，不能只换 sha 重传旧全文。远端解析采用 **fail-closed**：非法 root / items / id / 帖子状态 / 字段类型，或字段时间戳缺失、不是 ISO UTC，都拒绝整份状态并保留本机缓存提示；即使是缺时间戳的 destroyed 也不丢弃后写回。只有真实 404 视为文件缺失，API 失败或损坏 JSON 绝不当成空状态 PUT。未知字段忽略。picks 每次重试也重新 build：保留远端已有的非 feed 行与描述，patch 本机改过的 id / 显式阅读桶状态，并添加新 feed id（good 勾选，water / destroyed 不勾选）；销毁墓碑同时禁止旧行重新勾选。

- **Gitee**：手机直连主仓，状态与 picks / 队列写入该仓。
- **GitHub（推荐）**：手机只连私有数据面仓（例如 `davidsheh/spider-mobile`）；桌面 `scripts/mobile_sync.py` 的 pull / push 都会对全局 `reading_state.json` 双向字段合并，不限今天 / 昨天；push 会从 feed 剔除销毁正文。feed 由桌面推送，picks / 清洗队列由桌面拉回。后端 promote 允许人工回捞的 reject，但跳过 state bucket 为 water / destroyed 的条目。

### Gitee contents 接口的两个差异（真机踩过）

1. **不存在的路径返回 `200` + `[]`，不是 `404`**。把 `[]` 当文件读，`content` 就是 `undefined`，`atob('undefined')` 抛
   `The string to be decoded is not correctly encoded`。因此「文件缺失」必须是 `null` **或**数组（目录列表）。
   症状：`data/state/reading_state.json` 还没在仓库里时，每次开页都提示「阅读状态未取到」，
   「确认并保存决策」也必然失败——而且它永远写不出去，形成死锁。
2. **新建文件用 `POST /contents/{path}`，`PUT` 只用于更新且要带 `sha`**（GitHub 两者都用 `PUT`）。
   首次写入 `reading_state.json` / `clean_queue.json` 时必须走 `POST`，否则更新接口返回失败。

客户端统一用 `fileEntry()` / `fileText()` 把「缺失」与「文件」分开，并在失败时给出明确错误，
绝不把 `undefined` / 非 base64 内容交给 `atob`，也绝不把 API 异常当成远端空文件覆盖。

## 离线与本机隔离

本机 localStorage 以 **provider / owner / repo / branch** 为 scope，保存 feed、picks、全局阅读状态及**每个日期尚未确认保存的选择**。旧版不分仓的 `spider.feed.<date>` 缓存不会自动导入，避免串仓。

- 离线可读已缓存日期、操作已读 / 收藏 / 桶；保存失败不丢本机状态。顶部明确提示待同步与待确认日期。
- 刷新 / 切日期前暂存并提示；重新打开会恢复上次日期及其待确认改动。离线 picks 读取失败不会当成空文件覆盖缓存。
- 本版不会后台自动网络保存：联网后在「更多」中同步阅读状态，或明确确认并保存当前日期决策。其他日期的待确认选择需逐日确认。
- 浏览器可能清除存储，隐私模式 / 配额不足也可能写入失败；页面会明确告警，不能把 localStorage 当永久备份。
- PIN **只加密令牌**，不加密本机正文 / 状态缓存。请保护设备，勿公开业务文件或带凭据的输出。

## 安装与全屏

可将 `console/` 空壳单独部署到 HTTPS 静态站点（GitHub Pages、Netlify Drop、EdgeOne Pages 等），不要包含 `data/`、`daily/`、`archive/`。编辑源码本身不会发布，GitHub 一键发布命令见下节。

1. 打开空壳，选择 Gitee / GitHub，填写 `owner/repo`、可选分支及令牌。
2. 推荐 GitHub fine-grained PAT，仅授权数据面私有仓 `Contents: Read and write`。Gitee 私人令牌勾 `projects`，属账号级权限，建议用只协作该仓的专用账号。
3. PIN 可选；留空明文保存在本机，设置后每次打开解锁。
4. 浏览器菜单 → **添加到主屏幕**，从主屏幕启动。

manifest 使用 `display: fullscreen`，`display_override: [fullscreen, standalone]`，保留 Apple web-app-capable。实际全屏能力取决于浏览器 / 系统，普通浏览器不能无交互强制全屏。「更多 → 全屏阅读」仅在用户点击时调用 requestFullscreen，失败会提示安装到主屏幕；**不开页请求全屏**。Apple 平台可能仍以 standalone 呈现。

也可直接传 `index.html` 到手机用 `file://` 打开（浏览器需允许 API 访问），但没有 SW / PWA 安装，换版需重传。HTTPS 下生产 SW 使用 **spider-shell-<内容版本>**，源码直接运行使用开发缓存；只缓存首页、manifest 与图标，API、同源业务文件和版本检查均不缓存。业务离线缓存只在按仓库隔离的 localStorage 中。

## GitHub Pages 一键发布

从 spider 仓库根目录执行；不用网页登录或 `gh` CLI。目标读取 `config.yaml` 的 `publish`
（本机覆盖同其他配置），默认 `DavidSheh/spider-console / main / root`。

```yaml
publish:
  github_repo: "DavidSheh/spider-console"
  branch: "main"
  token_env: "SPIDER_PAGES_TOKEN"
  token_file: "~/.spider_pages_token"
```

在本机用户目录创建 `.spider_pages_token`（不是 `.txt`），UTF-8，仅一行实际 PAT。
Windows 路径通常为 `C:\Users\<用户名>\.spider_pages_token`。不把令牌写进 YAML、命令行、
手机页面或仓库。推荐 fine-grained PAT，仅选空壳仓、Contents 读写 + Pages 读写。
**这不是手机数据面的令牌，不要在手机填写它。**

凭据优先级：显式 `--token-file` > `SPIDER_PAGES_TOKEN` > 配置中的 token_file；不自动读取
`GITHUB_TOKEN` / `GH_TOKEN` / `~/.spider_gh_token`。首次准备令牌需要用户在 GitHub 创建，日常发布不需要浏览器登录。

```bash
python -B console/publish.py --dry-run  # 完全离线，不读 PAT；列文件、大小、版本
python -B console/publish.py --check    # 联网只读；检查来源、HTTPS、远端差异
python -B console/publish.py            # 真正写远端、请求构建并等待部署验证
```

发布仅上传五个源空壳文件，以及内存生成的 `release.json`、`.nojekyll`；不递归上传 console
目录，不上传 publish.py，更不上传任何业务数据或凭据。文件缺失、符号链接、疑似 PAT 都会阻止发布。
一次提交所有变化，保留远端其他文件；没变化不提交/不请求新构建，遇并发不强推。
不要求本地先 commit，不操作 spider 本地 Git，也不会提交已有暂存文件。

目标仓须已存在且公开、Pages 须是所配分支/root 来源。本工具不建仓、不改来源、域名或 HTTPS
开关。默认正式上传前必须验证实际站点的 HTTPS/TLS；证书错误、重定向或网络失败会停止。
若 Pages 未强制 HTTPS，会提醒手机只使用 HTTPS 链接；不能在公开 HTTP 页面中输入令牌。

若桌面无法连接站点，但用户已在手机确认同一 HTTPS URL 可以访问，可**显式**使用下列模式。
它另要求 GitHub API 的证书记录为 approved、未过期且覆盖该域名；不会关闭 API 的 TLS 验证。
不能把 HTTP 地址、其他站点或未经用户确认的地址拿来跳过默认预检。

```bash
# 先只读检查，再发布；URL 必须与 Pages 实际 HTTPS 地址完全一致
python -B console/publish.py --check --api-only --confirmed-https-url https://codebug.top/spider-console/
python -B console/publish.py --api-only --confirmed-https-url https://codebug.top/spider-console/
```

该模式只确认目标提交的 Pages 构建完成，**不会声称桌面线上 HTML/SW 验证成功**；手机仍需检查
「更多」中的版本号及阅读功能。默认命令依旧要求桌面 HTTPS 实际探测；此例外不能作为常驻配置。

默认等待 180 秒轮询目标提交的构建（单个网络请求最多 20 秒 socket 超时；DNS/网络可能使总时长更长）。
`--timeout 300` 可延长轮询等待；`--no-wait` 只是不等构建，不能据此认为已部署。
成功需构建提交匹配，且线上 release/HTML/SW 与本次版本一致。提交后构建失败或超时会明确提示，
先检查不要盲目重复发布。

版本按内容生成，不随发布时间或 Windows 换行变化。发布自动生成生产缓存版本；手机在「更多」
查看版本并检测更新，新版只提示，刷新必须手动触发。未确认选择、未同步阅读状态、其他日期草稿、
同步进行中或本机存储异常时不能刷新。新版本不会自动删除本机业务缓存。
旧 v2 页面没有新版检测逻辑，第一次升级可能需要先处理草稿再手动刷新；之后才有新版提示。
HTTP 与 HTTPS 的本机存储互相隔离，若原先使用 HTTP，请先妥善处理旧页面草稿，不能假定它们
自动出现在 HTTPS 中。不要在 HTTP 页面输入令牌。

## 离线验证

仓库根目录运行，无依赖、无联网、无真实业务文件写入：

```bash
node tests/test_mobile_console.cjs
```

Node VM 夹具覆盖桶默认值、收藏过滤、字段合并与重试、销毁、picks 行保留 / 重新构建、固定 UTC+8 确认、本机隔离与跨日期草稿、帖尾计时 / 页面可见性、未读卡片保留、明确确认与投递保护、SW 空壳白名单。它不替代真机的滚动 / 全屏 / PWA 安装和 API CORS 验证。
