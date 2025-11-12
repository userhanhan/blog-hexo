---
title: docker镜像源、代理配置
date: 2025-11-11 11:11:11
tags: [镜像源、docker代理]
categories: [docker]
---

要解决 Docker 拉取镜像时出现的 `Error response from daemon: Get "https://registry-1.docker.io/v2/": context deadline exceeded` 错误，可通过配置国内镜像源（镜像加速器）来优化网络连接。以下是详细步骤：

---

## **方法 1：修改 Docker 配置文件（推荐）**
### 步骤：
1. **编辑 Docker 配置文件**  
   创建或修改 `/etc/docker/daemon.json`（Linux/macOS）或 Docker Desktop 的配置（Windows）：
   ```bash
   sudo nano /etc/docker/daemon.json
   ```

2. **添加国内镜像源**  
   将以下内容粘贴到文件中（根据需求选择一个或多个镜像源）：
   ```json
   {
     "registry-mirrors": [
       "https://docker.mirrors.ustc.edu.cn",    // 中科大源
       "https://hub-mirror.c.163.com",          // 网易源
       "https://registry.docker-cn.com",         // Docker中国官方源（已停更，部分可用）
       "https://mirror.ccs.tencentyun.com"      // 腾讯云源
     ]
   }
   ```
   > **注**：建议优先使用 **中科大** 或 **网易** 源。

3. **保存并重启 Docker**  
   ```bash
   sudo systemctl daemon-reload    # 重新加载配置
   sudo systemctl restart docker   # 重启 Docker 服务
   ```

4. **验证配置是否生效**  
   ```bash
   docker info
   ```
   在输出中检查 `Registry Mirrors`，确认已列出配置的镜像源。

---

## **方法 2：临时使用镜像源（命令行指定）**
拉取镜像时直接指定镜像源：
```bash
docker pull registry.docker-cn.com/library/ubuntu:latest
```
> 注意：此方法需修改镜像 URL，仅对单个命令有效。

---


### **各镜像源地址参考**
| 镜像源             | 地址                                  |
|--------------------|---------------------------------------|
| **中科大**         | `https://docker.mirrors.ustc.edu.cn`  |
| **网易**           | `https://hub-mirror.c.163.com`        |
| **腾讯云**         | `https://mirror.ccs.tencentyun.com`   |
| **阿里云**         | 需登录控制台获取专属地址              |
| **Docker 中国**    | `https://registry.docker-cn.com`     |

---

完成配置后，再次运行 `docker pull` 命令即可通过国内镜像源加速下载。

## **方法 3：配置代理**

1. **打开代理配置文件**：
   ```bash
   sudo vi /etc/systemd/system/docker.service.d/http-proxy.conf
   ```

2. **确保格式正确**：
   ```ini
   [Service]
   Environment="HTTP_PROXY=http://代理服务器IP:端口"
   Environment="HTTPS_PROXY=http://代理服务器IP:端口"
   Environment="NO_PROXY=localhost,127.0.0.1,::1"
   ```

   **正确示例**（根据您的实际代理设置调整）：
   ```ini
   [Service]
   Environment="HTTP_PROXY=http://192.168.1.100:8080"
   Environment="HTTPS_PROXY=http://192.168.1.100:8080"
   Environment="NO_PROXY=localhost,127.0.0.1,::1,docker-registry.example.com"
   ```

### 🌐 特别说明：本地代理设置

如果您使用的是本地代理（如 Clash、V2Ray 等），通常应该使用：

```ini
Environment="HTTP_PROXY=http://127.0.0.1:7890"  # Clash 默认端口
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
```

### 🔧 验证和修复步骤

1. **检查当前配置**：
   ```bash
   sudo systemctl show --property=Environment docker
   ```

2. **修正配置文件**后，重新加载并重启 Docker：
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

3. **验证代理设置**：
   ```bash
   sudo systemctl show --property=Environment docker | grep PROXY
   ```

4. **测试连接**：
   ```bash
   docker pull hello-world
   ```

### 💡 如果问题仍然存在

如果修正后仍有问题，请尝试：

1. **检查代理服务器是否正常运行**：
   ```bash
   curl -x http://代理IP:端口 https://www.google.com --connect-timeout 5
   ```

2. **临时禁用 Docker 代理**：
   ```bash
   sudo rm /etc/systemd/system/docker.service.d/http-proxy.conf
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

3. **检查 DNS 设置**：
   ```bash
   cat /etc/resolv.conf
   ping -c 4 8.8.8.8  # 测试基本网络连接
   ```
### 💡 进一步排查建议

如果修正配置后问题依旧，请按以下步骤排查：

-   **测试代理服务器本身是否可用**：在终端中运行以下命令，测试您的代理服务器是否能正常访问Docker Hub。请将命令中的地址和端口替换为您的实际代理信息。
    ```bash
    curl -x http://192.168.10.86:10809 https://registry-1.docker.io/v2/
    ```
    -   如果返回 `200 OK` 或类似的成功信息，说明代理服务器工作正常。
    -   如果连接失败或超时，则问题出在代理服务器本身（如未启动、配置错误或网络不通），您需要检查并确保代理服务是正常运行的。

-   **检查代理认证**：如果您的代理服务器需要用户名和密码，则配置地址中需要包含认证信息，格式为：`http://用户名:密码@192.168.10.86:10809`。

-   **尝试另一种配置方法**：除了修改 `http-proxy.conf` 文件，您也可以尝试在 `/etc/docker/daemon.json` 文件中配置代理。请注意，如果 `daemon.json` 中配置了代理，其优先级可能会高于systemd的配置。
    ```json
    {
        "proxies": {
            "default": {
                "httpProxy": "http://192.168.10.86:10809",
                "httpsProxy": "http://192.168.10.86:10809",
                "noProxy": "localhost,127.0.0.1"
            }
        }
    }
    ```

    修改后同样需要重启Docker服务。

### 🔧 删除代理

1.  **移除代理配置**
    您可以通过以下任意一种方式移除代理配置：
    *   **删除配置文件**：直接删除之前创建的代理配置文件。
        ```bash
        sudo rm /etc/systemd/system/docker.service.d/http-proxy.conf
        ```
    *   **清空配置文件内容**：如果该文件中还有其他重要配置，您可以只清空代理设置。使用文本编辑器（如 `vi` 或 `nano`）打开文件，将 `[Service]` 部分下的 `Environment` 行删除或注释掉（在行首添加 `#` 号）。
    *   **临时禁用（推荐用于测试）**：如果您只是想暂时关闭代理，而不是永久删除配置，可以将配置文件重命名，例如加上 `.bak` 后缀。这在您未来可能还需要使用代理时会非常方便。
        ```bash
        sudo mv /etc/systemd/system/docker.service.d/http-proxy.conf /etc/systemd/system/docker.service.d/http-proxy.conf.bak
        ```

2.  **重新加载配置并重启Docker服务**
    **这是关键的一步**。只要修改了 Docker 守护进程的配置，就必须执行以下命令来重新加载 systemd 配置并完全重启 Docker 服务，更改才能生效。
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

3.  **验证代理是否已移除**
    配置重启后，您可以验证代理设置是否已清除。执行以下命令检查 Docker 的环境变量：
    ```bash
    sudo systemctl show --property=Environment docker
    ```
    如果输出结果中不再显示 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量，或者其值为空，即表示代理配置已成功移除。
