# 实验室 Linux 服务器无法联网？利用 Windows Clash Verge + SSH 反向代理解决 Conda 环境创建失败问题（完整实战）
(着急的话直接看 第五节操作)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/70ea66e7bc2b4120b80a911d3cc42d78.png)

## 一、问题背景

最近在实验室 GPU 服务器上准备进行 Qwen3 模型的 LoRA 微调实验，需要首先创建一个 Python 3.10 的 Conda 虚拟环境。

实验环境如下：

| 项目 | 环境信息 |
|---|---|
| 本地操作系统 | Windows |
| 远程服务器 | Linux（实验室 GPU 服务器） |
| 远程连接方式 | SSH |
| Python 环境管理工具 | Conda |
| 本地代理工具 | Clash Verge |
| 本地 Clash 端口 | 7897 |
| SSH 远程转发端口 | 1080 |
| 目标环境 | qwen3-lora（Python 3.10） |

本地 Windows 电脑可以正常访问互联网，但实验室 Linux 服务器无法直接连接外部软件源。

最初以为是 Conda 镜像源配置问题，尝试排查后发现，服务器原有代理已经失效，而且直接访问公网软件源也会失败。

最终采用 **SSH 反向端口转发（SSH Reverse Port Forwarding）**，让 Linux 服务器通过 Windows 本地 Clash 代理访问互联网，成功完成 Conda 虚拟环境创建。

本文记录完整的问题定位与解决过程。

## 二、问题现象：Conda 创建环境失败

### 2.1 原始命令

在实验室 Linux 服务器中执行：

```bash
conda create -n qwen3-lora python=3.10 -y
```

正常情况下，Conda 会从软件源获取包索引、解析依赖关系、下载软件包并创建虚拟环境。

但实际执行过程中，终端不断出现连接失败及重试信息：

```text
Retrying (Retry(total=2, ...)) after
connection broken by 'NewConnectionError(
"SOCKSHTTPSConnection(
host='mirrors.tuna.tsinghua.edu.cn',
port=443
): Failed to establish a new connection:
[Errno 111] Connection refused"
)'
```

经过多次重试后，出现：

```text
CondaHTTPError: HTTP 000 CONNECTION FAILED

for url:
https://mirrors.tuna.tsinghua.edu.cn/anaconda/
pkgs/r/linux-64/repodata.json
```

其中，几个值得关注的关键词是：

- `SOCKSHTTPSConnection`：当前请求可能正在通过 SOCKS 代理访问 HTTPS 网站。
- `Connection refused`：目标连接被拒绝，无法建立 TCP 连接。
- `HTTP 000 CONNECTION FAILED`：Conda 未能获得有效的 HTTP 响应。
- `repodata.json`：Conda 用于获取软件包版本和依赖信息的仓库元数据文件。

因此，问题发生在**软件包索引获取阶段**，而不是 Python 3.10 本身或依赖冲突。

### 2.2 初步怀疑

可能原因包括：

1. Conda 镜像源暂时无法访问；
2. 实验室服务器无法连接公网；
3. 服务器网络代理配置错误；
4. 原有 SOCKS5 代理服务未启动或已经失效。

为了确定具体原因，需要逐步排查。

## 三、问题排查：定位代理与网络连接异常

### 3.1 检查服务器代理环境变量

在 Linux 服务器执行：

```bash
env | grep -i proxy
```

实际输出：

```text
HTTPS_PROXY=socks5://127.0.0.1:1080
HTTP_PROXY=socks5://127.0.0.1:1080
ALL_PROXY=socks5://127.0.0.1:1080
```

可以发现服务器原先配置了 SOCKS5 代理，地址为：

```text
127.0.0.1:1080
```

这里需要注意：`127.0.0.1` 是回环地址，表示运行当前命令的计算机自身。

由于命令在 Linux 服务器执行，因此这里的 `127.0.0.1` 指的是 Linux 服务器，而不是 Windows 本地电脑。

