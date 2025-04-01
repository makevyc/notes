# polycube
---

## 文档解析
---
### 初始化
开启polycubed守护进程
+ sudo systemctl start polycubed

查看状态
+ sudo systemctl status polycubed

查看有哪些可用命令和服务
+ polycubectl --help

停止服务
+ sudo systemctl stop polycubed


### Polycube

---

#### **1. 架构概览**
Polycube 架构由四大核心组件构成，形成一个灵活、可扩展的网络功能虚拟化框架：

| 组件 | 核心作用 | 关键特性 |
|------|----------|----------|
| **polycubed** | 系统守护进程 | 服务无关的中央控制器，代理所有请求，支持本地/远程服务统一管理 |
| **polycubectl** | 命令行接口 | 服务无关的 CLI，基于 YANG 模型动态适配所有服务 |
| **Polycube Services** | 网络功能实现 | 模块化插件，包含数据平面（eBPF）、控制平面、慢速路径 |
| **libpolycube** | 公共基础库 | 提供日志、配置管理、通信等通用功能，加速服务开发 |

---

#### **2. 核心组件详解**

##### **(1) polycubed：系统守护进程**
- **核心角色**  
  **中央控制器**，负责服务生命周期管理（启动、配置、停止）和请求路由。
  
- **通信模型**  
  - **本地服务**：以共享库（`.so`）形式部署，通过直接函数调用交互。  
  - **远程服务**：以独立进程运行（可跨机器），通过 **gRPC** 协议通信。  
  - **统一接口**：对外暴露 REST API，隐藏服务部署细节（本地或远程对用户透明）。

- **关键能力**  
  - 动态加载/卸载服务模块  
  - 跨服务实例的资源协调  
  - 配置持久化与故障恢复  

```bash
# 本地服务目录示例（默认路径）
/usr/lib/x86_64-linux-gnu/polycube/services/
  ├── bridge.so    # 网桥服务
  ├── router.so    # 路由器服务
  └── firewall.so  # 防火墙服务
```

---

##### **(2) polycubectl：命令行接口**
- **服务无关性**  
  基于服务的 **YANG 数据模型** 动态生成命令结构，无需硬编码适配新服务。

- **代码生成工具**  
  提供自动化工具链，从 YANG 模型生成服务控制接口代码，简化开发者工作流：
  ```text
  YANG 模型 → 生成器 → REST API 绑定 + CLI 命令树
  ```

- **交互模式**  
  - 支持 **TAB 补全** 和 **上下文帮助**（`?` 符号）。  
  - 可解析 JSON/YAML 文件批量配置。

---

##### **(3) Polycube 服务（网络功能）**
每个服务包含三个核心部分：

| 组件 | 运行位置 | 功能描述 | 示例 |
|------|----------|----------|------|
| **数据平面** | Linux 内核 | 基于 eBPF 实现高性能包处理 | 网桥 MAC 学习、NAT 转换规则 |
| **控制平面** | 用户空间 | 通过 REST/gRPC 接收配置，更新数据平面 | 配置防火墙规则、路由表 |
| **慢速路径** | 用户空间 | 处理复杂逻辑（无法在内核完成） | 生成树协议（STP）、ARP 响应 |

**服务特性**：  
- **多版本共存**：同一功能可提供不同实现（如轻量版 vs 全功能版）。  
- **热插拔**：运行时动态加载/卸载服务模块。  
- **跨平台部署**：支持本地或分布式部署（通过 gRPC）。

---

##### **(4) libpolycube：公共库**
为服务开发提供基础功能，包括：
- **日志系统**：统一日志等级（`DEBUG`, `INFO`, `ERROR`）。  
- **配置管理**：解析命令行参数和配置文件。  
- **通信框架**：封装与 `polycubed` 的交互逻辑（REST/gRPC）。  
- **工具函数**：IP 地址处理、协议解析等。  

**开发者价值**：  
- 减少重复代码，聚焦业务逻辑。  
- 确保服务符合 Polycube 架构规范。

---

