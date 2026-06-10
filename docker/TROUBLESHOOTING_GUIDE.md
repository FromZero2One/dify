# Dify 本地部署常见问题解决指南

> **文档说明：** 本文档汇总了 Dify 本地部署时遇到的常见问题及解决方案，包括代理配置、Ollama 集成等。

---

## 目录

1. [Docker 容器访问宿主机服务的通用问题](#1-docker-容器访问宿主机服务的通用问题)
2. [插件安装慢或失败 - 代理配置](#2-插件安装慢或失败---代理配置)
3. [本地 Ollama / LM Studio 模型连接失败](#3-本地-ollama--lm-studio-模型连接失败)
4. [常见问题排查](#4-常见问题排查)
5. [参考命令速查](#5-参考命令速查)

---

## 1. Docker 容器访问宿主机服务的通用问题

### 核心问题

**Linux 系统中，Docker 容器无法通过 `localhost` 或 `127.0.0.1` 访问宿主机服务。**

### 原因分析

- 容器内的 `localhost` 指向容器自己，不是宿主机
- `host.docker.internal` 只在 Docker Desktop for Mac/Windows 中自动配置，Linux 上不可用
- 宿主机服务如果只监听 `127.0.0.1`，容器无法连接

### 通用解决方案

**两步走策略：**

1. **修改宿主机服务监听地址**
   ```bash
   # 将服务从监听 127.0.0.1 改为 0.0.0.0
   # 例如：代理服务器、Ollama 等
   ```

2. **使用 Docker 网络网关 IP 访问**
   ```bash
   # 获取网关 IP
   docker network inspect docker_default | grep Gateway
   
   # 示例输出："Gateway": "172.21.0.1"
   # 在容器配置中使用 http://172.21.0.1:端口
   ```

---

## 2. 插件安装慢或失败 - 代理配置

### 问题描述

在本地部署 Dify 时，安装插件速度很慢或失败，而官方云服务 (https://cloud.dify.ai/tools) 安装插件很快。

**根本原因：**
- Dify 需要访问国外的插件市场 (marketplace.dify.ai)
- 本地网络需要通过代理才能快速访问国外资源
- Docker 容器默认无法使用宿主机的代理配置

### 解决方案总览

需要完成以下三个关键配置：

1. ✅ 修改代理服务器监听地址（允许容器连接）
2. ✅ 配置 Dify 容器的代理环境变量
3. ✅ 重启服务应用配置

### 详细配置步骤

#### 步骤 1：检查代理服务器状态

```bash
# 检查代理端口是否在监听
ss -tlnp | grep 7890
```

**预期输出示例：**
```
LISTEN 0  4096  127.0.0.1:7890  0.0.0.0:*  users:(("xray",pid=150930,fd=3))
```

**关键问题识别：**
- ❌ 如果监听地址是 `127.0.0.1:7890`，Docker 容器**无法连接**
- ✅ 需要改为 `0.0.0.0:7890` 或 `*:7890`

---

#### 步骤 2：修改代理服务器配置（以 xray 为例）

##### 2.1 找到配置文件

```bash
# 查找 xray 进程
ps aux | grep xray | grep -v grep

# 根据进程信息找到配置文件路径
find /home/wsm/.local/share/v2rayN -name "config.json" -type f
```

##### 2.2 备份并修改配置

```bash
# 备份原配置
cp /home/wsm/.local/share/v2rayN/binConfigs/config.json \
   /home/wsm/.local/share/v2rayN/binConfigs/config.json.backup

# 修改监听地址（将 127.0.0.1 改为 0.0.0.0）
sed -i 's/"listen": "127.0.0.1"/"listen": "0.0.0.0"/' \
    /home/wsm/.local/share/v2rayN/binConfigs/config.json

# 验证修改
grep -A 2 '"tag": "socks"' /home/wsm/.local/share/v2rayN/binConfigs/config.json
```

**修改前后对比：**
```json
// 修改前
{
  "tag": "socks",
  "port": 7890,
  "listen": "127.0.0.1",  // ❌ 只监听本地回环
  ...
}

// 修改后
{
  "tag": "socks",
  "port": 7890,
  "listen": "0.0.0.0",    // ✅ 监听所有接口
  ...
}
```

##### 2.3 准备必要的数据文件

xray 启动需要 `geosite.dat` 和 `geoip.dat` 文件：

```bash
# 查找数据文件位置
find /home/wsm/.local/share/v2rayN -name "geosite.dat" -o -name "geoip.dat"

# 复制到 xray 二进制文件目录
cp /home/wsm/.local/share/v2rayN/bin/{geosite.dat,geoip.dat} \
   /home/wsm/.local/share/v2rayN/bin/xray/
```

##### 2.4 永久配置方案（推荐）

> ⚠️ **直接修改 `config.json` 不可靠！** v2rayN 会在切换节点或更新时重新生成该文件，覆盖手动修改。以下方案确保配置永久生效。

###### 2.4.1 修改 v2rayN 的 AllowLANConn 设置

v2rayN 的 `guiNConfig.json` 中 `AllowLANConn` 控制是否监听局域网：

```bash
python3 -c "import json; f=open('/home/wsm/.local/share/v2rayN/guiConfigs/guiNConfig.json','r'); d=json.load(f); d['Inbound'][0]['AllowLANConn']=True; f.close(); f=open('/home/wsm/.local/share/v2rayN/guiConfigs/guiNConfig.json','w'); json.dump(d,f,indent=2); f.close(); print('Done')"
```

也可以在 v2rayN GUI 中操作：**设置 → 允许来自局域网的连接 → 勾选**

###### 2.4.2 创建 Docker 专用 xray 配置和同步脚本

```bash
mkdir -p /home/wsm/.local/share/v2rayN/binConfigs/custom
cp /home/wsm/.local/share/v2rayN/binConfigs/config.json /home/wsm/.local/share/v2rayN/binConfigs/custom/xray-docker-proxy.json
sed -i 's/"listen": "127.0.0.1"/"listen": "0.0.0.0"/' /home/wsm/.local/share/v2rayN/binConfigs/custom/xray-docker-proxy.json
cat > /home/wsm/.local/share/v2rayN/binConfigs/custom/sync-xray-docker-config.sh << 'SCRIPT'
#!/bin/bash
SRC=/home/wsm/.local/share/v2rayN/binConfigs/config.json
DST=/home/wsm/.local/share/v2rayN/binConfigs/custom/xray-docker-proxy.json
[ -f "$SRC" ] && cp "$SRC" "$DST" && sed -i 's/"listen": "127.0.0.1"/"listen": "0.0.0.0"/' "$DST" && echo OK || echo ERROR
SCRIPT
chmod +x /home/wsm/.local/share/v2rayN/binConfigs/custom/sync-xray-docker-config.sh
```

###### 2.4.3 创建 systemd 用户服务（开机自启）

```bash
mkdir -p /home/wsm/.config/systemd/user
cat > /home/wsm/.config/systemd/user/xray-docker-proxy.service << 'EOF'
[Unit]
Description=Xray Proxy for Docker Containers (Listen on 0.0.0.0)
After=network.target

[Service]
Type=simple
ExecStartPre=/home/wsm/.local/share/v2rayN/binConfigs/custom/sync-xray-docker-config.sh
ExecStart=/home/wsm/.local/share/v2rayN/bin/xray/xray run -c /home/wsm/.local/share/v2rayN/binConfigs/custom/xray-docker-proxy.json
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable xray-docker-proxy.service
systemctl --user start xray-docker-proxy.service
loginctl enable-linger wsm
```

###### 2.4.4 验证服务状态

```bash
systemctl --user status xray-docker-proxy.service
ss -tlnp | grep 7890
```

**预期输出：**
```
LISTEN 0  4096  *:7890  0.0.0.0:*  users:(("xray",...))
```

看到 `*:7890` 表示成功！

###### 2.4.5 日常维护

```bash
# v2rayN 切换节点后，重启服务即可同步最新配置
systemctl --user restart xray-docker-proxy.service

# 查看服务日志
journalctl --user -u xray-docker-proxy.service -f
```

---

#### 步骤 3：获取 Docker 网络网关 IP

```bash
docker network inspect docker_default | grep Gateway
```

**示例输出：**
```json
"Gateway": "172.21.0.1"
```

记录这个 IP 地址，后续配置会用到。

---

#### 步骤 4：配置 Dify 环境变量

编辑 `docker/.env` 文件，添加代理配置：

```bash
cd /home/wsm/codes/dify/docker
vim .env
```

在文件末尾添加以下内容（**在 Nginx 配置之前**）：

```bash
# Proxy configuration for accessing external services (e.g., plugin marketplace)
HTTP_PROXY=http://172.21.0.1:7890
HTTPS_PROXY=http://172.21.0.1:7890
NO_PROXY=localhost,127.0.0.1,db_postgres,redis,weaviate,sandbox,ssrf_proxy,plugin_daemon,api,web,worker,worker_beat,api_websocket
```

**重要说明：**
- `172.21.0.1` 替换为您实际的 Docker 网络网关 IP
- `7890` 替换为您的代理端口
- `NO_PROXY` 包含所有内部服务名，确保它们不经过代理

---

#### 步骤 5：更新 docker-compose.yaml

为以下服务添加代理环境变量：
- api
- worker
- worker_beat
- plugin_daemon

编辑 `docker/docker-compose.yaml`，在每个服务的 `environment:` 部分添加：

```yaml
HTTP_PROXY: ${HTTP_PROXY:-}
HTTPS_PROXY: ${HTTPS_PROXY:-}
NO_PROXY: ${NO_PROXY:-}
```

---

#### 步骤 6：重启 Dify 服务

```bash
cd /home/wsm/codes/dify/docker

# 停止所有容器
docker compose down

# 重新启动
docker compose up -d

# 查看容器状态
docker compose ps
```

---

#### 步骤 7：验证配置

##### 7.1 检查容器环境变量

```bash
docker exec docker-api-1 env | grep -E "^(HTTP_PROXY|HTTPS_PROXY|NO_PROXY)="
```

**预期输出：**
```
HTTP_PROXY=http://172.21.0.1:7890
HTTPS_PROXY=http://172.21.0.1:7890
NO_PROXY=localhost,127.0.0.1,db_postgres,redis,...
```

##### 7.2 测试代理连通性

```bash
docker exec docker-api-1 python3 -c "
import urllib.request
proxy_handler = urllib.request.ProxyHandler({
    'http': 'http://172.21.0.1:7890',
    'https': 'http://172.21.0.1:7890'
})
opener = urllib.request.build_opener(proxy_handler)
opener.addheaders = [('User-Agent', 'Mozilla/5.0')]
try:
    response = opener.open('https://www.google.com', timeout=5)
    print(f'通过代理访问 Google: 成功 (状态码: {response.status})')
except Exception as e:
    print(f'访问失败: {type(e).__name__}: {e}')
"
```

**预期输出：**
```
通过代理访问 Google: 成功 (状态码: 200)
```

---

## 3. 本地 Ollama / LM Studio 模型连接失败

### 问题现象

在 Dify Web 界面添加本地 Ollama 或 LM Studio 模型时，验证凭据失败，报错：

```
CredentialsValidateFailedError: An error occurred during credentials validation: 
HTTPConnectionPool(host='localhost', port=11434): Max retries exceeded with url: /api/chat 
(Caused by NewConnectionError("HTTPConnection(host='localhost', port=11434): 
Failed to establish a new connection: [Errno 111] Connection refused"))
```

### 根本原因

- Ollama 默认监听在 `127.0.0.1:11434`，LM Studio 默认监听在 `127.0.0.1:1234`
- Dify 运行在 Docker 容器中
- 容器无法访问宿主机的 `127.0.0.1`

> ⚠️ **URL 配置关键提醒：** 在 Dify 中配置本地模型服务时，Base URL **必须**使用 Docker 网络网关 IP（如 `172.21.0.1`），**绝对不能**使用 `127.0.0.1` 或 `localhost`！
>
> | 服务 | ❌ 错误 URL | ✅ 正确 URL |
> |------|------------|------------|
> | Ollama | `http://127.0.0.1:11434` | `http://172.21.0.1:11434` |
> | LM Studio | `http://127.0.0.1:1234` | `http://172.21.0.1:1234` |
> | 代理服务器 | `http://127.0.0.1:7890` | `http://172.21.0.1:7890` |

### 解决方案

#### 步骤 1：修改服务监听地址

##### 1.1 Ollama - 创建 systemd 覆盖配置

```bash
# 创建覆盖目录
sudo mkdir -p /etc/systemd/system/ollama.service.d

# 创建配置文件
sudo tee /etc/systemd/system/ollama.service.d/override.conf > /dev/null << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
EOF
```

##### 1.2 Ollama - 重启服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama

# 等待服务启动
sleep 3

# 验证监听地址
ss -tlnp | grep 11434
```

**预期输出：**
```
LISTEN 0  4096  *:11434  0.0.0.0:*
```

看到 `*:11434` 表示成功（而不是 `127.0.0.1:11434`）

---

##### 1.3 LM Studio - 修改监听地址

LM Studio 默认监听 `127.0.0.1:1234`，需要改为 `0.0.0.0:1234`：

1. 打开 LM Studio 应用
2. 进入 **Local Server** 设置页面
3. 将 **Server Host** 从 `127.0.0.1` 改为 `0.0.0.0`
4. 确认 **Server Port** 为 `1234`（默认端口）
5. 重启 LM Studio 的本地服务

```bash
# 验证监听地址
ss -tlnp | grep 1234
```

**预期输出：**
```
LISTEN 0  4096  *:1234  0.0.0.0:*
```

看到 `*:1234` 表示成功（而不是 `127.0.0.1:1234`）

---

#### 步骤 2：获取 Docker 网络网关 IP

```bash
docker network inspect docker_default | grep Gateway
```

**示例输出：**
```json
"Gateway": "172.21.0.1"
```

---

#### 步骤 3：测试容器到服务的连通性

```bash
# 测试 Ollama 连通性
docker exec docker-plugin_daemon-1 curl -s --connect-timeout 3 \
  http://172.21.0.1:11434/api/tags | jq '.models[].name'

# 测试 LM Studio 连通性
docker exec docker-plugin_daemon-1 curl -s --connect-timeout 3 \
  http://172.21.0.1:1234/v1/models | jq '.data[].id'
```

**预期输出：**
应该列出您已下载的 Ollama 模型名称，如：
```
"qwen3:8b"
"qwen3.5:latest"
"98k:latest"
```

---

#### 步骤 4：在 Dify Web 界面配置模型服务

> ⚠️ **再次强调：配置 URL 时必须使用 Docker 网络网关 IP，不要使用 `localhost` 或 `127.0.0.1`！**

##### 4.1 配置 Ollama

1. 登录 Dify Web 界面
2. 进入 **工作区设置** → **模型提供商**
3. 找到 **Ollama** 提供商
4. 点击 **添加模型** 或编辑现有配置
5. **关键修改：**
   - ❌ Base URL: `http://localhost:11434` （错误）
   - ✅ Base URL: `http://172.21.0.1:11434` （正确）

   将 `172.21.0.1` 替换为您实际获取的网关 IP

6. 选择模型（如 `qwen3:8b`）
7. 点击 **验证并保存**

##### 4.2 配置 LM Studio

1. 登录 Dify Web 界面
2. 进入 **工作区设置** → **模型提供商**
3. 找到 **LM Studio** 提供商（或 OpenAI-API-compatible 类型）
4. 点击 **添加模型** 或编辑现有配置
5. **关键修改：**
   - ❌ Base URL: `http://localhost:1234/v1` （错误）
   - ❌ Base URL: `http://127.0.0.1:1234/v1` （错误）
   - ✅ Base URL: `http://172.21.0.1:1234/v1` （正确）

   将 `172.21.0.1` 替换为您实际获取的网关 IP

6. 选择已加载的模型
7. 点击 **验证并保存**

---

## 4. 常见问题排查

### 问题 1：容器无法连接代理或 Ollama

**症状：**
```
curl: (7) Failed to connect to 172.21.0.1 port XXXX: Connection refused
```

**排查步骤：**

1. 检查服务是否监听在 0.0.0.0：
   ```bash
   ss -tlnp | grep <端口号>
   ```
   应该看到 `*:XXXX` 而不是 `127.0.0.1:XXXX`

2. 检查防火墙规则：
   ```bash
   sudo iptables -L -n | grep <端口号>
   ```

3. 测试宿主机本地连接：
   ```bash
   curl http://127.0.0.1:<端口号>
   ```

**解决方案：**
- 重新执行配置步骤，确保服务监听 `0.0.0.0`
- 重启服务

---

### 问题 2：host.docker.internal 解析失败

**症状：**
```
socket.gaierror: [Errno -2] Name or service not known
```

**原因：**
`host.docker.internal` 只在 Docker Desktop for Mac/Windows 中自动配置，Linux 上不可用。

**解决方案：**
使用 Docker 网络网关 IP，不要使用 `host.docker.internal`。

---

### 问题 3：xray 启动失败

**症状：**
```
Failed to start: failed to load config files: failed to open geosite.dat
```

**原因：**
缺少 `geosite.dat` 或 `geoip.dat` 文件。

**解决方案：**
```bash
# 复制数据文件到 xray 目录
cp /home/wsm/.local/share/v2rayN/bin/{geosite.dat,geoip.dat} \
   /home/wsm/.local/share/v2rayN/bin/xray/
```

---

### 问题 4：Docker 拉取镜像失败

**症状：**
```
ERROR: failed to solve: proxyconnect tcp: dialing 127.0.0.1:7890: Connection refused
```

**原因：**
Docker 守护进程的代理配置问题。

**解决方案：**

配置 Docker 守护进程代理（与容器内部代理不同）：

```bash
# 编辑 /etc/systemd/system/docker.service.d/http-proxy.conf
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,172.17.0.0/16,172.21.0.0/16,m.daocloud.io,docker.m.daocloud.io"
EOF

# 重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

### 问题 5：Ollama 找不到模型列表

**症状：**
连接成功但返回空模型列表

**解决方案：**

1. 确认模型已下载：
   ```bash
   ollama list
   ```

2. 如果没有模型，先下载一个：
   ```bash
   ollama pull qwen3:8b
   ```

3. 刷新 Dify 页面重新验证

---

### 问题 6：代理配置后仍然很慢

**排查步骤：**

1. 确认代理本身速度快：
   ```bash
   curl -x http://127.0.0.1:7890 -w "Time: %{time_total}s\n" -o /dev/null -s https://www.google.com
   ```

2. 检查 NO_PROXY 是否包含了不该排除的地址

3. 查看容器日志：
   ```bash
   docker logs docker-plugin-daemon-1 --tail 50
   docker logs docker-api-1 --tail 50
   ```

---

## 5. 安全注意事项

### ⚠️ 重要警告

将服务设置为监听 `0.0.0.0` 会使其对**整个网络可见**。

**安全措施：**

1. **仅限本地网络访问：**
   ```bash
   # 使用 iptables 限制访问来源（以 11434 端口为例）
   sudo iptables -A INPUT -p tcp --dport 11434 -s 172.21.0.0/16 -j ACCEPT
   sudo iptables -A INPUT -p tcp --dport 11434 -s 127.0.0.1 -j ACCEPT
   sudo iptables -A INPUT -p tcp --dport 11434 -j DROP
   ```

2. **不要暴露到公网：**
   - 确保路由器/防火墙不转发相关端口
   - 不要在云服务器上开放这些端口

3. **定期审查配置：**
   ```bash
   # 查看当前监听的服务
   ss -tlnp
   
   # 查看 iptables 规则
   sudo iptables -L -n
   ```

---

## 6. 配置备份与恢复

### 备份配置

```bash
# 创建备份目录
mkdir -p ~/backup/dify-configs/$(date +%Y%m%d)

# 备份 xray 配置
cp /home/wsm/.local/share/v2rayN/binConfigs/config.json \
   ~/backup/dify-configs/$(date +%Y%m%d)/xray-config.json

# 备份 Ollama 配置
cp /etc/systemd/system/ollama.service.d/override.conf \
   ~/backup/dify-configs/$(date +%Y%m%d)/ollama-override.conf

# 备份 Dify 配置
cp /home/wsm/codes/dify/docker/.env \
   ~/backup/dify-configs/$(date +%Y%m%d)/dify-env
cp /home/wsm/codes/dify/docker/docker-compose.yaml \
   ~/backup/dify-configs/$(date +%Y%m%d)/dify-compose.yaml
```

### 快速恢复

```bash
# 恢复 xray 配置
cp ~/backup/dify-configs/YYYYMMDD/xray-config.json \
   /home/wsm/.local/share/v2rayN/binConfigs/config.json
kill $(ps aux | grep "[x]ray" | awk '{print $2}')
cd /home/wsm/.local/share/v2rayN/bin/xray
nohup ./xray run -c /home/wsm/.local/share/v2rayN/binConfigs/config.json > /tmp/xray.log 2>&1 &

# 恢复 Ollama 配置
cp ~/backup/dify-configs/YYYYMMDD/ollama-override.conf \
   /etc/systemd/system/ollama.service.d/override.conf
sudo systemctl daemon-reload && sudo systemctl restart ollama

# 恢复 Dify 配置
cp ~/backup/dify-configs/YYYYMMDD/dify-env /home/wsm/codes/dify/docker/.env
cp ~/backup/dify-configs/YYYYMMDD/dify-compose.yaml /home/wsm/codes/dify/docker/docker-compose.yaml
cd /home/wsm/codes/dify/docker && docker compose down && docker compose up -d
```

---

## 7. 参考命令速查

### 通用命令

```bash
# 查看 Docker 网络网关
docker network inspect docker_default | grep Gateway

# 查看所有运行中的容器
docker ps --filter "name=docker"

# 进入容器调试
docker exec -it docker-api-1 bash

# 查看容器日志
docker logs docker-api-1 --tail 100 -f
docker logs docker-plugin-daemon-1 --tail 100 -f
```

### 代理相关

```bash
# 检查代理状态
ss -tlnp | grep 7890

# 检查容器环境变量
docker exec docker-api-1 env | grep PROXY

# 测试代理连通性
docker run --rm --network docker_default alpine/curl \
  curl -s --connect-timeout 3 http://172.21.0.1:7890

# 重启代理
kill $(ps aux | grep "[x]ray" | awk '{print $2}')
cd /home/wsm/.local/share/v2rayN/bin/xray
nohup ./xray run -c /home/wsm/.local/share/v2rayN/binConfigs/config.json > /tmp/xray.log 2>&1 &
```

### Ollama 相关

```bash
# 检查 Ollama 状态
systemctl status ollama

# 查看 Ollama 监听地址
ss -tlnp | grep 11434

# 查看 Ollama 日志
journalctl -u ollama -f --tail 50

# 列出已下载的模型
ollama list

# 测试 Ollama API
curl http://172.21.0.1:11434/api/tags

# 从容器测试 Ollama
docker exec docker-plugin-daemon-1 curl -s http://172.21.0.1:11434/api/tags

# 重启 Ollama
sudo systemctl restart ollama
```

### Dify 相关

```bash
# 重启 Dify
cd /home/wsm/codes/dify/docker && docker compose down && docker compose up -d

# 查看容器状态
docker compose ps

# 查看特定服务日志
docker compose logs -f api
docker compose logs -f plugin_daemon
```

---

## 总结

### 核心原则

1. ✅ **宿主机服务必须监听 `0.0.0.0`**（而非 `127.0.0.1`）
2. ✅ **使用 Docker 网络网关 IP 访问宿主机**（而非 `localhost`）
3. ✅ **Linux 上 `host.docker.internal` 不可用**
4. ✅ **区分 Docker 守护进程代理和容器内部代理**
5. ✅ **注意安全风险，不要暴露到公网**

### 配置清单

| 服务 | 监听地址 | 访问地址 | 配置文件 |
|------|---------|---------|---------|
| 代理服务器 | 0.0.0.0:7890 | http://172.21.0.1:7890 | xray config.json / systemd service |
| Ollama | 0.0.0.0:11434 | http://172.21.0.1:11434 | systemd override |
| LM Studio | 0.0.0.0:1234 | http://172.21.0.1:1234/v1 | LM Studio GUI 设置 |
| Dify 容器 | - | 配置 HTTP_PROXY 等 | docker/.env |

按照本指南配置后，Dify 应该能够：
- ✅ 快速安装国外插件
- ✅ 正常连接本地 Ollama / LM Studio 模型
- ✅ 稳定运行所有功能

---

**最后更新时间：** 2026-06-09  
**适用版本：** Dify 1.14.2, Ollama latest, LM Studio latest  
**作者：** AI Assistant  
**维护者：** Dify 本地部署团队