如果服务器 1080 端口没有提供正常的代理服务，Conda 就无法通过这个地址访问外网。

### 3.2 检查 SOCKS5 端口是否监听

执行：

```bash
ss -lnt | grep ':1080 '
```

实际结果：没有任何输出。

这说明当时没有发现服务器本机 TCP 1080 端口处于监听状态。

结合前面的错误，可以判断：**服务器配置了 SOCKS5 代理地址，但该地址对应的本地代理服务不可用。**

### 3.3 临时取消代理配置

为了确认是否仅仅是代理问题，尝试在当前终端取消环境变量：

```bash
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
unset ALL_PROXY
unset all_proxy
```

检查 Conda 是否存在单独的代理配置：

```bash
conda config --show proxy_servers
```

实际输出：

```text
proxy_servers: {}
```

说明 Conda 没有通过该配置项单独指定代理。

### 3.4 测试服务器直连能力

取消代理后，分别测试清华大学镜像源、中科大镜像源以及 Anaconda 官方源：

```bash
curl -I --connect-timeout 5 \
https://mirrors.tuna.tsinghua.edu.cn

curl -I --connect-timeout 5 \
https://mirrors.ustc.edu.cn

curl -I --connect-timeout 5 \
https://repo.anaconda.com
```

实际结果：

```text
curl: (7) Failed to connect to
mirrors.tuna.tsinghua.edu.cn port 443
after 4 ms: Connection refused

curl: (7) Failed to connect to
mirrors.ustc.edu.cn port 443
after 5 ms: Connection refused

curl: (7) Failed to connect to
repo.anaconda.com port 443
after 6 ms: Connection refused
```

三个不同的网站均连接失败。

这说明问题并非单一镜像源异常。服务器的公网 HTTPS 出站连接存在障碍，可能与实验室网络访问策略、路由或防火墙有关。

需要强调的是，这些测试能够证明当前连接失败，但仅凭 `Connection refused`，还不能确定具体是哪一层网络设备实施了限制。

### 3.5 排查结论

经过以上测试，可以确定：

- 原先配置的 SOCKS5 代理端口 `1080` 没有正常监听；
- 取消代理后，服务器仍无法连接所测试的公网 HTTPS 网站；
- Conda 创建失败的直接原因是无法获取远程软件包元数据；
- 继续反复更换 Conda 镜像源，并不能从根本上解决当前的网络访问问题。

考虑到本地 Windows 电脑可以正常联网，因此决定使用 SSH 反向端口转发解决问题。

## 四、解决原理：SSH 反向代理是如何工作的？

### 4.1 整体思路

通常，我们使用 SSH 从 Windows 电脑远程登录 Linux 服务器。

但 SSH 不仅能够提供远程终端，还支持端口转发。

其中，`-R` 参数可以在远程服务器建立监听端口，并将连接通过 SSH 通道转发到本地电脑指定的地址和端口。

本次方案利用这一特性：

1. Windows 电脑运行 Clash Verge；
2. Clash 提供本地代理服务；
3. Windows 主动连接 Linux 服务器并建立 SSH 反向隧道；
4. Linux 服务器通过自己的 `127.0.0.1:1080` 访问隧道；
5. SSH 将连接转发到 Windows 的 `127.0.0.1:7897`；
6. Clash 代理最终负责访问外部软件源。

整个请求路径如下：

```text
Linux 服务器
    |
    | Conda / pip / curl
    v
127.0.0.1:1080
（服务器端监听端口）
    |
    | SSH 反向端口转发
    v
Windows 本地电脑
    |
    v
127.0.0.1:7897
（Clash Verge 代理）
    |
    v
公网软件源
Anaconda / PyPI / GitHub
```

**【插图位置：SSH 反向隧道网络架构示意图】**

### 4.2 为什么采用 SSH 反向转发？

因为实验室服务器本身不能正常直接访问公网，但 Windows 电脑可以建立到服务器的 SSH 连接。

利用已经允许的 SSH 连接，可以将服务器的代理请求转发到 Windows。

