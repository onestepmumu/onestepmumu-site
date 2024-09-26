+++
title = 'V2Ray + VLESS + SOCKS：Ubuntu 服务器完整搭建教程'
date = 2024-09-26T14:24:44+08:00
draft = false
+++

## 摘要

本文档详细介绍了 V2Ray 的配置过程，重点关注 VLESS 和 SOCKS 协议的实现。文档涵盖了基本的服务器设置、协议选择、安全考虑以及高级特性如 WebSocket 的使用。主要内容包括日志配置、入站和出站连接设置、流量探测（sniffing）的重要性，以及如何优化 V2Ray 以提高性能和安全性。本指南适合both新手和有经验的用户，旨在提供全面而实用的 V2Ray 配置知识。

## 1. 安装 V2Ray

使用官方提供的安装脚本：

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

## 2. 配置 V2Ray
编辑配置文件 /usr/local/etc/v2ray/config.json：

```json
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "debug"
  },
  "inbounds": [
    {
      "port": 10010,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id":"uuidgen 命令生成",
            "level": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp"
      }
    },
    {
      "port": 10086,
      "protocol": "socks",
      "settings": {
        "auth": "noauth"
      },
      "tag": "socks-in"
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag":"direct"
    }
  ],
  "dns": {
    "servers":[
      "1.1.1.1",
      "8.8.8.8"
    ]
  }
}
```
注意：将 UUID (xxx) 更改为您自己使用uuidgen命令生成的 UUID。

## 3. 配置 UFW
1. 安装 UFW

```bash
sudo apt install ufw
```
2. 确保 UFW 被禁用，以防止在配置过程中意外锁定自己
```bash
sudo ufw disable
```
3. 根据最小权限原则，设置默认策略为拒绝所有入站连接，允许所有出站连接
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
4. 启用日志，更好的监控防火墙活动
```bash
sudo ufw logging on
```
5. 定期审查规则，删除不需要的规则
```bash
sudo ufw status numbered
sudo ufw delete [rule number]
```

## 4. 添加防火墙规则并开放V2RAY服务使用到的端口
1. 为 V2Ray 配置的端口添加防火墙规则：

```bash
sudo ufw allow 10010/tcp
sudo ufw allow 10086/tcp
```
2. 检查规则设置
```bash
sudo ufw show added
```

3. 启用 UFW：

```bash
sudo ufw enable
```

4. 检查UFW状态和规则
```bash
sudo ufw status verbose
```

## 5. 配置 Linux IP 转发
```bash
cat /proc/sys/net/ipv4/ip_forward
```
如果输出为 0，表示 IP 转发当前是禁用的。如果输出为 1，则表示已启用。

### 5.1 启用 IP 转发
- 方法 1：临时启用（重启后失效）
  使用以下命令临时启用 IP 转发：

```bash
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```
注意：这种方法在系统重启后会失效。

- 方法 2：永久启用
  编辑 sysctl 配置文件：
```bash
sudo nano /etc/sysctl.conf
```
添加或修改以下行：
```
net.ipv4.ip_forward = 1
```
保存并关闭文件。
应用更改：
```bash
sudo sysctl -p
```
- 方法 3：使用 sysctl 命令
  使用 sysctl 命令直接修改内核参数：
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
注意：这种方法也是临时的，重启后会失效。

### 5.2 验证更改
无论使用哪种方法，都可以通过以下命令验证更改是否生效：

```bash
cat /proc/sys/net/ipv4/ip_forward
```
输出应该为 1，表示 IP 转发已启用。

### 5.3 安全注意事项
启用 IP 转发可能会增加系统的安全风险。确保配置适当的防火墙规则。
如果使用 UFW（Uncomplicated Firewall），可能需要额外配置以允许转发流量。
在云服务器上，可能需要在云平台的防火墙或安全组中做相应调整。

## 6. 启动 V2Ray 服务
启动 V2Ray 服务并设置为开机自启：

```bash
sudo systemctl start v2ray
sudo systemctl enable v2ray
```

## 7. 测试端口可访问性
安装 netcat：

```bash
sudo apt install netcat
```

测试 VLESS 端口（10010）：

```bash
nc -zv localhost 10010
```
测试 SOCKS 端口（10086）：

```bash
nc -zv localhost 10086
```
## 8. 检查 V2Ray 服务状态
检查 V2Ray 服务的状态：

```bash
sudo systemctl status v2ray
```
## 9. 查看日志
查看 V2Ray 的日志：

```bash
sudo tail -f /var/log/v2ray/access.log
sudo tail -f /var/log/v2ray/error.log
```
## 10. 更新 V2Ray
### 10.1 使用官方脚本更新
```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```
### 10.2 手动更新
检查最新版本：访问 V2Ray GitHub Releases
下载最新版本：
```bash
wget https://github.com/v2fly/v2ray-core/releases/download/v4.xx.x/v2ray-linux-64.zip
```
停止 V2Ray 服务：
```bash
sudo systemctl stop v2ray
```
解压并替换旧文件：
```bash
sudo unzip -o v2ray-linux-64.zip -d /usr/local/bin/v2ray
```
重新启动 V2Ray 服务：
```bash
sudo systemctl start v2ray
```
验证更新：
```bash
v2ray -version
```
## 10.3 更新后的注意事项
- 检查配置兼容性
- 测试服务是否正常
- 查看更新日志
- 备份重要数据

## 11. 安全性增强
- 定期更改 UUID 和端口
- 使用 TLS 加密（如果尚未使用）
- 启用防火墙并仅开放必要的端口
- 定期检查日志文件以发现潜在的安全问题