#### **3. 数据流与协作**
```text
+--------------+       REST API       +------------+
| polycubectl  |  ←—————→ | polycubed | ←—————→ [Service A]
+--------------+          +------------+    ↑
                                |           | gRPC/direct
                                ↓           ↓
                           [Service B]  [Service C (Remote)]
```

1. **用户操作**：通过 `polycubectl` 发送命令（如创建路由器实例）。  
2. **请求路由**：`polycubed` 解析请求并转发到目标服务（本地或远程）。  
3. **服务处理**：服务更新数据平面（eBPF程序）或慢速路径逻辑。  
4. **响应返回**：结果通过 `polycubed` 返回给用户。

---

#### **4. 核心优势**
- **动态扩展性**：新服务无需修改核心组件即可集成。  
- **异构部署**：混合本地与远程服务，适应不同场景。  
- **性能与灵活平衡**：关键路径 eBPF 加速，复杂逻辑用户态处理。  
- **开发友好**：自动代码生成和公共库降低开发门槛。

---

#### **5. 典型应用场景**
1. **云网络虚拟化**：快速部署虚拟路由器、负载均衡器。  
2. **网络安全**：动态插入防火墙、DDoS 缓解服务。  
3. **网络实验**：通过模块化服务构建复杂拓扑。  
4. **边缘计算**：轻量级服务适配资源受限环境。

---

#### **6. 扩展阅读**
- **YANG 模型**：定义服务配置结构的标准语言。  
- **eBPF 技术**：内核可编程性的核心实现基础。  
- **gRPC 协议**：跨语言服务通信的开放框架。  

```bash
# 查看系统中已注册的服务
polycubectl services show

# 监控 polycubed 与服务的通信（需启用 debug 模式）
journalctl -u polycubed -f | grep "Forwarding request"
```

### Polycubed

---

#### **1. Polycubed 概述**
**Polycubed** 是 Polycube 系统的核心守护进程，负责管理网络服务（称为 **Cubes**）的全生命周期，包括创建、更新和删除。它提供统一的 REST API 接口，用于配置所有网络功能，并通过 `polycubectl` 工具进行交互。

**核心功能**：
- **内核抽象层**：为不同网络服务提供统一的底层接口。
- **REST API 服务器**：通过 HTTP 接口暴露配置能力。
- **单点管理**：所有网络功能通过 Polycubed 集中配置。
- **特权运行**：需以 `root` 权限运行，且全系统只能运行一个实例。

---

#### **2. Systemd 服务管理**
Polycubed 可通过 `systemd` 管理，实现服务的启动、停止、自启等操作。

**常用命令**：
```bash
# 启动服务
sudo systemctl start polycubed

# 停止服务
sudo systemctl stop polycubed

# 重启服务（支持热重载配置）
sudo systemctl reload-or-restart polycubed

# 设置开机自启
sudo systemctl enable polycubed

# 查看服务状态
sudo systemctl status polycubed

# 查看日志（实时追踪日志用 `-f`）
journalctl -u polycubed
```

---

#### **3. 命令行参数**
启动 `polycubed` 时可通过参数配置其行为：

**常用参数**：
| 参数 | 说明 |
|------|------|
| `-p <PORT>` 或 `--port` | 指定 REST API 监听端口（默认 `9000`） |
| `-a <ADDR>` 或 `--addr` | 绑定监听的 IP 地址（默认 `localhost`） |
| `-l <LEVEL>` 或 `--loglevel` | 设置日志级别（如 `debug`, `info`, `error`） |
| `--logfile <PATH>` | 指定日志文件路径（默认 `/var/log/polycube/polycubed.log`） |
| `-d` 或 `--daemon` | 以守护进程模式运行（后台运行） |
| `--configfile <PATH>` | 指定配置文件路径（默认 `/etc/polycube/polycube.conf`） |

**示例**：
```bash
# 调试模式启动，监听所有 IP 的 8080 端口
polycubed --loglevel=debug --addr=0.0.0.0 --port=8080
```

---

#### **4. 配置文件**
配置文件用于持久化 `polycubed` 的配置，优先级低于命令行参数（命令行参数会覆盖配置文件）。

