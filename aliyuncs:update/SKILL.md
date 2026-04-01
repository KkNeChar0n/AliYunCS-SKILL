---
name: aliyuncs:update
description: 根据当前需求和任务完成情况，批量更新云效中需求、任务、子任务的状态
user_invocable: true
---

# 更新需求和任务状态

当用户执行 `/aliyuncs:update` 时，执行以下步骤：

## 工作流校验

读取 `.aliyuncs.json`，校验配置完整且 `requirements` 或 `tasks` 中至少有一项不为空。

## SDK 初始化

```python
from alibabacloud_devops20210625.client import Client
from alibabacloud_tea_openapi.models import Config

config = Config(
    access_key_id=accessKeyId,
    access_key_secret=accessKeySecret,
    endpoint='devops.cn-hangzhou.aliyuncs.com'
)
client = Client(config)
```

## 步骤 1：获取当前状态

遍历所有需求、任务、子任务，查询最新状态：

```python
response = client.get_work_item_info(organizationId, workitemIdentifier)
wi = response.body.workitem
# wi.status 是状态名称（如"待处理"、"已完成"）
# wi.status_stage_identifier 是阶段标识
```

## 步骤 2：展示状态总览

以树形结构展示所有工作项状态，然后用 AskUserQuestion 询问用户要更新哪些。

## 步骤 3：获取可用状态列表

```python
from alibabacloud_devops20210625.models import GetWorkItemWorkFlowInfoRequest

req = GetWorkItemWorkFlowInfoRequest(configuration_id='')
response = client.get_work_item_work_flow_info(organizationId, workitemIdentifier, req)
# 遍历 response.body.workflow.statuses
# 每个状态有：.name, .identifier, .workflow_stage_identifier, .workflow_stage_name
```

已知状态 identifier：
- `100005` = 待处理（确认阶段）
- `100010` = 处理中（处理阶段）— 仅任务类型
- `142838` = 开发中（开发阶段）— 仅需求类型
- `100014` = 已完成（正常结束）
- `141230` = 已取消（异常结束）

## 步骤 4：执行状态更新

**重要**：`update_work_item` 的签名是 `(organization_id, request)`，workitem identifier 在 request 内部，不是单独的参数。

```python
from alibabacloud_devops20210625.models import UpdateWorkItemRequest

request = UpdateWorkItemRequest(
    identifier=workitemIdentifier,  # 工作项ID放在 request 里
    field_type='status',
    property_key='status',
    property_value='100014'  # 目标状态的 identifier
)
client.update_work_item(organizationId, request)  # 只传2个参数！
```

**每次调用间加 `time.sleep(0.2)` 避免限流。**

## 步骤 5：更新本地配置

将状态变更同步到 `.aliyuncs.json`。

## 步骤 6：输出变更摘要

表格展示：工作项 | 原状态 | 新状态 | 结果
