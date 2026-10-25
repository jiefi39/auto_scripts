# auto_sb  烧饼论坛（sb.sb）签到脚本

> 站点：https://sb.sb （烧饼论坛）  
> 目标：每日签到领「烧饼」，结果推送到 Telegram

---

## 0、脚本做了什么

1. 打开 `https://sb.sb/login/`，用 **用户名 + 密码** 登录
2. 站点用的是 **Cap 人机验证**（`capjs.net`，含 PoW + 反自动化 instrumentation），
   所以脚本用 **真实 Chrome（SeleniumBase UC 模式 + Xvfb）** 点击验证，而不是纯 requests
3. 打开 `https://sb.sb/checkin/`，判断「今日已签到 / 立即签到」
4. 未签到 → 点「立即签到」→ 弹窗内完成人机验证 → 再次提交
5. 解析：今日奖励（站点一次性提示 `__pageFlash`）、连续签到、累计签到、钱包烧饼
6. Telegram 推送结果（可选）

---

## 1、环境变量

### 1.1 `SB_BATCH`（必填）账号

```
SB_BATCH='YOUR_USERNAME,YOUR_PASSWORD
username2,password2,1234567890:AAHxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,987654321'
```

**上面那对单引号是你自己在 GitHub Secrets 里加的时候不要写进去的**

每行一套，格式：

| 列数 | 格式 | 说明 |
| --- | --- | --- |
| 2 列 | `username,password` | 用全局 `TG_BOT_TOKEN` / `TG_CHAT_ID` 通知 |
| 4 列 | `username,password,tg_bot_token,tg_chat_id` | 该账号单独指定 TG |

### 1.2 `PRIVATE_REPO_TOKEN`（必填）

GitHub PAT（`repo` 权限），用于在 Actions 里拉取本私有库。

### 1.3 `TG_BOT_TOKEN` / `TG_CHAT_ID`（可选）

```
TG_BOT_TOKEN=1234567890:AAHxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TG_CHAT_ID=987654321
```

不填就不发通知（脚本会打印 `ℹ️ 未配置 Telegram，跳过通知`）。

### 1.4 `HY2_PROXY_URL` / `SOCKS_PORT`（可选）

```
HY2_PROXY_URL=hysteria2://505bc7c0-8f2a-44f7-bdf4-8e37a8f2e896@185.156.53.135:26160/?sni=www.bing.com&alpn=h3&insecure=1
SOCKS_PORT=51080
```

- 不设置 → 直连（GitHub Actions 美国 IP 可以直连 sb.sb）
- 设置了 → 脚本自己拉起 `hysteria client`，Chrome 走 `socks5://127.0.0.1:SOCKS_PORT`
- 注意：**中国大陆 IP 会被站点区域拦截**（返回 `not available in your region`），
  所以本机调试时一定要挂代理，Actions 里直连即可

---

## 2、GitHub Actions 权限

仓库 → **Settings → Actions → General → Workflow permissions**

选 ✅ **Read and write permissions**

（workflow 里会提交 `sb/time.txt` 保活，避免 60 天无提交导致定时任务被停用）

---

## 3、定时任务

在 `auto_scripts/.github/workflows/sb-checkin.yml` 里改 cron（UTC 时间）：

```yaml
schedule:
  - cron: "20 16 * * *"   # 次日 00:20 北京时间
  - cron: "20 4 * * *"    # 12:20 北京时间
```

---

## 4、本地调试

```bash
pip install requests seleniumbase pyvirtualdisplay pyautogui

# macOS 本机调试（先自己起好 socks5 代理，或用 HY2_PROXY_URL）
export SB_BATCH="YOUR_USERNAME,YOUR_PASSWORD"
export HY2_PROXY_URL="hysteria2://...."
export SOCKS_PORT=51080
python sb_checkin.py
```

Linux 无桌面环境时脚本会自动起 Xvfb；Actions 里是 `xvfb-run -a python sb_checkin.py`。

---

## 5、常见问题

| 现象 | 原因 / 处理 |
| --- | --- |
| `⚠️ 人机验证未通过` | Cap 的 instrumentation 判定为自动化。脚本会自动重试 3 次；仍失败就换个时间段跑（同一 IP 连续失败会被临时降权） |
| `⚠️ 登录失败` 且页面提示「人机验证未通过」 | 同上；Actions 里确认是 `xvfb-run` 启动的（headless 更容易被判定） |
| `🎁 今日奖励: 未知` | 当天已经签到过（站点不再给一次性提示），属正常；报告里仍有连续/累计/余额 |
| `💰 钱包烧饼: 未知` | `/bank/` 页面结构变动，改 `read_balance()` 里的正则即可 |
| 页面变成 `not available in your region` | 出口 IP 被区域限制，挂 `HY2_PROXY_URL` 或换 runner |