**配置文件路径**：
- 默认路径：`/etc/polycube/polycube.conf`
- 可通过 `--configfile` 指定自定义路径。

**配置格式**：
```ini
# 示例配置文件
loglevel: debug
daemon: true
port: 9000
```

**注意**：
- 仅支持长参数名（如 `loglevel`，而非 `-l`）。
- 修改配置文件后需重启服务生效。

---

#### **5. 持久化配置（Persistency）**
Polycubed 支持将当前网络拓扑和配置保存到磁盘，实现故障恢复或重启后自动加载。

**核心功能**：
1. **自动保存**：每次配置变更后自动转储到文件。
2. **启动加载**：默认加载上次关闭时的配置。
3. **自定义加载**：支持从指定文件加载或初始化空配置。

**相关参数**：
| 参数 | 说明 |
|------|------|
| `--cubes-dump-enable` | 启用持久化功能（必需） |
| `--cubes-dump-file <PATH>` | 指定自定义配置文件路径（默认 `/etc/polycube/cubes.yaml`） |
| `--cubes-dump-clean-init` | 启动时清空历史配置（初始化空拓扑） |

**示例**：
```bash
# 启用持久化并指定自定义配置文件
polycubed --cubes-dump-enable --cubes-dump-file ~/my_cubes.yaml

# 初始化空配置（覆盖默认文件）
polycubed --cubes-dump-enable --cubes-dump-clean-init
```

**限制**：
- 不支持 YANG 动作（如防火墙规则的 `append` 操作）。
- 部分服务可能无法一次性加载全部配置。

---

#### **6. 调试与日志**
通过调整日志级别或直接查看日志定位问题。

**调试模式启动**：
```bash
polycubed --loglevel=debug
```

**日志查看**：
```bash
# 查看实时日志
journalctl -u polycubed -f

# 查看历史日志文件
tail -f /var/log/polycube/polycubed.log
```

---

#### **7. REST API**
Polycubed 提供标准的 REST API 接口，可通过工具（如 `curl` 或 `polycubectl`）交互。

**API 文档**：
- 访问运行中的 `polycubed` 的 Swagger UI（默认地址 `http://localhost:9000/polycube/v1/api/docs`）。

---

#### **8. 最佳实践**
1. **服务管理**：优先使用 `systemd` 管理服务生命周期。
2. **持久化配置**：生产环境建议启用 `--cubes-dump-enable`。
3. **日志分级**：日常使用 `info` 级别，调试时切换为 `debug`。
4. **安全配置**：通过 `--cert` 和 `--key` 启用 HTTPS 加密通信。

---

**附：快速命令速查表**
```bash
# 启动服务并启用持久化
sudo systemctl start polycubed --cubes-dump-enable

# 查看实时日志
journalctl -u polycubed -f

# 修改配置后重启
sudo systemctl restart polycubed
```





### Polycubectl

---

#### **1. Polycubectl 概述**
**Polycubectl** 是 Polycube 的命令行接口（CLI），用于与 Polycube 中的网络服务（**Cubes**，如桥接、路由器等）交互。其核心特性包括：
- **服务无关性**：基于 YANG 数据模型，无需为新开发的 Cube 修改 CLI。
- **动态扩展**：支持自动识别新添加的 Cube。
- **灵活配置**：支持交互式帮助、文件输入、复杂参数传递等功能。

---

#### **2. 安装与前提条件**
- **默认安装**：安装 Polycube 时自动包含 `polycubectl`。
- **依赖要求**：需先启动 `polycubed` 守护进程。
  ```bash
  # 启动 polycubed
  sudo polycubed
  ```

---

#### **3. 基础命令结构**
命令遵循层级结构，格式为：  
`polycubectl [父资源] [操作] [子资源] [参数=值]`

| 组件        | 说明                                                                 |
|-------------|----------------------------------------------------------------------|
| **父资源**  | 操作的上下文路径（如 `router`、`firewall`）。                        |
| **操作**    | `add`（添加）、`del`（删除）、`show`（查看）、`set`（设置）或 YANG 动作。 |
| **子资源**  | 具体操作的目标资源（如 `ports`、`rules`）。                          |
| **参数**    | 附加配置参数（如 `loglevel=debug`）。                                |

