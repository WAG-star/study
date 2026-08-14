# Day3：HTTP 报文、状态码与方法

> 目标：会看真实 HTTP 报文、会用 curl 调接口
> 评分：92/100（报文四部分概念混了一次，实操全过）

## 1. HTTP 报文结构（四段式）

**请求报文**：
```
POST /api/login HTTP/1.1          ← 请求行：方法 + 路径 + 版本
Host: example.com                 ← 请求头
Content-Type: application/json
                                  ← 空行（分隔符，必须有）
{"user":"wag","pass":"123"}       ← 请求体
```

**响应报文**：
```
HTTP/1.1 200 OK                   ← 状态行：版本 + 状态码 + 短语
Content-Type: text/html           ← 响应头
                                  ← 空行
<html>...</html>                  ← 响应体
```

⚠️ **易错点**：请求报文第一段叫**请求行**，响应报文第一段叫**状态行**——别混成"响应头"。

## 2. nginx 安装与确认

```bash
apt install nginx -y
systemctl status nginx        # active (running)
ss -tlnp | grep 80            # 80 端口被监听
```
1核2G 跑 nginx 无压力。nginx 默认只放行 **GET/HEAD**，其他方法一律 405。

## 3. curl 三件套

```bash
curl http://localhost                # 只显示响应体
curl -i http://localhost             # 响应头 + 响应体（i = include）
curl -I http://localhost             # 只显示响应头（I = HEAD 请求）
curl -v http://localhost             # 全过程（v = verbose）
curl -s http://localhost             # 静默（去掉进度条）
```

## 4. curl -v 逐行解读（核心技能）

```
* Trying 127.0.0.1:80...            ← ① TCP 连接
> GET / HTTP/1.1                    ← ② 发出的请求行（> 发送）
> Host: localhost
> User-Agent: curl/7.81.0
> Accept: */*
>                                   ← ③ 空行：请求头结束
< HTTP/1.1 200 OK                   ← ④ 收到的状态行（< 接收）
< Server: nginx/1.30.4
< Content-Type: text/html
< Content-Length: 896
< ETag: "6a57c166-380"              ← 缓存标识（Day4 协商缓存用）
<                                   ← ⑤ 空行：响应头结束
<!DOCTYPE html>...                  ← ⑥ 响应体
```

💡 **记忆**：`>` 是发出去的，`<` 是收进来的。

## 5. 状态码实测记录

| 状态码 | 场景 | 实测结果 |
|--------|------|---------|
| 200 OK | 访问首页 | ✅ |
| 301 Moved Permanently | `curl -I http://jd.com` | ✅ Location: https://www.jd.com |
| 404 Not Found | `curl -i http://localhost/nonexistent` | ✅ 资源不存在 |
| 405 Not Allowed | `curl -X OPTIONS/POST localhost` | ✅ 方法不被允许 |

💡 **记忆**：
- **404 = 东西不存在，405 = 你这么干不被允许**（nginx 默认只放行 GET/HEAD）
- **301 的灵魂是 `Location` 头**——状态码说"挪走了"，Location 说"挪到哪"

⚠️ **易错点（实测打脸）**：别假设服务器行为——`curl -I http://www.baidu.com` 返回 **200**（百度没配重定向），`jd.com` 才返回 301。以实测为准。

## 6. HTTP 方法

| 方法 | 语义 | 幂等 |
|------|------|------|
| GET | 取资源 | ✅ |
| POST | 创建/提交 | ❌ |
| PUT | 整体替换 | ✅ |
| PATCH | 部分修改 | ❌ |
| DELETE | 删除 | ✅ |
| HEAD | 只要响应头 | ✅ |
| OPTIONS | 问支持什么方法 | ✅ |

**GET vs POST**：GET 参数在 URL（`?q=linux`），POST 参数在请求体（`-d "user=wag"`）。GET 是"看"（可重复），POST 是"交"（重复会重复下单）。

**幂等** = 同一请求执行 1 次和 N 次，对资源状态的改变一致。GET/DELETE 幂等，POST 不幂等。

## 7. 实战：本地造 301（可选进阶）

```bash
echo 'server { listen 8080; return 301 http://localhost/; }' > /etc/nginx/conf.d/redirect.conf
nginx -t && systemctl reload nginx
curl -I http://localhost:8080/    # 301 + Location: http://localhost/
```
💡 `return 301 地址` 是 nginx 重定向咒语。
