# 树莓派 5 搭建 LoRaWAN 网关与私有 ChirpStack 服务器完全指南

#### chirpstack官方部署教程 https://www.chirpstack.io/docs/getting-started/docker.html
**适用硬件**： Raspberry Pi 5 (8GB/4GB) + Waveshare SX1303 LoRaWAN Gateway HAT  
**适用系统**： Raspberry Pi OS (Debian 12 Bookworm / Debian 13 Trixie)  
**目标**： 实现纯本地、私有化的 LoRaWAN 网络覆盖与数据采集。

---

## 🛑 核心避坑清单 (必读)

在开始之前，请务必注意以下导致失败的“三大杀手”：

1.  **内存页大小不兼容**：树莓派 5 默认开启 16KB Page Size，会导致 Docker 里的数据库（Postgres/Redis）无限重启。**必须改为 4KB**。
2.  **Docker 镜像架构错误**：默认拉取的某些镜像（如 Mosquitto）可能是 AMD64 (PC版) 的，导致在树莓派 (ARM64) 上报 `exec format error`。**必须指定平台拉取**。
3.  **网关 ID 不匹配**：`global_conf.json` 里的 ID 必须与 ChirpStack 网页后台填写的 ID 完全一致（精确到每一位）。

---

## 第一阶段：系统基础配置

### 1. 开启 SPI 和 I2C 接口
SX1302 扩展板依赖这两个接口通信。

1.  终端输入 `sudo raspi-config`
2.  进入 **Interface Options** -> 开启 **SPI** 和 **I2C**。
3.  重启树莓派：
    ```bash
    sudo reboot
    ```

### 2. 🔥 关键修改：切换内核内存页大小 (Page Size)
这是树莓派 5 运行 Docker 数据库最关键的一步。

1.  编辑配置文件：
    ```bash
    sudo nano /boot/firmware/config.txt
    # (注：旧版系统可能是 /boot/config.txt)
    ```
2.  在文件末尾添加一行：
    ```ini
    kernel=kernel8.img
    ```
3.  保存退出并重启：
    ```bash
    sudo reboot
    ```
4.  **验证**：重启后运行 `getconf PAGE_SIZE`，输出 `4096` 即为成功。

---

## 第二阶段：编译并安装网关驱动

使用微雪 (Waveshare) 提供的驱动库。由于网络问题，建议使用国内镜像。

### 1. 下载与编译

```bash
# 1. 安装依赖
sudo apt update
sudo apt install -y git make

# 2. 下载并解压
wget https://files.waveshare.com/wiki/SX130X/demo/PI5/sx130x_hal_rpi5.zip
sudo unzip sx130x_hal_rpi5.zip

# 3. 编译基础库
cd sx1302_hal_rpi5-master/
sudo make clean all
sudo make all
cp tools/reset_lgw.sh util_chip_id/
cp tools/reset_lgw.sh  packet_forwarder/

# 4. 获取 ID
cd util_chip_id/
sudo ./chip_id


```

记录输出的 ID，例如：`0016c00**********`

---

## 第三阶段：部署 ChirpStack (Docker)

### 1. 安装 Docker
推荐使用官方脚本安装：

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
# 将当前用户加入 Docker 组 (免 sudo)
sudo usermod -aG docker $USER
newgrp docker
```

### 2. 下载 ChirpStack 配置

```bash
# 回到主目录
cd ~
# 克隆 Docker 部署库
git clone https://github.com/chirpstack/chirpstack-docker.git
# (如果 github 慢，请使用代理或下载 zip 包上传)

cd chirpstack-docker
```

### 3. 🔥 修改配置：指定频段
默认配置通常开启所有频段，建议只保留你硬件支持的频段（如 EU868 或 CN470）。

```bash
nano configuration/chirpstack/chirpstack.toml
```

找到 `[regions]` 部分。修改 `enabled_regions`，例如只保留 EU868：

```toml
enabled_regions=["eu868"]
```

### 4. 🔥 修改 docker-compose.yml (解决架构与权限问题)
编辑 `docker-compose.yml`，重点修改 Mosquitto 和 REST API 部分：

```yaml
services:
  # ... 其他服务保持不变 ...

  mosquitto:
    # 强制指定版本，避免拉到不兼容的镜像
    image: eclipse-mosquitto:2.0.18
    # ...

  chirpstack-rest-api:
    image: chirpstack/chirpstack-rest-api:4
    # 🔥 关键：添加 root 权限，解决 "unable to find user nobody" 报错
    user: "0:0"
    # ...
```

### 5. 强制拉取 ARM64 镜像并启动
为了防止下载到电脑版镜像，使用以下命令启动：

```bash
# 1. 强制拉取 ARM64 镜像
sudo DOCKER_DEFAULT_PLATFORM=linux/arm64 docker-compose pull

