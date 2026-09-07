# Wazuh 4.10 安装与排障全记录（CSDN 合规版）

> 记录部署 Wazuh（开源 SIEM/XDR 安全检测平台）时遇到的问题、根因与解决思路，供后续参考。

---

## 一、为什么写这篇

网上 Wazuh 装教程多，但大多是照官方文档复制，默认环境干净。实际部署总会遇到各种真实问题：下载组件网络不通、证书配置、磁盘不足、官方脚本 Bug……本文记录这些坑，帮后来人少走弯路。

## 二、选型：官方脚本 vs Docker

| 方式 | 说明 |
|---|---|
| 官方一键脚本（wazuh-install.sh -a）| 自动处理证书/密码，一条命令，推荐 |
| Docker Compose | 容器化，但需手动配镜像源/证书，坑多 |

**结论：直接官方一键脚本**，别在 Docker 上折腾。

## 三、环境准备

- 虚拟机：VMware + Ubuntu 24.04 Server，内存 8GB，磁盘 40GB（注意扩容，见坑 5）
- **务必勾选 Install OpenSSH server**（方便后续 SSH 管理，避免 VMware 窗口粘贴问题）
- 装完用 `ssh` 连入，之后所有操作用 SSH 执行（能粘贴命令，减少出错）

## 四、安装步骤

```bash
# 1. 扩容磁盘（Ubuntu LVM 默认只分 19G，先扩到 38G）
sudo apt clean
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# 2. 下载并安装
curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh
sudo -E bash wazuh-install.sh -a
```

装完末尾会打印 web 地址和 admin 密码（务必保存）。

## 五、踩坑全记录

### 坑 1：避免 Docker 部署
克隆失败、main 分支镜像不存在、证书手动配……坑太多，直接官方脚本。**教训：国内部署选官方脚本一步到位。**

### 坑 2：apt 更新报 502
**现象**：`apt update` 报 `502 Bad Gateway`。
**根因**：apt 访问镜像源时走了错误的网络路径。
**解决**：配置 apt 时对国内镜像源使用直连，境外组件源走代理（区分开）。

### 坑 3：GPG key 导入失败
**现象**：`gpg: no valid OpenPGP data found`。
**根因**：安装脚本内部 curl 下载 key 时网络未正确配置，拿到无效内容。
**定位**：手动 `curl` 下载 key + `gpg --import` 能成功 → 确认为脚本下载环节问题。
**解决**：给 curl 配置全局代理即可。

### 坑 4：indexer 初始化失败（最曲折）
**现象**：`ERROR: Cannot initialize Wazuh indexer cluster`，秒级回滚。
**排查**：`journalctl -u wazuh-indexer` 显示服务其实 Started（无错误），是初始化脚本连不上 9200。
**根因**：① curl 连本地 `127.0.0.1:9200` 检查时也走代理 → 连不上；② 加 `--retry` 会和 `--output /dev/null` 冲突报 `Failed to truncate file`。
**解决**：① 配置 curl 对本地地址直连（no_proxy）；② 去掉 `--retry`。
**关键认知**：curl 连本地 9200 收到 HTTP 401 是**正常**（admin 密码还没设），不是失败，继续走。

### 坑 5：磁盘空间不足
**现象**：装 dashboard 报 `No space left on device`。
**根因**：Ubuntu LVM 默认根分区只 19G（物理盘 40G），Wazuh 全套 + 缓存超过。
**解决**：装之前先扩容（lvextend + resize2fs），把 19G 扩到 38G。

### 坑 6：重跑报 already installed / 端口占用
**现象**：`Wazuh manager already installed` / `Port 1515/55000 is being used`。
**根因**：回滚不彻底，`/var/ossec` 残留、进程占端口。
**解决**：`sudo pkill -9 -f wazuh && sudo rm -rf /var/ossec`。dpkg 卸载脚本报错则挪走 .prerm/.postrm 后再 purge。

### 坑 7：Wazuh API 用户不存在（官方 Bug）
**现象**：`The Wazuh API user wazuh does not exist`。
**根因**：官方 Bug（GitHub Issue #66）。脚本只替换已存在用户密码，不添加缺失用户，而 4.10 默认 internal_users.yml 不含 wazuh 用户。
**解决**：改脚本，在改密码前先检查用户是否存在、不存在就添加（末尾附修复脚本）。

## 六、排障思路

1. **代理只给境外组件用，国内镜像源直连**——否则 apt 502
2. **curl 连本地（127.0.0.1）必须直连**，别让代理拦截本地检测
3. **磁盘先扩容再装**，否则装到 dashboard 才爆
4. **排查三板斧**：`journalctl -u 服务`、`ps aux | grep apt`、`df -h`——别蒙着改

## 七、结语

这套排障思路，比装成功本身更有价值。遇到类似的代理/证书/磁盘/脚本 Bug，都知道怎么定位了。