这种方式有以下特点：

- 不需要修改 Conda 源码；
- 通常不需要 Linux 服务器的 root 权限；
- 不需要在服务器安装 Clash；
- 可以用于 Conda、pip 等工具；
- 远程端口可以限定在服务器回环地址，避免开放给其他网络主机。

需要注意，服务器 SSH 服务端必须允许远程 TCP 端口转发，并且该使用方式应符合实验室网络管理规定。

## 五、具体操作：从 Windows 到 Linux 的完整配置

### 5.1 在 Windows 中检查 Clash Verge 代理端口

首先打开 Clash Verge，进入：

**设置 → Clash 设置 → 端口设置**

在本次实验中，看到端口值为：

```text
7897
```

系统代理已开启。

**【插图位置：Clash Verge 设置页面截图，标注端口 7897】**

不同版本或用户配置的 Clash 端口可能不同，因此不能直接假设使用 7890。

### 5.2 测试 Windows 本地代理是否正常

打开 Windows PowerShell，执行：

```powershell
curl.exe -I -x http://127.0.0.1:7897 https://repo.anaconda.com
```

这里：

- `curl.exe`：使用 Windows 中的 curl 可执行程序；
- `-I`：仅请求 HTTP 响应头；
- `-x`：指定代理服务器；
- `http://127.0.0.1:7897`：Clash 在 Windows 本地提供的 HTTP 代理入口。

实际返回：

```text
HTTP/1.1 200 Connection established

HTTP/1.1 200 OK
```

说明 Windows 本地 Clash 代理可以成功连接 Anaconda 官方网站。

这一步非常重要：应当先验证本地代理可用，再尝试建立 SSH 隧道，否则后续难以判断究竟是代理本身的问题，还是端口转发的问题。

**【插图位置：Windows PowerShell 中本地代理测试成功的截图】**

### 5.3 建立 SSH 反向端口转发

仍然在 Windows PowerShell 中执行：

```powershell
ssh -N -o ExitOnForwardFailure=yes -R 127.0.0.1:1080:127.0.0.1:7897 username@<服务器地址>
```

这里的 `<服务器地址>` 需要替换为实际 SSH 主机名或 IP 地址。

命令参数解释：

| 参数 | 含义 |
|---|---|
| `ssh` | 建立 SSH 连接 |
| `-N` | 不执行远程 Shell 命令，仅建立连接 |
| `-R` | 配置远程端口转发 |
| `127.0.0.1:1080` | Linux 服务器监听的代理入口 |
| `127.0.0.1:7897` | Windows 本地 Clash 代理地址 |
| `ExitOnForwardFailure=yes` | 远程端口转发建立失败时退出，而不是继续保持 SSH 连接 |

执行后，系统要求输入 SSH 登录密码。

输入密码后，PowerShell 没有新的输出，也没有返回命令提示符。

这是正常现象。

因为使用了 `-N` 参数，SSH 不会打开远程交互终端，而是保持连接用于端口转发。

此时应保持这个 PowerShell 窗口运行，不要关闭，也不要按 `Ctrl+C`。

**【插图位置：Windows PowerShell 中 SSH 反向隧道正在运行的截图】**

### 5.4 在 Linux 服务器验证隧道

另外打开一个 PowerShell 或 VS Code Remote SSH 终端，正常登录 Linux 服务器。

首先检查 1080 端口：

```bash
ss -lnt | grep ':1080 '
```

成功时应能看到本地监听记录。

然后，通过这个端口访问 Anaconda：

```bash
curl -I -x http://127.0.0.1:1080 https://repo.anaconda.com
```

本次实际返回：

```text
HTTP/1.1 200 Connection established

HTTP/2 200
```

这一步是整个解决过程最关键的验证。

因为此时请求已经不是从 Windows 直接发出，而是从 Linux 服务器发出，经过 SSH 反向隧道，再通过 Windows Clash 访问 Anaconda。

因此可以确认：

**Linux 服务器 → SSH 隧道 → Windows Clash → Anaconda 官方网站**

