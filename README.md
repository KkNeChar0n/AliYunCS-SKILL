# AliYunCS-SKILL

阿里云云效（Yunxiao）Claude Code Skills 套件。用斜杠命令管理云效项目的迭代、需求、任务，**并和飞书 PRD 文档双向联动**：在云效创建需求时自动建飞书 wiki 节点，PRD 写完后飞书 URL 自动回写到云效需求描述。

> 配套使用：[PRD-SKILL](https://github.com/KkNeChar0n/PRD-SKILL)（提供 `/prd-feishu` 把 PRD 写入飞书文档）。两个 skill 套件配合，可以做到「云效创建需求 → 写飞书 PRD → URL 自动回填云效」全程不重复输入。

## Skills 列表

| 命令 | 功能 |
|------|------|
| `/aliyuncs:init` | 初始化项目配置（组织 ID、项目标识、AccessKey），生成项目级 `.aliyuncs.json` |
| `/aliyuncs:create-iteration` | 创建迭代，命名规则 `项目名称 V1.X.0`，自动递增版本号；**可选填飞书迭代父节点 URL 启用联动** |
| `/aliyuncs:create-requirement` | 根据对话内容创建产品需求，关联到当前迭代；**配置了飞书联动后自动在飞书父节点下建空 docx 子节点** |
| `/aliyuncs:create-task` | 根据需求拆解研发任务，自动创建子任务（前端 / 后端 / 测试） |
| `/aliyuncs:update` | 批量查询和更新需求、任务、子任务的状态；支持改 `status` / `subject` / `description` 字段 |

## 工作流

### 基本工作流（仅云效）

```
init
 └─ create-iteration       # 创建迭代
     └─ create-requirement # 创建需求（多个）
         └─ create-task    # 拆任务（多个）
             └─ update     # 推进状态（贯穿整个迭代）
```

### 扩展工作流（云效 ↔ 飞书 PRD 联动）

需要同时安装 [PRD-SKILL](https://github.com/KkNeChar0n/PRD-SKILL)，并跑过 `/prd-feishu-init` 配好飞书自建应用凭证。

```
1. /aliyuncs:init                          # 仅首次
2. /aliyuncs:create-iteration              # 顺便填一次「飞书迭代父节点 URL」
3. /aliyuncs:create-requirement <subject>  # 自动在飞书父节点下建空 docx 子节点 + 把 URL 存进 .aliyuncs.json
4. /prd-feishu <subject>                   # 写 PRD 到飞书节点 + URL 自动回写云效需求描述顶部
5. /aliyuncs:create-task                   # 按需拆任务
6. /aliyuncs:update                        # 推状态
```

整套流程跑下来：
- 飞书 URL 只手动给过一次（`create-iteration` 时的父节点 URL）
- 云效需求 ID 全程零输入（通过 subject 模糊匹配）
- 飞书空文档零手动创建（`create-requirement` 自动建）
- 云效 PRD 链接零手动填（`prd-feishu` 自动写）

## 安装

```bash
git clone git@github.com:KkNeChar0n/AliYunCS-SKILL.git
cd AliYunCS-SKILL
cp -r aliyuncs:* ~/.claude/skills/
```

带 `:` 的目录名需要保留（Claude Code 用它作为 skill 命名空间）。

## 初始化

```
/aliyuncs:init
```

按引导依次填入：

| 字段 | 说明 | 在哪找 |
|------|------|--------|
| `organizationId` | 云效组织 ID | 云效任意页面 URL `https://devops.aliyun.com/projex/project/{spaceIdentifier}/...?organizationId=xxx` 里的 `organizationId` |
| `spaceIdentifier` | 云效项目唯一标识 | 同上 URL 的 `/project/{spaceIdentifier}/` 段 |
| `projectName` | 项目显示名称 | 用于迭代命名（`{projectName} V1.X.0`） |
| `accessKeyId` | 阿里云 AccessKey ID | [AccessKey 管理](https://ram.console.aliyun.com/manage/ak) |
| `accessKeySecret` | AccessKey Secret | 同上，**仅创建时显示一次** |

skill 会自动调 `ListOrganizationMembers` 取第一个成员的 `accountId` 作为 `staffAccountId`（创建迭代和需求时作为「负责人」必填字段），写入 `.aliyuncs.json`，并把该文件加入 `.gitignore`。

## `.aliyuncs.json` 配置文件

项目根目录下的状态文件。包含密钥，**已自动加入 `.gitignore`**。

```json
{
  "organizationId": "xxxx",
  "spaceIdentifier": "xxxx",
  "projectName": "证书自动化SaaS",
  "accessKeyId": "LTAI...",
  "accessKeySecret": "...",
  "apiBase": "https://devops.aliyun.com",
  "staffAccountId": "205243275789820602",

  "currentIteration": {
    "identifier": "97c02a3ecdfe03fc72caa846d3",
    "name": "证书自动化SaaS V1.1.0",
    "status": "TODO",
    "feishu_parent_url": "https://xxx.feishu.cn/wiki/{token}",
    "feishu_parent_node_token": "{token}",
    "feishu_space_id": "{space_id}"
  },

  "requirements": [
    {
      "identifier": "工作项ID",
      "subject": "需求标题",
      "status": "待处理",
      "feishu_url": "https://xxx.feishu.cn/wiki/{node_token}",
      "tasks": [
        { "identifier": "任务ID", "subject": "任务标题", "status": "处理中", "subtasks": [...] }
      ]
    }
  ]
}
```

`currentIteration.feishu_*` 三个字段是可选的，配了才会启用飞书联动；每个 `requirement.feishu_url` 是被 `create-requirement` 或后续 `prd-feishu` 自动写入的。

## 飞书联动配置（可选）

### 前置

1. 安装 [PRD-SKILL](https://github.com/KkNeChar0n/PRD-SKILL) 并跑过 `/prd-feishu-init`（飞书自建应用凭证存到 `~/.prd-feishu/config.json`）
2. 飞书自建应用需要开启权限：`docx:document` / `docx:document:create` / `drive:drive` / `wiki:wiki`（最后一个新增）
3. 想存放 PRD 的飞书知识库节点（如「迭代 V1.1.0」节点），右上角分享 → 链接共享 →「**组织内获得链接的人可编辑**」（让应用通过链接共享获得读写权）

### 启用

跑 `/aliyuncs:create-iteration` 时填写飞书父节点 URL（一次性，存进 `.aliyuncs.json` 的 currentIteration）。skill 会自动调 `wiki/v2/spaces/get_node` 校验应用对该节点的访问权，校验通过才写入。

之后 `create-requirement` 会自动调 `wiki/v2/spaces/{space_id}/nodes` 创建空 docx 子节点，把 URL 写回 requirement 条目；`prd-feishu` 写完 PRD 后回查 `.aliyuncs.json` 拿 requirement.identifier，调云效 SDK 把 `📄 [PRD 文档](URL)` 插到需求描述顶部。

### 跳过

`create-iteration` 时选「跳过」就不启用联动。所有 skill 仍可独立工作，飞书节点 / PRD URL 需要自己在飞书建好并手动填入 `.aliyuncs.json` 对应 requirement 的 `feishu_url` 字段（不推荐，自动化更香）。

## 完整示例：一个迭代周期

```bash
# 在项目根目录执行（.aliyuncs.json 会落在这里）
cd ~/my-project

# 1. 一次性配置（首次）
/aliyuncs:init
# → 填 5 个字段，写入 .aliyuncs.json

# 2. 创建本迭代
/aliyuncs:create-iteration
# → 询问飞书父节点 URL（可选），自动建 V1.X.0 迭代

# 3. 创建需求（按需多次）
/aliyuncs:create-requirement
# → 跟 Claude 对话整理 subject + description，确认后创建
# → 自动在飞书父节点下建同名空 docx
# 重复 N 次创建 N 个需求

# 4. 为每个需求写 PRD（PRD-SKILL）
/prd-feishu 应用管理详情页
# → 模糊匹配 requirement → 拿 feishu_url → 写内容 → 同步云效描述

# 5. 拆任务
/aliyuncs:create-task
# → 选一个需求拆，自动建前端/后端/测试子任务

# 6. 推进状态
/aliyuncs:update
# → 树形展示所有工作项，AskUserQuestion 选要改的，批量改 status
```

## 依赖

- Python 3.7+
- `alibabacloud-devops20210625` SDK（首次使用时自动 `pip3 install`）
- 阿里云 AccessKey ID / Secret
- （可选）飞书自建应用 + [PRD-SKILL](https://github.com/KkNeChar0n/PRD-SKILL)

## 已知约束 / 踩过的坑

- **`CreateSprintRequest` 必传 `staff_ids`**（否则 400）— init 时已经自动取并存到 `staffAccountId`
- **`create_workitem_v2` 必传 `assigned_to`**（否则 400「负责人不能为空」）
- **`update_work_item` 调用签名只有 2 个参数**：`(organization_id, request)`，workitem identifier 在 request 内部
- **SDK 不支持启动 / 完成迭代**（无 `UpdateSprint` API）—— 需在云效 UI 手动操作
- **更新 description 是全量覆盖**：要追加内容必须先 `get_work_item_info` 读出原内容再拼接写回
- **每个 SDK 调用间建议 `time.sleep(0.2)`** 避免限流
- **状态 identifier 因项目工作流不同而异**：通用状态（待处理 100005 / 已完成 100014 / 已取消 141230）大部分项目一致；其他状态先调 `GetWorkItemWorkFlowInfoRequest` 查实际可用状态

## 认证

使用阿里云 AccessKey ID + Secret 通过 SDK 签名认证调用云效 API。AccessKey 建议用 RAM 子账号 + 最小权限策略（仅授权云效相关 Action）。

## 许可证

MIT
