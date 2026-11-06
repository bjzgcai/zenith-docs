---
title: CLI 与 Agent Skills 工作流
description: 安装并安全使用 Zenith CLI 和 Agent Skills，完成查询、开发机操作、存储、SSH 与操作恢复。
---

# CLI 与 Agent Skills 工作流

Zenith CLI 是面向脚本和自动化的个人 API 客户端。Agent Skills 使用同一个 CLI，帮助支持的 Agent 按照相同的权限和恢复规则执行任务。本页覆盖当前已验证的个人工作流；平台管理员、项目管理员和厂商 API 不会因为安装 CLI 而获得额外权限。

## 开始前准备

- 使用 **Node.js 24** 和 npm 11（随 Node.js 24 提供）。当前只验证 macOS 和 Linux；Windows 尚未验证。
- 从有权限访问的私有 `cli-v<版本>` 客户端 Release 下载同一版本的 CLI、Skills、清单、安装说明、NOTICE 和 SHA-256 文件。
- 在控制台 **账号设置 → API Token** 新建个人 Token。Token 只显示一次，默认只读且有效期 90 天，最长 365 天；按下方命令表选择最小范围，写操作还需要相应的动作范围。
- 准备一个受信任的 HTTPS 平台地址。不要把 `/pt/api` 加到 origin，也不要在命令参数、聊天消息或代码中写入 Token。

下载后先在空目录校验完整的客户端文件集。以下以 `0.1.0-dev.1` 为例，请替换为实际下载的完整文件名；使用本人可写的 npm prefix，无需以系统管理员身份运行终端：

```bash
# Linux
sha256sum --check zenith-cli-SHA256SUMS
# macOS
shasum -a 256 --check zenith-cli-SHA256SUMS
node --version
npm --version
npm install --global --prefix "$HOME/.local" --engine-strict --ignore-scripts ./zenith-cli-0.1.0-dev.1.tgz
export PATH="$HOME/.local/bin:$PATH"
zenith --version
zenith --help
zenith api operations --output json
```

后续终端会话也需要将该 prefix 的 `bin` 目录加入 `PATH`。

