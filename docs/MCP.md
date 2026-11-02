# MCP 服务

AI 客户端可查询、创建、管理和导出任务，与桌面共用同一数据库和队列。

## 连接

1. 打开“功能 → MCP 服务”，点击“启动服务”。默认地址为 `http://127.0.0.1:8766/mcp`。
2. 在客户端选择 **Streamable HTTP**，填写地址；也可使用面板里的“复制连接配置”。
3. 启用鉴权时，将下方 `<API_KEY>` 替换为面板里的密钥；未启用时删除 `headers`。

```json
{
  "mcpServers": {
    "feed-collector": {
      "type": "http",
      "url": "http://127.0.0.1:8766/mcp",
      "headers": {"Authorization": "Bearer <API_KEY>"}
    }
  }
}
```

## 工具

| 用途 | 工具 |
| --- | --- |
| 应用状态、任务列表与详情 | `get_app_status`、`list_tasks`、`get_task_progress` |
| 搜索账号、查询聊天记录 | `search_accounts`、`query_chat_records` |
| 创建链接、账号、关键词、聊天记录任务 | `add_video_task`、`add_account_task`、`add_keyword_task`、`add_message_task` |
| 暂停、继续、取消 | `pause_task`、`resume_task`、`cancel_task` |
| 导出视频、评论、弹幕 | `export_task_data` |
| 日志状态、脱敏诊断 ZIP | `get_diagnostics_info`、`export_diagnostic_bundle` |

完整参数通过 `tools/list` 获取。以下是常用约定：

| 参数 | 说明 |
| --- | --- |
| `scope` | 默认 `job`（账号、关键词、聊天记录主任务）；链接和自动下载用 `download` |
| `task_id` | 使用创建结果或 `list_tasks` 返回的 ID；主任务子视频统一由主任务控制 |
| `offset` / `limit` | 本地查询分页；默认 50 条，每页最多 200 条，以返回的 `offset` 和总数继续查询 |
| `data_kind` | `get_task_progress` 和 `export_task_data` 可选 `videos`（默认）、`comments`、`bullets` |
| `path` | 创建主任务时为保存根目录；导出时为**服务所在电脑**的文件路径，表格使用 `.xlsx` / `.csv`，诊断包必须是绝对路径 `.zip` |

`get_task_progress` 查询主任务视频时返回 `job` 和分页 `videos`；每条视频含本地已保存的 `collected_comment_count`、`collected_bullet_count`（评论含回复、按记录去重，未采集为 0），并保留公开 `comment_count`。查询单条下载时保留状态与 `progress` 百分比，扩展字段位于 `video`。评论、弹幕仍返回 `items`、`total`、`offset`、`states`，包含采集状态、二级回复及响应统计。查询和导出只读取本地已有数据，导出包含全部已采集记录，不受查询分页限制。桌面不再展示评论/弹幕正文，本接口不变。

## 采集选项

四种创建工具都支持以下参数：

| 参数 | 默认与含义 |
| --- | --- |
| `collect_comments` / `collect_bullets` | 默认 `false`；启用后采集评论（含二级回复）/ 弹幕 |
| `interaction_max_pages` | 默认 `0`（不限），最大 10000；**每个视频、每种互动**分别计算，二级回复请求也计入 |
| `interaction_max_items` | 默认 `0`（不限），最大 1000000；每个视频、每种互动分别计算，评论条数包含回复 |

页数和条数上限任一先达到即停止。关键词任务的 `page_limit` 是**搜索结果页数**（1–100），与互动页数上限分开计算。主任务可设置 `download_files: false` 只保存已有作品信息，启用评论或弹幕后仍会请求对应接口。

例如，调用 `add_keyword_task`，采集前 2 页作品，每个视频最多保存 100 条评论：

```json
{
  "keyword": "示例关键词",
  "page_limit": 2,
  "download_files": false,
  "collect_comments": true,
  "interaction_max_items": 100
}
```

调用 `export_task_data` 导出评论：

```json
{"task_id": "创建结果中的ID", "data_kind": "comments", "path": "D:\\Exports\\评论.xlsx"}
```

扩展字段包括账号资料、媒体规格、作品和互动统计、评论用户及回复关系、弹幕类型与来源。缺失值留空，`0` / `false` 保留原值，复杂结构保留 JSON；Excel 超长内容拆为相邻“续”列。旧缓存不会自动联网补字段。