这条通信链路已经成功建立。

**【插图位置：Linux 服务器通过 1080 端口访问 Anaconda 成功的截图】**

### 5.5 在 Linux 中配置代理环境变量

确认隧道可用后，在 Linux 服务器执行：

```bash
export HTTP_PROXY=http://127.0.0.1:1080
export HTTPS_PROXY=http://127.0.0.1:1080
export ALL_PROXY=http://127.0.0.1:1080

export http_proxy=$HTTP_PROXY
export https_proxy=$HTTPS_PROXY
export all_proxy=$ALL_PROXY
```

其中：

- `HTTP_PROXY`：设置 HTTP 请求使用的代理；
- `HTTPS_PROXY`：设置 HTTPS 请求使用的代理；
- `ALL_PROXY`：为支持它的应用提供通用代理配置；
- 小写变量：兼容使用小写环境变量的程序。

注意：这里配置的是 **HTTP 代理入口**，即使访问的是 HTTPS 网站，也可以通过 HTTP CONNECT 建立隧道。

检查配置：

```bash
env | grep -i proxy
```

应当看到相关变量已经指向：

```text
http://127.0.0.1:1080
```

由于使用 `export` 设置，这些变量只在当前 Shell 及其子进程中有效，不会自动永久生效。

### 5.6 重新创建 Conda 虚拟环境

完成代理配置后，重新运行 Conda。

为了避免此前配置的软件源造成额外干扰，本次显式使用默认官方源：

```bash
conda create -n qwen3-lora python=3.10 -y \
  --override-channels \
  -c defaults
```

参数说明：

- `-n qwen3-lora`：指定环境名称；
- `python=3.10`：指定 Python 版本；
- `-y`：自动确认安装；
- `--override-channels`：忽略 Conda 当前配置的其他软件源；
- `-c defaults`：使用 Conda 默认官方软件源。

与之前不断发生连接失败的情况不同，这次 Conda 可以正常获取软件包索引，并完成环境创建。

最终，**`qwen3-lora` 环境创建成功**。

激活环境：

```bash
conda activate qwen3-lora
```

检查 Python 版本：

```bash
python --version
```

检查 Python 的实际路径：

```bash
which python
```

正常情况下，Python 版本应为 `3.10.x`，其路径应指向新建的 `qwen3-lora` 环境。

**【插图位置：Conda 环境成功创建及 Python 版本验证截图】**

## 六、常见问题及排查方法

### 6.1 Clash 的 7890 端口无法使用

最开始曾尝试：

```powershell
curl.exe -I -x socks5h://127.0.0.1:7890 https://repo.anaconda.com
```

结果：

```text
curl: (7) Failed to connect to
127.0.0.1 port 7890
Could not connect to server
```

原因是本次 Clash Verge 实际配置的端口为 `7897`，并非 `7890`。

解决方法：查看 Clash 设置中的实际端口，不要直接照搬教程中的端口号。

### 6.2 SSH 输入密码后没有反应

如果使用：

```powershell
ssh -N -R ...
```

登录后终端没有输出，通常是正常的。

`-N` 参数告诉 SSH 不运行远程命令，因此不会进入 Linux Shell。

可以另开一个终端，在服务器检查远程监听端口并通过 curl 验证转发是否正常。

### 6.3 SSH 提示远程端口转发失败

可能提示：

```text
Warning: remote port forwarding failed
for listen port 1080
```

可能原因：

- 1080 端口已经被其他进程占用；
- SSH 服务端禁止远程端口转发；
- 服务器端口监听策略存在限制。

可以先尝试将远程端口改为 `18080`：

```powershell
ssh -N -o ExitOnForwardFailure=yes -R 127.0.0.1:18080:127.0.0.1:7897 yeshun@<服务器地址>
```

并同步修改 Linux 中的代理端口。

如果依然失败，应咨询服务器管理员，确认是否允许 TCP 端口转发。

### 6.4 服务器 curl 成功，但 Conda 仍然无法联网

