# auto_v2ex  V2EX 每日签到脚本

> 站点：https://v2ex.com  
> 目标：每天领「每日登录奖励」（铜币），结果推送到 Telegram

---

## 0、脚本做了什么

1. 先读**加密**缓存 `v2ex_cookie.enc.json`（AES/Fernet，存放在**公开库** `auto_scripts/v2ex/`），有效就直接签到 —— **免验证码**
2. 缓存无效 / 不存在 → 用户名密码登录：
   - 解析登录页动态字段名（V2EX 每次会话字段名都是随机哈希）
   - 抓 `/_captcha` 图片，用 **ddddocr** 识别（实测大小写不敏感）
   - 识别失败自动换图重试，默认 8 次
3. 登录成功 → 把 A2 Cookie **加密后**写入 `v2ex_cookie.enc.json`，workflow 自动提交回**公开库**，
   下次运行免验证码
4. `GET /mission/daily` → 取 `redeem?once=` 链接 → 领取奖励
5. 解析：今日奖励铜币（含日期）、账户余额（银币 / 铜币）
6. Telegram 推送

---

## 1、环境变量

> ⚠️ 下面所有示例都是占位符。真实账号密码 / token **只填到 GitHub Secrets**，
> 不要写进仓库文件、不要写进 README、不要提交。

### 1.1 `V2EX_BATCH`（必填）

```
V2EX_BATCH='YOUR_EMAIL,YOUR_PASSWORD
username2,password2,<tg_bot_token>,<tg_chat_id>'
```

**上面那对单引号是你自己在 GitHub Secrets 里加的时候不要写进去的**

| 列数 | 格式 |
| --- | --- |
| 2 列 | `username,password`（用全局 TG） |
| 4 列 | `username,password,tg_bot_token,tg_chat_id` |

### 1.2 `V2EX_COOKIE`（强烈推荐）

浏览器 F12 → Application → Cookies → `v2ex.com` → 复制 `A2` 的值：

```
A2=2|1:0|10:1792...
```

填了它 → 优先走 Cookie，正常情况**完全不碰验证码**；万一失效会自动回退账号密码登录。

> **登录回退顺序（脚本行为）**：
> 1. `V2EX_COOKIE`（Secret）→ 2. 本地加密缓存 → 3. **Cookie 全失效就自动改用账号密码 + 验证码登录**
>
> Cookie 失效**不会单独发告警**，就是按上面顺序往下走，正常出签到报告。
>
> **验证码 OCR 实测（2026-10-25，Actions 真实跑过，8 张人工标注样本）**：
>
> | 方案 | 单张命中率 |
> | --- | --- |
> | `denoise`（中值滤波+放大3x）+ `ddddocr beta` | **25%（2/8，最优）** |
> | 原图 + `ddddocr beta` | 12% |
> | 原图 / 放大 / Otsu 二值化 + std | 0% |
> | 多方案置信度投票 | 0%（置信度和长度先验都救不回来） |
>
> 单张只有 ~25%，但脚本会**换图重试**：默认 `V2EX_OCR_RETRIES=15`，理论成功率 ≈ 98%。
> 真实登录探针实测：**第 8 次尝试登录成功**，耗时 23s。
> 所以 Cookie 失效后回退账号密码是**真的能救回来**的，只是慢一点。

> 配了有效 Cookie 后实测：`🍪 使用Secret Cookie：有效 → ☑️ 今日奖励已领取 → ✅ TG 发送成功`。

### 1.3 Cookie 缓存加密（`V2EX_COOKIE_KEY` / `V2EX_COOKIE_PASS`）

**落盘的 Cookie 一律加密，绝不写明文。** 两种方式二选一：

**方式 A：Fernet 密钥（推荐）**

```bash
python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())"
# 输出形如：Yh3n...（44 位 urlsafe base64）
```

```
V2EX_COOKIE_KEY=YOUR_FERNET_KEY
```

**方式 B：口令（PBKDF2-SHA256，390000 次迭代派生）**

```
V2EX_COOKIE_PASS=YOUR_LONG_PASSPHRASE
```

行为约定：

| 场景 | 结果 |
| --- | --- |
| 配了密钥 | Cookie 加密写 `v2ex_cookie.enc.json`（权限 `0600`），文件里没有明文 Cookie / 邮箱；CI 会把它放到公开库 `v2ex/` 下 |
| 没配密钥 | **完全不落盘**，Cookie 只在本次进程内存里，日志会提示 |
| 密钥不对 | 解密失败 → 视为无缓存，走密码登录，不会崩 |
| 存在旧版明文 `v2ex_cookie.json` | 自动加密迁移并删除明文文件 |

加密文件长这样（`data` 是密文）：

```json
{
  "v": 1,
  "alg": "fernet-aes128-cbc-hmac",
  "kdf": "raw-key",
  "salt": "",
  "updated": "2026-01-01 09:00:00",
  "data": "gAAAAAB..."
}
```

### 1.4 `TG_BOT_TOKEN` / `TG_CHAT_ID`（可选）

