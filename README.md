# EngiFTP

**简体中文** | [English](README.en.md)

**EngiFTP** 是使用 Rust 编写的全功能 FTP 服务器,自带 Web 管理界面,支持用户管理、细粒度权限配置与 S3 对象存储。
基于 tokio 异步运行时从零实现 FTP 协议,Web 后台基于 axum。

## 功能特性

### FTP 协议(RFC 959 / 2389 / 2428 / 3659)

- **认证**:USER / PASS / REIN / ACCT,支持匿名登录(可开关,强制只读),连续 3 次密码错误自动断开
- **传输模式**:被动模式(PASV / EPSV)与主动模式(PORT / EPRT),支持 IPv6 扩展命令;ASCII 与二进制类型(TYPE A / I)
- **文件操作**:RETR / STOR / APPE / STOU(唯一文件名)/ DELE / RNFR+RNTO / SIZE / MDTM
- **断点续传**:REST + RETR / STOR
- **目录操作**:CWD / CDUP / PWD / MKD / RMD,以及 X 前缀兼容命令(XPWD 等)
- **列目录**:LIST(unix ls 风格)/ NLST / MLSD / MLST(机器可读格式,带 perm 事实)/ STAT
- **其他**:FEAT / OPTS UTF8 / SYST / HELP / NOOP / ABOR / MODE / STRU / ALLO / SITE CHMOD
- **多编码支持**:文件名落盘统一 UTF-8;旧设备/旧版 Windows 客户端用 GBK 等编码上送的文件名自动转码。
  服务器级编码可配置(自动检测 / UTF-8 / GBK / GB18030 / Big5 / Shift_JIS / EUC-KR / Windows-1252),
  也可在用户管理中为指定用户**强制**某种编码(优先级最高,覆盖自动检测与 OPTS UTF8,避免误检测)。
  默认"自动检测":UTF-8 优先,非法 UTF-8 字节按 GB18030(GBK 超集)解码,并以同编码回送应答与目录列表;
  现代客户端发送 `OPTS UTF8 ON` 后该会话固定 UTF-8
- **安全**:每个用户被锁定在自己的主目录内(虚拟根目录牢笼,`..` 无法越出);空闲超时;最大连接数限制;单条命令行长度上限 8KB
- **FTPS(显式 TLS,RFC 4217)**:支持 AUTH TLS / PBSZ / PROT,**明文与加密两种模式同时可用**,控制连接与数据连接均可加密;首次启用自动在数据目录生成自签名证书(cert.pem/key.pem,私钥 0600),可替换为正式证书;系统设置可开关

### 存储管理

- **存储池**:支持两种类型——`local`(本地磁盘目录)与 `s3`(**API 直连**对象存储,
  自实现 SigV4 签名,零外部依赖,兼容 AWS S3 / MinIO / 阿里云 OSS 等)
- **不挂载本地文件系统**:FTP 操作直接翻译为对象存储 API——列目录走 ListObjectsV2
  **流式分页输出**(先回 150 开数据连接,边取分页边发送,目录再大客户端也不会
  因长时间收不到数据而超时),下载走 GetObject(Range 支持断点续传),
  上传小文件单次 PUT、大文件 8MB 分段上传(内存有界),目录用标记对象表达,
  重命名为 Copy+Delete(目录递归)
- 每个用户可指定使用哪个存储池,主目录 = 池路径(或 bucket 前缀)/池内子目录
  (留空默认用户名;填 `/` 则直接使用**池根目录**,多个用户可共享池根);
  不选池则直接使用自定义路径(兼容旧配置)
- **存储统计**:概览页显示文件总数与存储占用,存储管理页按池显示文件数、占用大小与统计时间,
  支持手动「刷新统计」;S3 池用流式遍历(内存 O(1)),海量对象也不占内存;统计后台异步进行不阻塞界面
- s3 池配置:bucket(可带 `:/前缀`)、Region、自定义 Endpoint、AK/SK
  (管理接口不回传 Secret),Web 一键连接测试。
  **MinIO 等兼容存储直接填用户名/密码**(即 AccessKey/SecretKey),
  Endpoint 填如 `http://<minio-host>:9000`,已对真实 MinIO 跑通全量端到端验证
