# 深度学习服务器环境配置指南

> **硬件**: NVIDIA RTX 5080 (16GB) | **系统**: Ubuntu 24.04 LTS (桌面版) | **框架**: PyTorch 2.9 + CUDA 12.8 | **方向**: CV 计算机视觉

---

## 版本兼容性对照表

| 显卡 | 架构 | 最低驱动 | CUDA | cuDNN | PyTorch | Python |
|------|------|---------|------|-------|---------|--------|
| RTX 5090 | Blackwell | 560+ | 12.8+ | 9.x | 2.6+ | 3.11/3.12 |
| RTX 5080 | Blackwell | 560+ | 12.8+ | 9.x | 2.6+ | 3.11/3.12 |
| RTX 5070 Ti | Blackwell | 560+ | 12.8+ | 9.x | 2.6+ | 3.11/3.12 |
| RTX 5070 | Blackwell | 560+ | 12.8+ | 9.x | 2.6+ | 3.11/3.12 |

> RTX 50 系列基于 Blackwell 架构 (sm_120)，必须使用 CUDA 12.8 及以上。NVIDIA 驱动向后兼容，590 驱动可以运行 CUDA 12.6/12.8/13.0 的程序。

---

## 配置流程总览

```
Phase 1: 系统安装 → Phase 2: GPU 驱动 → Phase 3: Python 环境 → Phase 4: 远程开发 → Phase 5: 实用技巧
```

---

## Phase 01: 系统安装与基础配置

### Step 1: 安装 Ubuntu 24.04 LTS

**制作 USB 启动盘:**
- 下载 Ubuntu 24.04 LTS: https://ubuntu.com/download/desktop
- 用 Rufus (Windows) 写入 U 盘 (至少 8GB)
- Rufus 设置: 分区类型 GPT, 目标系统 UEFI

**BIOS 设置:**
- 开机按 F2/DEL/F12 进入 BIOS
- **关闭 Secure Boot** (安全启动) ← 非常重要，不关会导致 NVIDIA 驱动无法加载！
- 将 USB 设为第一启动项
- 确认 UEFI 模式

**分区建议 (选 "Something else" 手动分区):**

| 挂载点 | 文件系统 | 大小 | 说明 |
|--------|---------|------|------|
| /boot/efi | EFI | 512 MB | UEFI 引导 |
| /boot | ext4 | 1 GB | 内核引导文件 |
| swap | swap | = 内存大小 | 交换分区 |
| / | ext4 | 剩余全部 | 根分区 |

> 如果只装 Linux 不保留 Windows，选 "Erase disk and install Ubuntu" 最简单。

### Step 2: 系统基础配置

**更换清华源:**

```bash
# 备份
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# 重写源文件 (一步到位，避免 sed 替换不完整)
sudo tee /etc/apt/sources.list.d/ubuntu.sources << 'EOF'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu/
Suites: noble noble-updates noble-backports noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
```

> **踩坑经验**: Ubuntu 24.04 的源文件格式是 DEB822 格式 (.sources)，不是传统的 sources.list。用 sed 只替换域名可能遗漏第二段 security 源，建议直接用 `tee` 重写整个文件。

**更新系统:**

```bash
sudo apt update && sudo apt upgrade -y
```

**安装常用工具:**

```bash
sudo apt install -y git curl wget vim htop tmux \
  build-essential net-tools openssh-server \
  software-properties-common apt-transport-https
```

> **踩坑经验**: 如果 htop、tmux、build-essential 报 "无法定位软件包"，说明源没配对。先检查 `cat /etc/apt/sources.list.d/ubuntu.sources` 确认内容正确，再 `sudo apt update`。

**配置 SSH 服务:**

```bash
# 启动 SSH 并设为开机自启
sudo systemctl enable --now ssh

# 验证
sudo systemctl status ssh

# 查看本机 IP
ip addr show | grep "inet "
```

**配置防火墙:**

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8888/tcp    # Jupyter 端口
sudo ufw enable
```

> **警告**: 启用 UFW 前务必先放行 SSH (22 端口)，否则远程连接会被断开！

---

## Phase 02: NVIDIA 驱动安装

### Step 3: 安装 NVIDIA 驱动

```bash
# 查看推荐驱动
ubuntu-drivers devices

# 安装 open kernel module 版本
sudo apt install -y nvidia-driver-580-open

# 重启
sudo reboot

