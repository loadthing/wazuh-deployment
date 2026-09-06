# Wazuh 4.10 国内环境部署手册（照着做就能成功）

> **用途**：可复用的部署参考手册。下次（或别人）重新部署 Wazuh，照着这份做就能成功；遇到问题翻后面的「踩坑附录」。
> **环境**：VMware + Ubuntu 24.04 Server + Clash 代理（国内网络）。

---

## 一、快速部署清单（照着勾，全打勾就成功了）

- [ ] 虚拟机：内存 8GB、磁盘 40GB
- [ ] Ubuntu 24.04 Server，**勾选 `Install OpenSSH server`**
- [ ] 宿主机 Clash：Allow LAN 开、绑定 `*`、防火墙放行 7890
- [ ] 虚拟机能走宿主机代理（`curl -x` 测试返回 401）
- [ ] 配好系统代理（/etc/environment）、apt 代理（95proxy）、curl 代理（/etc/curlrc）
- [ ] 磁盘扩容（LVM 从 19G 扩到 38G）——**装之前做**
- [ ] `curl -sO` 下载 wazuh-install.sh，`sudo -E bash wazuh-install.sh -a` 安装
- [ ]（若报 API 用户 Bug）先跑 python 修复脚本再装
- [ ] 装成，保存 admin 密码
- [ ] 验证：`systemctl status` 四个服务 active，dashboard 能登录

---

## 二、详细部署步骤

### 2.1 环境准备

**虚拟机配置**（VMware）：
- 内存 **8GB**（低于此容易 OOM）
- 磁盘 **40GB**（注意：Ubuntu LVM 默认只分 19G，**先扩容**，见 2.3）

**Ubuntu 安装**：
- Ubuntu 24.04 Server，**务必勾选 `Install OpenSSH server`** ← 关键

> ⚠️ 为什么必须勾 OpenSSH：VMware 窗口**不能从宿主粘贴命令**。没装 OpenSSH 你只能手敲长命令，极易出错。勾上它，装完从 Windows `ssh` 连进去，就能**粘贴**命令了。这一步是后面所有命令能否顺利执行的基石。

**SSH 连入**（Windows PowerShell）：
```bash
ssh <用户名>@<虚拟机IP>
```
输入 `yes` + 密码。之后所有操作都在 SSH 里做。

### 2.2 代理配置（核心前提）

Wazuh 从境外下载，必须走代理。分三步：

**① 宿主机 Clash**：
- Allow LAN 开，绑定地址填 `*`
- Windows 防火墙放行 7890：
```powershell
netsh advfirewall firewall add rule name="Clash7890" dir=in action=allow protocol=TCP localport=7890
```

**② 测代理连通**（虚拟机里）：
```bash
curl -x http://<宿主IP>:7890 --max-time 10 -s -o /dev/null -w "%{http_code}\n" https://registry-1.docker.io/v2/
```
返回 **401** = 通。宿主 IP 一般是 `192.168.254.1`（用 `ip route | grep default` 看网关，宿主通常是 `.1`）。

**③ 三个配置**（系统级 / apt / curl）：
```bash
# 系统级（curl/wget 走代理）
sudo tee -a /etc/environment <<'EOF'
http_proxy=http://<宿主IP>:7890
https_proxy=http://<宿主IP>:7890
EOF
source /etc/environment

# apt 代理（关键：wazuh 源走代理、Ubuntu 源直连）
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

# curl 全局代理（wazuh-install.sh 内部 curl 也走代理）
sudo tee -a /etc/curlrc <<'EOF'
proxy = "http://<宿主IP>:7890"
EOF
```

> **核心原则：** Wazuh 组件（境外）走代理，Ubuntu 源（国内）**必须直连**（`DIRECT`）。否则 apt 访问国内源走代理会 502（见踩坑 2）。

### 2.3 安装

**先扩容磁盘**（避免装到一半爆空间）：
```bash
sudo apt clean
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h /   # 确认从 19G 变 ~38G
```

**下载并安装**：
```bash
curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh
sudo -E bash wazuh-install.sh -a
```

**若报 `The Wazuh API user wazuh does not exist`**（官方 Bug）→ 先跑修复脚本再装（见踩坑 7）。

### 2.4 装完保存 + 验证

装成功末尾会打印：
```
INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
    User: admin
    Password: <随机密码>
```
**保存 admin 密码**。

验证服务健康：
```bash
systemctl status wazuh-indexer wazuh-manager wazuh-dashboard filebeat | grep -E "active|inactive"
```
全部 `active (running)` = 正常。

登录 dashboard：浏览器 `https://<虚拟机IP>`（443），admin + 密码。能看到 Overview、MITRE ATT&CK、FIM 等模块。

---

## 三、踩坑附录（遇到问题翻这里）

### 坑 1：避免用 Docker 部署（除非你闲）
Docker 方式（wazuh-docker）国内坑极多：克隆失败、main 分支是 5.x（镜像不存在）、证书手动配。**国内 + 新手直接官方脚本**。
若非要用 Docker：checkout 到 `v4.10.0`、配国内镜像加速器、证书用 wazuh-certs-generator（0.0.2 镜像 + config.yml 放 `/config/certs.yml`）。

### 坑 2：apt 访问 Ubuntu 源 502
**现象**：`apt update` 报 `502 Bad Gateway [IP: 宿主IP:7890]`。
**根因**：系统代理误伤 apt，apt 访问国内 Ubuntu 源也走代理 → 502。
**解决**：apt 代理里 Ubuntu 源设 `DIRECT`（见 2.2③）。apt 走 wazuh 源（境外代理）、直连国内源。