- **凭证透传**:s3 池可开启"使用 FTP 登录凭证作为 S3 AK/SK"——每个用户用自己的
  对象存储密钥登录(用户名=AccessKey,密码=SecretKey),登录时通过一次最小 S3
  请求校验凭证,权限由对象存储侧控制,服务器不存储任何密钥
- **目录缓存与后台扫描**:S3 目录列表与元数据带 TTL 缓存(默认 60 秒,系统设置可调,0=关闭)。
  后台任务自动预热池根目录、定期刷新热点目录(最近访问过的)、按预算渐进爬取子目录;
  缓存父目录列表时同步预填充子项元数据,CWD/SIZE 零请求返回 —— 实测生产桶
  根目录列表与 CWD 从 11 秒降至毫秒级。数据实时性三层保证:本服务器的写操作
  **即时失效**相关缓存(含全部祖先目录)、缓存命中时触发后台异步刷新
  (stale-while-revalidate)、其他系统直传 S3 的数据最多延迟一个 TTL 可见
- S3 池的已知限制:不支持 APPE 追加与 REST 断点续传上传(对象存储无法原地写),
  会明确返回 504;SITE CHMOD 仅本地池有效;会话被强行中断时可能残留未完成的
  分段上传,建议在 bucket 上配置 AbortIncompleteMultipartUpload 生命周期规则

### Web 管理界面(默认 http://127.0.0.1:8080)

- **概览**:在线会话数、用户数、运行时长、服务配置摘要(5 秒自动刷新)
- **用户管理**:新增 / 编辑 / 删除 / 启用禁用,每用户独立主目录,6 项权限开关(列目录、下载、上传、删除、重命名、建目录),可指定存储池与交互编码
- **存储管理**:存储池列表(类型、路径、状态、使用用户数)、新建/编辑/删除、S3 连接测试
- **在线会话**:实时查看客户端地址、登录用户、当前目录、最后命令、上传下载流量,可一键踢出
- **系统设置**:端口、被动端口范围、PASV 对外地址(NAT)、最大连接、空闲超时、欢迎语、匿名开关,在线保存
- **授权管理**:序列号在线激活 / 无网设备离线激活(导入 `license.dat`),展示机器码与授权状态,免费版 ↔ 企业版切换(详见下文「授权」)
- **运行日志**:最近 500 条日志在线查看
- 管理员密码、用户密码均以加盐 SHA-256 哈希存储;登录态使用 HttpOnly Cookie,有效期 12 小时

## 快速开始

直接运行对应平台的程序即可,**无需安装任何依赖**(Linux 为 musl 静态链接,拷贝即跑):

| 平台 | 程序文件 |
|---|---|
| Linux x86_64 | `engiftp-linux-amd64` |
| Linux ARM64 | `engiftp-linux-arm64` |
| macOS Intel | `engiftp-macos-amd64` |
| macOS Apple Silicon | `engiftp-macos-arm64` |
| Windows x86_64 | `engiftp-windows-amd64.exe` |

**Linux / macOS**:

```bash
chmod +x engiftp-linux-amd64
./engiftp-linux-amd64                 # 默认数据目录 ./data
./engiftp-linux-amd64 /etc/engiftp    # 或指定数据目录
```

> macOS 首次运行若被 Gatekeeper 拦截:执行 `xattr -d com.apple.quarantine engiftp-macos-arm64` 后再运行。

**Windows**:在终端运行 `engiftp-windows-amd64.exe [数据目录]`(或直接双击,数据目录为当前目录下 `data`)。

首次启动会生成默认配置并打印管理员初始账号:

```
Web 管理界面: http://127.0.0.1:8080
管理员账号: admin  初始密码: admin123   ← 请立即登录修改
```

浏览器打开 Web 管理界面,按初始化向导设置管理员与基本参数,然后创建 FTP 用户,即可用任意 FTP 客户端连接:

```bash
ftp <服务器IP> 2121
# 或
curl -T file.txt ftp://<服务器IP>:2121/ --user alice:密码
```

