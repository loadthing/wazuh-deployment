# Wazuh 4.10 国内环境安装踩坑全记录（从 Docker 到官方脚本）

> 一句话：**在国内网络 + 代理环境下，Wazuh 到底怎么装才不翻车。**
> 这篇是我从早上折腾到下午、把 Docker 和官方脚本两条路都踩了一整遍后的完整记录，包含遇到的每一个坑和解决思路。
> 适合：想搭 Wazuh（开源 SIEM/XDR）做安全检测平台的运维/安全从业者、学习者。

---

## 目录

1. 为什么写这篇
2. 选型：Docker 还是官方脚本？（先看结论）
3. 环境准备（VMware + Ubuntu）
4. 核心前提：网络代理（Clash）
5. 官方一键安装步骤
6. 踩坑全记录（7 个坑，附每个的排错思路）
7. FAQ（常见问题）
8. 装好后怎么办（验证 + 接入 agent）
9. 经验总结

---

## 1. 为什么写这篇

国内外讲 Wazuh 安装的文章不少，但**绝大多数是"照官方文档复制粘贴"**，默认你网络干净、环境标准。而真实场景是：国内访问境外源受阻、要配代理、代理又误伤国内源、磁盘不够、官方脚本还有 Bug……

这篇文章的价值在于**把这些真实坑一个个贴出来**：现象什么样、根因在哪、怎么定位、怎么解决。这篇能帮你省下一整天。

---

## 2. 选型：Docker 还是官方脚本？

| 方式 | 优点 | 缺点（国内 + 新手）|
|---|---|---|
| **Docker Compose**（wazuh-docker）| 容器化、组件隔离 | 镜像拉不动、克隆失败、版本/证书手动配，坑极多 |
| **官方一键脚本**（wazuh-install.sh -a）| 自动处理证书/密码，一条命令 | 需要配好代理（下文详述）|

> **结论：国内直接官方一键脚本 + 配好代理。** Docker 方式我踩了一整遍（克隆、镜像、证书全出问题），最后放弃。别走这条弯路。

---

## 3. 环境准备（VMware + Ubuntu 24.04）

### 3.1 虚拟机配置
- **内存 8GB**（Wazuh all-in-one 需要，低于这个容易 OOM）
- **磁盘 40GB**（注意：Ubuntu LVM 默认只分 19G，剩下的 21G 用不上，后面要扩容——见坑 5）

### 3.2 安装 Ubuntu 时关键一步
- **务必勾选 `Install OpenSSH server`**

> ⚠️ 这一步很关键：VMware 窗口**不能从宿主机粘贴命令**（要装 VMware Tools 才支持剪贴板）。如果没装 OpenSSH，你只能手敲那些长命令，极易出错。**勾上它，装完直接从 Windows `ssh` 连进去，就能粘贴了。**

### 3.3 SSH 连入（装完系统后）
```bash
# Windows PowerShell 里
ssh <你的用户名>@<虚拟机IP>
```
输入 yes + 密码。之后所有操作都在 SSH 里做（能粘贴命令）。

---

## 4. 核心前提：网络代理（Clash）

Wazuh 从境外的 `packages.wazuh.com` 下载组件，**国内直连会失败/超时，必须走代理**。

### 4.1 宿主机 Clash 配置
1. **开启 Allow LAN**，绑定地址填 `*`（表示所有接口，这样虚拟机才能访问）
2. **Windows 防火墙放行代理端口**（默认 7890）
```powershell
netsh advfirewall firewall add rule name="Clash7890" dir=in action=allow protocol=TCP localport=7890
```

### 4.2 确认代理能连（虚拟机里测）
```bash
# 另开 SSH 窗口，测代理能否访问 Docker Hub（返回 401 即通）
curl -x http://<宿主IP>:7890 --max-time 10 -s -o /dev/null -w "%{http_code}\n" https://registry-1.docker.io/v2/
```
> 宿主 IP 在 VMware NAT 下通常是 `192.168.254.1`（用 `ip route | grep default` 看网关，宿主一般是 `.1`）。返回 **401** = 代理通了。

