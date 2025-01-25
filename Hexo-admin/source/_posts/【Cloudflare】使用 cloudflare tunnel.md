---
title: 使用 cloudflare tunnel
date: 2025-01-13 19:54:06
categories: cloudflare
tags: [tunnel, cloudflare]


---

使用 cloudflare tunnel

<!-- more --> 

#### 1. **创建多个 Tunnel**

每个 Tunnel 都需要独立创建。以 SSH 和 HTTPS 为例：

1. **创建 SSH Tunnel**

   ```bash
   cloudflared tunnel create ssh-tunnel
   ```

2. **创建 HTTPS Tunnel**

   ```bash
   cloudflared tunnel create https-tunnel
   ```

每个 Tunnel 都会生成一个唯一的 `credentials-file`，通常位于 `~/.cloudflared/` 目录下。

------

#### 2. **配置多个 Tunnel 的服务**

为每个 Tunnel 创建独立的配置文件。例如：

##### **SSH Tunnel 配置**

创建或编辑 `~/.cloudflared/ssh-config.yml`：

```yaml
tunnel: ssh-tunnel
credentials-file: /home/your-user/.cloudflared/ssh-tunnel.json

ingress:
  - hostname: ssh.example.com
    service: ssh://localhost:22
  - service: http_status:404
```

##### **HTTPS Tunnel 配置**

创建或编辑 `~/.cloudflared/https-config.yml`：

```yaml
tunnel: https-tunnel
credentials-file: /home/your-user/.cloudflared/https-tunnel.json

ingress:
  - hostname: www.example.com
    service: http://localhost:80
  - service: http_status:404
```

------

#### 3. **创建DNS记录**

```
sudo cloudflared tunnel route dns ssh-tunnel ssh.example.com

sudo cloudflared tunnel route dns https-tunnel www.example.com
```



#### 4. **启动多个 Tunnel**

可以使用以下命令分别启动每个 Tunnel：

- 启动 SSH Tunnel：

  ```bash
  cloudflared tunnel --config ~/.cloudflared/ssh-config.yml run ssh-tunnel
  ```

- 启动 HTTPS Tunnel：

  ```bash
  cloudflared tunnel --config ~/.cloudflared/https-config.yml run https-tunnel
  ```

------

#### 5. **使用 Systemd 管理多个 Tunnel**

为每个 Tunnel 配置一个 Systemd 服务，以便在系统启动时自动运行。

1. **创建 Systemd 服务文件**

   - 为 SSH Tunnel 创建服务文件 `/etc/systemd/system/ssh-tunnel.service`：

     ```ini
     [Unit]
     Description=Cloudflare Tunnel for SSH
     After=network.target
     
     [Service]
     ExecStart=/usr/local/bin/cloudflared tunnel --config /home/your-user/.cloudflared/ssh-config.yml run ssh-tunnel
     Restart=always
     User=your-user
     
     [Install]
     WantedBy=multi-user.target
     ```

   - 为 HTTPS Tunnel 创建服务文件 `/etc/systemd/system/https-tunnel.service`：

     ```ini
     [Unit]
     Description=Cloudflare Tunnel for HTTPS
     After=network.target
     
     [Service]
     ExecStart=/usr/local/bin/cloudflared tunnel --config /home/your-user/.cloudflared/https-config.yml run https-tunnel
     Restart=always
     User=your-user
     
     [Install]
     WantedBy=multi-user.target
     ```

2. **启动和启用服务**

   - 启动 SSH Tunnel 服务：

     ```bash
     sudo systemctl start ssh-tunnel
     sudo systemctl enable ssh-tunnel
     ```

   - 启动 HTTPS Tunnel 服务：

     ```bash
     sudo systemctl start https-tunnel
     sudo systemctl enable https-tunnel
     ```

------

#### 6. **配置本地的 ssh **

需要在本地ssh配置代理

```
Host ssh.example.com
ProxyCommand C:\\Users\\29308\\.ssh\\cloudflared-windows-amd64.exe access ssh --hostname %h
```

