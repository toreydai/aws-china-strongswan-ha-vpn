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
2. Slave 进入 MASTER 状态，**先**通过 AWS CLI 将 EIP 从 Master 漂移到自身
3. EIP 迁移完成（sleep 3s）后，Slave 重启 StrongSwan、发起 IKE 协商建立隧道
4. Slave 更新 VPC 路由表，使对端流量经由 Slave 转发
5. SNS 发送故障告警邮件

**手动回切流程（Failback）**：
1. 确认 Master StrongSwan 已恢复：`sudo systemctl start strongswan`（在 Master 执行）
2. 在 Slave 上停止 Keepalived：`sudo systemctl stop keepalived`
3. Master 收到 priority 0 广播（或超时），进入 MASTER 状态，**先**漂回 EIP，再重建隧道
4. 确认 EIP 和隧道恢复后，重启 Slave Keepalived：`sudo systemctl start keepalived`

> **设计说明**：Master 配置了 `nopreempt`，故障恢复后不会自动抢占，需手动 failback。
> 这样可以避免网络抖动时的频繁切换（脑裂风险），运维人员可在确认 Master 完全健康后再执行回切。

---

## CloudFormation 模板

| 模板 | 用途 |
|------|------|
| `strongswan-main-china-new-vpc.yaml` | HA VPN 主端，**自动新建 VPC**（推荐快速体验） |
| `strongswan-on-prem-test-new-vpc.yaml` | 模拟对端（无真实本地机房时），自动新建 VPC |

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
| IPSecSharedSecret | 预共享密钥（两端一致，≥20 位） | 自定义 |

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

用步骤 2 的 `VPNPublicIP` 更新步骤 1 模板的 `RemoteServerIP` 参数，重新部署（Stack Update）。

### 步骤 4：验证连通性

登录 Master VPN 实例（SSM Session Manager）：

```bash
# 查看 StrongSwan 隧道状态（期望 ESTABLISHED）
swanctl --list-sas

# 查看 Keepalived 状态（Master 应为 MASTER STATE，priority 120）
journalctl -u keepalived -n 20

# Ping 对端私有 IP 验证连通性
ping <对端 VPN 实例私有 IP>
```

登录 Slave VPN 实例确认待机状态：

```bash
# Slave StrongSwan 守护进程应为 active（无活跃隧道，start_action = none）
systemctl is-active strongswan

# Slave Keepalived 应为 BACKUP STATE，priority 100
journalctl -u keepalived -n 10
```

---

## 故障切换测试

### 1. 模拟 Master 故障

在 Master 节点上停止 StrongSwan：

```bash
sudo systemctl stop strongswan
```

Keepalived 每 3s 检测一次，连续 10 次失败（约 30s）后触发切换。观察日志：

```bash
# Master 上查看 priority 下降
sudo journalctl -u keepalived -f
# 期望：Changing effective priority from 120 to 90

# Slave 上观察接管过程
sudo journalctl -u keepalived -f
# 期望：Changing effective priority from 100 to 100（StrongSwan 保持运行）
# 期望：received lower priority (90) advert from <MasterIP> - discarding
# 期望：Entering MASTER STATE
```

切换成功后验证（约 30s～3min）：
- AWS 控制台确认 EIP 已关联到 Slave 实例
- 检查 SNS 告警邮件
- 从 on-prem 端 Ping Slave 私有 IP，验证连通性恢复

### 2. 手动 Failback

```bash
# 第 1 步：在 Master 上恢复 StrongSwan（确保 Keepalived 健康检查通过）
sudo systemctl start strongswan      # 在 Master 执行

# 第 2 步：在 Slave 上停止 Keepalived，触发 Master 接管
sudo systemctl stop keepalived       # 在 Slave 执行

# 第 3 步：等待 EIP 漂回 Master（约 1～3 min），验证连通性
ping <对端私有 IP>                   # 在 on-prem 或 Master 执行

# 第 4 步：恢复 Slave Keepalived，使其重回 BACKUP 待机状态
sudo systemctl start keepalived      # 在 Slave 执行
```

---

## 常用运维命令

```bash
# 查看 VPN 隧道状态
swanctl --list-sas

# 重新加载配置（不中断隧道）
swanctl --load-all

# 手动重启隧道
systemctl restart strongswan
swanctl --load-all
swanctl --initiate --child net

# 查看 IKE 协商日志
journalctl -u strongswan -f

# 查看 Keepalived 状态与切换日志
systemctl status keepalived
journalctl -u keepalived -f

# 查看 EIP 当前关联实例（在任意实例执行）
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)
aws ec2 describe-addresses \
  --filters "Name=public-ip,Values=<VPN_EIP>" \
  --query 'Addresses[0].{InstanceId:InstanceId,AssocId:AssociationId}' \
  --region $REGION
```

