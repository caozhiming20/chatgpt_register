# ChatGPT 批量自动注册工具 - 完整项目说明

## 📋 项目概述

这是一个基于 Python 的自动化工具，使用 DuckMail 临时邮箱服务批量注册 ChatGPT 账号，并自动获取 OAuth Token（Codex 协议）。支持并发注册、自动验证码识别、代理配置等功能。

### 核心功能
- 🔄 **批量并发注册** - 支持多线程并发注册多个账号
- 📧 **临时邮箱集成** - 自动创建 DuckMail 临时邮箱
- 🔐 **自动验证** - 自动接收和识别 OTP 验证码
- 🎫 **OAuth Token** - 自动获取 Access Token 和 Refresh Token
- 🌐 **代理支持** - 支持 HTTP/HTTPS 代理配置
- 📤 **CPA 面板集成** - 可选上传到 CPA 管理面板

---

## 🏗️ 项目架构

### 技术栈
- **语言**: Python 3.8+
- **核心依赖**: curl_cffi (模拟浏览器请求)
- **邮箱服务**: DuckMail API (https://api.duckmail.sbs)
- **目标服务**: ChatGPT / OpenAI Auth

### 目录结构
```
chatgpt_register/
├── chatgpt_register.py      # 主程序
├── config.json               # 配置文件
├── README.md                 # 项目说明
├── PROJECT_GUIDE.md          # 本文档
├── codex/                    # Codex 协议相关
│   ├── config.json
│   └── protocol_keygen.py
├── registered_accounts.txt   # 注册成功的账号
├── ak.txt                    # Access Tokens
├── rk.txt                    # Refresh Tokens
└── codex_tokens/             # Token JSON 文件目录
    ├── email1@duckmail.sbs.json
    └── email2@duckmail.sbs.json
```

---

## 🔄 核心流程与关键逻辑

### 1. 整体流程图

```
开始
  ↓
加载配置 (config.json)
  ↓
创建线程池 (并发数)
  ↓
┌─────────────────────────────────────┐
│  每个线程执行以下流程 (并发)        │
├─────────────────────────────────────┤
│ 1. 创建 DuckMail 临时邮箱           │
│    ├─ POST /accounts (创建账号)     │
│    └─ POST /token (获取邮箱 Token)  │
│                                     │
│ 2. ChatGPT 注册流程                 │
│    ├─ 访问首页                      │
│    ├─ 获取 CSRF Token               │
│    ├─ 提交邮箱 (Signin)             │
│    ├─ 授权跳转 (Authorize)          │
│    ├─ 注册账号 (Register)           │
│    ├─ 发送 OTP                      │
│    ├─ 等待验证码邮件 (轮询)         │
│    ├─ 验证 OTP                      │
│    ├─ 填写个人信息                  │
│    └─ 完成回调                      │
│                                     │
│ 3. OAuth Token 获取 (可选)          │
│    ├─ 初始化 OAuth 会话             │
│    ├─ 提交邮箱                      │
│    ├─ 验证密码                      │
│    ├─ OTP 验证 (如需要)             │
│    ├─ Workspace/Org 选择            │
│    ├─ 获取 Authorization Code       │
│    └─ 交换 Access/Refresh Token     │
│                                     │
│ 4. 保存结果                         │
│    ├─ registered_accounts.txt       │
│    ├─ ak.txt / rk.txt               │
│    ├─ codex_tokens/*.json           │
│    └─ 上传到 CPA 面板 (可选)        │
└─────────────────────────────────────┘
  ↓
等待所有线程完成
  ↓
输出统计结果
  ↓
结束
```

### 2. 关键模块详解

#### 2.1 DuckMail 临时邮箱模块

**功能**: 创建临时邮箱并接收验证码

**关键函数**:
```python
def create_temp_email():
    """创建 DuckMail 临时邮箱"""
    # 1. 生成随机邮箱地址 (8-13位随机字符@duckmail.sbs)
    # 2. POST /accounts - 创建邮箱账号
    # 3. POST /token - 获取邮箱访问 Token
    # 返回: (email, password, mail_token)
```

**API 调用**:
- `POST https://api.duckmail.sbs/accounts`
  - Body: `{"address": "xxx@duckmail.sbs", "password": "xxx"}`
  - 无需 API Key (可选配置以获取私有域名)
  
- `POST https://api.duckmail.sbs/token`
  - Body: `{"address": "xxx@duckmail.sbs", "password": "xxx"}`
  - 返回: `{"token": "邮箱访问令牌"}`

**验证码接收**:
```python
def wait_for_verification_email(mail_token, timeout=120):
    """轮询等待验证码邮件"""
    while time.time() - start_time < timeout:
        # 1. GET /messages - 获取邮件列表
        # 2. GET /messages/{id} - 获取邮件详情
        # 3. 正则提取 6 位验证码
        # 4. 每 3 秒轮询一次
```

**验证码提取规则**:
```python
patterns = [
    r"Verification code:?\s*(\d{6})",
    r"code is\s*(\d{6})",
    r">\s*(\d{6})\s*<",
    r"(?<![#&])\b(\d{6})\b",
]
```

#### 2.2 ChatGPT 注册模块

**功能**: 完成 ChatGPT 账号注册流程

**关键步骤**:

1. **访问首页** (`visit_homepage`)
   ```python
   GET https://chatgpt.com/
   # 目的: 初始化 Session，获取 Cookies
   ```

2. **获取 CSRF Token** (`get_csrf`)
   ```python
   GET https://chatgpt.com/api/auth/csrf
   # 返回: {"csrfToken": "..."}
   ```

3. **提交邮箱** (`signin`)
   ```python
   POST https://chatgpt.com/api/auth/signin/openai
   # Body: {"callbackUrl": "...", "csrfToken": "...", "json": "true"}
   # 返回: {"url": "授权 URL"}
   ```

4. **授权跳转** (`authorize`)
   ```python
   GET https://auth.openai.com/api/accounts/authorize?...
   # 自动跳转到注册或登录页面
   ```

5. **注册账号** (`register`)
   ```python
   POST https://auth.openai.com/api/accounts/user/register
   # Body: {"username": "email", "password": "password"}
   # Headers: 包含 Datadog Trace 信息
   ```

6. **发送 OTP** (`send_otp`)
   ```python
   GET https://auth.openai.com/api/accounts/email-otp/send
   # 触发发送验证码邮件
   ```

7. **验证 OTP** (`validate_otp`)
   ```python
   POST https://auth.openai.com/api/accounts/email-otp/validate
   # Body: {"code": "123456"}
   ```

8. **创建账号** (`create_account`)
   ```python
   POST https://auth.openai.com/api/accounts/create_account
   # Body: {"name": "John Doe", "birthdate": "1990-01-01"}
   ```

9. **完成回调** (`callback`)
   ```python
   GET https://chatgpt.com/api/auth/callback/openai?code=...
   # 完成注册流程
   ```

#### 2.3 OAuth Token 获取模块

**功能**: 获取 Codex 协议的 Access Token 和 Refresh Token

**PKCE 流程**:
```python
# 1. 生成 PKCE 参数
code_verifier = base64.urlsafe_b64encode(secrets.token_bytes(64))
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier).digest()
)

# 2. 授权请求
GET /oauth/authorize?
    response_type=code&
    client_id=app_EMoamEEZ73f0CkXaXp7hrann&
    redirect_uri=http://localhost:1455/auth/callback&
    code_challenge={code_challenge}&
    code_challenge_method=S256

# 3. 获取 Authorization Code (通过登录流程)

# 4. 交换 Token
POST /oauth/token
Body: {
    "grant_type": "authorization_code",
    "code": "{authorization_code}",
    "redirect_uri": "...",
    "client_id": "...",
    "code_verifier": "{code_verifier}"
}
```

**关键步骤**:
1. 初始化 OAuth 会话
2. 提交邮箱 (`/api/accounts/authorize/continue`)
3. 验证密码 (`/api/accounts/password/verify`)
4. OTP 验证 (如果需要)
5. Workspace/Organization 选择
6. 获取 Authorization Code
7. 交换 Access Token 和 Refresh Token

#### 2.4 Sentinel Token 生成模块

**功能**: 生成 OpenAI 的反爬虫 Token (PoW - Proof of Work)

**算法**:
```python
class SentinelTokenGenerator:
    def generate_token(self, seed, difficulty):
        # 1. 生成配置数组 (浏览器指纹信息)
        config = [
            "1920x1080",           # 屏幕分辨率
            timestamp,             # 当前时间
            random_number,         # 随机数
            user_agent,            # UA
            # ... 更多浏览器指纹
        ]
        
        # 2. 工作量证明 (PoW)
        for nonce in range(MAX_ATTEMPTS):
            config[3] = nonce
            data = base64_encode(config)
            hash_hex = fnv1a_32(seed + data)
            
            # 3. 检查难度
            if hash_hex[:len(difficulty)] <= difficulty:
                return "gAAAAAB" + data + "~S"
```

**使用场景**:
- 注册账号时 (`flow="authorize_continue"`)
- 密码验证时 (`flow="password_verify"`)
- 添加到请求头: `openai-sentinel-token: {token}`

#### 2.5 浏览器指纹模拟

**Chrome 版本池**:
```python
_CHROME_PROFILES = [
    {"major": 131, "impersonate": "chrome131", ...},
    {"major": 133, "impersonate": "chrome133a", ...},
    {"major": 136, "impersonate": "chrome136", ...},
    {"major": 142, "impersonate": "chrome142", ...},
]
```

**关键 Headers**:
```python
headers = {
    "User-Agent": "Mozilla/5.0 ... Chrome/131.0.6778.123 ...",
    "sec-ch-ua": '"Google Chrome";v="131", "Chromium";v="131"',
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": '"Windows"',
    "sec-ch-ua-arch": '"x86"',
    "sec-ch-ua-bitness": '"64"',
    "sec-ch-ua-full-version": '"131.0.6778.123"',
    "sec-ch-ua-platform-version": '"10.0.0"',
}
```

**Datadog Trace Headers**:
```python
{
    "traceparent": "00-{uuid}-{parent_id}-01",
    "tracestate": "dd=s:1;o:rum",
    "x-datadog-origin": "rum",
    "x-datadog-sampling-priority": "1",
    "x-datadog-trace-id": "{trace_id}",
    "x-datadog-parent-id": "{parent_id}",
}
```

#### 2.6 并发控制模块

**线程池实现**:
```python
def run_batch(total_accounts, max_workers=3):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {}
        for idx in range(1, total_accounts + 1):
            future = executor.submit(_register_one, idx, ...)
            futures[future] = idx
        
        for future in as_completed(futures):
            ok, email, err = future.result()
            # 处理结果
```

**线程安全**:
```python
_print_lock = threading.Lock()  # 控制台输出锁
_file_lock = threading.Lock()   # 文件写入锁

# 使用示例
with _file_lock:
    with open("registered_accounts.txt", "a") as f:
        f.write(f"{email}----{password}\n")
```

---

## ⚙️ 配置说明

### config.json 完整配置

```json
{
  "_comment": "ChatGPT 批量注册配置",
  
  "total_accounts": 5,
  "duckmail_api_base": "https://api.duckmail.sbs",
  "duckmail_bearer": "",
  "proxy": "http://127.0.0.1:7890",
  
  "output_file": "registered_accounts.txt",
  
  "enable_oauth": true,
  "oauth_required": true,
  "oauth_issuer": "https://auth.openai.com",
  "oauth_client_id": "app_EMoamEEZ73f0CkXaXp7hrann",
  "oauth_redirect_uri": "http://localhost:1455/auth/callback",
  
  "ak_file": "ak.txt",
  "rk_file": "rk.txt",
  "token_json_dir": "codex_tokens",
  
  "upload_api_url": "http://localhost:8317/v0/management/auth-files",
  "upload_api_token": "your_cpa_panel_password"
}
```

### 配置项详解

| 配置项 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `total_accounts` | int | 是 | 注册账号数量 |
| `duckmail_api_base` | string | 是 | DuckMail API 地址 |
| `duckmail_bearer` | string | 否 | DuckMail API Key (可选，用于私有域名) |
| `proxy` | string | 否 | 代理地址，格式: `http://host:port` |
| `output_file` | string | 是 | 账号输出文件路径 |
| `enable_oauth` | bool | 是 | 是否启用 OAuth Token 获取 |
| `oauth_required` | bool | 是 | OAuth 失败时是否终止注册 |
| `oauth_issuer` | string | 是 | OAuth 授权服务器地址 |
| `oauth_client_id` | string | 是 | OAuth 客户端 ID |
| `oauth_redirect_uri` | string | 是 | OAuth 回调地址 |
| `ak_file` | string | 是 | Access Token 保存文件 |
| `rk_file` | string | 是 | Refresh Token 保存文件 |
| `token_json_dir` | string | 是 | Token JSON 文件目录 |
| `upload_api_url` | string | 否 | CPA 面板上传 API 地址 |
| `upload_api_token` | string | 否 | CPA 面板登录密码 |

### 环境变量支持

所有配置项都支持通过环境变量覆盖：

```bash
# Linux/Mac
export DUCKMAIL_BEARER="dk_your_api_key"
export PROXY="http://127.0.0.1:7890"
export TOTAL_ACCOUNTS=10

# Windows
set DUCKMAIL_BEARER=dk_your_api_key
set PROXY=http://127.0.0.1:7890
set TOTAL_ACCOUNTS=10
```

---

## 📖 使用说明

### 1. 环境准备

#### 1.1 安装 Python
```bash
# 检查 Python 版本 (需要 3.8+)
python --version

# 如果没有安装，请访问 https://www.python.org/downloads/
```

#### 1.2 安装依赖
```bash
pip install curl_cffi
```

#### 1.3 配置代理 (可选但推荐)
```bash
# 确保代理服务正在运行
# 常见代理软件: Clash, V2Ray, Shadowsocks 等
# 默认端口通常是 7890 或 1080
```

### 2. 配置文件设置

编辑 `config.json`:

```json
{
  "total_accounts": 5,           // 修改为你需要的数量
  "proxy": "http://127.0.0.1:7890",  // 修改为你的代理地址
  "duckmail_bearer": "",         // 留空即可 (可选)
  "enable_oauth": true,          // 是否获取 Token
  "oauth_required": true         // Token 失败是否终止
}
```

### 3. 运行程序

#### 3.1 交互式运行
```bash
python chatgpt_register.py
```

程序会提示你输入：
1. 是否使用代理
2. 注册账号数量
3. 并发数

#### 3.2 非交互式运行
```bash
# 使用环境变量
export TOTAL_ACCOUNTS=10
export PROXY="http://127.0.0.1:7890"
python chatgpt_register.py
```

### 4. 查看结果

#### 4.1 账号信息
```bash
cat registered_accounts.txt
```

格式: `邮箱----ChatGPT密码----邮箱密码----oauth状态`

示例:
```
aerioacyt@duckmail.sbs----l$Z&zvFLdpDU9s----13iFNoaYtS@ld!----oauth=ok
```

#### 4.2 Token 文件
```bash
# Access Tokens
cat ak.txt

# Refresh Tokens
cat rk.txt

# 完整 Token JSON
ls codex_tokens/
cat codex_tokens/aerioacyt@duckmail.sbs.json
```

Token JSON 格式:
```json
{
  "type": "codex",
  "email": "aerioacyt@duckmail.sbs",
  "expired": "2026-04-08T12:34:56+08:00",
  "id_token": "eyJhbGc...",
  "account_id": "user-xxx",
  "access_token": "eyJhbGc...",
  "last_refresh": "2026-03-08T12:34:56+08:00",
  "refresh_token": "v1.Mxxx..."
}
```

---

## 🔧 高级用法

### 1. 自定义并发数

```python
# 修改 main() 函数中的默认值
workers_input = input("并发数 (默认 3): ").strip()
max_workers = int(workers_input) if workers_input.isdigit() else 3

# 建议:
# - 网络较好: 5-10
# - 网络一般: 3-5
# - 网络较差: 1-3
```

### 2. 使用 DuckMail API Key

访问 https://domain.duckmail.sbs 获取 API Key，可以：
- 获取更多域名选择
- 创建私有域名邮箱
- 提高请求配额

配置:
```json
{
  "duckmail_bearer": "dk_70da50da343ff329daaa4c271c9159fd743763bd73adcb9c661f85fff752f2d6"
}
```

### 3. CPA 面板集成

如果你部署了 CPA Dashboard (https://github.com/dongshuyan/CPA-Dashboard):

```json
{
  "upload_api_url": "http://your-cpa-panel:8317/v0/management/auth-files",
  "upload_api_token": "your_panel_password"
}
```

注册成功后会自动上传 Token JSON 到面板。

### 4. 仅注册不获取 Token

```json
{
  "enable_oauth": false
}
```

这样只会注册账号，不会获取 OAuth Token，速度更快。

### 5. 自定义邮箱域名

如果有 DuckMail API Key 和私有域名:

```python
# 修改 create_temp_email() 函数
email = f"{email_local}@your-private-domain.com"
```

---

## 🐛 常见问题

### 1. 网络连接超时

**问题**: `Resolving timed out after 15001 milliseconds`

**解决**:
- 检查代理是否正常运行
- 确认代理地址和端口正确
- 尝试更换代理节点

### 2. DuckMail API 限制

**问题**: `429 Too Many Requests`

**解决**:
- 降低并发数
- 增加请求间隔
- 申请 API Key 提高配额

### 3. 验证码接收失败

**问题**: `未能获取验证码`

**解决**:
- 检查邮箱是否创建成功
- 增加超时时间 (默认 120 秒)
- 检查 DuckMail 服务状态

### 4. OAuth Token 获取失败

**问题**: `OAuth 获取失败`

**解决**:
- 检查账号是否注册成功
- 确认密码正确
- 查看详细日志定位问题

### 5. Sentinel Token 生成失败

**问题**: `sentinel token 获取失败`

**解决**:
- 检查网络连接
- 更新浏览器指纹配置
- 重试请求

---

## 📊 性能优化建议

### 1. 并发数设置

| 网络质量 | 建议并发数 | 预计速度 |
|---------|-----------|---------|
| 优秀 | 8-10 | 20-25 秒/账号 |
| 良好 | 5-7 | 25-30 秒/账号 |
| 一般 | 3-5 | 30-40 秒/账号 |
| 较差 | 1-3 | 40-60 秒/账号 |

### 2. 代理选择

- 优先使用稳定的付费代理
- 避免使用免费公共代理
- 选择延迟低的节点 (<200ms)

### 3. 错误重试

当前版本不支持自动重试，建议：
- 记录失败的账号索引
- 手动重新运行失败的部分
- 或修改代码添加重试逻辑

---

## 🔒 安全注意事项

### 1. 账号安全
- 生成的密码强度高 (14 位，包含大小写字母、数字、特殊字符)
- 建议定期更换密码
- 不要在公共场合泄露账号信息

### 2. API Key 安全
- 不要将 API Key 提交到公共仓库
- 使用环境变量存储敏感信息
- 定期轮换 API Key

### 3. 代理安全
- 使用可信的代理服务
- 避免使用不明来源的代理
- 注意代理日志可能记录请求信息

### 4. 合规使用
- 遵守 OpenAI 服务条款
- 不要用于恶意目的
- 注意账号使用频率限制

---

## 📝 输出文件说明

### 1. registered_accounts.txt
```
格式: 邮箱----ChatGPT密码----邮箱密码----oauth状态
用途: 快速查看所有注册成功的账号
```

### 2. ak.txt
```
格式: 每行一个 Access Token
用途: 直接用于 API 调用
有效期: 通常 90 天
```

### 3. rk.txt
```
格式: 每行一个 Refresh Token
用途: 刷新 Access Token
有效期: 通常 1 年
```

### 4. codex_tokens/*.json
```json
{
  "type": "codex",
  "email": "邮箱地址",
  "expired": "过期时间",
  "id_token": "ID Token",
  "account_id": "账号 ID",
  "access_token": "访问令牌",
  "last_refresh": "最后刷新时间",
  "refresh_token": "刷新令牌"
}
```

---

## 🛠️ 开发与扩展

### 1. 添加新的邮箱服务

实现以下接口:
```python
def create_temp_email():
    """创建临时邮箱"""
    return email, password, mail_token

def wait_for_verification_email(mail_token, timeout):
    """等待验证码"""
    return otp_code
```

### 2. 自定义输出格式

修改 `_register_one()` 函数中的保存逻辑:
```python
with _file_lock:
    with open(output_file, "a", encoding="utf-8") as out:
        # 自定义格式
        out.write(f"{email},{password},{oauth_ok}\n")
```

### 3. 添加通知功能

在注册成功后发送通知:
```python
def send_notification(email, success):
    # 发送邮件、Telegram、Discord 等
    pass
```

---

## 📚 相关资源

- **DuckMail 官网**: https://duckmail.sbs
- **DuckMail API 文档**: https://www.duckmail.sbs/api-docs
- **DuckMail 域名管理**: https://domain.duckmail.sbs
- **CPA Dashboard**: https://github.com/dongshuyan/CPA-Dashboard
- **curl_cffi 文档**: https://github.com/yifeikong/curl_cffi

---

## 📄 许可证

本项目仅供学习和研究使用，请遵守相关法律法规和服务条款。

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

**最后更新**: 2026-03-08
**版本**: 1.0.0