需要确认是否在执行 Conda 的同一个终端配置了环境变量：

```bash
env | grep -i proxy
```

此外，还可以检查 Conda 软件源：

```bash
conda config --show channels
```

如果软件源存在问题，可继续使用：

```bash
conda create -n qwen3-lora python=3.10 -y \
  --override-channels \
  -c defaults
```

如果仍失败，需要根据 Conda 的具体错误判断，不能将所有故障都归因于网络连接。

### 6.5 关闭 Windows 后，服务器又无法访问网络

因为 SSH 隧道运行在 Windows 本地 PowerShell 中。

当 PowerShell 关闭、电脑断网或 SSH 连接断开时，服务器端的反向转发也会失效。

因此，在创建环境和下载依赖时，需要保持：

- Windows 电脑正常联网；
- Clash Verge 正常运行；
- SSH 反向隧道持续连接。

SSH 隧道是当前连接会话提供的临时网络通道，并不是服务器永久联网方案。

## 七、进一步优化：复用 SSH 配置

如果需要经常在实验室服务器上安装 Python 依赖，每次手动输入一长串 SSH 命令并不方便。

可以考虑使用 Windows 用户目录中的 SSH 配置文件：

```text
C:\Users\<用户名>\.ssh\config
```

加入以下配置：

```sshconfig
Host enine-proxy
    HostName <服务器地址>
    User yeshun
    RemoteForward 127.0.0.1:1080 127.0.0.1:7897
    ExitOnForwardFailure yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

保存后，在 Windows PowerShell 中执行：

```powershell
ssh -N enine-proxy
```

便可建立与之前相同的反向隧道。

其中，`ServerAliveInterval` 和 `ServerAliveCountMax` 用于帮助检测失效连接，但不能保证断网后自动恢复。

对于多人共享的实验室服务器，更建议使用独立的 SSH 别名并按需建立代理，避免不必要的长期端口占用。

## 八、总结与经验

本次问题看似是 Conda 创建 Python 环境失败，但根本障碍在于服务器无法正常访问外部软件源。

实际排查过程可以概括为：

```text
Conda 创建环境失败
        ↓
发现 SOCKSHTTPSConnection 报错
        ↓
检查服务器代理环境变量
        ↓
发现 127.0.0.1:1080 没有监听
        ↓
取消代理后测试公网软件源
        ↓
清华、中科大、Anaconda 均无法连接
        ↓
确认 Windows 本地 Clash 代理正常
        ↓
利用 SSH -R 建立反向端口转发
        ↓
Linux 服务器通过 1080 端口访问公网
        ↓
配置 HTTP_PROXY 等环境变量
        ↓
成功创建 Conda 虚拟环境
```

这次实践带来几个重要经验。

**第一，软件安装失败不一定是软件本身的问题。**

遇到 `CondaHTTPError` 时，应先判断是否是网络、代理或 DNS 等基础环境问题，而不是立即重装 Conda。

**第二，更换镜像源并不能解决所有网络问题。**

如果服务器无法建立公网连接，无论使用官方源还是国内镜像，都可能失败。应先验证网络链路。

**第三，127.0.0.1 始终指向当前设备自身。**

Windows 的 `127.0.0.1:7897` 和 Linux 的 `127.0.0.1:1080` 是两个不同的本地地址，需要通过 SSH 隧道建立联系。

**第四，SSH 不只是远程登录工具。**

利用端口转发，可以在无需修改服务器系统网络配置的情况下，在授权范围内完成开发环境的网络访问需求。

本次最终通过 Windows Clash Verge + SSH 反向端口转发，成功解决了实验室服务器无法创建 Conda 环境的问题。

对于需要通过 SSH 使用计算节点、GPU 服务器或远程 Linux 开发环境的科研人员，这是一种值得掌握的实用方法。

---

**关键词：** Linux、Conda、SSH 反向隧道、Remote Port Forwarding、Clash Verge、网络代理、GPU 服务器、Python 环境配置
