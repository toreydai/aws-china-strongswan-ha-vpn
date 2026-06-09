# AWS 中国区 StrongSwan 高可用 VPN

基于 **StrongSwan + Keepalived** 在 AWS 中国区构建站点到站点高可用 IPSec VPN。主节点故障时，EIP 自动漂移至备节点，并通过 SNS 发送告警邮件。

> **技术升级说明**：本方案从已停止维护的 Openswan（2014 年 EOL）升级为 StrongSwan，采用 IKEv2 + AES-256-GCM 现代加密套件，底座从 Amazon Linux 2 升级为 **Amazon Linux 2023**。

---

## 架构

```
               对端网络（本地机房 / 另一 VPC）
                    |   IKEv2 + AES-256-GCM
               -----+-----
              /            \
    ┌─────────────┐    ┌─────────────┐
    │  Master EC2  │◄──►│  Slave EC2  │  Keepalived VRRP
    │  StrongSwan  │    │  StrongSwan │  (单播模式，不依赖组播)
    └──────┬──────┘    └──────┬──────┘
           │                  │
           └────┬─────────────┘
                │ EIP（故障自动漂移）
                │
         AWS 中国区 VPC（跨 AZ 部署）
```

**故障切换流程（Failover）**：
1. Keepalived 每 3s 检测 `systemctl is-active strongswan`，连续 10 次失败（约 30s）触发切换
2. Slave 进入 MASTER 状态，通过 AWS CLI 将 EIP 从 Master 漂移到自身（使用 AllocationId）
3. Slave 重启 StrongSwan 重建 VPN 隧道
4. SNS 发送故障告警邮件

**手动回切流程（Failback）**：
1. 确认 Master 节点已完全恢复
2. 在 Slave 上停止 Keepalived：`sudo systemctl stop keepalived`
3. Master 检测到 Slave 不再发送 VRRP 报文，进入 MASTER 状态
4. Master 的 `notify_master` 脚本自动重启 StrongSwan、漂回 EIP、发送告警
5. Slave 重新启动 Keepalived：`sudo systemctl start keepalived`

> **设计说明**：Master 配置了 `nopreempt`，故障恢复后不会自动抢占，需手动 failback。
> 这样可以避免网络抖动时的频繁切换（脑裂风险），运维人员可在确认 Master 完全健康后再执行回切。

---

## CloudFormation 模板

| 模板 | 用途 |
|------|------|
| `strongswan-main-china-new-vpc.yaml` | HA VPN 主端，**自动新建 VPC**（推荐快速体验） |
| `strongswan-main-china-existing-vpc.yaml` | HA VPN 主端，接入已有 VPC |
| `strongswan-on-prem-test-new-vpc.yaml` | 模拟对端（无真实本地机房时），自动新建 VPC |
| `strongswan-on-prem-test-existing-vpc.yaml` | 模拟对端，接入已有 VPC |

---

## 快速开始（两端均用 AWS 模拟）

### 步骤 1：部署模拟对端（on-prem 测试环境）

在 CloudFormation 中部署 `strongswan-on-prem-test-new-vpc.yaml`：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| VPCCIDR | 模拟本地端 VPC 网段（不与 HA VPN 端重叠） | `172.16.0.0/16` |
| OnPremPrivateIP | 模拟本地端 VPN 实例私有 IP | `172.16.0.8` |
| LocalSubnet | 模拟本地端网段 | `172.16.0.0/16` |
| RemoteServerIP | **先填占位符**，步骤 2 部署后再更新 | `1.2.3.4` |
| RemoteSubnet | HA VPN 端 VPC 网段 | `10.50.0.0/16` |
| IPSecSharedSecret | 预共享密钥（两端一致，≥8 位） | 自定义 |

部署完成后，记录 Output `OnPremPublicIP`。

### 步骤 2：部署 HA VPN 主端

在 CloudFormation 中部署 `strongswan-main-china-new-vpc.yaml`：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| VPCCIDR | HA VPN 端 VPC 网段 | `10.50.0.0/16` |
| MasterPrivateIP | Master 节点私有 IP | `10.50.0.8` |
| SlavePrivateIP | Slave 节点私有 IP | `10.50.1.8` |
| RemoteServerIP | 步骤 1 的 OnPremPublicIP | `<步骤1输出>` |
| RemoteSubnet | 模拟本地端网段 | `172.16.0.0/16` |
| IPSecSharedSecret | 预共享密钥（与步骤 1 一致） | 同步骤 1 |
| KeepalivedAuthPass | VRRP 认证密码（8 位） | 自定义 |
| SNSEmail | 故障告警邮件 | 你的邮箱 |