---

## 5. 官方一键安装步骤

### 5.1 配系统级代理（curl/wget 走代理）
```bash
sudo tee -a /etc/environment <<'EOF'
http_proxy=http://<宿主IP>:7890
https_proxy=http://<宿主IP>:7890
EOF
source /etc/environment
```

### 5.2 配 apt 代理（关键：wazuh 源走代理、Ubuntu 源直连）
```bash
sudo tee /etc/apt/apt.conf.d/95proxy <<'EOF'
Acquire::http::Proxy "http://<宿主IP>:7890";
Acquire::https::Proxy "http://<宿主IP>:7890";
Acquire::http::Proxy::cn.archive.ubuntu.com "DIRECT";
Acquire::https::Proxy::cn.archive.ubuntu.com "DIRECT";
Acquire::http::Proxy::archive.ubuntu.com "DIRECT";
Acquire::https::Proxy::archive.ubuntu.com "DIRECT";
Acquire::http::Proxy::security.ubuntu.com "DIRECT";
Acquire::https::Proxy::security.ubuntu.com "DIRECT";
Acquire::http::Proxy::mirrors.tuna.tsinghua.edu.cn "DIRECT";
Acquire::https::Proxy::mirrors.tuna.tsinghua.edu.cn "DIRECT";
EOF
```

> 原理：Wazuh 组件（境外）走代理，Ubuntu 源（清华/security/archive，国内）**必须直连**，让 apt 不走代理访问国内源（否则 502，见坑 2）。

### 5.3 配 curl 全局代理（wazuh-install.sh 内部的 curl 也能走代理）
```bash
sudo tee -a /etc/curlrc <<'EOF'
proxy = "http://<宿主IP>:7890"
EOF
```

### 5.4 下载并安装
```bash
curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh
sudo -E bash wazuh-install.sh -a
```
> `-E` 保留环境变量（让 sudo 里 curl 也走代理）。

### 5.5 装完保存信息
装成功会在末尾打印：
```
INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
    User: admin
    Password: <随机密码>
```
**务必把 `admin` 密码保存好**（Login dashboard 用）。

📸 **截图**：装成功的 Summary（含密码）一定要截。

---

## 6. 踩坑全记录（7 个坑）

### 坑 1：Docker 方式踩雷（已放弃）
- **现象**：`git clone wazuh-docker` 失败（gitclone 给残缺版）；镜像 `wazuh-*:5.x` 不存在（main 分支是 5.x 开发版）；证书生成报 `is a directory`
- **教训**：国内 + 新手，**别碰 Docker 部署 Wazuh**。直接官方脚本。
- **如果非要用 Docker**：checkout 到 `v4.10.0`、配国内镜像加速器、证书用 wazuh-certs-generator（0.0.2 镜像 + config.yml 放对位置）。

### 坑 2：apt 访问 Ubuntu 源报 502
- **现象**：`apt update` 报 `502 Bad Gateway [IP: <宿主IP>:7890]`
- **根因**：系统级 `http_proxy` 全局代理让 **apt 也走 Clash**，而 Clash 访问 Ubuntu 源（国内镜像）返回 **502**。
- **解决**：apt 代理里把 **Ubuntu 源设为 `DIRECT`**（见 5.2）。apt 访问 wazuh 源走代理、访问国内源直连。
- **排错思路**：看报错的 IP 是宿主代理，即 apt 走了代理 → 让国内源 DIRECT。

### 坑 3：GPG key 导入失败 `Cannot import Wazuh GPG key`
- **现象**：`gpg: no valid OpenPGP data found`，导入失败
- **根因**：wazuh-install.sh 内部用 `common_curl` 下载 key，**curl 没走代理**，拿到的是无效内容（HTML/错误页）。
- **定位**：手动 `curl -sOL https://packages.wazuh.com/key/GPG-KEY-WAZUH && sudo gpg --import GPG-KEY-WAZUH` —— 手动能导入成功，说明是脚本里 curl 不走代理。
- **解决**：配 `/etc/curlrc` 全局代理（见 5.3），curl 都走代理，key 下载就正常了。

