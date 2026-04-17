# 内网渗透测试网络拓扑与工具链总结

## 1. 网络拓扑结构

本次渗透测试环境涉及 Web 服务器与 Active Directory 域环境，整体网络结构如下：

```
攻击机（Kali）
    │
    │ proxychains / socks 代理（1080）
    ▼
Web服务器（10.0.0.10）
    │
    │（内网访问 / 双网卡转发）
    ▼
内网网段（10.0.0.0/24）
    │
    ├── DC01 域控（10.0.0.2）
    │       ├── corp.local 域
    │       ├── SMB（445）
    │       ├── Kerberos（88）
    │       └── WMI / RPC
    │
    └── ...
```

---

## 2. 攻击路径流程

```
目录扫描（dirsearch）
        ↓
发现 /config.php.bak、product.php、upload.php
        ↓
SQL Injection（product.php）
        ↓
获取数据库与用户凭据（user1等）
        ↓
文件上传漏洞（upload.php）
        ↓
RCE → 获得 www-data shell
        ↓
内网探测（ip neigh / arp）
        ↓
发现 DC01（10.0.0.2）
        ↓
SMB 认证（user1:Password123）
        ↓
远程命令执行（netexec / wmiexec / psexec）
        ↓
SYSTEM 权限获取
        ↓
NTDS.dit 数据提取（域控哈希）
        ↓
创建 backdoor 用户（持久化）
```

---

## 3. 使用工具链总结

### 3.1 信息收集阶段
- dirsearch（目录枚举）
- 浏览器访问验证
- config.php.bak 文件探测

### 3.2 漏洞利用阶段（Web）
- sqlmap（SQL注入利用）
- curl / browser（接口验证）
- upload.php 文件上传利用

### 3.3 初始访问
- bash reverse shell
- netcat（nc -lvnp）

---

### 3.4 内网渗透阶段
- ip neigh（内网主机发现）
- arp（网络邻居分析）
- proxychains（流量转发）

---

### 3.5 横向移动
- netexec smb（SMB认证与命令执行）
- wmiexec（远程WMI执行）
- impacket-psexec（服务创建执行）

---

### 3.6 域环境攻击
- netexec --ntds（NTDS.dit 提取）
- impacket-lookupsid（SID枚举）
- Kerberos 票据相关工具（klist / ccache）

---

### 3.7 持久化访问
- net user backdoor /add
- SMB / psexec 远程登录验证

---

## 4. 关键资产与权限变化

### 初始阶段
- www-data（Web低权限用户）

### 内网阶段
- corp\\user1（域用户）

### 域控阶段
- NT AUTHORITY\\SYSTEM（DC01）

### 域权限
- Administrator / krbtgt hash 获取

---

## 5. 风险点总结

- config.php.bak 泄露敏感信息
- SQL 注入导致数据库泄露
- 文件上传导致远程代码执行
- 内网无访问控制（Web → DC）
- SMB 明文/弱口令认证
- 域控可直接被远程执行命令
- NTDS.dit 可被远程提取
- 可创建持久化后门账户

---

## 6. 总体结论

本次测试中，攻击者通过 Web 层漏洞成功进入内网，并在缺乏横向隔离与访问控制的情况下逐步提升权限，最终完全控制 Active Directory 域环境。整个攻击链条从初始访问到域控沦陷过程完整，且未遭遇有效检测或阻断机制，说明当前网络在 Web 安全、内网隔离及域控防护方面均存在较高风险。
