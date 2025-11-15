---
slug: 2025-11-08-DisableSWAP
title: 彻底禁用 Fedora 系统的 SWAP 和 ZRAM
authors: xiaonya
tags: ["Fedora","SWAP","ZRAM","tutorial"]
---

K8S安装时需要禁用 SWAP，否则会报错。Fedora 默认启用了 ZRAM 作为 SWAP 的一种形式。本文记录一下彻底禁用 Fedora 系统的 SWAP 和 ZRAM的过程
<!-- truncate -->
## 彻底禁用 ZRAM 和 Swap
```bash

sudo swapoff -a

cat /etc/systemd/zram-generator.conf
cat /usr/lib/systemd/zram-generator.conf

sudo systemctl status systemd-zram-setup@zram0.service

# 屏蔽 ZRAM 启动服务
sudo systemctl mask systemd-zram-setup@zram0.service

# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 再次禁用 swap 以防万一
sudo swapoff -a

# 确认 swap 已禁用
swapon --show
free -h

```

> 重启后禁用失败，可能在 /etc/systemd/zram-generator.conf 没有创建配置文件来覆盖默认设置。系统正在使用 /usr/lib/systemd/zram-generator.conf 这个默认配置文件。

```bash
# 创建配置文件
sudo touch /etc/systemd/zram-generator.conf

# 编辑文件并添加配置，设置大小为 0
echo "[zram0]" | sudo tee /etc/systemd/zram-generator.conf > /dev/null
echo "zram-size = 0" | sudo tee -a /etc/systemd/zram-generator.conf > /dev/null

cat /etc/systemd/zram-generator.conf
# 输出应为：
# [zram0]
# zram-size = 0

# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 停止 ZRAM setup 服务
sudo systemctl stop systemd-zram-setup@zram0.service

# 再次临时关闭所有 swap
sudo swapoff -a

# 检查 Swap 状态，确保当前已关闭
free -h
sudo swapon --show
# 如果没有输出，表示 swap 已成功禁用
```