---
name: aliyuncs:create-requirement
description: 根据当前对话中已总结的内容，在阿里云云效中自动创建需求（工作项），并关联到当前迭代
user_invocable: true
---

# 在云效中创建需求

当用户执行 `/aliyuncs:create-requirement` 时，执行以下步骤：

## 工作流校验

读取 `.aliyuncs.json`，依次校验：
1. 文件存在且包含完整配置（含 accessKeyId、accessKeySecret、staffAccountId）
2. `currentIteration` 不为 null

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

## 步骤 1：整理需求内容

根据当前对话上下文整理需求的 subject（标题）和 description（Markdown 描述）。如果对话中没有明确内容，使用 AskUserQuestion 询问。展示给用户确认后再创建。

## 步骤 2：获取工作项类型

```python
from alibabacloud_devops20210625.models import ListProjectWorkitemTypesRequest

request = ListProjectWorkitemTypesRequest(
    space_type='Project',
    category='Req'
)
response = client.list_project_workitem_types(organizationId, spaceIdentifier, request)
# 遍历 response.body.workitem_types，找 category_identifier == 'Req' 的
# 取其 .identifier 作为 workitemTypeIdentifier
```

## 步骤 3：创建需求

**重要**：`assigned_to` 必填，否则 API 返回 400「字段【负责人】不能为空」。

```python
from alibabacloud_devops20210625.models import CreateWorkitemV2Request

request = CreateWorkitemV2Request(
    subject='需求标题',
    description='需求描述（Markdown格式）',
    space_identifier=spaceIdentifier,
    category='Req',
    workitem_type_identifier=workitemTypeIdentifier,
    sprint_identifier=currentIteration['identifier'],
    assigned_to=staffAccountId  # 必填！从 .aliyuncs.json 读取
)
response = client.create_workitem_v2(organizationId, request)
workitem_id = response.body.workitem_identifier  # 注意：不是 response.body.workitem.identifier
```

**响应字段说明**：
- 成功时 `response.body.workitem_identifier` 是工作项ID
- 不存在 `response.body.workitem` 对象，直接用 `body.workitem_identifier`

## 步骤 4：更新本地配置

将需求信息追加到 `.aliyuncs.json` 的 `requirements` 数组：

```json
{
  "identifier": "工作项ID",
  "subject": "需求标题",
  "status": "待处理",
  "tasks": []
}
```

输出创建成功信息，提示下一步：`/aliyuncs:create-task`
