# 快速开始指南

## 🚀 5 分钟快速上手

### 第一步: 安装依赖

```bash
pip install curl_cffi
```

### 第二步: 配置代理

编辑 `config.json`，修改代理地址:

```json
{
  "proxy": "http://127.0.0.1:7890"
}
```

> 💡 如果没有代理，可以留空 `""`，但可能会失败

### 第三步: 运行程序

```bash
python chatgpt_register.py
```

按提示输入:
- 是否使用代理: `y`
- 注册数量: `2` (建议先测试 2 个)
- 并发数: 直接回车 (使用默认值 3)

### 第四步: 查看结果

```bash
# 查看注册的账号
cat registered_accounts.txt

# 查看 Access Token
cat ak.txt

# 查看 Refresh Token
cat rk.txt
```

---

## 📋 输出格式说明

### registered_accounts.txt
```
邮箱----ChatGPT密码----邮箱密码----oauth状态
```

示例:
```
test123@duckmail.sbs----Abc123!@#Xyz----Def456!@#Uvw----oauth=ok
```

### 使用账号登录

1. 访问 https://chatgpt.com
2. 使用邮箱和 ChatGPT 密码登录
3. 如果需要验证码，使用邮箱密码登录 DuckMail 查看

---

## ⚙️ 常用配置

### 只注册不获取 Token (更快)

```json
{
  "enable_oauth": false
}
```

### 增加注册数量

```json
{
  "total_accounts": 10
}
```

### 使用环境变量

```bash
# Linux/Mac
export TOTAL_ACCOUNTS=10
export PROXY="http://127.0.0.1:7890"

# Windows
set TOTAL_ACCOUNTS=10
set PROXY=http://127.0.0.1:7890
```

---

## 🐛 遇到问题?

### 网络超时
- 检查代理是否运行
- 确认代理地址正确

### 验证码接收失败
- 等待时间更长 (最多 120 秒)
- 检查 DuckMail 服务状态

### OAuth 失败
- 设置 `"oauth_required": false` 继续注册
- 查看详细日志定位问题

---

## 📖 完整文档

查看 `PROJECT_GUIDE.md` 了解详细说明。

---

**祝你使用愉快! 🎉**