# 验证
nvidia-smi
```

> **踩坑经验 1 — 必须用 open 版本**: RTX 50 系列 (Blackwell) 要求使用 open kernel module 版本的驱动。如果装了非 open 版本 (如 nvidia-driver-580)，`nvidia-smi` 会显示 `No devices were found`，内核日志 (`sudo dmesg | grep -i nvidia`) 会反复出现 `Installed in this system requires use of the NVIDIA open kernel modules`。解决方法: 卸载后重装 open 版。

```bash
# 如果装错了，先卸载再装 open 版
sudo apt purge -y 'nvidia-*'
sudo apt autoremove -y
sudo apt install -y nvidia-driver-580-open
sudo reboot
```

> **踩坑经验 2 — Secure Boot 必须关闭**: 如果 Secure Boot 开着，NVIDIA 内核模块签名验证会失败。`sudo dmesg | grep -i nvidia` 会看到 `module verification failed: signature or/and required key Missing`。必须进 BIOS 关闭 Secure Boot。

> **踩坑经验 3 — 驱动版本与 PyTorch 的关系**: 驱动版本 (570/580/590) 与 CUDA 版本是两回事。NVIDIA 驱动向后兼容，580 驱动支持 CUDA 13.0，自然兼容 PyTorch 2.9 要求的 CUDA 12.8。不存在"驱动太新不适配"的问题。

### Step 4 & 5: CUDA Toolkit & cuDNN (可跳过)

PyTorch pip 包自带 CUDA 运行时和 cuDNN，不需要单独安装系统级 CUDA Toolkit。以下场景才需要单独装:
- 需要 `nvcc` 编译自定义 CUDA 算子
- 使用 TensorRT
- 编译某些论文代码

如需安装:

```bash
# CUDA Toolkit 12.8
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-8

# 环境变量
echo 'export PATH=/usr/local/cuda-12.8/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
nvcc --version

# cuDNN
sudo apt install -y cudnn-cuda-12
```

---

## Phase 03: Python 环境配置

### Step 6: 安装 Miniconda

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
# 安装过程一路回车，最后 "update your shell profile" 输入 yes

source ~/.bashrc
conda --version
```

**换清华源:**

```bash
# conda 源
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
conda config --set show_channel_urls yes

# pip 源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

> **踩坑经验**: 首次用 conda 创建环境时可能报 `CondaTOSNonInteractiveError`，需要先接受服务条款:

```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### Step 7: 创建环境 & 安装 PyTorch

```bash
# 创建环境
conda create -n cawa python=3.11 -y
conda activate cawa

# 安装 PyTorch 2.9 (CUDA 12.8)
pip install torch==2.9.0 torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

**验证 GPU:**

```python
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
# 期望输出:
# 2.9.0+cu128
# True
# NVIDIA GeForce RTX 5080
```

> 如果 `torch.cuda.is_available()` 返回 False: 1) 检查 nvidia-smi 是否正常 2) 确认装的是 cu128 版而不是 CPU 版 3) 确认驱动版本 ≥ 560

### Step 8: CV 常用库

```bash
# 基础科学计算
pip install numpy scipy pandas scikit-learn matplotlib seaborn

# CV 核心库
pip install opencv-python opencv-python-headless
pip install albumentations          # 数据增强
pip install timm                    # 预训练模型 (ResNet, ViT, ConvNeXt 等 700+)
pip install ultralytics             # YOLOv8/v11 目标检测

# 交互式开发
pip install jupyterlab ipywidgets
```

---

## Phase 04: 远程操作与开发工具

### Step 9: Windows SSH 连接 (局域网)

**首次连接 (Windows PowerShell):**

```powershell
ssh chenjie@10.16.173.29
# 输入 yes，然后输入 Linux 密码
```

**配置免密登录:**

```powershell
# 1. 生成密钥
ssh-keygen -t ed25519
# 一路回车

