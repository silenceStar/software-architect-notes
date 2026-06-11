# Mac + MonoProxy + GitHub Push 问题排查总结

## 问题现象

执行 Git Push 时出现：

```bash
fatal: unable to access 'https://github.com/xxx/xxx.git':
Failed to connect to github.com port 443
```

或者：

```bash
curl -I https://github.com
```

一直卡住。

---

## 问题定位

### 1. DNS正常

```bash
ping github.com
```

结果：

```text
64 bytes from github.com
```

说明：

* DNS解析正常
* 网络可达GitHub

---

### 2. HTTPS异常

```bash
curl -I https://github.com
```

无返回，一直卡住。

```bash
nc -vz github.com 443
```

不通。

说明：

```text
Ping正常
HTTPS(443)异常
```

---

### 3. 检查系统代理

```bash
scutil --proxy
```

发现：

```text
HTTPProxy  : 127.0.0.1
HTTPPort   : 8118

HTTPSProxy : 127.0.0.1
HTTPSPort  : 8118

SOCKSProxy : 127.0.0.1
SOCKSPort  : 8119
```

说明使用 MonoProxy。

---

### 4. 验证代理可用

查看代理监听：

```bash
lsof -iTCP:8118 -sTCP:LISTEN
```

结果：

```text
MonoProxy ... localhost:8118 (LISTEN)
```

测试代理：

```bash
curl -x http://127.0.0.1:8118 -I https://github.com
```

返回：

```text
HTTP/2 200
```

说明：

```text
MonoProxy正常
GitHub正常
```

---

## 原因

Git、Curl 默认未使用 MonoProxy 代理。

```text
浏览器
  ↓
MonoProxy
  ↓
GitHub
（正常）

Git
  ↓
直连GitHub
  ↓
443连接失败
```

---

## 解决方案

### 当前终端临时生效

```bash
export HTTP_PROXY=http://127.0.0.1:8118
export HTTPS_PROXY=http://127.0.0.1:8118
```

验证：

```bash
git ls-remote https://github.com/用户名/仓库.git
```

---

### Git全局代理（慎用）

```bash
git config --global http.proxy http://127.0.0.1:8118
git config --global https.proxy http://127.0.0.1:8118
```

取消：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

> 注意：全局代理会影响公司Git仓库访问。

---

## GitHub认证问题

错误：

```text
Password authentication is not supported
```

原因：

GitHub已不支持账号密码认证。

解决：

* Personal Access Token（PAT）
* SSH Key（推荐）

---

## 仓库冲突问题

错误：

```text
non-fast-forward
Updates were rejected because the remote contains work that you do not have locally
```

原因：

GitHub仓库初始化时自动生成README，本地仓库也有独立提交历史。

解决：

### 合并历史

```bash
git pull origin main --allow-unrelated-histories --no-rebase
git push origin main
```

### 强制覆盖（仅适用于新仓库）

```bash
git push -f origin main
```

---

## 常用排查命令

```bash
# 测试GitHub
ping github.com

# 测试HTTPS
curl -I https://github.com

# 测试443端口
nc -vz github.com 443

# 查看系统代理
scutil --proxy

# 查看Git代理
git config --global --list

# 查看远程仓库
git remote -v

# 测试仓库访问
git ls-remote https://github.com/用户名/仓库.git
```

---

## 最终结论

问题链路：

```text
Git Push失败
    ↓
443连接失败
    ↓
MonoProxy仅浏览器生效
    ↓
Git未走代理
    ↓
配置代理后恢复
    ↓
PAT认证通过
    ↓
解决远程仓库历史冲突
    ↓
Push成功
```