---

## 关键参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| IKE 加密套件 | `aes256-sha256-ecp256` | IKEv2，AES-256 + SHA-256 + ECP-256（ECDH）|
| ESP 加密套件 | `aes256gcm128` | AES-256-GCM，AEAD 无需单独认证算法 |
| IKE 生命周期 | `86400s`（24h） | |
| SA 生命周期 | `3600s`（1h） | |
| Keepalived 检测 | 每 3s，连续 10 次失败（约 30s）触发切换 | |
| Keepalived 模式 | Master/Slave 均为 `state BACKUP`；Master priority 120 + `nopreempt`；Slave priority 100 | |
| Master start_action | `start` | 主动发起隧道 |
| Slave start_action | `none` | 守护进程运行但不主动发起，由 notify_master 显式 initiate |

---

## HA 设计说明

### Slave StrongSwan 为何保持运行而不停止？

**背景**：Keepalived 使用 `weight -30` 的健康检查脚本（`systemctl is-active strongswan`）决定优先级：

| 节点 | 正常 priority | StrongSwan 宕机后 priority |
|------|--------------|--------------------------|
| Master | 120 | 90（120 - 30） |
| Slave | 100 | 70（100 - 30） |

如果在 `notify_backup.sh` 中停止 Slave 的 StrongSwan，则：
- Slave priority 降至 70 < Master degraded priority 90
- Master 仍赢得选举，**Failover 永远不会触发**

**正确设计**：`notify_backup.sh` 仅通过 `swanctl --terminate` 断开活跃隧道，保持 StrongSwan 守护进程运行。Slave 的 StrongSwan 配置为 `start_action = none`，不会主动向对端发起连接（对端 SG 也只允许来自 EIP 的流量，天然阻断），健康检查持续通过，priority 维持 100。

当 Master StrongSwan 宕机：
- Master priority 90 < Slave priority 100 → Slave 赢得选举 → Failover 正常触发 ✓

### notify_master.sh 中 EIP 迁移须在 StrongSwan 之前

**背景**：对端的安全组只允许来自 VPN EIP 的 UDP 500/4500 流量。如果 StrongSwan 先于 EIP 迁移启动，Slave 发出的 IKE_INIT 源 IP 是其原有公网 IP，会被对端 SG 拒绝，StrongSwan 5 次重传后放弃，导致隧道建立失败。

**正确顺序**：
1. 迁移 EIP 到本实例
2. `sleep 3`（等待 EIP 绑定生效）
3. 重启 StrongSwan + 发起隧道

Master 的 `notify_master.sh`（failback 场景）同理。

### 为什么使用 nopreempt？

默认 VRRP 行为：Master 宕机 → Slave 接管 → Master 恢复 → Master **自动抢占**。

自动抢占的问题：
- Master 恢复时若服务不稳定，会造成 EIP 来回漂移，VPN 频繁中断
- 抢占期间 EIP 未绑定，隧道无法立即重建

本模板将 Master 和 Slave 都配置为 `state BACKUP`，Master 使用更高 priority 并启用 `nopreempt`，故障恢复后不自动抢占，运维人员确认 Master 完全健康后再执行手动 failback。

### EIP 漂移机制

使用 AllocationId + AssociationId 方式操作 EIP（VPC 推荐方式）：
1. `describe-addresses` 获取当前 AllocationId 和 AssociationId
2. `disassociate-address --association-id` 解绑当前关联
3. `associate-address --allocation-id` 绑定到新实例

---

## 实测数据（cn-northwest-1，2026-06-09）

| 指标 | 数值 |
|------|------|
| 正常状态隧道延迟（OnPrem → Master） | 0.27 ms |
| Keepalived 检测到故障时间 | 约 30s（10 × 3s） |
| Failover 总耗时（停 StrongSwan → 连通性恢复） | 约 3 min |
| 故障切换后延迟（OnPrem → Slave） | 0.8 ms |
| Failback 总耗时（停 Slave Keepalived → 连通性恢复） | 约 2.5 min |
| Failback 后延迟（OnPrem → Master） | 0.23 ms |

> Failover/Failback 耗时包含：Keepalived 检测（~30s）+ EIP 迁移 + StrongSwan 重启 + IKE 协商。

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