# 2. 传公钥到服务器
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh chenjie@10.16.173.29 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# 输入最后一次密码
```

**配置别名 (SSH config):**

```powershell
# 用 PowerShell 写入 (避免记事本编码问题)
Set-Content -Path "$env:USERPROFILE\.ssh\config" -Value "Host cawa`n    HostName 10.16.173.29`n    User chenjie`n    IdentityFile ~/.ssh/id_ed25519" -Encoding UTF8
```

> **踩坑经验**: 不要用记事本 (notepad) 编辑 SSH config，可能出现编码乱码导致 `Could not resolve hostname`。用 PowerShell `Set-Content` 或 VS Code 直接编辑更安全。

```powershell
# 以后一条命令直接连
ssh cawa
```

### Step 10: VS Code Remote SSH

1. Windows 上安装 VS Code
2. 安装扩展: **Remote - SSH**
3. `Ctrl+Shift+P` → `Remote-SSH: Connect to Host` → 选 **cawa**
4. 首次连接选 Linux，VS Code 自动在服务器上安装 server 组件
5. File → Open Folder → 选项目目录
6. `Ctrl+`` ` 打开终端 → `conda activate cawa` → 开始写代码

**推荐远程扩展:** Python, Pylance, Jupyter, GitLens

**日常工作流:**
1. 打开 VS Code → 连 cawa
2. 编辑器里写代码 (文件直接在服务器上)
3. 终端里跑训练
4. 长时间训练用 tmux 防止中断

### Step 11: Jupyter Lab 远程访问

```bash
# 在服务器上
conda activate cawa
jupyter lab --generate-config
jupyter lab password

# 在 tmux 里启动
tmux new -s jupyter
conda activate cawa
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
# Ctrl+B 再按 D 断开 tmux
```

Windows 浏览器访问: `http://10.16.173.29:8888`

**更安全的方式 — SSH 端口转发:**

```powershell
# Windows PowerShell
ssh -L 8888:localhost:8888 cawa
# 然后浏览器访问 http://localhost:8888
```

### Step 12: Tailscale 外网访问

**服务器端:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# 浏览器打开给出的链接登录
sudo systemctl enable tailscaled    # 开机自启
tailscale ip -4                     # 记下 100.x.x.x
```

**Windows 端:**
- 下载 https://tailscale.com/download
- 安装 → 用同一个账号登录

**SSH config 添加 Tailscale 别名:**

```
Host cawa-ts
    HostName 100.x.x.x
    User chenjie
    IdentityFile ~/.ssh/id_ed25519
```

```powershell
# 在家 (局域网)
ssh cawa

# 在外面 (外网)
ssh cawa-ts
```

---

## Phase 05: 实用技巧

### Step 13: GPU 监控

```bash
watch -n 1 nvidia-smi              # 基础
pip install gpustat && gpustat -i 1  # 简洁
pip install nvitop && nvitop         # 最全
```

### Step 14: tmux (防止训练中断)

```bash
tmux new -s train          # 创建会话
python train.py            # 启动训练
# Ctrl+B 再按 D            # 断开 (训练继续)
tmux attach -t train       # 重新连接
tmux ls                    # 查看所有会话
tmux kill-session -t train # 杀掉会话
```

**快捷键速查:**

| 快捷键 | 功能 |
|--------|------|
| Ctrl+B → D | 断开会话 |
| Ctrl+B → C | 新建窗口 |
| Ctrl+B → N/P | 下/上一个窗口 |
| Ctrl+B → % | 左右分屏 |
| Ctrl+B → " | 上下分屏 |
| Ctrl+B → 方向键 | 切换分屏 |

### Step 15: 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| `nvidia-smi` 报 No devices found | 驱动未用 open 版 | `sudo apt install nvidia-driver-580-open` |
| `nvidia-smi` 报 command not found | 驱动未安装/未重启 | 安装驱动后 `sudo reboot` |
| 驱动模块签名验证失败 | Secure Boot 开着 | 进 BIOS 关闭 Secure Boot |
| `torch.cuda.is_available()` = False | PyTorch 装了 CPU 版 | 用 `--index-url .../cu128` 重装 |
| CUDA out of memory | 显存不足 | 减小 batch_size / 用混合精度 |
| SSH 连接被拒绝 | SSH 服务未启动 / 防火墙 | `sudo systemctl start ssh` / `sudo ufw allow 22` |
| apt 找不到软件包 | 源配置不对 | 重写 ubuntu.sources 后 `sudo apt update` |
| ToDesk 远程黑屏 | Wayland 不兼容 | 编辑 `/etc/gdm3/custom.conf` 设 `WaylandEnable=false` |
| SSH config 别名不识别 | 记事本编码问题 | 用 PowerShell Set-Content 写入 |
| conda 创建环境报 TOS 错误 | 未接受服务条款 | `conda tos accept --override-channels --channel ...` |

---

## 实际配置结果

```
系统:    Ubuntu 24.04.4 LTS (内核 6.17.0-14-generic)
驱动:    NVIDIA 580.126.09 (open kernel module)
CUDA:    13.0 (驱动支持) / 12.8 (PyTorch 使用)
GPU:     NVIDIA GeForce RTX 5080 (16GB)
Python:  3.11.14
PyTorch: 2.9.0+cu128
环境名:  cawa
```