### 坑 4：indexer 初始化失败 `Cannot initialize Wazuh indexer cluster`（最曲折）
- **现象**：装完 indexer 后，初始化集群安全设置失败，秒级回滚。日志只有 `ERROR: Cannot initialize Wazuh indexer cluster`。
- **排查**：`journalctl -u wazuh-indexer --no-pager | tail` 看到服务其实 `Started` 了（没有错误），说明是**初始化脚本连不上 9200**。
- **根因（两个叠加）**：
  1. curl 连本地 `127.0.0.1:9200` 检查 indexer 就绪时，**也走了代理** → 代理连不上本地 indexer → 失败。
     **解决**：`no_proxy=127.0.0.1,localhost`（可加进 `/etc/environment`），curl 本地直连。
  2. 我曾给 curl 加 `--retry`，但和 `--output /dev/null` 冲突报 `Failed to truncate file`。
     **解决**：去掉 `--retry`。curl 收到 indexer 的 **HTTP 401** 响应（admin 密码还没设，**正常**）即判定就绪，继续走。
- **关键认知**：curl 连本地 9200 收到 **401** 是**正常**的（indexer 安全插件还没初始化），不是失败，wazuh-install.sh 会继续。别把 401 当问题。

### 坑 5：磁盘空间不足 `No space left on device`
- **现象**：装 dashboard 时 `E: You don't have enough free space in /var/cache/apt/archives/`，wazuh 全套 + apt 缓存把分区塞满。
- **根因**：Ubuntu LVM **默认只给根分区 19G**（物理盘 40G，剩 21G 没用上）。Wazuh all-in-one（indexer+manager+dashboard+缓存）超过 19G。
- **定位**：`df -h` 看根分区是 19G。
- **解决**：**扩容 LVM 逻辑卷**（把 19G 扩到 38G）：
  ```bash
  sudo apt clean
  sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
  ```
  > 装之前就先扩容，免得到 dashboard 才爆。

### 坑 6：回滚残留 `already installed` / 端口占用
- **现象**：装失败回滚后重跑，报 `Wazuh manager already installed` 或 `Port 1515/55000 is being used`。
- **根因**：回滚清理不彻底：`/var/ossec` 目录残留、wazuh 进程（authd/apid，进程名 wazuh）还占着端口。
- **解决**：
  ```bash
  sudo pkill -9 -f wazuh     # 杀残留进程
  sudo rm -rf /var/ossec     # 删残留目录
  ```
- **dpkg 卸载脚本报错**（pre-removal/post-removal script `exit 127/1`，常有 `find /var/ossec/api/: No such file`）：把脚本挪走跳过：
  ```bash
  sudo mv /var/lib/dpkg/info/wazuh-manager.prerm /var/lib/dpkg/info/wazuh-manager.prerm.bak
  sudo mv /var/lib/dpkg/info/wazuh-manager.postrm /var/lib/dpkg/info/wazuh-manager.postrm.bak
  sudo dpkg --purge --force-all wazuh-manager
  ```

