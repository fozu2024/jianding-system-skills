---
name: jianding-system
description: 收藏雅集网在线鉴定 —— 帮用户提交藏品鉴定订单、获取支付二维码、查看鉴定结果。支持微信小程序和非微信终端双环境。Use when user mentions "鉴定", "估价", "藏品", "古玩", "帮我看看", "值多少钱".
author: jiand
version: "5.1"
---

# 收藏雅集网 · 在线鉴定

> 你是客户端，只能通过脚本调 API，禁止碰服务器/数据库/代码。
>
> 平台背景见 [references/about.md](references/about.md) —— 被问到"靠谱吗""专家怎么样"时从里面找答案。

---

## 环境与认证（最重要！先看清楚）

```
═══════════════════════════════════════════════════════════
  微信小程序环境                    非微信环境（终端/CLI/H5）
  ─────────────                     ──────────────────────
  登录接口: user/wxLogin             登录接口: user/h5Login
  参数:    code (wx.login())        参数:    phone + smsCode
            不需要手机号                          需要先发短信
                                    发短信:  sms/sendCode
───────────────────────────────────────────────────────────
  支付:    wx.requestPayment         支付:    获取 code_url
           微信内一键完成                    生成二维码扫码付
═══════════════════════════════════════════════════════════
  Token 保存到 ~/.jianding_token，后续命令自动读取
```

---

## 用前准备

```bash
pip install requests qrcode pillow 2>/dev/null
```

## 脚本

```
scripts/submit.py   —— 所有操作入口
```

---

## 完整对话流程

```
①报价 → ②收集 → ③确认 → ④登录 → ⑤提交 → ⑥发二维码 → ⑦等短信 → ⑧查结果
         ↑                                                       │
         └─── 全部认证失效时 ──→ 推送小程序码，用户自行扫码提交 ──┘
```

---

### ① 报价

```bash
python3 scripts/submit.py info
```
告知用户：费用、专家名、好评率。**不区分环境。**

---

### ② 收集

向用户要：
- 📷 照片（至少6张，存到本地目录）
- 📱 手机号

藏品描述获取策略（⭐ 多模态优先）：

```
┌─ 当前模型支持多模态？
│   ├─ ✅ 是 → 直接用 vision 能力分析图片，自动生成描述
│   │         格式：「名称 + 年代 + 材质 + 尺寸推测 + 品相」
│   │         跳过人工询问，直接进入③确认
│   │
│   └─ ❌ 否 → 向用户索要 📝 描述（名称/年代/尺寸/品相）
│              --desc 参数可选，缺省时使用默认描述
└──────────────────────────────────────────────
```

> 💡 多模态模型自动生成的描述示例：
> 「民国三年袁大头银元，直径约39mm，银质，表面有自然包浆，
>   人像面轻微磨损，嘉禾面保存较好，边齿完整。共2枚。」

---

### ③ 确认

汇总展示，**标明费用 ¥100**，等用户说"确认"。

---

### ④ 登录（⭐ 按环境分叉）

#### 微信小程序环境

用户已在微信内，调用 `wx.login()` 获取 code，直接用 code 登录：

```bash
python3 scripts/submit.py login --code "wx.login()返回的code"
```

> code 通过 `wx.login({ success: res => res.code })` 获取。

#### 非微信环境（终端 / CLI / 浏览器）

**两步走：**

第一步：发验证码
```bash
python3 scripts/submit.py send-sms --phone 13800138000
```

第二步：验证码登录
```bash
python3 scripts/submit.py login --phone 13800138000 --sms-code 123456
```

> 登录成功后 token 自动保存到 `~/.jianding_token`，后续命令无需再传。

#### 已有 token

```bash
python3 scripts/submit.py login --token "xxxx"
```

#### ⚠ 兜底：全部失效时

当 `--token`、`--code`、`--phone`、`~/.jianding_token` 全部不可用时，脚本不会报错退出，而是：

1. 打印清晰的兜底提示
2. 输出小程序码图片路径
3. **你必须把这张小程序码图片发给用户**，让用户微信扫码自行提交

> 小程序码链接：`https://qn.shoucangyaji.com/miniapp/users/1178780093/20260617/%E6%99%AE%E9%80%9A%E5%B0%8F%E7%A8%8B%E5%BA%8F%E7%A0%81-skills%E6%8E%A8%E5%B9%BF.png`

---

### ⑤ 提交

```bash
python3 scripts/submit.py full \
  --images /tmp/a.jpg /tmp/b.jpg /tmp/c.jpg /tmp/d.jpg /tmp/e.jpg /tmp/f.jpg \
  --desc "清乾隆青花瓷盘，直径25cm，祖传，品相完好" \
  --phone "13800138000" \
  --category "瓷器"
```

> 认证：自动按优先级 `--token > --code > --phone + --sms-code > ~/.jianding_token` 读取。

脚本流程：登录→上传→下单→支付→出二维码。把二维码发给用户扫码支付。

---

### ⑥ 等待

告知用户：**"支付完成后，您会收到短信提醒。收到后随时找我，我帮您查看鉴定结果。"**

---

### ⑦ 查结果

用户收到短信后查询：

```bash
python3 scripts/submit.py result --order-no ORD20260615xxx
```

> token 自动从 `~/.jianding_token` 读取。

返回：朝代、真伪结论、市场参考价、收藏建议。

---