```
TG_BOT_TOKEN=<your_bot_token>
TG_CHAT_ID=<your_chat_id>
```

### 1.5 其它（可选）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `V2EX_COOKIE_FILE` | `v2ex_cookie.enc.json` | 加密缓存文件名 |
| `V2EX_OCR_RETRIES` | `15` | 验证码识别重试次数（单张 ~25%，15 次≈98%） |
| `HY2_PROXY_URL` | 空 | `hysteria2://...` 链接，配了脚本就自己拉起本地 SOCKS5（优先级最高） |
| `SOCKS_PORT` | `51081` | Hy2 本地 SOCKS5 监听端口 |
| `PROXY_URL` | 空 | 已有代理时直接填，如 `socks5h://127.0.0.1:51080` |

代理优先级：`HY2_PROXY_URL`（脚本自建）> `PROXY_URL` > 系统/环境代理。
只要配了显式代理，脚本会设 `trust_env=False`，避免被系统代理覆盖。

> ⚠️ `socks5://` / `socks5h://` 需要 **PySocks**，`requirements.txt` 已写 `requests[socks]`，
> 缺了会报 `Missing dependencies for SOCKS support`。

---

## 2、GitHub Actions 权限

仓库 → **Settings → Actions → General → Workflow permissions** → ✅ **Read and write permissions**

workflow 会做两件事（都在**公开库**里提交）：

1. 把**加密的** `v2ex/v2ex_cookie.enc.json` 提交回公开库（下次免验证码）
2. 更新 `v2ex/time.txt`（保活，避免 60 天无提交被停用定时任务）

> **为什么放公开库**：私有库可能多人共用，Cookie 放公开库反而只暴露**密文**；
> 解密密钥 `V2EX_COOKIE_KEY` 只在 GitHub Secrets 里，公开库看不到明文。
> workflow 里有防呆：明文 `v2ex_cookie.json` 一旦出现会被自动删除并 `git rm --cached`，
> `.gitignore` 也挡了一层。

---

## 3、定时任务

在 `auto_scripts/.github/workflows/v2ex-checkin.yml` 里改 cron（UTC）：

```yaml
schedule:
  - cron: "25 0 * * *"    # 08:25 北京时间（V2EX 奖励 08:00 后重置）
  - cron: "25 12 * * *"   # 20:25 北京时间
```

---

## 4、本地调试

```bash
pip install requests ddddocr pillow cryptography

export V2EX_BATCH="YOUR_EMAIL,YOUR_PASSWORD"
export V2EX_COOKIE_KEY="$(python -c 'from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())')"
python v2ex_checkin.py
```

---

## 5、常见问题

| 现象 | 原因 / 处理 |
| --- | --- |
| `❌ 登录失败（验证码识别失败或密码错误）` | Cookie 失效且回退登录时 15 次 OCR 都没蒙对（概率约 2%）。把 `V2EX_OCR_RETRIES` 调到 20，或更新 `V2EX_COOKIE` |
| `⚠️ 该账号开启了 2FA` | 脚本不处理 2FA，请配 `V2EX_COOKIE` |
| `🔒 未配置 V2EX_COOKIE_KEY / V2EX_COOKIE_PASS，为安全起见不落盘` | 正常提示：没配密钥就不缓存 Cookie，每天都会走一次密码登录 |
| `⚠️ Cookie 解密失败（密钥不匹配？）` | 换过密钥 / 从别的库拷来的缓存。删掉 `v2ex_cookie.enc.json` 重新登录一次即可 |
| `☑️ 今日已签到过了` | 正常，当天奖励已领 |
| `🎁 今日奖励: 未知` | `/balance` 首行没有「每日登录奖励」记录，稍后重跑即可 |
| Cookie 没回写私有库 | 检查 `PRIVATE_REPO_TOKEN` 是否有 `repo` 权限；私有库默认分支必须是 `main` |
| `Missing dependencies for SOCKS support` | 缺 PySocks，`pip install "requests[socks]"`（workflow 已装） |
| `🌐 IP 信息: xx.xx.xx.xx (US) [⚠️ 直连]` | 代理没生效：确认 `HY2_PROXY_URL` 已配成 Secret，且 workflow 的「📡 安装 Hysteria2」步骤成功 |

---

## 6、安全提示

- `v2ex/v2ex_cookie.enc.json` 放在公开库，但内容是**密文**；没有 `V2EX_COOKIE_KEY` / `V2EX_COOKIE_PASS`
  解不开。**密钥只放 GitHub Secrets，永远不要提交到任何仓库**（公开库尤其不能）。
- 公开库可见性意味着密文人人可下载，所以密钥强度直接决定安全：用 `Fernet.generate_key()` 生成的
  128bit 随机密钥即可；不要用弱口令当 `V2EX_COOKIE_PASS`。
- 如果怀疑泄露：V2EX → 设置 → 安全 → 「登出所有设备」，旧 Cookie 立即失效；
  再换一个 `V2EX_COOKIE_KEY` 即可。
- `.gitignore` 已忽略 `v2ex_cookie.json`（明文），避免误提交。
