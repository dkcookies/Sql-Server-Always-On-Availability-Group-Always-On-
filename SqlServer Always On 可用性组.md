---
# SQL Server Always On 两节点高可用部署手册

> **版本**：v1.0  
> **适用环境**：Windows Server + **SQL Server 企业版（Enterprise Edition）**  
> **架构**：两台 SQL Server + 文件共享见证服务器 + WSFC + Always On 可用性组 + Listener
> ⚠️ **重要前提：本手册仅适用于 SQL Server 企业版。**  
> Always On 可用性组的完整功能（多副本、自动故障转移、可读辅助副本、多数据库 AG）**仅在企业版中提供**。  
> SQL Server 标准版（Standard Edition）仅支持功能受限的“基本可用性组”（Basic Availability Group），**不支持**本手册中的完整高可用架构。  
> 在开始部署前，请先确认你的 SQL Server 版本：
> ```sql
> SELECT SERVERPROPERTY('Edition') AS Edition;
> ```
> 预期返回：`Enterprise Edition (64-bit)` 或类似企业版标识。
---

## 目录

1. [最终架构](#一最终架构)
2. [准备工作](#二准备工作)
3. [准备 Active Directory 域](#三准备-active-directory-域)
4. [设置主机名并加域](#四设置主机名并加域)
5. [网络连通性测试](#五网络连通性测试)
6. [安装 SQL Server](#六安装-sql-server)
7. [安装 SSMS](#七安装-ssms)
8. [安装 WSFC 功能](#八安装-wsfc-功能)
9. [验证 WSFC](#九验证-wsfc)
10. [创建 WSFC 集群](#十创建-wsfc-集群)
11. [配置文件共享见证](#十一配置文件共享见证)
12. [检查 WSFC 状态](#十二检查-wsfc-状态)
13. [检查 SQL Server 版本](#十三检查-sql-server-版本)
14. [启用 Always On](#十四启用-always-on)
15. [确认 Always On 已启用](#十五确认-always-on-已启用)
16. [创建数据库镜像 Endpoint](#十六创建数据库镜像-endpoint)
17. [开放防火墙 5022](#十七开放防火墙-5022)
18. [配置服务账号权限](#十八配置服务账号权限)
19. [准备数据库](#十九准备数据库)
20. [主库完整备份](#二十主库完整备份)
21. [复制备份文件](#二十一复制备份文件)
22. [从库还原数据库](#二十二从库还原数据库)
23. [创建可用性组](#二十三创建可用性组)
24. [配置同步模式](#二十四配置同步模式)
25. [配置故障转移模式](#二十五配置故障转移模式)
26. [配置数据同步方式](#二十六配置数据同步方式)
27. [创建 Listener](#二十七创建-listener)
28. [最终架构](#二十八最终架构)
29. [Java 应用连接](#二十九java-应用连接)
30. [故障转移测试](#三十故障转移测试)
31. [检查 AG 状态](#三十一检查-ag-状态)
32. [生产环境监控指标](#三十二生产环境监控指标)
33. [常见故障排查](#三十三常见故障排查)
34. [重要提醒](#三十四重要提醒)
35. [完整执行 Checklist](#三十五完整执行-checklist)

---

## 一、最终架构

```
Java应用
                       │
                       ▼
             SQL-AG-LISTENER
                  10.0.0.20
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
        SQL01               SQL02
     10.0.0.11            10.0.0.12
       Primary             Secondary
          │                   ▲
          │   事务日志同步     │
          └───────────────────┘
```

WSFC 集群：

```
SQLCLUSTER
     │
┌────┴────┐
│         │
SQL01   SQL02
```

```
文件共享见证服务器
      │
      │  SMB 共享文件夹
      │  授予 CNO 完全控制
      ▼
SQLCLUSTER$
```

> ****说明****：Always On AG 在 Windows 上依赖 WSFC，同一 AG 的副本必须位于同一 WSFC 的不同节点。双节点集群属于偶数节点，必须配置文件共享见证作为仲裁投票，否则一台服务器宕机时另一台无法自动接管。
>
> ****AG****：Always On Availability Group（Always On 可用性组）
>
> **WSFC**：Windows Server Failover Cluster（Windows Server 故障转移集群）
>
> **SMB**：Server Message Block，是 Windows 上用来**共享文件夹**的网络协议
>
> **CNO**：Cluster Name Object（集群名称对象），创建 WSFC 集群时，AD 里自动生成的一个计算机对象，名字是 `集群名$`

---

## 二、准备工作

| 项目     | SQL01       | SQL02       |
| -------- | ----------- | ----------- |
| 主机名   | SQL01       | SQL02       |
| IP       | 10.0.0.11   | 10.0.0.12   |
| SQL 角色 | Primary     | Secondary   |
| SQL 实例 | MSSQLSERVER | MSSQLSERVER |

**两台服务器配置尽量一致**：CPU、内存、磁盘、Windows 版本、SQL Server 版本及补丁。

---

## 三、准备 Active Directory 域

```
Domain Controller
       │
       ├── SQL01.corp.local
       │
       └── SQL02.corp.local
```

> ⚠️ **不要把 SQL Server 安装成域控制器。——数据库服务和域控制器不能在同一台服务器** 
> 如果公司已有 AD 域，直接使用现有域。

---

## 四、设置主机名并加域

```powershell
# SQL01
Rename-Computer -NewName "SQL01"
Restart-Computer

# SQL02
Rename-Computer -NewName "SQL02"
Restart-Computer
```

加域：系统属性 → 计算机名 → 更改 → 域 → 输入 `corp.local` → 输入域管理员账号。

---

## 五、网络连通性测试

```powershell
# 互相 ping
ping SQL02
ping SQL01

# 测试 SQL 端口
Test-NetConnection SQL02 -Port 1433
Test-NetConnection SQL01 -Port 1433

# 测试 AG 通信端口
Test-NetConnection SQL02 -Port 5022
```

> **注意**：1433 是 SQL 客户端端口，5022 是 AG 镜像端点端口，两者不同。

---

## 六、安装 SQL Server

两台服务器都安装，要求：

- 版本一致
- CU 补丁一致
- 实例名一致（`MSSQLSERVER`）
- 排序规则一致
- 数据库文件路径尽量一致

---

## 七、安装 SSMS

安装 **SQL Server Management Studio**，后续操作大部分通过它完成。

---

## 八、安装 WSFC 功能

WSFC（Windows Server Failover Clustering）是 ****Windows Server**** 的操作系统功能，不是 SQL Server 的功能。两台 SQL Server 节点都必须安装。

### 方式一：图形界面安装

#### 1. 打开服务器管理器

在 SQL01 上，点击"开始"菜单 → **"服务器管理器"**。

#### 2. 启动添加角色和功能向导

在服务器管理器右上角，点击 **"管理"** → **"添加角色和功能"**。

#### 3. 开始之前

直接点击 **"下一步"**。

#### 4. 安装类型

选择 **"基于角色或基于功能的安装"**，点击 **"下一步"**。

#### 5. 服务器选择

选择 **"从服务器池中选择服务器"**，确认选中 **SQL01**，点击 **"下一步"**。

#### 6. 服务器角色

**保持全空，什么都不勾选**，直接点击 **"下一步"**。

> ⚠️ 故障转移群集是"功能"，不是"角色"。这里不需要选任何角色。

#### 7. 功能

在功能列表中，找到并勾选：

- ✅ **故障转移群集**（Failover Clustering）

弹出提示框时，点击 **"添加功能"**。

如果需要图形管理工具，同时勾选：

- ✅ **故障转移群集工具**（Failover Cluster Management Tools）

点击 **"下一步"**。

#### 8. 确认安装

核对配置，点击 **"安装"**。等待安装完成，点击 **"关闭"**。

#### 9. 在 SQL02 上重复

在 SQL02 上，把上面的步骤**完整重复一遍**。

![](C:/Users/admin/AppData/Roaming/marktext/images/626602be88ac9236ddf33c1db6db41d13b20f976.png)

### 方式二：PowerShell 安装（推荐）

以**管理员身份**打开 PowerShell，在**两台机器**上分别执行：

```powershell
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
```

---

## 九、<mark>验证 WSFC</mark>

### 1. 打开故障转移群集管理器

在 SQL01 上，点击“开始”菜单 → **“Windows 管理工具”** → **“故障转移群集管理器”**。

### 2. 启动验证向导

在管理器左侧控制台树中，确保选中 **“故障转移群集管理”**。  
在右侧的 **“管理”** 区域下，点击 **“验证配置…”**。

### 3. 添加服务器节点

在“开始之前”页点击“下一步”。  
在 **“选择服务器或群集”** 页面，直接输入服务器名称（如 `SQL01`）后点击 **“添加”**，再输入 `SQL02` 并添加。  
确认两个节点都在下方列表中后，点击“下一步”。

### 4. 选择测试范围

在 **“测试选项”** 页面，**强烈建议选择“运行所有测试(推荐)”**。  
首次搭建时，全量验证最稳妥。点击“下一步”开始测试。

### 5. 查看并分析结果

测试完成后，在 **“摘要”** 页面点击 **“查看报告”**。  
报告文件也会自动保存在 `C:\Windows\Cluster\Reports\` 目录下。

---

### 🔍 如何解读验证结果

| 结果               | 含义         | 处理方式                                                      |
| ------------------ | ------------ | ------------------------------------------------------------- |
| ✅ 绿色对勾         | 测试通过     | 无需处理                                                      |
| ⚠️ 黄色三角（警告） | 通常可以接受 | 没有共享存储时，存储相关警告可忽略，不影响 Always On          |
| ❌ 红色 X（错误）   | **必须解决** | 任何红色 X 都意味着集群无法正常创建，需根据报告修复后重新验证 |

---

### 💡 针对你的环境提醒

因为你搭的是 Always On，重点确保 **“系统配置”** 和 **“网络”** 这两大类测试**没有红色 X**。  
测试可能会跑几分钟，耐心等待即可。

![](C:/Users/admin/AppData/Roaming/marktext/images/e02774b20623e17c66af1a12c8455518f8bbfe99.png)

---

## 十、创建 WSFC 集群

### 1. 打开创建群集向导

在 SQL01 上打开 **故障转移群集管理器**，在右侧"管理"区域点击 **"创建群集…"**。

### 2. 添加服务器节点

在"选择服务器"页面，输入 `SQL01` 后点击"添加"，再输入 `SQL02` 并添加。确认两个节点都在列表中后，点击"下一步"。

### 3. 验证配置（可选但推荐）

向导会询问是否运行验证。如果你在第九步已经完整验证过，可以选择 **"否，我不需要运行验证"**，直接进入下一步。

首次搭建建议选择 **"是"**，让向导重新验证一遍。

### 4. 指定群集名称和 IP

- **群集名称**：输入 `SQLCLUSTER`
- **IP 地址**：为群集分配一个未被占用的静态 IP（例如 `10.0.0.20`，与节点同网段但不同 IP）

> ⚠️ 这个 IP 是集群的管理地址，不是 Listener 的 IP。

### 5. 确认并完成创建

在"确认"页面核对配置，点击"下一步"开始创建。等待创建完成后，点击"完成"。

### 6. 验证群集状态

```powershell
Get-Cluster
Get-ClusterNode
```

---

## 十一、配置文件共享见证

双节点集群属于**偶数节点**，必须配置见证，否则一台服务器宕机时另一台无法自动接管。

### 1. 在文件共享见证服务器上创建共享文件夹

在文件共享见证服务器上（独立于 SQL01 和 SQL02），执行：

```powershell
# 创建文件夹
New-Item -Path "D:\ClusterWitness" -ItemType Directory -Force
```

### 2. 创建 SMB 共享

1. 右键 `D:\ClusterWitness` → **属性** → **共享** → **高级共享**
2. 勾选 **“共享此文件夹”**
3. 点击 **权限**

### 3. 配置共享权限

1. 在“权限”窗口中，**删除**默认的 `Everyone`
2. 点击 **添加** → **对象类型**勾选“计算机” → 输入 `SQLCLUSTER$` → 确定
3. 勾选 **完全控制** → 确定

### 4. 配置 NTFS 权限

1. 回到文件夹属性，切换到 **安全** 选项卡 → **编辑** → **添加**
2. 点击 **对象类型**勾选“计算机” → 输入 `SQLCLUSTER$` → 确定
3. 勾选 **完全控制** → 确定

> ⚠️ **共享权限和 NTFS 权限都必须授予 CNO 完全控制**，缺一不可。CNO 名称格式为 `集群名$`（如 `SQLCLUSTER$`）。

### 5. 在集群中配置仲裁见证

在故障转移群集管理器中：

1. 右键集群名称 → **“更多操作”** → **“配置群集仲裁设置”**
2. 选择 **“选择仲裁见证”**
3. 选择 **“配置文件共享见证”**
4. 输入共享路径：`\\文件共享服务器\ClusterWitness`
5. 完成配置

### 6. 验证见证配置

```powershell
Get-ClusterQuorum
```

预期输出显示已配置见证资源。

---

## 十二、检查 WSFC 状态

```powershell
Get-Cluster
Get-ClusterNode
```

预期输出：

```
SQL01   Up
SQL02   Up
```

---

## 十三、检查 SQL Server 版本

```sql
SELECT
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Edition') AS Edition;
```

确认是否为 Enterprise Edition（完整 AG 功能）。

---

## 十四、启用 Always On

```
SQL Server Configuration Manager
    ↓
SQL Server Services
    ↓
SQL Server (MSSQLSERVER)
    ↓
Properties
    ↓
Always On High Availability
    ↓
勾选 Enable Always On Availability Groups
```

**两台都要做，做完重启 SQL Server 服务。**

---

## 十五、确认 Always On 已启用

```sql
SELECT SERVERPROPERTY('IsHadrEnabled') AS IsHadrEnabled;
```

两台都返回 `1` 才算成功。

---

## 十六、创建数据库镜像 Endpoint

**SQL01 和 SQL02 都执行：**

```sql
CREATE ENDPOINT [Hadr_endpoint]
STATE = STARTED
AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (ROLE = ALL);
GO
```

检查：

```sql
SELECT name, state_desc, role_desc
FROM sys.database_mirroring_endpoints;
```

---

## 十七、开放防火墙 5022

```powershell
New-NetFirewallRule `
    -DisplayName "SQL Server AG 5022" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 5022 `
    -Action Allow
```

测试：

```powershell
Test-NetConnection SQL02 -Port 5022
Test-NetConnection SQL01 -Port 5022
```

确保 `TcpTestSucceeded : True`。

---

## 十八、配置服务账号权限

生产环境建议使用专门的**域账号**（如 `CORP\sqlservice`），不要用 Administrator。

**推荐隔离方案**:同域 + 不同 OU + 不同账号（推荐）

---

## 十九、准备数据库

```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'OrderDB';
```

必须是 `FULL`，否则：

```sql
ALTER DATABASE OrderDB SET RECOVERY FULL;
```

---

## 二十、主库完整备份

```sql
BACKUP DATABASE OrderDB
TO DISK = 'D:\Backup\OrderDB_full.bak'
WITH INIT, COMPRESSION, STATS = 10;

BACKUP LOG OrderDB
TO DISK = 'D:\Backup\OrderDB_log.trn'
WITH INIT, COMPRESSION, STATS = 10;
```

---

## 二十一、复制备份文件

将 `OrderDB_full.bak` 和 `OrderDB_log.trn` 从 SQL01 复制到 SQL02 的 `D:\Backup\`。

---

## 二十二、从库还原数据库

**SQL02 执行：**

```sql
RESTORE DATABASE OrderDB
FROM DISK = 'D:\Backup\OrderDB_full.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

RESTORE LOG OrderDB
FROM DISK = 'D:\Backup\OrderDB_log.trn'
WITH NORECOVERY, STATS = 10;
```

> ⚠️ **不要执行 WITH RECOVERY**，数据库应显示 `Restoring...`。
>
> 因为 AG 加入数据库时，要求从库的数据库处于**“正在还原”状态**
>
> 加入 AG 之后由AG 接管自动变为ONLINE（可读），数据库正式可用

---

## 二十三、创建可用性组

SSMS 连接 SQL01：

```
在 SQL01（主副本）上通过 SSMS 向导创建可用性组。

### 操作前确认

| 检查项 | 要求 |
|--------|------|
| OrderDB 恢复模式 | FULL |
| OrderDB 完整备份 | 已完成 |
| 两台服务器数据库文件路径 | 建议完全一致 |
| SQL02 上 OrderDB 状态 | RESTORING（已用 NORECOVERY 还原） |
| 当前登录账号权限 | sysadmin 或 CREATE AVAILABILITY GROUP 权限 |

### 向导操作步骤

## 向导操作步骤

### 1. 启动向导

在 SSMS 的“对象资源管理器”中，连接到 SQL01。依次展开 “Always On 高可用性” 节点和 “可用性组” 节点，右键点击 “可用性组”，选择 “新建可用性组向导”。

### 2. 指定名称

在“指定可用性组名称”页面，输入你计划好的名称：`ProductionAG`。注意这个名称在域内必须是唯一的。

### 3. 选择数据库

在“选择数据库”页，勾选 `OrderDB`。如果 `OrderDB` 前面有绿色对勾且状态显示“满足先决条件”，说明可以直接进入下一步。

### 4. 配置副本

这是最关键的一步，点击 “添加副本”，连接并输入 `SQL02`。添加后，在“副本”选项卡中，为 SQL01 和 SQL02 分别设置：

- **可用性模式**：同步提交
- **故障转移模式**：自动（这是实现自动切换的关键）

### 5. 数据同步

在“选择初始数据同步”页，因为你已经手动在 SQL02 上用 `NORECOVERY` 还原了备份，所以选择 **“仅联接”**。这样向导就不会重复做备份和还原，直接利用现有状态把数据库拉入 AG。

### 6. 创建侦听器

在“侦听器”选项卡，勾选 “创建可用性组侦听器”。填入你规划好的信息：DNS 名称（如 `SQL-AG`）、端口（`1433`）和静态 IP（如 `10.0.0.20`）。

### 7. 完成向导

完成验证后，点击“完成”，向导会执行创建操作。
```

查看AG的SQL

```
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
```

预期结果：

| AGName       | replica_server_name | role_desc | connected_state_desc | synchronization_health_desc |
| ------------ | ------------------- | --------- | -------------------- | --------------------------- |
| ProductionAG | SQL01               | PRIMARY   | CONNECTED            | HEALTHY                     |
| ProductionAG | SQL02               | SECONDARY | CONNECTED            | HEALTHY                     |

### 检查数据库同步状态

```
SELECT
    DB_NAME(database_id) AS DatabaseName,
    synchronization_state_desc,
    synchronization_health_desc,
    database_state_desc
FROM sys.dm_hadr_database_replica_states;
```

预期结果：

| DatabaseName | synchronization_state_desc | synchronization_health_desc | database_state_desc |
| ------------ | -------------------------- | --------------------------- | ------------------- |
| OrderDB      | SYNCHRONIZED               | HEALTHY                     | ONLINE              |

## 二十四、配置同步模式

| 场景           | 模式                    | 说明                           |
| -------------- | ----------------------- | ------------------------------ |
| 同机房、低延迟 | **Synchronous commit**  | 数据一致性强，支持自动故障转移 |
| 跨机房、高延迟 | **Asynchronous commit** | 性能好，但可能丢数据           |

### SSMS 图形界面操作

1. 在对象资源管理器中，连接到承载主副本的 SQL01
2. 展开 **“Always On 高可用性”** → **“可用性组”**
3. 右键 `ProductionAG` 下的副本（如 SQL02），点击 **“属性”**
4. 在“可用性副本属性”对话框中，将“可用性模式”改为 `同步提交`
5. 如需自动故障转移，将“故障转移模式”改为 `自动`

---

## 二十五、配置故障转移模式

**目标**：SQL01 挂了，SQL02 自动接管。

```text
SQL01: Primary + Synchronous + Automatic Failover
SQL02: Secondary + Synchronous + Automatic Failover
```

### 操作前提

SSMS 已连接到 SQL01（当前的主副本）。可用性组 `ProductionAG` 已创建成功。当前账号有 `sysadmin` 或 `ALTER AVAILABILITY GROUP` 权限。

### 1. 连接 SQL01

打开 SSMS，在“连接到服务器”窗口输入 SQL01 的实例名，用有权限的账号登录。

### 2. 展开 Always On 节点

在左侧“对象资源管理器”中，依次点开：

```
SQL01 (你的实例)
  └── Always On 高可用性
        └── 可用性组
              └── ProductionAG
```

### 3. 展开 ProductionAG，找到副本

展开 `ProductionAG` 后，你会看到：

```
ProductionAG
  ├── 可用性副本
  │     ├── SQL01      ← 主副本
  │     └── SQL02      ← 辅助副本
  ├── 可用性数据库
  │     └── OrderDB
  └── 可用性组侦听器
        └── SQL-AG
```

### 4. 右键 SQL02 副本 → 属性

在“可用性副本”下，**右键 `SQL02`**，选择 **“属性”**。

### 5. 修改 SQL02 的配置

```
弹出“可用性副本属性”对话框后，找到这两个下拉框，分别改为：
可用性模式 → 同步提交（Synchronous commit）
故障转移模式 → 自动（Automatic）
改完点 “确定” 保存。
```

### 6. 对 SQL01 副本重复

```
回到“可用性副本”列表，右键 SQL01，选择 “属性”。
同样把：
可用性模式 → 同步提交
故障转移模式 → 自动
改完点 “确定”。
```

### 操作后验证

在 SQL01 上新建查询，执行：

```sql
SELECT 
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc
FROM sys.availability_replicas ar
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id
WHERE ag.name = 'ProductionAG';
```

预期结果：

| replica_server_name | availability_mode_desc | failover_mode_desc |
| ------------------- | ---------------------- | ------------------ |
| SQL01               | SYNCHRONOUS_COMMIT     | AUTOMATIC          |
| SQL02               | SYNCHRONOUS_COMMIT     | AUTOMATIC          |

### 常见问题

| 问题                       | 原因                       | 解决                                                       |
| -------------------------- | -------------------------- | ---------------------------------------------------------- |
| “故障转移模式”下拉框是灰的 | 可用性模式还没选“同步提交” | 先把可用性模式改成同步提交                                 |
| 改完保存报错               | 辅助副本未同步             | 等 `synchronization_state_desc` 变成 `SYNCHRONIZED` 后再改 |
| 找不到“可用性副本”节点     | AG 未展开或权限不足        | 确认 AG 已创建，账号有 sysadmin 权限                       |

### 一句话总结

在 SSMS 里，右键 SQL02 副本改配置，再右键 SQL01 副本改一遍，两台都设成“同步提交 + 自动”，自动故障转移才会生效。

## 二十六、配置数据同步方式

Wizard 选项：

- **Full**：首次搭建推荐
- **Join only**：已手动还原时使用
- **Skip initial data synchronization**：跳过初始化

---

## 二十七、创建 Listener

```
Listener DNS Name: SQL-AG
端口: 1433
静态 IP: 10.0.0.20
```

> Listener DNS 名称必须在域中唯一，推荐使用静态 IP。

---

## 二十八、最终架构

```
Java
                     │
                     ▼
              SQL-AG.corp.local
                 10.0.0.20
                     │
             ┌───────┴───────┐
             │               │
             ▼               ▼
          SQL01            SQL02
        10.0.0.11         10.0.0.12
         PRIMARY          SECONDARY
             │               ▲
             │               │
             └───────────────┘
                 5022
              日志同步
```

---

## 二十九、Java 应用连接

```properties
# ❌ 错误：直连具体服务器
jdbc:sqlserver://10.0.0.11:1433;databaseName=OrderDB

# ✅ 正确：连接 Listener
jdbc:sqlserver://SQL-AG:1433;databaseName=OrderDB
```

Spring Boot 示例：

```yaml
spring:
  datasource:
    url: jdbc:sqlserver://SQL-AG:1433;databaseName=OrderDB;multiSubnetFailover=true
    username: your_user
    password: your_password
```

---

## 三十、故障转移测试

1. **手动 Failover**：SSMS → Availability Groups → ProductionAG → Failover → 选 SQL02
2. **模拟故障**：`Stop-Service MSSQLSERVER`（在 SQL01 上）
3. **验证 Java 重连**：观察应用是否自动连到 SQL02

---

## 三十一、检查 AG 状态

```sql
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
```

理想状态：`PRIMARY` / `SECONDARY` / `CONNECTED` / `HEALTHY`

---

## 三十二、生产环境监控指标

- AG synchronization state
- AG synchronization health
- Redo Queue
- Log Send Queue
- Transaction Log usage
- Database state
- Replica role
- Connection status
- WSFC cluster health

> **Log Send Queue 持续增大**说明从库跟不上主库。

---

## 三十三、常见故障排查

### ① Secondary 一直不是 SYNCHRONIZED

检查：5022 端口、防火墙、Endpoint、服务账号、DNS、网络。

### ② SQL02 连不上 SQL01

```powershell
Test-NetConnection SQL01 -Port 5022
```

### ③ Listener 创建失败

检查：DNS、AD 权限、Cluster 权限、Listener IP 是否冲突。

---

## 三十四、重要提醒

> **Always On ≠ 备份。**
>
> 即使搭好了 AG，仍然必须做：Full Backup、Differential Backup、Transaction Log Backup，并配置异地备份、备份保留策略和恢复测试。

---

## 三十五、完整执行 Checklist

- [ ] ① 最终架构
- [ ] ② 准备工作
- [ ] ③ 准备 Active Directory 域
- [ ] ④ 设置主机名并加域
- [ ] ⑤ 网络连通性测试
- [ ] ⑥ 安装 SQL Server
- [ ] ⑦ 安装 SSMS
- [ ] ⑧ 安装 WSFC 功能
- [ ] ⑨ 验证 WSFC
- [ ] ⑩ 创建 WSFC 集群
- [ ] ⑪ 配置文件共享见证
- [ ] ⑫ 检查 WSFC 状态
- [ ] ⑬ 检查 SQL Server 版本
- [ ] ⑭ 启用 Always On
- [ ] ⑮ 确认 Always On 已启用
- [ ] ⑯ 创建数据库镜像 Endpoint
- [ ] ⑰ 开放防火墙 5022
- [ ] ⑱ 配置服务账号权限
- [ ] ⑲ 准备数据库
- [ ] ⑳ 主库完整备份 
- [ ] ㉑ 复制备份文件
- [ ] ㉒ 从库还原数据库
- [ ] ㉓ 创建可用性组
- [ ] ㉔ 配置同步模式
- [ ] ㉕ 配置故障转移模式
- [ ] ㉖ 配置数据同步方式
- [ ] ㉗ 创建 Listener
- [ ] ㉘ 最终架构
- [ ] ㉙ Java 应用连接
- [ ] ㉚ 故障转移测试
- [ ] ㉛ 检查 AG 状态
- [ ] ㉜ 生产环境监控指标
- [ ] ㉝ 常见故障排查
- [ ] ㉞ 重要提醒
- [ ] ㉟ 完整执行 Checklist

---

> **文档结束**