# 2. 启动服务
sudo docker-compose up -d

# 3. 检查状态 (必须全绿 Up)
sudo docker-compose ps
```

---

## 第四阶段：网关与服务器对接

### 1. 配置网关转发程序
回到驱动目录，配置网关将数据发给本机的 Docker。

```bash
cd ~/SX1302_LoRa_Gateway_HAT/packet_forwarder/

# 1. 复制对应的模板 (根据你的硬件频率，如 EU868)
sudo cp global_conf.json.sx1250.EU868 global_conf.json

# 2. 编辑配置
sudo nano global_conf.json
```

修改 `gateway_conf` 部分：

```json
"gateway_conf": {
    "gateway_ID": "0016c001******",     /* 填写第二阶段获取的真实 ID */
    "server_address": "127.0.0.1",        /* 指向本机 */
    "serv_port_up": 1700,                 /* 必须是 1700 */
    "serv_port_down": 1700,               /* 必须是 1700 */
    ...
}
```

### 2. 启动网关

```bash
sudo ./lora_pkt_fwd
```

观察日志：如果出现 `PUSH_ACK: 100.00%`，说明连接成功。

---

## 进阶配置：设置开机自启 (可选)

我们需要新建一个配置文件来告诉系统如何管理这个程序。

### 1. 创建服务文件

在终端输入：

```bash
sudo nano /etc/systemd/system/lora-gateway.service
```

### 2. 填入配置内容

请将下面的内容完整复制进去。 (注意：我根据你之前的路径 `/home/mh/lora/...` 写好了绝对路径，请仔细核对 User 是否为你现在的用户名 `XX`)

```ini
[Unit]
Description=LoRaWAN Packet Forwarder Service
After=network.target

[Service]
# 以 root 权限运行 (访问 SPI/GPIO 需要)
User=root
# 工作目录 (程序去哪里找 global_conf.json)
WorkingDirectory=/home/mh/lora/sx1302_hal_rpi5-master/packet_forwarder
# 启动命令
ExecStart=/home/mh/lora/sx1302_hal_rpi5-master/packet_forwarder/lora_pkt_fwd
# 崩溃或退出后自动重启
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

保存退出：`Ctrl + O` -> `Enter` -> `Ctrl + X`。

### 3. 启用并启动服务

执行下面三行命令：

```bash
# 1. 重新加载系统配置
sudo systemctl daemon-reload

# 2. 设置开机自启
sudo systemctl enable lora-gateway.service

# 3. 立即启动服务
sudo systemctl start lora-gateway.service
```

### 4. 检查是否成功

查看服务状态：

```bash
sudo systemctl status lora-gateway.service
```

如果你看到绿色的 `active (running)`，说明成功了！

以后你想看网关日志，可以用这个命令：

```bash
sudo journalctl -u lora-gateway.service -f
```

---

## 第五阶段：Web 端配置

1.  **登录后台**：访问 `http://<树莓派IP>:8080` (账号/密码: `admin` / `admin`)。
2.  **添加网关**：
    *   菜单 **Gateways** -> **Add gateway**。
    *   **Gateway ID**: 填入 `0016c001f15d3e21` (注意：不要多填0，必须和 `chip_id` 输出的一模一样)。
    *   提交后，状态应显示为绿色 **Online**。
3.  **配置应用**：
    *   创建 **Device Profile** (选择对应频率，如 EU868, LoRaWAN 1.0.3)。
    *   创建 **Application**。
    *   在 Application 中添加 **Device** (填入你真实节点的 DevEUI 和 AppKey)。
4.  **数据集成 (可选)**：
    *   在 **Application** -> **Integrations** 中添加 HTTP 集成，填入你的后端地址 (如 SpringBoot 接口)，实现数据推送。

---

## 🎉 常见报错速查

| 现象 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| **Docker 容器无限 Restarting** | 树莓派 5 内存页大小不兼容 | 修改 `/boot/firmware/config.txt` 添加 `kernel=kernel8.img` 并重启。 |
| **Log 提示 exec format error** | 镜像架构不对 (AMD64 vs ARM64) | 修改 `docker-compose.yml` 显式指定版本，并用 `DOCKER_DEFAULT_PLATFORM=linux/arm64` 重新 pull。 |
| **Log 提示 unable to find user nobody** | 系统用户权限差异 | 在 `docker-compose.yml` 对应服务下添加 `user: "0:0"`。 |
| **网页显示 "Never seen" 但 Docker 全绿** | 网关 ID 填错 / 端口不对 | 检查网页填写的 ID 是否多/少了 0；检查 `global_conf.json` 端口是否为 1700。 |
| **PUSH_ACK 0.00%** | 服务器 Mosquitto 没启动 | 检查 Docker 容器状态，确保 Mosquitto 是 Up 状态。 |
