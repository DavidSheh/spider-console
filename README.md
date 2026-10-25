# console —— 手机端控制台（ADR-0012）

一个**没有后端**的单页应用：直接调 Gitee API 读写你的私有仓。

- **页面不含任何内容**，公开托管也读不到你的帖子（没有令牌就只有登录框）；
- 数据面直连 `gitee.com`（国内 CDN），只有空壳本身需要一次加载；
- 唯一凭据是 Gitee 私人令牌，只存在手机上（可选 PIN 加密）。

## 1. 部署空壳（三选一，都不用域名）

先看一眼实测（本机，和手机同一网络）：

| 域名 | 结果 |
| --- | --- |
| `github.com` | ❌ 超时（建不了仓、也申请不了令牌） |
| `*.github.io` | ✅ 0.5s |
| `vercel.app` / `*.pages.dev` | ❌ 不通 |
| `app.netlify.com/drop`、`netlify.app` | ✅ 0.43s |
| `pages.edgeone.ai`、`edgeone.app` | ✅ 0.8~1.5s（腾讯 CDN） |
| `davidsheh.gitee.io` | 404（Gitee Pages 2024-05 已下线） |

于是有三个选项：

**A. Netlify Drop（最快，免登录）**

打开 <https://app.netlify.com/drop>，把 `console/` 文件夹（或只把 `index.html`）拖进去，
立刻得到一个 `https://xxx.netlify.app` 链接。建议随后注册认领，避免匿名站点被回收。

**B. EdgeOne Pages（腾讯，国内最快）**

<https://pages.edgeone.ai/zh/drop> → 上传 `console/` 里的文件 → 得到 `https://xxx.edgeone.app`。
需要腾讯云账号（实名）。免费额度够个人用。

**C. 不托管（最私密）**

把 `console/index.html` 这一个文件传到手机（微信文件传输助手 / 华为云盘），用浏览器打开即可：
空壳不联网加载，数据直接读 `gitee.com`。换版本时要重传一次；`file://` 下没有 Service Worker（已做降级）。

> 打包上传用：`cd console && zip -r ../console.zip index.html sw.js manifest.webmanifest icon.svg`

## 2. 建令牌（建议用专用账号）

Gitee 令牌**没有仓库级权限**，所以推荐「专用协作者账号」这条路：

1. 新建一个 Gitee 小号，把主号仓库 `davidsheh/spider` 加它为**协作者**；
2. 用这个小号建私人令牌：Gitee → 设置 → 私人令牌 → 只勾 `projects`；
3. 令牌泄露时只影响这一个仓，且随时可撤。

## 3. 手机上用

1. 浏览器打开 `https://<账号>.github.io/spider-console/`；
2. 填仓库 `davidsheh/spider`、粘贴令牌；PIN 可留空（明文存在本机）或设 4~6 位（加密保存）；
3. 浏览器菜单 →「添加到主屏幕」，之后像 App 一样打开。

数据源有两种（对应 `config.yaml` 的 `mobile.provider`）：

- **gitee**：直连主仓 `davidsheh/spider`。令牌是账号级权限，泄露等于全仓可写。
- **github**（推荐）：只连私有小仓 `davidsheh/spider-mobile`，用 fine-grained PAT
  （仅该仓 `Contents: Read and write`）。桌面 `scripts/mobile_sync.py` 负责推 feed、拉 picks。

每天的动线：**看 → 把不是干货的点成「已剔除」→ 保存**（默认全保留，所以多数日子一次都不用点）：

- **保存**：把 `daily/<date>/picks.md` 写回仓库，并写入 `# confirmed:` 时间戳；
- **投递清洗**：把保留项写进 `data/state/clean_queue.json`，桌面下次 `./run.sh` 消费，
  效果与在电脑上清洗完全一致（ADR-0012 第 6 条）。

桌面侧看结果：

```bash
git pull
python3 scripts/promote.py --date 2026-10-24    # 读 picks.md 的 [x]
# 然后在 pi 会话里说「清洗今天的」
```

## 注意

- `picks.md` **只由人和这个网页写**，机器只在缺失时生成一次；所以别在电脑上「重新渲染」期待它更新。
- 仓库刚被桌面推过时，保存可能撞 409，页面会自动重取 `sha` 重试一次。
- 令牌在浏览器里，任何能打开这台手机的人都能用；PIN 只能挡住「拿到备份文件」的场景。
