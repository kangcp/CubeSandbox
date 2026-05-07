# CubeSandbox 部署与问题修复指南

> 本文档记录了在 QEMU 开发环境中部署 CubeSandbox 时遇到的问题及修复方案，适用于从源码编译并部署的场景。

---

## 目录

- [环境要求](#环境要求)
- [问题概述](#问题概述)
- [修复方案](#修复方案)
  - [修复 1：Shim Vsock 超时（10s → 120s）](#修复-1shim-vsock-超时10s--120s)
  - [修复 2：Probe 探测超时（30s → 120s）](#修复-2probe-探测超时30s--120s)
- [编译步骤](#编译步骤)
  - [Step A：构建 Docker Builder 镜像](#step-a构建-docker-builder-镜像)
  - [Step B：编译 CubeShim](#step-b编译-cubeshim)
  - [Step C：编译 cubemastercli](#step-c编译-cubemastercli)
- [部署步骤](#部署步骤)
  - [Step 1：准备开发环境](#step-1准备开发环境)
  - [Step 2：安装 CubeSandbox 服务](#step-2安装-cubesandbox-服务)
  - [Step 3：同步编译产物到 VM](#step-3同步编译产物到-vm)
  - [Step 4：重启服务并创建模板](#step-4重启服务并创建模板)
  - [Step 5：运行测试代码](#step-5运行测试代码)
- [常见问题排查](#常见问题排查)
- [修改文件清单](#修改文件清单)

---

## 环境要求

| 项目 | 要求 |
|------|------|
| **宿主机 OS** | x86_64 Linux（支持 KVM 嵌套虚拟化） |
| **Docker** | 已安装并运行 |
| **QEMU** | 已安装（`qemu-system-x86_64`） |
| **嵌套 KVM** | 宿主机需启用嵌套虚拟化 |
| **Rust 工具链** | 1.77.2（通过 Docker Builder 自动管理） |
| **Go** | 1.24.8（通过 Docker Builder 自动管理） |
| **网络** | 需要访问 GitHub、Docker Hub、Ubuntu Archive 等 |

---

## 问题概述

在 QEMU 开发环境中按照 README 执行到 Step 3（创建 Code Interpreter 模板）时，可能遇到以下问题：

### 问题 1：Shim Vsock 通信超时

```
Create shim task: Others(Other: Create sandbox failed: Recive event timeout after 10000ms)
```

**原因**：`CubeShim` 中 vsock 通信等待超时设置为 10 秒，在 QEMU 嵌套虚拟化环境下，VM 启动速度较慢，10 秒不够用。

### 问题 2：Probe 健康检查失败

```
template creation failed: Get "http://192.168.1.84:49999/health": dial tcp 192.168.1.84:49999: connect: connection refused
```

**原因**：`cubemastercli` 中 probe 的默认总超时为 30 秒，沙箱内的 code interpreter 服务启动时间超过 30 秒。

### 问题 3：去掉 probe 后沙箱不可用

如果去掉 `--probe 49999` 创建模板，模板创建会成功，但使用 SDK 创建沙箱时会返回 **502 Bad Gateway**。这是因为 probe 的作用是确保服务完全启动后才拍快照，没有 probe 时快照在服务启动之前就拍了。

---

## 修复方案

### 修复 1：Shim Vsock 超时（10s → 120s）

**文件**：`CubeShim/shim/src/sandbox/sb.rs`（约第 780 行）

**修改前**：
```rust
let ev = ch
    .wait_notify(Duration::from_nanos(1000 * 1000 * 1000 * 10 as u64))
    .await?;
```

**修改后**：
```rust
let ev = ch
    .wait_notify(Duration::from_nanos(1000 * 1000 * 1000 * 120 as u64))
    .await?;
```

**说明**：将 vsock 等待超时从 10 秒增加到 120 秒，给 QEMU 嵌套虚拟化环境下的 VM 足够的启动时间。

### 修复 2：Probe 探测超时（30s → 120s）

**文件**：`CubeMaster/cmd/cubemastercli/commands/cubebox/template.go`（约第 1327 行）

**修改前**：
```go
overrides.Probe = &types.Probe{
    ProbeHandler: &types.ProbeHandler{
        HttpGet: &types.HTTPGetAction{
            Path: &probePath,
            Port: int32(probePort),
            Host: &host,
        },
    },
    TimeoutMs:        30000,
    PeriodMs:         500,
    FailureThreshold: 60,
    SuccessThreshold: 1,
}
```

**修改后**：
```go
overrides.Probe = &types.Probe{
    ProbeHandler: &types.ProbeHandler{
        HttpGet: &types.HTTPGetAction{
            Path: &probePath,
            Port: int32(probePort),
            Host: &host,
        },
    },
    TimeoutMs:        120000,
    PeriodMs:         500,
    FailureThreshold: 240,
    SuccessThreshold: 1,
}
```

**说明**：
- `TimeoutMs`：总探测超时从 30 秒增加到 120 秒
- `FailureThreshold`：最大失败次数从 60 增加到 240（配合 500ms 重试间隔，覆盖 120 秒窗口）
- Probe 会每 500ms 尝试一次 HTTP GET `http://<sandbox-ip>:49999/health`，最多等待 120 秒

---

## 编译步骤

> **前提**：以下所有命令在 **宿主机** 上的 `CubeSandbox` 项目根目录下执行。

### Step A：构建 Docker Builder 镜像

Builder 镜像包含 Rust、Go、protoc 等所有编译依赖。首次编译前需要构建（后续可跳过）。

```bash
cd /path/to/CubeSandbox

# 注意：如果在海外环境（非中国大陆），需要加 GITHUB_ACTIONS=true 参数
# 以跳过 Tencent 镜像源（SSL 证书问题），改用官方 Ubuntu 源
docker build \
  --build-arg GITHUB_ACTIONS=true \
  --build-arg RUSTUP_DIST_SERVER=https://static.rust-lang.org \
  --build-arg RUSTUP_UPDATE_ROOT=https://static.rust-lang.org/rustup \
  -t cube-sandbox-builder:latest \
  -f docker/Dockerfile.builder .
```

**中国大陆环境**可以直接使用默认镜像源：
```bash
make builder-image
```

> ⚠️ 构建 builder 镜像需要下载约 5GB 的依赖，首次构建可能需要 10-30 分钟。

### Step B：编译 CubeShim

编译修改了 vsock 超时的 shim 组件：

```bash
cd /path/to/CubeSandbox
mkdir -p _output/bin

make builder-run BUILDER_CMD='mkdir -p /workspace/_output/bin && \
  cd /workspace/CubeShim && \
  cargo build --release --locked && \
  install -m 0755 /workspace/CubeShim/target/release/containerd-shim-cube-rs /workspace/_output/bin/containerd-shim-cube-rs && \
  install -m 0755 /workspace/CubeShim/target/release/cube-runtime /workspace/_output/bin/cube-runtime'
```

编译产物：
- `_output/bin/containerd-shim-cube-rs`（约 14MB）
- `_output/bin/cube-runtime`（约 9MB）

> 编译耗时约 2-3 分钟（首次可能更长，需要下载 crates）。
> Rust 工具链版本：**1.77.2**（定义在 `CubeShim/rust-toolchain.toml`）。

### Step C：编译 cubemastercli

编译修改了 probe 超时的 CLI 工具：

```bash
cd /path/to/CubeSandbox

make builder-run BUILDER_CMD='mkdir -p /workspace/_output/bin && \
  cd /workspace/CubeMaster && \
  go mod download && \
  go build -o /workspace/_output/bin/cubemastercli ./cmd/cubemastercli'
```

编译产物：
- `_output/bin/cubemastercli`（约 32MB）

---

## 部署步骤

### Step 1：准备开发环境

```bash
cd /path/to/CubeSandbox/dev-env

# 首次运行：下载并初始化 VM 镜像
./prepare_image.sh

# 启动 VM（保持此终端打开）
./run_vm.sh
```

在**第二个终端**中登录 VM：
```bash
cd /path/to/CubeSandbox/dev-env
./login.sh
```

VM 信息：
- 用户：`opencloudos`，密码：`opencloudos`
- SSH 端口：`10022`
- Cube API：`http://127.0.0.1:13000` → guest:3000
- CubeProxy HTTP：`http://127.0.0.1:11080` → guest:80
- CubeProxy HTTPS：`https://127.0.0.1:11443` → guest:443

### Step 2：安装 CubeSandbox 服务

在 VM 内（通过 `login.sh` 登录后）执行：

**Global 用户**：
```bash
curl -sL https://github.com/tencentcloud/CubeSandbox/raw/master/deploy/one-click/online-install.sh | bash
```

**中国大陆用户**：
```bash
curl -sL https://cnb.cool/CubeSandbox/CubeSandbox/-/git/raw/master/deploy/one-click/online-install.sh | MIRROR=cn bash
```

### Step 3：同步编译产物到 VM

回到**宿主机**终端，将编译好的二进制文件同步到 VM：

```bash
cd /path/to/CubeSandbox/dev-env

# 同步 shim 组件（containerd-shim-cube-rs 和 cube-runtime）
./sync_to_vm.sh bin containerd-shim-cube-rs cube-runtime

# 同步 cubemastercli
./sync_to_vm.sh bin cubemastercli
```

`sync_to_vm.sh` 会自动：
1. 通过 SSH/SCP 上传文件到 VM
2. 备份旧文件为 `*.bak`
3. 安装到正确的目录并设置权限

各组件的安装路径：

| 组件 | VM 内路径 |
|------|----------|
| `containerd-shim-cube-rs` | `/usr/local/bin/containerd-shim-cube-rs` |
| `cube-runtime` | `/usr/local/bin/cube-runtime` |
| `cubemastercli` | `/usr/local/services/cubetoolbox/CubeMaster/bin/cubemastercli` |

### Step 4：重启服务并创建模板

在 **VM 内**执行：

```bash
# 重启所有服务
systemctl restart cube-sandbox-oneclick.service

# 创建模板（注意必须带 --probe 49999）
cubemastercli tpl create-from-image \
  --image ccr.ccs.tencentyun.com/ags-image/sandbox-code:latest \
  --writable-layer-size 1G \
  --expose-port 49999 \
  --expose-port 49983 \
  --probe 49999
```

> ⚠️ `--probe 49999` 是**必须的**！它确保沙箱内的 code interpreter 服务完全启动后才拍快照。

监控构建进度：
```bash
cubemastercli tpl watch --job-id <job_id>
```

等待输出 `template_status: READY`，记录 `template_id`。

> 整个模板创建过程（下载镜像 + 构建 ext4 + 分发 + 快照）可能需要 5-10 分钟，请耐心等待。

### Step 5：运行测试代码

在 **VM 内**执行：

```bash
# 安装 Python SDK
yum install -y python3 python3-pip
pip install e2b-code-interpreter

# 设置环境变量
export E2B_API_URL="http://127.0.0.1:3000"
export E2B_API_KEY="dummy"
export CUBE_TEMPLATE_ID="<your-template-id>"  # 替换为 Step 4 中获得的 template_id
export SSL_CERT_FILE="/root/.local/share/mkcert/rootCA.pem"

# 运行测试
python3 -c "
import os
from e2b_code_interpreter import Sandbox

with Sandbox.create(template=os.environ['CUBE_TEMPLATE_ID']) as sandbox:
    result = sandbox.run_code(\"print('Hello from Cube Sandbox, safely isolated!')\")
    print(result)
"
```

---

## 常见问题排查

### Q1：Docker builder-image 构建失败（Tencent 镜像 SSL 证书错误）

```
Certificate verification failed: The certificate is NOT trusted.
```

**解决**：使用官方源构建：
```bash
docker build \
  --build-arg GITHUB_ACTIONS=true \
  --build-arg RUSTUP_DIST_SERVER=https://static.rust-lang.org \
  --build-arg RUSTUP_UPDATE_ROOT=https://static.rust-lang.org/rustup \
  -t cube-sandbox-builder:latest \
  -f docker/Dockerfile.builder .
```

### Q2：`make shim` 重新触发 builder-image 构建

`make shim` 依赖 `builder-image` target，会重新构建 Docker 镜像。

**解决**：直接使用 `make builder-run BUILDER_CMD='...'` 跳过 builder-image 依赖。

### Q3：模板创建时 `connection refused`

```
Get "http://192.168.1.84:49999/health": dial tcp 192.168.1.84:49999: connect: connection refused
```

**原因**：Probe 超时不够，沙箱内服务还没启动完。

**解决**：增大 probe 超时（本指南中的修复 2）。

### Q4：SDK 调用返回 502 Bad Gateway

**原因**：模板创建时未使用 `--probe`，快照在服务启动前就拍了。

**解决**：必须带 `--probe 49999` 重新创建模板。

### Q5：如何查看 VM 内的服务日志

```bash
# 查看所有服务状态
systemctl status cube-sandbox-oneclick.service

# 查看 cubelet 日志
journalctl -u cubelet --no-pager -n 100

# 查看 cubemaster 日志
journalctl -u cubemaster --no-pager -n 100
```

---

## 修改文件清单

| # | 文件路径 | 修改内容 | 原始值 | 修改后 |
|---|---------|---------|--------|--------|
| 1 | `CubeShim/shim/src/sandbox/sb.rs` | vsock 等待超时 | `1000 * 1000 * 1000 * 10` (10s) | `1000 * 1000 * 1000 * 120` (120s) |
| 2 | `CubeMaster/cmd/cubemastercli/commands/cubebox/template.go` | `TimeoutMs` | `30000` (30s) | `120000` (120s) |
| 3 | 同上 | `FailureThreshold` | `60` | `240` |

---

## 完整操作流程（快速参考）

```bash
# === 宿主机操作 ===

# 1. 克隆项目
git clone https://github.com/tencentcloud/CubeSandbox.git
cd CubeSandbox

# 2. 应用代码修改
# 修改 CubeShim/shim/src/sandbox/sb.rs:
#   将 .wait_notify(Duration::from_nanos(1000 * 1000 * 1000 * 10 as u64))
#   改为 .wait_notify(Duration::from_nanos(1000 * 1000 * 1000 * 120 as u64))
#
# 修改 CubeMaster/cmd/cubemastercli/commands/cubebox/template.go:
#   将 TimeoutMs: 30000 改为 TimeoutMs: 120000
#   将 FailureThreshold: 60 改为 FailureThreshold: 240

# 3. 构建 builder 镜像
docker build \
  --build-arg GITHUB_ACTIONS=true \
  --build-arg RUSTUP_DIST_SERVER=https://static.rust-lang.org \
  --build-arg RUSTUP_UPDATE_ROOT=https://static.rust-lang.org/rustup \
  -t cube-sandbox-builder:latest \
  -f docker/Dockerfile.builder .

# 4. 编译 shim
mkdir -p _output/bin
make builder-run BUILDER_CMD='mkdir -p /workspace/_output/bin && \
  cd /workspace/CubeShim && cargo build --release --locked && \
  install -m 0755 /workspace/CubeShim/target/release/containerd-shim-cube-rs /workspace/_output/bin/containerd-shim-cube-rs && \
  install -m 0755 /workspace/CubeShim/target/release/cube-runtime /workspace/_output/bin/cube-runtime'

# 5. 编译 cubemastercli
make builder-run BUILDER_CMD='mkdir -p /workspace/_output/bin && \
  cd /workspace/CubeMaster && go mod download && \
  go build -o /workspace/_output/bin/cubemastercli ./cmd/cubemastercli'

# 6. 准备并启动 VM（终端 1）
cd dev-env
./prepare_image.sh
./run_vm.sh

# 7. 登录 VM（终端 2）
cd dev-env && ./login.sh

# === VM 内操作 ===

# 8. 安装 CubeSandbox
curl -sL https://github.com/tencentcloud/CubeSandbox/raw/master/deploy/one-click/online-install.sh | bash

# === 回到宿主机（终端 3）===

# 9. 同步编译产物
cd dev-env
./sync_to_vm.sh bin containerd-shim-cube-rs cube-runtime
./sync_to_vm.sh bin cubemastercli

# === 回到 VM 内 ===

# 10. 重启服务
systemctl restart cube-sandbox-oneclick.service

# 11. 创建模板
cubemastercli tpl create-from-image \
  --image ccr.ccs.tencentyun.com/ags-image/sandbox-code:latest \
  --writable-layer-size 1G \
  --expose-port 49999 \
  --expose-port 49983 \
  --probe 49999

# 12. 监控进度
cubemastercli tpl watch --job-id <job_id>

# 13. 测试
export E2B_API_URL="http://127.0.0.1:3000"
export E2B_API_KEY="dummy"
export CUBE_TEMPLATE_ID="<template_id>"
export SSL_CERT_FILE="/root/.local/share/mkcert/rootCA.pem"
yum install -y python3 python3-pip
pip install e2b-code-interpreter
python3 -c "
import os
from e2b_code_interpreter import Sandbox
with Sandbox.create(template=os.environ['CUBE_TEMPLATE_ID']) as sandbox:
    print(sandbox.run_code(\"print('Hello from Cube Sandbox!')\"))
"
```

---

> **文档版本**：2026-05-07  
> **适用项目**：CubeSandbox (https://github.com/tencentcloud/CubeSandbox)  
> **测试环境**：EC2 x86_64 + QEMU 嵌套虚拟化 + OpenCloudOS 9