部署完成后，记录 Output `VPNPublicIP`。

### 步骤 3：回填对端 IP 并更新 on-prem 端

用步骤 2 的 `VPNPublicIP` 更新步骤 1 模板的 `RemoteServerIP` 参数，重新部署。

### 步骤 4：验证连通性

登录任一 VPN 实例（SSM Session Manager 或 SSH）：

```bash
# 查看 StrongSwan 状态
swanctl --list-sas

# 期望输出（ESTABLISHED 表示隧道已建立）
# site-to-site{1}: INSTALLED, TUNNEL ...
# site-to-site[1]: ESTABLISHED 23 seconds ago

# 查看 SA 详情
swanctl --list-conns
swanctl --list-sas

# Ping 对端私有 IP 验证连通性
ping <对端 VPN 实例私有 IP>
```

---

## 故障切换测试

在 Master 节点上停止 StrongSwan 模拟故障：

```bash
sudo systemctl stop strongswan
```

观察 Keepalived 状态（约 30s 触发切换）：

```bash
sudo systemctl status keepalived -l
sudo journalctl -u keepalived -f
```

切换成功后验证：
- 控制台查看 EIP 已关联到 Slave 实例
- 检查 SNS 告警邮件
- 从 on-prem 端 Ping 对端，验证连通性恢复

**手动 Failback**（测试完后）：

```bash
# 1. 先恢复 Master StrongSwan（会被 notify_master 自动处理，此步可跳过）
# 2. 在 Slave 上停止 Keepalived，触发 Master 接管
sudo systemctl stop keepalived   # 在 Slave 上执行

# 3. 确认 EIP 漂回 Master 后，重新启动 Slave Keepalived
sudo systemctl start keepalived  # 在 Slave 上执行
```

---

## 常用运维命令

```bash
# 查看 VPN 状态
swanctl --list-sas

# 重新加载配置（不中断隧道）
swanctl --load-all

# 手动重启隧道
systemctl restart strongswan
swanctl --load-all
swanctl --initiate --child net

# 查看 IKE 协商日志
journalctl -u strongswan -f

# 查看 Keepalived 状态
systemctl status keepalived

# 查看切换日志
journalctl -u keepalived -f
```

---

## 关键参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| IKE 加密套件 | `aes256-sha2_256-ecp256` | IKEv2，AES-256 + SHA-256 + ECP-256（ECDH）|
| ESP 加密套件 | `aes256gcm128` | AES-256-GCM，AEAD 无需单独认证算法 |
| IKE 生命周期 | `86400s`（24h） | |
| SA 生命周期 | `3600s`（1h） | |
| Keepalived 检测 | 每 3s，连续 10 次失败（约 30s）触发切换 | |
| Keepalived 模式 | Master/Slave 均为 `state BACKUP`，Master priority 更高并启用 `nopreempt` | 手动 failback，避免网络抖动时频繁切换 |

---

## HA 设计说明

### 为什么使用 nopreempt？

默认 VRRP 行为：Master 宕机 → Slave 接管 → Master 恢复 → Master **自动抢占**。

自动抢占的问题：
- Master 恢复时若服务不稳定，会造成 EIP 来回漂移，VPN 频繁中断
- 更严重的是：抢占触发后，Master 没有 EIP 绑定期间的连接状态，隧道可能无法立即重建

本模板将 Master 和 Slave 都配置为 `state BACKUP`，Master 使用更高 priority 并启用 `nopreempt`。这样 `nopreempt` 不会被 Keepalived 清除。

使用该模式后：
- Master 宕机后由 Slave 稳定承载，不自动回切
- 运维人员确认 Master 完全健康后，通过停止 Slave 的 Keepalived 触发手动 failback
- `notify_master` 脚本确保 EIP 漂回和 StrongSwan 正确重启

### EIP 漂移机制

使用 AllocationId + AssociationId 方式操作 EIP（VPC 推荐方式）：
1. `describe-addresses` 获取当前 AllocationId 和 AssociationId
2. `disassociate-address --association-id` 解绑当前关联
3. `associate-address --allocation-id` 绑定到新实例

---

## 环境清理

```bash
# 删除两个 CloudFormation Stack（自动清理 EC2、EIP、安全组、VPC 等）
aws cloudformation delete-stack --stack-name <ha-vpn-stack-name> --region cn-northwest-1
aws cloudformation delete-stack --stack-name <onprem-test-stack-name> --region cn-northwest-1
```

---

## 免责声明

本项目仅供学习与技术参考，不构成生产部署方案。运行本实验将产生 AWS 资源费用，实验结束后请及时清理资源。