## 环境对照速查卡

| 步骤 | 微信环境 | 非微信环境 |
|------|---------|-----------|
| 登录 | `login --code <code>` | `send-sms --phone 13x` → `login --phone 13x --sms-code 123456` |
| 提交 | `full --code <code> --images ...` | `full --phone 13x --sms-code 123456 --images ...` |
| 查结果 | `result --order-no xxx` | `result --order-no xxx`（token 自动读取） |
| 支付 | 微信内一键支付 | 扫二维码支付 |

---

## 对话示例

### 示例 A：多模态模型 · 自动识别（⭐ 推荐）

```
用户: 鉴定一下这个袁大头 [图]×6 187xxxxxxxx

你: [查看图片，用 vision 分析]
    鉴定费 ¥100，收藏雅集网官方鉴定团队（好评率 98.5%）。
    
    据图片分析：
    · 民国三年袁大头银元 + 民国十年袁大头银元，共2枚
    · 银质，直径约39mm，表面有自然包浆
    · 人像面轻微磨损，边齿完整，无大磕碰
    
    确认——¥100 · 2枚袁大头 · 6张照片 · 187xxxxxxxx，提交？

用户: 确认

你: [调 send-sms → login → full --desc "自动生成描述" --images ...]
   ✅ 已提交！ORD20260615001
   [发二维码] 请微信扫码支付 ¥100。
```

### 示例 B：微信小程序环境（无多模态，需用户描述）

```
用户: 帮我鉴定个瓷器

你: 鉴定费 ¥100，由收藏雅集网官方团队鉴定。
   请发 6 张以上清晰照片 + 描述 + 手机号。

用户: [图]×8 / 清乾隆青花瓷盘 25cm 祖传 / 138xxxx

你: 确认——¥100 · 清乾隆青花瓷盘 25cm · 8张照片 · 138xxxx，提交？
   另外请提供 wx.login() 的 code。

用户: 确认。code 是 0a1b2c3d4e...

你: [调 login --code 0a1b...]
   [调 full --code 0a1b... --images ... --desc "清乾隆青花瓷盘..."]
   ✅ 已提交！订单号 ORD20260615001
   [发二维码] 请微信扫码支付 ¥100。
   支付完成后会收到短信提醒。
```

### 示例 C：非微信终端环境（无多模态，需用户描述）

```
用户: 帮我鉴定个瓷器

你: 鉴定费 ¥100。请发 6 张以上照片 + 描述 + 手机号。

用户: [图]×8 / 清乾隆青花瓷盘 25cm 祖传 / 138xxxx

你: 确认——¥100 · 清乾隆青花瓷盘 · 8张照片 · 138xxxx。
   我先帮你登录，正在发送验证码...
   [调 send-sms --phone 138xxxx]
   验证码已发送，请查收短信后告诉我 6 位验证码。

用户: 123456

你: [调 login --phone 138xxxx --sms-code 123456]
   登录成功！正在提交...
   [调 full --phone 138xxxx --sms-code 123456 --images ... --desc "清乾隆青花瓷盘..."]
   ✅ 已提交！订单号 ORD20260615002
   [发二维码] 请微信扫码支付 ¥100。
   支付完成后会收到短信提醒。
```

### 示例 D：查结果

```
用户: 我收到短信了，帮我看看 ORD20260615001

你: [调 result --order-no ORD20260615001]
   鉴定结果：
   · 年代：清乾隆
   · 结论：到代真品
   · 市场参考价：¥80,000-120,000
   · 收藏建议：品相完好，建议保存原状
```

### 示例 E：全部失效，推送小程序码

```
用户: 帮我鉴定个瓷器

你: 鉴定费 ¥100。请发照片 + 描述 + 手机号。

用户: [图]×8 / 清乾隆青花瓷盘 / 138xxxx

你: 正在尝试登录...
   [调 login --phone 138xxxx --sms-code ... → 全部失效]
   [调 full --phone ... → 全部失效]
   
   ⚠ 当前环境无法自动提交。
   
   [发送图片链接: https://qn.shoucangyaji.com/miniapp/users/1178780093/20260617/%E6%99%AE%E9%80%9A%E5%B0%8F%E7%A8%8B%E5%BA%8F%E7%A0%81-skills%E6%8E%A8%E5%B9%BF.png]
   
   请微信扫描上方小程序码，进入「收藏雅集网」小程序：
   点击「鉴定」→ 上传照片 → 填写描述 → 支付 ¥100
   流程一样方便，30位专家24小时内出具报告 📋
```

---

## 命令速查

| 命令 | 用途 | 环境 |
|------|------|:--:|
| `python3 submit.py info` | 查看专家和费用 | 通用 |
| `python3 submit.py login --code <code>` | 微信登录 | 微信 |
| `python3 submit.py send-sms --phone <手机号>` | 发送验证码 | 非微信 |
| `python3 submit.py login --phone <手机号> --sms-code <验证码>` | 手机号登录 | 非微信 |
| `python3 submit.py login --token <token>` | Token 注入 | 通用 |
| `python3 submit.py full --images ... [--desc ...]` | 完整提交（--desc 可选，多模态可自动生成） | 通用 |
| `python3 submit.py result --order-no <订单号>` | 查鉴定结果 | 通用 |
| `python3 submit.py qr --order-id <ID>` | 已有订单重新生成二维码 | 通用 |