### 坑 3：GPG key 导入失败
**现象**：`gpg: no valid OpenPGP data found`。
**根因**：wazuh-install.sh 内部 curl 没走代理，下载 key 拿到无效内容。
**定位**：手动 `curl -sOL https://packages.wazuh.com/key/GPG-KEY-WAZUH && sudo gpg --import GPG-KEY-WAZUH`——手动成功即确是脚本 curl 不走代理。
**解决**：配 `/etc/curlrc` 全局代理。

### 坑 4：indexer 初始化失败（最曲折）
**现象**：`ERROR: Cannot initialize Wazuh indexer cluster`，秒级回滚。
**排查**：`journalctl -u wazuh-indexer --no-pager | tail` 看到服务其实 `Started`（无错误），是初始化脚本连不上 9200。
**根因**：① curl 连本地 `127.0.0.1:9200` 检查也走代理 → 连不上；② 加 `--retry` 会和 `--output /dev/null` 冲突报 `Failed to truncate file`。
**解决**：① `no_proxy=127.0.0.1,localhost`（加进 /etc/environment）；② 去掉 `--retry`。
**关键认知**：curl 连本地 9200 收到 **HTTP 401** 是**正常**（admin 密码还没设），不是失败，继续走。

### 坑 5：磁盘空间不足
**现象**：装 dashboard 报 `No space left on device` / `You don't have enough free space in /var/cache/apt/archives/`。
**根因**：Ubuntu LVM 默认只给根分区 19G（物理盘 40G），Wazuh 全套 + 缓存超过。
**解决**：装之前就扩容（见 2.3）。别等 dashboard 才爆。

### 坑 6：回滚残留（重跑报 already installed / 端口占用）
**现象**：重跑报 `Wazuh manager already installed` / `Port 1515/55000 is being used`。
**根因**：回滚不彻底，`/var/ossec` 残留、wazuh 进程占端口。
**解决**：
```bash
sudo pkill -9 -f wazuh
sudo rm -rf /var/ossec
```
dpkg 卸载脚本报错（exit 127/1）→ 挪走脚本：
```bash
sudo mv /var/lib/dpkg/info/wazuh-manager.prerm /var/lib/dpkg/info/wazuh-manager.prerm.bak
sudo mv /var/lib/dpkg/info/wazuh-manager.postrm /var/lib/dpkg/info/wazuh-manager.postrm.bak
sudo dpkg --purge --force-all wazuh-manager
```

### 坑 7：Wazuh API 用户不存在（官方 Bug）
**现象**：`The Wazuh API user wazuh does not exist` / `wazuh-wui does not exist`。
**根因**：官方 Bug（GitHub Issue #66）。脚本用 awk **只替换已有用户密码，不添加缺失用户**，而 4.10 默认 internal_users.yml **不含 wazuh/wazuh-wui**。
**解决**：改 wazuh-install.sh，改密码前先补上缺失用户：
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
print('OK 已修复' if inserted else 'NOT FOUND')
PYEOF
```
再重跑 `sudo -E bash wazuh-install.sh -a`。

---

## 四、FAQ

**装到一半失败能重跑吗？** 能。先清残留：`sudo pkill -9 -f wazuh && sudo rm -rf /var/ossec`。报 `already installed` 用 `-o`（覆盖）。

**怎么确认没卡住？** 另开窗口 `ps aux | grep -E "apt|wget|curl|dpkg" | grep -v grep`——有进程在下载=正常，没有=卡住。

**密码忘了？** `grep -i "password" /var/log/wazuh-install.log`。

**dashboard 证书报错？** 自签名，点"高级→继续访问"。确认 `systemctl status wazuh-dashboard` 在跑。

---

## 五、命令速查表

| 场景 | 命令 |
|---|---|
| 查磁盘 | `df -h` |
| 扩容 LVM | `sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv && sudo resize2fs /dev/ubuntu-vg/ubuntu-lv` |
| 清 apt 缓存 | `sudo apt clean` |
| 杀残留进程 | `sudo pkill -9 -f wazuh` |
| 删残留目录 | `sudo rm -rf /var/ossec` |
| 清残留包 | `sudo dpkg --purge --force-all wazuh-manager`（先挪 .prerm/.postrm）|
| 查服务状态 | `systemctl status wazuh-indexer wazuh-manager wazuh-dashboard filebeat` |
| 看安装日志 | `journalctl -u wazuh-indexer --no-pager \| tail` 或 `tail -f /var/log/wazuh-install.log` |
| 看是否在下载 | `ps aux \| grep -E "apt\|wget\|curl\|dpkg" \| grep -v grep` |

---

## 六、经验总结

1. 国内装 Wazuh，直接官方一键脚本 + 配好代理，别折腾 Docker
2. 代理只给境外组件用，Ubuntu 源必须直连（否则 apt 502）
3. curl 连本地（127.0.0.1）必须 `no_proxy` 直连，别让代理拦截本地检测
4. 磁盘 40G 物理盘 LVM 只分 19G，**先扩容再装**
5. Wazuh 官方脚本有 Bug（API 用户），遇到搜 GitHub Issue，改脚本绕开
6. 排查三板斧：`journalctl -u 服务` / `ps aux | grep apt` / `df -h`，别蒙着改
