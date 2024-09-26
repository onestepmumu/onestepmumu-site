+++
title = '【2024最新】Ubuntu WARP SOCKS5代理部署：Docker和UFW完整配置指南'
date = 2024-09-26T01:28:29+08:00
draft = false
+++
## 1. 简介
本教程将详细指导您如何在Ubuntu服务器上使用Docker部署WARP SOCKS5代理，配置UFW防火墙，以及如何在客户端（如Shadowrocket）中使用此代理来提升网络访问速度。本指南适用于Ubuntu 20.04和22.04版本。

## 2. Ubuntu服务器配置
### 2.1 安装Docker
首先，我们需要在Ubuntu系统上安装Docker。按顺序执行以下命令：

```bash
# 更新包索引
sudo apt-get update

# 安装必要的依赖
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# 添加Docker的官方GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 设置Docker稳定版仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 再次更新包索引
sudo apt-get update

# 安装Docker引擎
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# 启动Docker服务
sudo systemctl start docker

# 设置Docker开机自启
sudo systemctl enable docker

# 验证Docker安装
sudo docker run hello-world
```

验证Docker版本：
```bash
docker --version
```
### 2.2 部署WARP SOCKS5 Docker容器
使用以下命令来运行WARP SOCKS5 Docker容器：

```bash
sudo docker run --privileged --restart=always -itd \
    --name warp_socks \
    --cap-add NET_ADMIN \
    --cap-add SYS_MODULE \
    --sysctl net.ipv6.conf.all.disable_ipv6=0 \
    --sysctl net.ipv4.conf.all.src_valid_mark=1 \
    -v /lib/modules:/lib/modules \
    -v /etc/localtime:/etc/localtime:ro \
    -v /etc/timezone:/etc/timezone:ro \
    -p 1080:1080 \
    monius/docker-warp-socks
```
这个命令会创建一个名为warp_socks的容器，并在主机的1080端口上暴露SOCKS5代理服务。

### 2.3 验证WARP容器运行状态
运行以下命令来确认容器正在运行：

```bash
sudo docker ps | grep warp_socks
```
### 2.4 配置UFW防火墙
Ubuntu默认安装了UFW防火墙。如果没有安装，可以使用以下命令安装：

```bash
sudo apt-get update
sudo apt-get install ufw
```
配置UFW以允许WARP SOCKS5代理端口：

```bash
# 允许SSH连接（如果需要）
sudo ufw allow ssh

# 允许WARP SOCKS5代理端口
sudo ufw allow 10086/tcp

# 启用UFW
sudo ufw enable

# 检查UFW状态和规则
sudo ufw status verbose
```
注意：确保在启用UFW之前允许SSH连接，否则可能会断开与服务器的连接。

### 2.5 配置 Linux IP 转发
```bash
cat /proc/sys/net/ipv4/ip_forward
```
如果输出为 0，表示 IP 转发当前是禁用的。如果输出为 1，则表示已启用。

#### 2.5.1 启用 IP 转发
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

#### 2.5.2 验证更改
无论使用哪种方法，都可以通过以下命令验证更改是否生效：

```bash
cat /proc/sys/net/ipv4/ip_forward
```
输出应该为 1，表示 IP 转发已启用。

#### 2.5.3 安全注意事项
启用 IP 转发可能会增加系统的安全风险。确保配置适当的防火墙规则。
如果使用 UFW（Uncomplicated Firewall），可能需要额外配置以允许转发流量。
在云服务器上，可能需要在云平台的防火墙或安全组中做相应调整。

## 3. 客户端配置（Shadowrocket为例）
### 3.1 添加新的代理配置
1. 打开Shadowrocket应用。
2. 点击右上角的"+"按钮。
3. 选择"类型"为"Socks5"。
4. 填写服务器信息：
   1. "地址"：填写运行Docker容器的服务器IP或域名
   2. "端口"：10086
   3. "用户名"和"密码"留空