---

#### **4. 常用操作示例**

##### **服务管理**
```bash
# 添加路由器实例 r0，并设置日志级别为 debug
polycubectl router add r0 loglevel=debug

# 查看所有运行中的 Cube
polycubectl cubes show

# 删除防火墙实例 fw1
polycubectl firewall del fw1
```

##### **端口配置**
```bash
# 为路由器 r0 添加端口 port1，配置 IP 和子网掩码
polycubectl r0 ports add port1 ip=10.1.0.1 netmask=255.255.0.0

# 设置端口 peer 设备为 veth1
polycubectl r0 ports port1 set peer=veth1

# 删除端口
polycubectl r0 ports del port1
```

##### **YANG 动作**
```bash
# 在防火墙 fw1 的入站链末尾添加规则（源 IP 为 10.0.0.1，动作为丢弃）
polycubectl firewall fw1 chain ingress append src=10.0.0.1 action=DROP
```

---

#### **5. 输入方式**
支持从命令行、标准输入或文件加载配置。

##### **直接输入复杂配置**
```bash
# 通过 JSON 创建 helloworld 实例
polycubectl helloworld add hw0 << EOF
{
  "loglevel": "debug",
  "action": "forward"
}
EOF
```

##### **从文件加载配置**
```bash
# 从 YAML 文件创建路由器
polycubectl router add r0 < r0.yaml

# 从 JSON 文件批量添加 Cubes
polycubectl cubes add < cubes.json
```

---

#### **6. 帮助系统**
通过 `?` 或 `--help` 获取上下文帮助，动态探索命令结构。

**示例：配置路由器的交互式帮助**
```bash
# 查看 router 支持的顶层操作
polycubectl router ?

# 查看如何添加路由器实例
polycubectl router add ?

# 输出示例：
Keyword             Type     Description
<name>              string   Name of the router service
loglevel=value      string   Logging level (e.g., INFO, DEBUG)
```

---

#### **7. 输出控制**
通过标志调整输出格式和内容。

##### **格式控制**
```bash
# 以 JSON 格式显示详细信息
polycubectl router r0 show -json

# 以 YAML 格式显示简要信息
polycubectl router r0 show -yaml -brief
```

##### **隐藏字段**
```bash
# 隐藏端口信息
polycubectl router r0 show -hide=ports

# 隐藏端口 UUID 和 MAC 地址
polycubectl router r0 show -hide=ports.uuid,ports.mac
```

---

#### **8. 配置选项**
支持通过配置文件或环境变量自定义行为。

##### **配置文件**
默认路径：`~/.config/polycube/polycubectl_config.yaml`  
```yaml
debug: false     # 显示 HTTP 请求/响应详情
expert: true     # 启用运行时添加服务
url: http://localhost:9000/polycube/v1/  # polycubed 地址
cert: ""         # 客户端证书
key: ""          # 私钥
cacert: ""       # CA 证书
```

##### **环境变量**
覆盖配置文件中的设置：
```bash
export POLYCUBECTL_URL="http://10.0.0.1:9000/polycube/v1/"
export POLYCUBECTL_DEBUG="true"
```

---



#### **9. 最佳实践**
1. **利用帮助系统**：通过 `?` 实时探索命令结构。
2. **配置文件持久化**：将常用配置（如 `url`）写入配置文件。
3. **批量操作**：使用 YAML/JSON 文件管理复杂拓扑。
4. **输出调试**：开启 `debug` 模式排查通信问题。

---

#### **10. 命令速查表**
```bash
# 查看所有可用服务类型
polycubectl services show

# 显示网络设备列表
polycubectl netdevs show

# 查看拓扑结构
polycubectl topology show

# 连接两个端口
polycubectl connect br1:port1 router1:eth0

# 附加透明 Cube 到网络接口
polycubectl attach firewall1 eth0
```

---

**附：调试技巧**
```bash
# 查看 polycubectl 与 polycubed 的通信细节
export POLYCUBECTL_DEBUG=true
polycubectl router r0 show

# 实时监控 polycubed 日志
journalctl -u polycubed -f
```

