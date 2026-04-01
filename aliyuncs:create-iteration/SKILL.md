---
name: aliyuncs:create-iteration
description: 在云效中创建迭代，命名规则为【项目名称 V1.X.0】，自动递增版本号。启动迭代需在云效界面手动操作。
user_invocable: true
---

# 在云效中创建迭代

当用户执行 `/aliyuncs:create-iteration` 时，执行以下步骤：

## 工作流校验

读取 `.aliyuncs.json`，校验文件存在且包含完整配置（organizationId、spaceIdentifier、projectName、accessKeyId、accessKeySecret、staffAccountId），否则提示先执行 `/aliyuncs:init`。

## 前置依赖检查

```bash
pip3 list 2>/dev/null | grep alibabacloud-devops20210625
```
如果未安装，执行 `pip3 install alibabacloud-devops20210625`。

## SDK 初始化（所有步骤共用）

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

## 步骤 1：查询现有迭代

```python
from alibabacloud_devops20210625.models import ListSprintsRequest

request = ListSprintsRequest(
    space_identifier=spaceIdentifier,
    space_type='Project'
)
response = client.list_sprints(organizationId, request)
# 遍历 response.body.sprints，每个 sprint 有 .name, .identifier, .status 属性
```

从迭代名称中找出符合 `${projectName} V1.X.0` 模式的最大版本号 X，新迭代为 X+1。

## 步骤 2：创建迭代

**重要**：`CreateSprintRequest` 必须传 `staff_ids` 参数，否则 API 返回 400 错误。无 `space_type` 参数。

```python
from alibabacloud_devops20210625.models import CreateSprintRequest

request = CreateSprintRequest(
    space_identifier=spaceIdentifier,
    name='迭代名称',
    staff_ids=[staffAccountId]  # 必填！从 .aliyuncs.json 读取
)
response = client.create_sprint(organizationId, request)
sprint_identifier = response.body.sprint.identifier  # 迭代ID
sprint_name = response.body.sprint.name
sprint_status = response.body.sprint.status  # 'TODO'
```

## 步骤 3：更新本地配置

将迭代信息写入 `.aliyuncs.json` 的 `currentIteration` 字段：

```json
{
  "currentIteration": {
    "identifier": "迭代ID",
    "name": "项目名称 V1.X.0",
    "status": "TODO"
  }
}
```

## 步骤 4：确认完成

输出迭代名称和ID。

**注意**：SDK 不支持启动/完成迭代操作（无 UpdateSprint API）。请提示用户在云效界面手动点击「开始迭代」。

提示下一步：`/aliyuncs:create-requirement`
