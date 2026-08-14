# Day4：Cookie / Token / 会话鉴权

> 目标：搞懂"登录后怎么记住你"——Cookie、Session、Token 三兄弟
> 评分：30/30（概念）+ 实操全过（Cookie 生命周期 + JWT 签发/验签/篡改）

## 1. HTTP 无状态 → 为什么需要 Cookie

HTTP 协议不记人（像鱼，记忆只有 7 秒）。Cookie 帮它记住：

```
① 登录成功 → 服务器响应头 Set-Cookie: session=abc123
② 浏览器保存纸条
③ 每次请求 → 自动带上 Cookie: session=abc123
④ 服务器看纸条 → 认识你
```

## 2. Cookie 属性

| 属性 | 含义 |
|------|------|
| `Expires` / `Max-Age` | 过期时间（不设 = 会话级，关浏览器就没了） |
| `Domain` / `Path` | 哪些域名/路径才带 |
| `HttpOnly` | 禁止 JS 读取（防 XSS 偷 Cookie） |
| `Secure` | 只有 HTTPS 才传输 |
| `SameSite` | 跨站请求带不带（防 CSRF） |

💡 **记忆**：HttpOnly 防"偷"（XSS），SameSite 防"抢"（CSRF）。

## 3. Cookie 生命周期实操（curl 三件套）

```bash
curl -c /root/cookies.txt http://localhost:3000/login   # -c 存纸条（写文件）
curl -b /root/cookies.txt http://localhost:3000/me      # -b 带纸条（读文件）
curl -v -b cookies.txt http://localhost:3000/me         # -v 看请求头 Cookie 行
```

实测输出：
```
① 无 Cookie 访问 /me  →  no cookie            （服务器不认识你）
② 登录响应            →  Set-Cookie: session=abc123; HttpOnly
③ 保存后携带访问 /me  →  your cookie: session=abc123  （被认出）
```

💡 **记忆**：`-c` 存纸条、`-b` 带纸条、`-v` 看纸条长啥样。

## 4. Session vs Token（两条路线）

| | Session | Token/JWT |
|---|---|---|
| 状态存哪 | **服务器**（本子/数据库） | **客户端**（票上） |
| 服务器要不要查库 | 要 | 不要，验签即可 |
| 缺点 | 占内存、多机要共享、跨域差 | **吊销难**（被盗后在有效期内都能用） |
| 优点 | 可随时吊销 | 无状态、跨域/多端方便 |

💡 **终极记忆**：Session 是"服务器记你"（餐厅会员本），Token 是"你自己带着证明"（防伪门票）。

## 5. JWT 三段结构

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoid2FnIiwiZXhwIjoxNzg2NzAyMjQyfQ._CfjFbtn_RTfPr-Br0om-Ky5UeYc5n_-HDgVxXT6UE4
└──────── header ────────┘.└────── payload ───────┘.└────── signature ──────┘
```

| 段 | 内容 | 说明 |
|----|------|------|
| header | `{"alg":"HS256","typ":"JWT"}` | 算法声明 |
| payload | `{"user":"wag","exp":...}` | **明文数据**（base64 可解，谁都能看） |
| signature | HMAC-SHA256(header.payload, secret) | **防伪签名** |

💡 **记忆**：**数据能看（base64 可解）但改不了（签名对不上）**——篡改 payload 后验签必失败。

## 6. base64url（JWT 的编码）

`base64url` = 标准 base64 的 URL 安全变体：**`+`→`-`、`/`→`_`、去掉 `=`**（URL 里不能有 `+/=`）。

⚠️ **易错点（实测翻车）**：**Node 12 不支持 `base64url` 编码**（14.18+ 才有）——服务启动正常、一收到请求就崩 `ERR_UNKNOWN_ENCODING`。手写兼容版：
```js
const b64 = obj => Buffer.from(JSON.stringify(obj)).toString('base64')
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
const b64d = s => Buffer.from(s.replace(/-/g, '+').replace(/_/g, '/'), 'base64');
```

## 7. 签名防伪验证实操

```bash
# ① 签发
curl -s http://localhost:3001/login-token
# token: eyJhbG...三段...

# ② 验证（正确 token → VALID）
curl -H "Authorization: Bearer <token>" http://localhost:3001/verify
# VALID. payload: {"user":"wag","exp":...}

# ③ 篡改 payload 段 → INVALID
curl -H "Authorization: Bearer <token中间段改成XXXX>" http://localhost:3001/verify
# INVALID. signature mismatch
```

## 8. 环境坑记录（本次实测）

**① nohup 教训**：`node server.js &` 只防占终端，**SSH 断开时 shell 发 SIGHUP 杀掉后台进程**。必须：
```bash
nohup node server.js > server.log 2>&1 &
```
💡 服务器跑服务标准姿势：`nohup 命令 > 日志.log 2>&1 &`。`disown` 是 nohup 的替代品。

**② Node 升级（12 → 18）**：Ubuntu 默认 nodejs 12.x，apt 默认源升不了。NodeSource 源：
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt install nodejs -y
node -v    # v18.x
```
⚠️ dpkg 冲突 `trying to overwrite` = 新旧包抢同一文件，解法：**先卸旧包全家桶再装**：
```bash
apt remove --purge libnode-dev libnode72 nodejs npm -y
apt -f install -y && apt install nodejs -y
```
（Node 12 拆成 libnode-dev/libnode72/nodejs/npm 多个包，漏卸一个继续撞车）

**③ 退出码诊断**：Exit 127 = 命令找不到（查 PATH）；Exit 1 = 程序自己挂了（查日志）。**日志永远是第一现场**。

**④ Ctrl-S 冻结终端**：Ctrl-S（XOFF）暂停终端，一切输入无反应，**Ctrl-Q 解冻**。终端"死了"先试 Ctrl-Q，比 Ctrl-C 优先。