CLI 和 Skills 是独立归档，版本、API 版本和契约摘要必须来自同一次构建。安装 CLI 不会自动注册 Skills；请按[Skills 安装说明](https://github.com/bjzgcai/Zenith/blob/main/skills/INSTALL.md)将解压后的 `zenith/` 目录复制到项目 `.agents/skills/zenith` 或个人 `~/.agents/skills/zenith`。保留已有目录中的本地修改。Claude Code 的安装路径已记录，但执行效果尚未验证。

升级前保存当前完整文件集，升级时同时安装匹配的 CLI 与 Skills。回退时重新校验并安装保存的 CLI 归档、恢复同版本 Skills，保留本地 Skills 修改和未决操作记录。普通 CI 只验证包，不提供预览下载；客户端 Release 与平台部署版本独立。

## 配置 profile 和 Token

Profile 保存平台 origin 和本地凭据选择；它不是新的权限边界。先创建 profile，再通过隐藏提示或标准输入登录：

```bash
zenith profiles create lab --origin https://zenith.example.com/pt/api2
zenith --profile lab auth login
zenith --profile lab auth status --output json
zenith profiles use lab
```

自动化环境可使用安全注入的 `ZENITH_API_TOKEN`，或执行 `auth login --token-stdin --no-input` 从标准输入读取。CLI 只接受控制台生成的 `znt_` 个人 Token，会在发送请求前拒绝浏览器会话 JWT 和平台管理员凭据。`auth logout` 只删除本地 profile 凭据；轮换或彻底撤销仍需回到控制台：创建新 Token → 更新安全凭据 → 用最小查询验证 → 停用旧 Token → 确认无遗漏后删除旧 Token。

`--profile` 优先于 `ZENITH_PROFILE` 和保存的默认 profile；`ZENITH_API_TOKEN` 优先于本地凭据且不会写入凭据文件。轮换时要同时更新自动化环境中的值，避免新凭据被旧环境变量遮盖。`profiles list` 不输出凭据。本地登录凭据是仅本人可访问的目录（`0700`）中的明文文件（`0600`），应保存在本人受限目录。

Token 范围、资源所有权、项目成员关系、配额和厂商能力由 Platform 在每次请求中重新判断。平台管理员创建的个人 Token 也不会继承平台或项目管理权限。

## 查询、资源区与 JSON 脚本

一次操作先读取当前身份、范围与额度，再选择确切目标：

```bash
zenith capabilities --output json
zenith whoami --output json
zenith quota show --output json
zenith card-hours show --output json
zenith projects list --output json
zenith offerings list --output json
zenith offerings images "$OFFERING_ID" --workload DEV_INSTANCE --output json
zenith instances list --all --output json
```

从结果中选择并设置后续示例的 `PROJECT_ID`、`OFFERING_ID`、`VENDOR` 和 `PROVIDER_ID`。开发机目标使用同一行的 `vendor` 与 `vendor_instance_id`，不能用显示名称或 Platform 内部 `id` 代替 provider ID。`--scope projects` 可显式读取本人及已加入项目可见的机器，但不增加操作或 SSH 权限；开发机操作和 SSH 仍限本人所有。

API **2.11.0** 可先从 `capabilities` 的 `data.resource_zones` 读取资源区 `id`、`label` 和 `storage_scope`（`resource-zone:<id>`）。这个目录不含厂商字段，也不代表已分配规格或可创建资源；已有厂商坐标仍保留兼容。不同资源区的文件相互独立，同一个 `storage_scope` 也不保证区内所有资源池共用文件系统，选择资源区不会自动迁移文件。

资源区编号沿用控制台公开命名。下表是资源区与 API 中 `vendor` 字段的兼容映射，只有 Platform 返回明确 provider 区域时才使用对应关系；不要从 `vendor` 值或存储路径猜测具体区域：

| 控制台资源区 | API 中的 `vendor` 值 |
| --- | --- |
| 资源区 1–4 | `ctyun`；仅在实际 provider 区域可归属时使用对应编号 |
| 资源区 5 | `volcengine` |
| 资源区 6 | `lanqi` |
| 资源区 10 | `baidu` |

`--resource-zone` 用于 `offerings admission` 的明确选择，并非所有命令通用的筛选参数：

```bash
zenith offerings admission "$OFFERING_ID" --workload instance \
  --project-id "$PROJECT_ID" --resource-zone 5 --execution-class standard --output json
```

准入预检返回 `allowed`、`blocked` 或 `unknown`，并保留原因与 `retryable`；它不预留容量。有效编号仍可能没有匹配路由、在线工作节点或可用额度。收到 `blocked` / `unknown` 时不要擅自换区域或优先级。`standard` 和 `low_priority` 是当前有效优先级，旧值 `normal` 会被拒绝；还要核对所选规格在该资源区支持的执行类别。存储清单没有已核实的资源区信息，CLI 不会补造映射。

读取规格的 `capabilities_by_resource_zone` 和开发机的 `capabilities` 时，保留 `worker_status`（`online` / `draining` / `offline`）与可空的操作标志。标志 `true` 只表示技术支持，`false` 表示不支持，`null` 或缺失表示未知，均不授予权限或保证容量。规格只列出存在路由的资源区；某区缺少一种工作负载路由时，该类型创建标志为 `false`、执行类别为空数组。执行类别在工作节点离线时仍可能有值，不能用它推断在线。规格的 `resize` 也不证明已有开发机能改配到它。任务能力可通过个人 HTTP API 读取，CLI 任务读取命令仍在规划中。

准入结果的 `data.routes[].storage` 是存储配置预览：个人/公共可用标志和路径说明配置，`project_supported` 与路径模板只说明项目存储支持，不证明当前项目已有授权绑定。路径和 `storage_scope` 可能为 `null`，应保留未知；`mount_verified=false`、`preview_only=true`，`unchecked` 数组仍包含 `storage_mounts`。预检不创建文件、挂载或预留资源，有路径也不能证明实际挂载成功。字段说明见[个人 API 与 AI 接入](/reference/personal-api)。

脚本使用 `--output json`，不要解析表格、品牌标志或进度文字。JSON 格式版本为 `schema_version: 1`：

| `kind` | 应读取的字段 |
| --- | --- |
| `success` | `data` 中的查询结果 |
| `operation` | `operation` 中的回执；开发机操作另看 `operation.verification` |
| `error` | `error.code`、可选 `status` / `retry_after`，以及可能保留的 `operation` |

例如空存储列表是成功结果，不是权限失败：

```json
{"schema_version":1,"kind":"success","data":{"disks":[]}}
```

JSON 结果（包括错误）写入标准输出；同时检查进程退出码。退出 0 表示命令完成，仍须核对回执状态、准入结果和 `effect_verified`。退出 2 为语法错误或未实现，3 为认证失败，4 为权限不足，5 为冲突，6 为请求/网络/超时等错误，7 为待定或 `UNKNOWN`，130 为本地中断，1 为内部错误。不得仅因得到可解析 JSON 就判定成功。

开发机列表默认每页 50 条，`--all` 仍受 100 页、10,000 条上限限制。读取 `data.pagination.complete`、`truncated` 与 `next_page`；达到上限不等于完整清单。列表只包含状态为“平台管理”、未删除、未过期的观测记录，保留 `last_seen_at` 和 `stale`；完整分页也不代表实时完整云端库存。镜像查询没有可靠总数，分页完整性可能为 `null`，不能把未知当作完整。

## 开发机启停、重启与改配

先核对目标所有权、当前状态、配额和所选规格。每次新的授权操作使用新的 intent，同一次意图的恢复必须保持原 intent 和参数；写操作要求显式 `--yes`，不会再弹出确认提示：

```bash
zenith instances start "$VENDOR" "$PROVIDER_ID" \
  --intent start-dev-session --yes --wait --timeout-ms 90000 --output json
zenith instances stop "$VENDOR" "$PROVIDER_ID" \
  --intent stop-dev-session --yes --wait --timeout-ms 90000 --output json
zenith instances restart "$VENDOR" "$PROVIDER_ID" \
  --intent restart-dev-session --yes --wait --timeout-ms 90000 --output json
zenith instances resize "$VENDOR" "$PROVIDER_ID" \
  --offering-id "$OFFERING_ID" --intent resize-dev-session \
  --yes --wait --timeout-ms 90000 --output json
```

以上是分别执行的操作示例，不是要求连续运行的脚本。停止、重启会中断进程，改配也可能中断服务。改配只接受已分配的规格，不会自动替你执行停机再开机。Platform 在请求时重新判断权限、配额、卡时与厂商资格。

先查看改配计划时使用：

```bash
zenith instances resize "$VENDOR" "$PROVIDER_ID" \
  --offering-id "$OFFERING_ID" --dry-run --output json
```

`--dry-run` 仅做允许的读取，不需要 `--yes` 或 `--intent`，不能与 `--wait` 共用；它不锁定容量或保证最终准入。每个子命令的 `--help` 提供当前安装版本的精确语法。

`--wait` 的总时限包含前置读取、提交/恢复、回执轮询和新观测确认，默认 30 秒。成功回执后，CLI 要求比首次成功后观测更新、且 `stale=false` 的状态，才确认启动或停止效果。重启只能确认更新的 `RUNNING` 观测，不能证明重启过程；恢复的改配无法从旧回执重建原 GPU 规格，两者均可能返回 `verification.status=limited` 和 `effect_verified=false`。回执成功不证明机器已可 SSH 或应用已健康。

## 命令、范围与效果速查

### 原始 25 项的当前状态

最初的 25 项包含两项手动挂载操作；它们已经撤出个人 API，不能为了完成旧清单恢复 Token 权限。下面逐项列出其当前状态。已实现行的范围包含 CLI 前置读取与回执恢复所需的最小范围，`+` 表示都需要；有效 Token、当前账号角色、所有权、项目授权和准入检查仍必须满足。

| operationId | CLI 命令 | Token 范围 | 效果 |
| --- | --- | --- | --- |
| `getCapabilities` | `zenith capabilities` | 无额外 scope（仍需 Token） | 读取身份、Token 范围、能力与契约版本；需有效 Token。 |
| `getMyProfile` | `zenith whoami` | `profile:read` | 读取本人资料。 |
| `getMyQuotaUsage` | `zenith quota show` | `profile:read` | 读取有效配额与用量，不增加额度。 |
| `getMyCardHours` | `zenith card-hours show` | `profile:read` | 读取卡时及扣费来源，不是现金余额。 |
| `listMyProjects` | `zenith projects list` | `projects:read` | 读取本人项目列表。 |
| `getMyProject` | `zenith projects get` | `projects:read` | 按稳定项目 ID 读取详情。 |
| `listComputeOfferings` | `zenith offerings list` | `catalog:read` | 读取平台管理员已分配规格，不预留容量。 |
| `listDevMachines` | `zenith instances list` | `instances:read` | 读取本人或明确项目可见的开发机观测列表。 |
| `getDevMachine` | `zenith instances get` | `instances:read` | 按厂商和 provider ID 读取开发机观测。 |
| `startDevMachine` | `zenith instances start` | `instances:start` + `instances:read` + `operations:read` | 启动本人开发机，仍需通过准入及卡时检查。 |
| `stopDevMachine` | `zenith instances stop` | `instances:stop` + `instances:read` + `operations:read` | 停止本人开发机，会中断工作负载。 |
| `restartDevMachine` | `zenith instances restart` | `instances:restart` + `instances:read` + `operations:read` | 重启本人开发机，会中断工作负载。 |
| `resizeDevMachine` | `zenith instances resize` | `instances:resize` + `instances:read` + `operations:read` + `catalog:read` | 变更为已分配规格，可能中断服务；不会自动增加停启步骤。 |
| `getMyProjectStoragePaths` | `zenith projects storage-paths` | `storage:read` | 仅 typed HTTP API；CLI 命令尚未实现。 |
| `setMyProjectStoragePath` | `zenith projects set-storage-path` | `storage:configure` | 仅 typed HTTP API；CLI 命令尚未实现。 |
| `listMyMountDisks` | `zenith storage list` | `storage:read` | 读取已分配逻辑磁盘，不改变挂载或机器状态。 |
| `attachStorageAndRestart` | 已撤出个人 API，无 CLI 命令 | `storage:mount` 已退役 | 仅平台管理员控制台可调整存储并重启；会中断进程。 |
| `configureStoppedInstanceMounts` | 已撤出个人 API，无 CLI 命令 | `storage:mount` 已退役 | 仅平台管理员控制台可配置已停止开发机的挂载；提交后保持停机。 |
| `getMySshAccess` | `zenith ssh info` | `ssh:read` | 读取本人的结构化 SSH 信息，不建立连接。 |
| `listMyPublicKeys` | `zenith ssh keys list` | `ssh-keys:read` | 读取本人公钥元数据和指纹。 |
| `registerMyPublicKey` | `zenith ssh keys import` | `ssh-keys:write` + `ssh-keys:read` | 核对已有指纹后登记本地公钥，不注入机器。 |
| `getMyPublicKey` | `zenith ssh keys get` | `ssh-keys:read` | 读取一条本人公钥记录。 |
| `deleteMyPublicKeyRegistration` | `zenith ssh keys remove` | `ssh-keys:write` | 删除平台公钥登记；已安装公钥和已有 SSH 会话保留。 |
| `requestMyPublicKeyInjection` | `zenith ssh inject` | `ssh:inject` + `ssh-keys:read` | 请求异步补注入本人公钥；使用单独状态接口。 |
| `getMyPublicKeyInjectionStatus` | `zenith ssh injection-status` | `ssh-keys:read` | 读取公钥注入状态，不再次请求注入。 |
| `getMyOperation` | `zenith operations get` | `operations:read` | 读取本人操作回执，不创建操作。 |
| `findMyOperationByKey` | `zenith operations recover` | `operations:read` | 用保存的 intent 或原键找回回执，不再次提交。 |

`zenith operations wait` 复用回执读取，`zenith ssh connect` 复用 SSH 信息并启动本地 OpenSSH，均不增加一个 API operation。

### 后续增加的 42 项

当前 API **3.18.0** 的认证操作总数为 **65 = 25 − 2 + 42**，其中 **32 项已有 CLI 实现，33 项规划中**。下表逐项区分后续扩展；规划中行的范围仅说明 HTTP 契约，CLI 返回 `NOT_IMPLEMENTED` 且不发送请求。生成的 HTTP 客户端支持某接口不等于 CLI 已实现该命令。

| operationId | CLI 命令 | Token 范围 | 效果 |
| --- | --- | --- | --- |
| `listOfferingImages` | `zenith offerings images` | `catalog:read` | 已实现；读取规格镜像，分页完整性可能未知。 |
| `listMyBucketMounts` | `zenith storage bucket-mounts` | `storage:read` | 规划中；HTTP 可列出当前用户授权的项目存储桶挂载（资源区5、资源区10），CLI 不发送请求。 |
| `getMyInstanceMounts` | `zenith instances mount-info` | `instances:read`、`storage:read` | 规划中；HTTP 可读取自己的开发机内挂载目录，CLI 不发送请求。 |
| `getMyInstanceMountsById` | `zenith instances mount-info-by-id` | `instances:read`、`storage:read` | 规划中；通过平台实例 ID 读取挂载目录，CLI 不发送请求。 |
| `listDatasets` | `zenith datasets list` | `datasets:read` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `getDataset` | `zenith datasets get` | `datasets:read` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `listDatasetFiles` | `zenith datasets files` | `datasets:read` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `readDatasetFile` | `zenith datasets read` | `datasets:read` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `writeDatasetFile` | `zenith datasets write` | `datasets:write` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `createDatasetDirectory` | `zenith datasets mkdir` | `datasets:write` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `renameDatasetFile` | `zenith datasets rename` | `datasets:write` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `deleteDatasetFile` | `zenith datasets delete` | `datasets:write` | 规划中；HTTP 数据集接口已可用，CLI 不发送请求。 |
| `createDevMachine` | `zenith instances create` | `instances:create` | 规划中；HTTP 可创建并开机，CLI 不发送请求。 |
| `createTrainingJob` | `zenith jobs create` | `jobs:create` | 规划中；HTTP 可提交训练，CLI 不发送请求。 |
| `listTrainingJobs` | `zenith jobs list` | `jobs:read` | 规划中；HTTP 可列出本人任务，CLI 不发送请求。 |
| `getTrainingJob` | `zenith jobs get` | `jobs:read` | 规划中；HTTP 可读取本人任务，CLI 不发送请求。 |
| `cancelTrainingJob` | `zenith jobs cancel` | `jobs:cancel` + `operations:read` | 已实现；取消本人任务，保留历史、日志与存储。 |
| `cancelTrainingJobCreation` | `zenith jobs cancel-creation` | `jobs:cancel` + `operations:read` | 已实现；对尚未返回任务 ID 的创建操作持久化取消意图。 |
| `previewOfferingAdmission` | `zenith offerings admission` | `catalog:read` | 已实现；只读准入预检，不锁卡、不创建资源。 |
| `getTrainingJobLogs` | `zenith jobs logs` | `jobs:read` | 规划中；HTTP 可读取本人任务日志，CLI 不发送请求。 |
| `getTrainingMetrics` | `zenith jobs metrics`（规划中） | `jobs:read` |
| `getTrainingMetricsById` | `zenith jobs metrics-by-id`（规划中） | `jobs:read` |
| `getTrainingJobReplicas` | `zenith jobs replicas` | `jobs:read` | 规划中；HTTP 可读取本人任务副本诊断，CLI 不发送请求。 |
| `listMyObjectSources` | `zenith storage sources` | `storage:read` | 已实现；可按项目筛选已授权来源。 |
| `listMyObjects` | `zenith storage objects` | `storage:read` | 已实现；按来源列出一页相对对象，支持游标。 |
| `presignMyObjectUpload` | `zenith storage upload-presign` | `storage:write` | 已实现；需 --yes，将上传 URL 写入新建私有文件，不传文件。 |
| `presignMyObjectDownload` | `zenith storage download-presign` | `storage:read` | 已实现；将下载 URL 写入新建私有文件，不传文件。 |
| `issueMyBucketSts` | `zenith storage sts` | `storage:read` + `storage:write` + `storage:sts` | 已实现；整桶读写授权，需 --yes；凭据写入新建 0600 文件，不输出明文。 |
| `getDevMachineById` | `zenith instances get-by-id` | `instances:read` | 规划中；HTTP 可按 Platform ID 读取开发机。 |
| `startDevMachineById` | `zenith instances start-by-id` | `instances:start` | 规划中；HTTP 可按 Platform ID 启动本人开发机。 |
| `stopDevMachineById` | `zenith instances stop-by-id` | `instances:stop` | 规划中；HTTP 可按 Platform ID 停止本人开发机。 |
| `restartDevMachineById` | `zenith instances restart-by-id` | `instances:restart` | 规划中；HTTP 可按 Platform ID 重启本人开发机。 |
| `resizeDevMachineById` | `zenith instances resize-by-id` | `instances:resize` | 规划中；HTTP 可按 Platform ID 改配本人开发机。 |
| `getMySshAccessById` | `zenith ssh info-by-id` | `ssh:read` | 规划中；HTTP 可按 Platform ID 读取本人 SSH 信息。 |
| `requestMyPublicKeyInjectionById` | `zenith ssh inject-by-id` | `ssh:inject` | 规划中；HTTP 可按 Platform ID 请求本人公钥补注入。 |
| `getMyPublicKeyInjectionStatusById` | `zenith ssh injection-status-by-id` | `ssh-keys:read` | 规划中；HTTP 可按 Platform ID 读取注入状态。 |
| `getTrainingJobById` | `zenith jobs get-by-id` | `jobs:read` | 规划中；HTTP 可按 Platform job ID 读取本人任务。 |
| `getTrainingJobLogsById` | `zenith jobs logs-by-id` | `jobs:read` | 规划中；HTTP 可按 Platform job ID 读取本人日志。 |
| `getTrainingJobReplicasById` | `zenith jobs replicas-by-id` | `jobs:read` | 规划中；HTTP 可按 Platform job ID 读取本人副本诊断。 |
| `cancelTrainingJobById` | `zenith jobs cancel-by-id` | `jobs:cancel` | 规划中；HTTP 可按 Platform job ID 取消本人任务。 |

取消命令的最小范围为 `jobs:cancel` + `operations:read`；另授予 `jobs:read` 才能通过[个人 API](/reference/personal-api)查询任务实际状态。当前 `jobs get` 尚未实现。`--wait` 只等取消回执，不能证明训练已经停止。

### 单独的匿名文档发现

`capabilities` 始终需要有效个人 Token。匿名发现仅指 OpenAPI 文档和站点维护状态，不计入上述 65 项：

```bash
zenith api schema --output json
zenith --profile lab api schema --remote --output json
zenith api operations --output json
```

`api schema` 默认离线读取内置契约；`--remote` 从所选 profile 的 HTTPS origin 匿名读取 `/pt/api2/openapi.json`，不发送 Token，也不会替换内置契约。`api operations` 离线列出当前安装包的命令、scope 和实现状态。升级后以该清单及子命令帮助核对安装版本，并比较平台能力中的 `api_version` / `schema_sha256`，不要假定不同版本兼容。

## 存储与两种挂载边界

普通个人工作流可以查看已分配的逻辑挂载盘：

```bash
zenith storage list --output json
```

这只读取 `storage:read` 范围下的分配结果，返回逻辑磁盘 ID、容器路径和授权来源。它不会证明某台机器已经挂载，也不会泄露物理存储根或猜测资源区。

当前两种改变挂载的场景均通过控制台会话操作：本人可调整同资源区的项目及公共存储，个人盘和自定义路径由平台管理员操作，可从开发机列表的“调整存储挂载”进入（见[开发机管理](/compute/instances)）：

- **重启后挂载**会在存储变更后重启开发机；
- **停机配置**只调整已停止开发机的挂载，提交后保持停机。

个人 Token（包括平台管理员创建的 Token）不能调用这两类管理接口；CLI 不提供手工挂载、原始存储单元、任意路径或 provider 文件系统 ID 的绕过命令。资源区1–4 个人存储和项目存储的专属空间、硬配额和正式挂载契约仍在等待云端确认，不能从分配列表推断个人/项目隔离已就绪；自动个人存储仍按限制阻止挂载，不能回退挂载整个底层存储根。后续进展跟踪 [#31](https://github.com/bjzgcai/Zenith/issues/31)。

## SSH 工作流

私钥始终在本地生成和保存，只上传公钥。先选择尚未使用的密钥文件名，按提示设置口令，保留已有密钥：

```bash
ssh-keygen -t ed25519 -f "$HOME/.ssh/zenith_work"
zenith ssh keys import --public-key-file "$HOME/.ssh/zenith_work.pub" \
  --name work-laptop --output json
zenith ssh inject "$VENDOR" "$PROVIDER_ID" --yes --wait --timeout-ms 90000 --output json
zenith ssh injection-status "$VENDOR" "$PROVIDER_ID" --wait --timeout-ms 90000 --output json
zenith ssh info "$VENDOR" "$PROVIDER_ID" --output json
zenith ssh connect "$VENDOR" "$PROVIDER_ID" \
  --identity-file "$HOME/.ssh/zenith_work"
```

项目成员可见的机器不等于有权登录。`ssh inject` 使用独立状态 URL；超时或中断后先查询 `ssh injection-status`，不要盲目再次提交。`ssh connect` 保留 OpenSSH known-hosts 校验，只使用结构化端点字段，不执行厂商返回的命令或备注。接受新主机前应通过独立可信渠道核对目标与跳板指纹，主机密钥变化会被拒绝。脚本请使用 `ssh info --output json` 并将连接信息作为私密数据；交互式连接需要终端和本地 OpenSSH，不能与 JSON 或 `--no-input` 共用。

删除平台公钥登记、`auth logout` 或撤销 API Token 都不会移除机器已安装的公钥，也不会结束已有 SSH 会话。需要回收登录权限时须另行清理机器授权公钥。

## 回执恢复

网络超时、响应丢失或 `UNKNOWN` 时，保留同一个 intent 和原始幂等键：

```bash
zenith operations recover --intent start-dev-session --wait --output json
zenith operations get "$OPERATION_ID" --output json
zenith operations wait "$OPERATION_ID" --timeout-ms 120000 --output json
```

重复原开发机命令、原参数和原 intent 可恢复原操作并继续观察效果；`operations recover` 只恢复回执，不核实机器状态。记录绑定平台 origin 和实际账号，同一账号更换 Token 后仍可恢复；切换账号或平台不能读取其他上下文的记录。

不能换一个新 intent 重复提交。`UNKNOWN` 只表示结果待核实；即使后来找不到回执，也不能据此认定原操作未执行。超时和 Ctrl-C 只结束本地等待，不会取消远端操作。等待通常每 5 秒检查一次，并遵循服务端 `Retry-After`。

确认操作已处于 `SUCCEEDED` 或 `FAILED` 且无需恢复后，才可以 `operations forget --intent NAME` 清理本地记录。清理不会删除远端资源，但再次使用同名 intent 会生成新键，可能再次执行。取消任务同样需要新的专用 intent，并且只允许取消本人任务。

## 当前范围和明确未实现项

API 2.14.0 has 50 authenticated operations: 32 have CLI bindings and 18 are planned. The planned
bindings are instance creation; job creation/list/detail/logs/replicas;
eight Platform instance-ID aliases; and four Platform job-ID aliases. Generated HTTP types do not
make these CLI commands available. Existing instance, SSH and job-cancellation commands still
use vendor/provider coordinates. Alias commands return `NOT_IMPLEMENTED` without a request.

Instance deletion, raw jobs/logs routes, arbitrary files or migration, administrator and token
management, browser terminals, raw vendor APIs and manual mounts remain outside this CLI preview.
Follow [CLI Epic #50](https://github.com/bjzgcai/Zenith/issues/50) for staged expansion. Use an
authorized console entry or the documented personal HTTP contract; do not construct private routes.

完整命令表、权限范围、退出码和契约细节见[个人 API 与 AI 接入](/reference/personal-api)；安装、归档内容、回滚和 Skills 主机路径见仓库的 [CLI 安装说明](https://github.com/bjzgcai/Zenith/blob/main/cli/INSTALL.md) 和 [Skills 安装说明](https://github.com/bjzgcai/Zenith/blob/main/skills/INSTALL.md)。


## 查看我的项目并申请存储凭据

```sh
zenith projects list --output json
zenith projects get "$PROJECT_ID" --output json
zenith storage sources --project-id "$PROJECT_ID" --output json
zenith storage objects --source-id "$SOURCE_ID" --prefix checkpoints --output json
zenith storage sts --source-id "$SOURCE_ID" --duration-seconds 900 --credentials-file ./sts.json --yes --output json
```

使用项目返回的稳定 ID，及该项目来源返回的 `id`；不要用名称或物理桶名替代。
Token 需 `projects:read` 读取我的项目，`storage sts` 签发临时凭据还需三项存储范围和现有整桶读写授权。
凭据仅写入新建 0600 文件，不进入终端输出。不要将文件内容粘贴到 AI 对话。
默认有效期 900 秒，最长 43200 秒，Token 需额外预留 60 秒有效期；撤销 Token 不会
立即撤销已签发云凭据。签发不会自动重试。