### Linux 安装为系统服务(systemd)

将 Linux 程序拷到服务器,用 `install` 子命令一键安装为开机自启服务(需 root):

```bash
scp dist/engiftp-linux-amd64 server:/tmp/engiftp
ssh server 'sudo /tmp/engiftp install'         # 默认数据目录 /var/lib/engiftp
# 指定数据目录: sudo /tmp/engiftp install /自定义/路径
```

`install` 会:创建数据目录、安装到 `/usr/local/bin/engiftp`、写入
`/etc/systemd/system/engiftp.service`(含 `Restart=on-failure`、`LimitNOFILE=65536`)、
`systemctl enable --now` 开机自启。常用运维命令:

```bash
systemctl status engiftp     # 查看状态
systemctl restart engiftp    # 重启
journalctl -u engiftp -f     # 查看日志
```

**升级到新版本**(停服务 → 备份旧版 → 替换 → 重启;新版启动失败会**自动回滚**旧版并恢复服务):

```bash
scp dist/engiftp-linux-amd64 server:/tmp/engiftp-new
ssh server 'sudo /tmp/engiftp-new update'      # 新程序自我安装
# 或: sudo engiftp update /tmp/engiftp-new      # 由已安装程序执行
```

**卸载**:`sudo engiftp uninstall`(保留数据目录)。

## 默认配置

| 配置项 | 默认值 | 说明 |
|---|---|---|
| FTP 端口 | 2121 | 监听 0.0.0.0(使用 21 端口需 root 权限) |
| 被动端口范围 | 50000–50999 | 防火墙/NAT 需放行此范围,决定并发传输上限 |
| Web 管理端口 | 8080 | 监听 0.0.0.0,局域网内可直接访问 |
| 最大并发连接 | 2000 | 实测 1000+ 在线会话约占 26MB 内存(25KB/会话) |
| 用户主目录 | ./ftp_root/<用户名> | 创建用户时可自定义 |
| 空闲超时 | 600 秒 | 控制连接无命令则断开 |

启动时会自动抬升进程文件描述符上限(逐档尝试至 65536),支撑千级以上并发;
被动端口按轮转方式分配,避免高并发下的线性扫描开销。

所有配置保存在 `<数据目录>/config.json`,用户数据保存在 `<数据目录>/users.json`,
也可直接编辑文件后重启。监听地址 / 端口类改动需要重启进程才能生效,其余配置即时生效。
用户与存储池的变更对**已连接的在线会话**同样实时生效:下一条命令前自动热重载
(权限/编码立即收紧、存储池切换立即换后端、禁用或删除账户立即断开)。

## 安全

经系统性安全审计并加固,要点:

- **密码哈希**:用户与管理员密码用 **Argon2id**(抗 GPU 暴力破解);兼容旧格式校验,改密即升级
- **凭证文件**:`config.json`/`users.json`/`pools.json` 自动设为 `0600`,数据目录 `0700`(启动时对已存在文件也会收紧)
- **FTP 协议**:PORT/EPRT 强制目标 IP == 控制连接对端(防 FTP bounce);被动数据连接校验来源 IP(防端口劫持);未登录连接 30 秒认证超时(防 slowloris)
- **路径安全**:本地后端对每次访问做 canonicalize 边界校验 + 写入 `O_NOFOLLOW`,拦截符号链接逃逸
- **改密吊销**:修改管理员密码后吊销所有登录令牌,强制重新登录
- **SSRF**:S3 endpoint 始终拒绝云元数据端点(169.254.169.254 等);设 `ENGIFTP_BLOCK_PRIVATE_S3=1` 可进一步拦截所有内网/回环地址(默认放行以兼容内网 MinIO)

- **强制改密**:仍使用出厂默认密码 `admin/admin123` 时,管理员登录后立即弹窗要求修改,改完强制重新登录
- **FTPS**:FTP 支持显式 TLS 加密(见上),敏感数据可走加密通道;Web 管理面无内建 HTTPS

**部署说明**:管理面默认监听 `0.0.0.0:8080`(适配无桌面服务器的远程管理)。若暴露公网,建议经反向代理加 HTTPS,或用防火墙限制来源 IP;首次启动后请立即按弹窗提示修改管理员密码。