### 坑 7：Wazuh API 用户不存在 `The Wazuh API user wazuh does not exist`
- **现象**：最后一步报 `The Wazuh API user wazuh does not exist` / `wazuh-wui does not exist`。
- **根因**：**官方 Bug（GitHub Issue #66）**——wazuh-install.sh 用 `awk` **只替换已存在用户的密码，不会添加缺失用户**，而 4.10 的默认 `internal_users.yml` **不含 `wazuh`/`wazuh-wui`** 两个 API 用户，所以永远创建不出来。
- **定位**：看 `passwords_changePassword` 函数，确认它只 awk 替换不添加。
- **解决**：改 wazuh-install.sh，在改密码前**先检查用户是否存在，不存在就添加**（用 python 在含 `hashes[i]` 的 awk 行前插一段逻辑）：
  ```bash
  sudo python3 << 'PYEOF'
  path = 'wazuh-install.sh'
  lines = open(path).read().split('\n')
  out = []
  inserted = False
  for line in lines:
      if not inserted and 'awk -v new=' in line and 'hashes[i]' in line:
          indent = line[:len(line) - len(line.lstrip())]
          out.append(indent + 'grep -q "^${users[i]}:" /etc/wazuh-indexer/backup/internal_users.yml || echo -e "\\n${users[i]}:\\n  hash: \\"${hashes[i]}\\"\\n  reserved: true" >> /etc/wazuh-indexer/backup/internal_users.yml')
          inserted = True
      out.append(line)
  open(path, 'w').write('\n'.join(out))
  print('OK' if inserted else 'NOT FOUND')
  PYEOF
  ```
  之后重跑 `sudo -E bash wazuh-install.sh -a`，脚本会先补上这两个用户，检查就能过。

---

## 7. FAQ

**Q1：装到一半失败了，能直接重跑吗？**
能。wazuh-install.sh 支持重试，但回滚可能留残留。重跑前先清残留（见坑 6）：`sudo pkill -9 -f wazuh && sudo rm -rf /var/ossec`。如果报 `already installed` 用 `-o` 参数覆盖（`-o` 会清掉旧配置）。

**Q2：怎么确认没卡住（日志不动了）？**
另开窗口 `ps aux | grep -E "apt|wget|curl|dpkg" | grep -v grep`——有 apt/wget 进程说明在下载，正常等；没有才是卡住。

**Q3：docker 装 Wazuh 到底行不行？**
能，但国内 + 新手别碰。官方脚本一条命令，docker 我要踩十几个坑（克隆/镜像/证书），除非你闲。

**Q4：装成功的密码忘了怎么办？**
Wazuh all-in-one 的 admin 密码在安装日志里。装完 `grep -i "password" /var/log/wazuh-install.log` 能看到。

**Q5：dashboard 打不开/证书报错？**
自签名证书，浏览器点"高级 → 继续访问"。确认 `wazuh-dashboard` 服务在跑：`systemctl status wazuh-dashboard`。

---

## 8. 装好后怎么办（验证 + 接入 agent）

### 8.1 确认服务健康
```bash
systemctl status wazuh-indexer wazuh-manager wazuh-dashboard filebeat | grep -E "active|inactive"
```
全部 `active (running)` 就正常。

### 8.2 登录 dashboard
浏览器访问 `https://<虚拟机IP>`（443），账号 `admin`，密码见安装日志。能看到 Overview、MITRE ATT&CK、FIM 等模块。

### 8.3 接入第一个 agent（开始监控端点）
dashboard 里点 **Add agent**，选择要监控的系统（如 Linux），会给出安装 wazuh-agent 的命令，在目标机器上执行后，就能在 Wazuh 里看到这台机器的安全日志和告警了。

---

## 9. 经验总结

1. **国内装 Wazuh，直接官方一键脚本 + 配好代理**，别折腾 Docker
2. **代理只给 Wazuh 组件（境外）用，Ubuntu 源（国内）必须直连**——否则 apt 502
3. **curl 连本地（127.0.0.1）必须 `no_proxy` 直连**，别让代理拦截本地检测
4. **磁盘 40G 物理盘，LVM 只分 19G，先扩容再装**，否则装到 dashboard 才爆
5. **Wazuh 官方脚本有坑**（API 用户 Bug），遇到就去搜 GitHub Issue，别死磕
6. **排查三板斧**：`journalctl -u <服务>` 看服务日志、`ps aux | grep apt` 看是否在下载、`df -h` 先看磁盘。别蒙着改配置

---

*这套排坑思路，比装成功本身更有价值。下次遇到类似的（代理/证书/磁盘/脚本 Bug），都知道怎么定位了。*

*原创，转载请注明出处。*