### Cubes

---

#### **1. Cube 基础概念**
**Cube** 是 Polycube 的核心逻辑实体，包含以下组件：
- **数据平面（Data Plane）**：基于 eBPF 实现高性能包处理（内核态）。
- **控制/管理平面**：通过 REST/gRPC 接收配置（用户态）。
- **慢速路径（Slow Path）**：处理 eBPF 无法实现的复杂逻辑（如泛洪、循环操作）。

---

#### **2. Cube 类型**
Polycube 定义两类 Cube，适应不同网络功能需求：

##### **(1) 标准 Cube (Standard Cube)**
- **核心特性**  
  - 具备**转发能力**（如路由器、网桥）。  
  - 定义**端口（Ports）**，通过端口连接其他 Cube 或网络设备。  
  - 遵循**中间盒模型**（多端口网络功能）。

- **架构示意图**  
  ```text
          +--------------+
  port1---|              |---port3
          |  核心处理逻辑 |  
  port2---|              |---port4
          +--------------+
  ```

- **典型服务**  
  `router`, `bridge`, `loadbalancer`

##### **(2) 透明 Cube (Transparent Cube)**
- **核心特性**  
  - **无端口**，直接附加到现有端口或网络设备（如 `eth0` 或 `router1:port1`）。  
  - 支持**流量方向处理**：  
    - **入口（Ingress）**：流量进入目标端口前的处理（如防火墙过滤）。  
    - **出口（Egress）**：流量离开目标端口后的处理（如 NAT 转换）。  
  - 支持**堆叠**（类似功能链）。

- **架构示意图**  
  ```text
       +-------------------+
       |    +---------+    |
       |  ->| ingress |->  |
  <--->|*   +---------+   *|<--->
       |  <-| egress  |<-  |
       |    +---------+    |
       +-------------------+
  ```

- **典型服务**  
  `firewall`, `nat`, `packetmonitor`

---

#### **3. 影子 Cube (Shadow Cube)**
- **核心特性**  
  - **仅限标准 Cube**：通过 `shadow=true` 参数创建。  
  - **关联 Linux 网络命名空间**：实现与 Linux 网络栈的深度集成。  
  - **双向配置同步**：可通过 `polycubectl` 或传统 Linux 命令（如 `ip`）配置端口。

- **应用场景**  
  - 混合配置（部分参数由 Polycube 管理，部分由 Linux 管理）。  
  - 调试流量（结合 Span 模式捕获流量）。

- **架构示意图**  
  ```text
  Linux 命名空间                  Polycube 影子 Cube
  +--------------+               +--------------+
  | port1---eth0 |               | port1---核心 |
  | port2---veth1| <---> Shadow  | port2---逻辑 |
  +--------------+               +--------------+
  ```

- **操作示例**  
  ```bash
  # 创建影子路由器
  polycubectl router add r1 shadow=true

  # 通过 Linux 命令配置 IP
  ip netns exec pcn-r1 ifconfig port1 10.0.0.1/24

  # 通过 polycubectl 配置 IP（等效）
  polycubectl r1 ports port1 set ip=10.0.0.1/24
  ```

---

#### **4. 端口操作**
端口是 Cube 与外部通信的接口，需连接后才可传输流量。

##### **(1) 创建端口**
```bash
# 在路由器 r1 上创建端口并配置 IP
polycubectl r1 ports add port1 ip=10.0.1.1/24

# 在网桥 br1 上创建简单端口
polycubectl br1 ports add port2
```

##### **(2) 连接端口**
两种方式实现端口连接：`set peer` 或 `connect`。

- **方法 1：`set peer`**  
  ```bash
  # 连接到网络设备（veth1）
  polycubectl r1 ports port1 set peer=veth1

  # 连接到其他 Cube 的端口
  polycubectl r1 ports port2 set peer=br1:port2
  polycubectl br1 ports port2 set peer=r1:port2
  ```

- **方法 2：`connect`**  
  ```bash
  # 连接到网络设备
  polycubectl connect r1:port1 veth1

  # 连接到其他 Cube 的端口
  polycubectl connect r1:port2 br1:port2
  ```