5. 为配置命名，如"WARP SOCKS5"。
6. 点击"完成"保存。

### 3.2 使用WARP代理
1. 在Shadowrocket主界面选择刚创建的配置。
2. 打开Shadowrocket的开关启用代理。
### 3.3 配置规则（可选）
如果只想代理特定流量，可以在Shadowrocket的"配置"标签页中添加规则，例如：
```
DOMAIN-SUFFIX,openai.com,PROXY
DOMAIN-SUFFIX,ai.com,PROXY
FINAL,DIRECT
```
## 4. WARP代理监控和日志
### 4.1 查看实时日志
```bash
sudo docker logs -f warp_socks
```

### 4.2 查看最近的日志
```bash
sudo docker logs --tail 100 warp_socks
```

### 4.3 查看特定时间段的日志
```bash
sudo docker logs --since 2024-01-01T00:00:00 warp_socks
```

### 4.4 过滤日志
```bash
sudo docker logs warp_socks | grep ERROR
```
### 4.5 将日志保存到文件
```bash
sudo docker logs warp_socks > warp_logs.txt
```
## 5. WARP SOCKS5代理故障排除
1. 检查UFW防火墙状态和规则：
```bash
sudo ufw status verbose
```
2. 如果UFW阻止了连接，可以临时禁用它进行测试：
```bash
sudo ufw disable
```

3. 检查Docker容器是否正在运行：
```bash
sudo docker ps | grep warp_socks
```

4. 如果容器没有运行，尝试重启：
```bash
sudo docker restart warp_socks
```

5. 查看详细的错误日志：
```bash
sudo docker logs warp_socks
```

6. 确保服务器有足够的系统资源（CPU、内存、网络带宽）。
7. 如果使用NAT或位于防火墙后，确保正确设置了端口转发。
## 6. WARP代理安全注意事项
1. 仅在可信网络上使用此代理。
2. 定期更新Docker镜像以获取最新的安全补丁：
```bash
sudo docker pull monius/docker-warp-socks
sudo docker stop warp_socks
sudo docker rm warp_socks
# 然后重新运行2.2中的docker run命令
```
3. 考虑为SOCKS5代理添加身份验证。
4. 监控服务器的资源使用情况和安全日志。
5. 定期检查并更新UFW规则：
```bash
sudo ufw status numbered
```
6. 考虑使用fail2ban来防止暴力攻击。
## 7. WARP SOCKS5代理性能优化
1. 如果遇到性能问题，考虑调整Docker容器的资源限制。
2. 使用docker stats命令监控容器的资源使用情况：
```bash
sudo docker stats warp_socks
```
3. 如果代理速度慢，可能需要选择地理位置更近的服务器。
4. 考虑使用TCP BBR拥塞控制算法来提高网络性能：
```bash
# 检查是否已启用
sudo sysctl net.ipv4.tcp_congestion_control

# 如果未启用，可以通过以下命令启用
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
## 8. 常见问题解答（FAQ）
1. Q: 为什么我的WARP代理连接速度很慢？
    A: 可能是因为服务器位置距离您较远。尝试使用地理位置更近的服务器，或者启用TCP BBR算法。
2. Q: 如何确认WARP SOCKS5代理是否正常工作？
    A: 您可以使用在线IP查询工具，通过代理连接后查看您的IP地址是否改变。
3. Q: 我可以同时运行多个WARP SOCKS5代理实例吗？
    A: 是的，您可以在不同的端口上运行多个实例。只需修改Docker run命令中的端口映射即可。
4. Q: 如何更新WARP SOCKS5 Docker镜像？
    A: 使用sudo docker pull monius/docker-warp-socks命令拉取最新镜像，然后停止并删除旧容器，最后使用新镜像重新创建容器。
5. Q: 为什么我的客户端无法连接到WARP代理？
    A: 检查UFW防火墙设置，确保10086端口已开放。同时，验证服务器的安全组或网络ACL是否允许该端口的入站流量。