## 部署建议

- **公网部署**:Web 管理界面默认监听 0.0.0.0,暴露公网前建议改为 127.0.0.1(系统设置或 config.json 中的 `web_listen`),通过 SSH 隧道或反向代理(加 HTTPS)访问
- **NAT 后部署**:在系统设置中填写"PASV 对外公布地址"为公网 IP,并放行被动端口范围
- FTP 明文传输敏感场景,建议启用 **FTPS**(系统设置开关,显式 TLS)或配合 VPN/内网使用

## 授权

**个人用户免费使用**(免费版);商业、营利或组织生产环境使用需获得**企业版授权**,
请联系 **service@xnet.ltd** 获取序列号。

### 激活为企业版

在 Web 管理界面 **系统设置 → 授权管理** 中操作。页面会显示本机**机器码**(随机生成、
持久化于数据目录 `device_id`,授权与之绑定)。

- **在线激活**:输入序列号点「激活授权」。客户端先做本地序列号格式校验,再向授权服务器
  申请,服务端校验并签发与机器码绑定的授权令牌,本地保存到数据目录 `license.json`。
  之后每次启动离线复验该令牌即可恢复企业授权,无需再次联网。
- **离线激活(无网设备)**:在「离线激活」面板点开**离线激活页**,填入序列号与机器码,
  下载签名版 `license.dat`,把内容粘贴回面板「导入授权文件」。客户端用内置公钥验签 +
  设备绑定校验后即升级企业版。
- 激活成功后,界面各处标识(品牌徽标、概览、登录页)显示「企业版」,并切换为
  《EngiFTP 企业版授权协议》。可在授权管理中「解除授权」回退为免费版。

> 授权接入参数(产品名 / 产品 ID / HMAC 密钥 / 公钥 / 服务器地址 / 离线激活页)
> 已编译进二进制,亦可用环境变量覆盖:`ENGIFTP_LICENSE_SERVER`、
> `ENGIFTP_LICENSE_PRODUCT`、`ENGIFTP_LICENSE_PRODUCT_ID`、`ENGIFTP_LICENSE_SECRET`、
> `ENGIFTP_LICENSE_PUBKEY`、`ENGIFTP_LICENSE_OFFLINE_URL`。

### 遥测

程序在后台**静默**上报运行心跳(产品 ID、版本、机器码、操作系统/架构),启动即报、
之后每 6 小时一次。完全不阻塞主流程,任何网络/上报失败都被忽略且不打印日志。
仅在配置了服务器地址时启用。

### 多语言界面

- **Web 管理界面 / Web 运行日志**:支持简体中文与英文。点击登录页或侧边栏底部的
  **EN / 中文** 开关即可切换,选择持久化在浏览器,下次访问自动应用。
- **命令行输出 / 控制台日志**:由环境变量 `ENGIFTP_LANG` 控制(`en` / `zh`);
  不设置则跟随系统 locale,默认中文。例如:`ENGIFTP_LANG=en ./engiftp`。

## 命令行用法

```
engiftp [数据目录]          启动服务(默认数据目录 ./data)
engiftp install [数据目录]  安装为 systemd 服务并启用(Linux,需 root)
engiftp update [新程序]     更新已安装的程序并重启服务(失败自动回滚)
engiftp uninstall           停用并移除 systemd 服务(保留数据目录)
engiftp help                显示帮助
```

## 数据与文件

所有数据都在你指定的**数据目录**内,迁移/备份直接拷该目录即可:

| 文件 | 用途 |
|---|---|
| `config.json` | 服务器配置(端口、编码、FTPS 等) |
| `users.json` | FTP 用户、权限、密码哈希 |
| `pools.json` | 存储池配置 |
| `license.json` / `device_id` | 企业授权凭证与机器码 |
| `cert.pem` / `key.pem` | FTPS 自签证书(可替换为正式证书) |
| `audit.redb` / `index.redb` | 操作审计日志 / S3 全局索引 |

> 含凭证的文件会自动收紧为 `0600`,数据目录为 `0700`。

---