- **断开连接**  
  ```bash
  polycubectl r1 ports port1 set peer=""
  ```

---

#### **5. 附加透明 Cube**
使用 `attach`/`detach` 将透明 Cube 绑定到端口或网络设备。

```bash
# 将防火墙附加到路由器的 port2
polycubectl attach firewall1 r1:port2

# 将监控器附加到网络设备 veth1
polycubectl attach monitor0 veth1

# 分离透明 Cube
polycubectl detach firewall1
```

---

#### **6. eBPF 钩子点（Hook Points）**
指定 Cube 的 eBPF 程序挂载位置，影响性能和兼容性。

| 钩子类型   | 特性                              | 适用场景                   |
|------------|-----------------------------------|--------------------------|
| **TC**     | 兼容性最佳，性能中等              | 通用场景                 |
| **XDP_SKB**| 高性能，依赖驱动支持              | 高吞吐量处理             |
| **XDP_DRV**| 最高性能，需 NIC 驱动支持         | 极限性能需求（如 DDoS 防御）|

**操作示例**  
```bash
# 创建使用 XDP_SKB 钩子的路由器
polycubectl router add r1 type=XDP_SKB
```

---

#### **7. 流量调试：Span 模式**
- **功能**：复制 Cube 端口的流量到关联的 Linux 命名空间，供 `tcpdump` 捕获。  
- **代价**：高资源消耗，建议仅在调试时启用。

**操作流程**  
```bash
# 启用 Span 模式
polycubectl r1 set span=true

# 进入命名空间抓包
ip netns exec pcn-r1 tcpdump -i port1

# 关闭 Span 模式
polycubectl r1 set span=false
```

**注意事项**  
- 避免在启用 Span 时激活 Linux 内核转发（如 `ip_forward`），防止流量环路。  
- 仅适用于影子 Cube。

---

#### **8. 典型拓扑示例**
```text
   veth1                                      veth3
     |                                          |
+---------+        +---------+            +---------+
| bridge1 |--[fw0] | router1 |--[ddos0]--| router2 |--veth5
+---------+        +---------+            +---------+
     |                                          |
   veth2                                      veth4
```

**构建命令**  
```bash
# 创建基础服务
polycubectl bridge add br1
polycubectl router add r1
polycubectl router add r2

# 连接端口
polycubectl connect br1:port1 veth1
polycubectl connect br1:port2 veth2
polycubectl connect r1:port1 br1:port3
polycubectl connect r1:port2 r2:port1
polycubectl connect r2:port2 veth3

# 附加透明服务
polycubectl attach firewall0 r1:port1
polycubectl attach ddos0 r1:port2
```

---

#### **9. 常用命令速查**
```bash
# 查看所有 Cube 实例
polycubectl cubes show

# 查看端口连接状态
polycubectl r1 ports show

# 检查 eBPF 钩子类型
polycubectl r1 show type

# 批量导入配置
polycubectl cubes add < topology.yaml
```




## yang数据模型









## 生成代码框架

### 安装依赖项
```bash
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

### 准备目录结构
```bash
# 创建输入/输出目录
mkdir -p ~/polycube-codegen/input
mkdir -p ~/polycube-codegen/output

# 复制自定义YANG模型到输入目录
cp my-service.yang ~/polycube-codegen/input/
```

---

### 配置基础环境变量
```bash
# 设置基础YANG模型路径（默认值）
export POLYCUBE_BASEMODELS="/usr/local/include/polycube/datamodel-common/"

# 如果路径不存在，需要从源码安装基础模型
git clone https://github.com/polycube-network/polycube
cd polycube
sudo make install-base  # 安装核心基础模型
```

---

### 拉取代码生成镜像
```bash
docker pull polycubenets/polycube-codegen
```

---

### 配置Docker代理
如果需通过代理访问Docker Hub：

#### 1. 配置系统级代理
```bash
# 创建systemd代理配置目录
sudo mkdir -p /etc/systemd/system/docker.service.d

# 编辑代理配置文件
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf

# 添加内容示例：
[Service]
Environment="HTTP_PROXY=http://192.168.88.1:7890"
Environment="HTTPS_PROXY=http://192.168.88.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"

# 重载服务配置
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 2. 验证代理连通性
```bash
# 测试Docker Hub HTTP访问
curl -x http://192.168.88.1:7890 -I https://registry-1.docker.io/v2/
```

---

### 执行代码生成
```bash
docker run -it --user `id -u` \
  -v $POLYCUBE_BASEMODELS:/polycube-base-datamodels \
  -v ~/polycube-codegen/input:/input \
  -v ~/polycube-codegen/output:/output \
  polycubenets/polycube-codegen \
  -i /input/bridge.yang \
  -o /output/bridge_stub
```

---

### 问题排查

#### Docker拉取失败
```bash
# 临时设置会话级代理（仅限终端会话）
export HTTP_PROXY=http://192.168.88.1:7890
export HTTPS_PROXY=http://192.168.88.1:7890
sudo -E docker pull polycubenets/polycube-codegen
这次应该能拉取成功
```

#### 挂载路径错误问题
```bash
# 检查环境变量有效性
echo $POLYCUBE_BASEMODELS  # 预期输出完整路径

#如果输出错误 执行下面命令
export POLYCUBE_BASEMODELS="/usr/local/include/polycube/datamodel-common/"

# 验证路径存在性
ls -l $POLYCUBE_BASEMODELS  # 应显示core.yang等文件

# 权限验证（需root所有）
ls -l /etc/systemd/system/docker.service.d/http-proxy.conf
# 正确权限示例：
-rw-r--r-- 1 root root 216 Mar 28 15:30 http-proxy.conf
```

---

#### 代码生成成功验证
```
# 检查输出目录生成的代码框架
ls -R ~/polycube-codegen/output
# 预期看到 include/ src/ CMakeLists.txt 等文件
```

---

---

## 编译与部署Polycube服务

### 代码编译流程
#### 1. 创建构建目录并编译代码
```bash
# 在生成的代码框架下执行
mkdir build && cd build
cmake ..
make -j $(getconf _NPROCESSORS_ONLN)  # 并行编译，提升速度
sudo make install                      # 安装到系统目录
```

#### 2. 验证编译产物
检查是否生成 `.so` 动态库文件：
```bash
# 在 build/src/ 目录下查看生成的动态库
ls build/src/libpcn-helloworld.so

# 动态库应被安装到系统目录
ls /usr/lib/libpcn-helloworld.so
```

---

### 服务管理命令
#### 1. 加载服务到Polycube
```bash
# 加载动态库并为服务命名
polycubectl services add type=lib \
  uri=/absolute/path/to/libpcn-service_name.so \
  name=service_name


polycubectl services add type=lib \
  uri=/root/polycube/build/src/services/pcn-bridge/src/libpcn-bridge.so \
  name=bridge

# 查看所有已加载服务（验证是否成功）
polycubectl services show
```

#### 2. 创建服务实例
```bash
# 创建服务实例（此处实例名任意）
polycubectl service_name add instance_name

# 查看实例配置信息
polycubectl instance_name show
```

#### 3. 卸载服务
```bash
# 移除服务及其所有实例
polycubectl services del service_name

#添加实例
polycubectl router add r1
#删除实例
polycubectl router del r1

#查看实例列表
polycubectl router list

```

---

### 编译错误处理
#### 报错示例
```
/usr/include/polycube/services/table.h:25:10: fatal error: 
  ./../../../polycubed/src/utils/utils.h: No such file or directory
```

#### 解决方案
1. **查找缺失文件路径**
```bash
find / -name utils.h  # 预期输出类似：/root/polycube/src/polycubed/src/utils/utils.h
```

2. **修复头文件路径**
```bash
# 确认 /usr/include/polycube 目录存在
ls /usr/include/polycube

# 将缺失文件复制到系统包含目录
cp -r /root/polycube/src/polycubed/ /usr/include/
```

3. **重新编译**
```bash
cd build
rm -rf *    
cmake ..
make -j $(getconf _NPROCESSORS_ONLN)
sudo make install
```

---































