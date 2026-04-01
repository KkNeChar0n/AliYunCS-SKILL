# AliYunCS-SKILL

阿里云云效（Yunxiao）Claude Code Skills 套件，用于在 Claude Code 中通过斜杠命令管理云效项目的迭代、需求和任务。

## Skills 列表

| 命令 | 功能 |
|------|------|
| `/aliyuncs:init` | 初始化项目配置（组织ID、项目标识、AccessKey） |
| `/aliyuncs:create-iteration` | 创建迭代，命名规则 `项目名称 V1.X.0`，自动递增版本号 |
| `/aliyuncs:create-requirement` | 根据对话内容创建产品需求，关联到当前迭代 |
| `/aliyuncs:create-task` | 根据需求拆解研发任务，自动创建子任务（前端/后端/测试） |
| `/aliyuncs:update` | 批量查询和更新需求、任务、子任务的状态 |

## 工作流

```
init → create-iteration → create-requirement → create-task
                                ↑
                          update（随时可用）
```

## 安装

将各 `aliyuncs:xxx/` 目录复制到 `~/.claude/skills/` 下即可全局使用：

```bash
cp -r aliyuncs:* ~/.claude/skills/
```

## 依赖

- Python 3
- `alibabacloud-devops20210625` SDK（首次使用时自动安装）
- 阿里云 AccessKey ID / Secret

## 认证

使用阿里云 AccessKey ID + AccessKey Secret，通过 SDK 签名认证调用云效 API。
