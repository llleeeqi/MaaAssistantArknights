# VisionDump 二开计划（给外部 AI 决策）

目标：把 MAA 改造成可给外部 AI 提供结构化界面信息的执行引擎。

闭环目标：

1. MAA 识别当前界面
2. 导出结构化信息（文本/候选按钮/坐标）
3. 外部 AI 决策下一步
4. 外部程序调用 MAA 点击或切任务

## 一、可行性结论

- MaaCore 已具备：
  - 回调机制（`AsstMsg` + `details_json`）
  - 截图能力（`AsstAsyncScreencap` / `AsstGetImage`）
  - 点击能力（`AsstAsyncClick`）
- 当前缺口：
  - 缺少“通用 UI 识别结果流”（全量 OCR + 按钮候选 + 坐标）对外稳定输出。

因此建议在 MaaCore 增加 `VisionDump` 风格事件。

## 二、最小可用方案（MVP）

先不追求全量识别，先做可跑闭环：

1. 新增一个 `SubTaskExtraInfo` 事件类型：`VisionDump`
2. 事件字段先给最小集：
   - `screen_tag`
   - `texts`（文本 + bbox + score）
   - `buttons`（id/text/x/y/bbox/score）
   - `timestamp`
3. 先在主界面/常见导航场景产出（例如任务、公招、基建）
4. 外部程序据此做 AI 决策，再调用 `AsstAsyncClick`

## 三、建议 JSON 结构

```json
{
  "class": "VisionDump",
  "timestamp": "2026-03-25T12:34:56",
  "screen_tag": "main_menu",
  "texts": [
    {"text": "公开招募", "bbox": [100, 200, 260, 248], "score": 0.98}
  ],
  "buttons": [
    {
      "id": "recruit",
      "text": "公开招募",
      "x": 180,
      "y": 224,
      "bbox": [100, 200, 260, 248],
      "score": 0.96
    }
  ]
}
```

## 四、代码改造建议点位

> 下面是建议点位，具体实现需结合现有识别任务管线。

1. **消息定义层**
   - 文件：`src/MaaCore/Common/AsstMsg.h`
   - 说明：无需新增顶层枚举也可先复用 `SubTaskExtraInfo`，通过 `details.class = VisionDump` 区分。

2. **回调消息构造层**
   - 在已有子任务识别完成后，构造 `details_json` 并附带 `VisionDump` 字段。
   - 优先选择已有 OCR/模板匹配节点，避免重复识别。

3. **协议文档层**
   - 文件：`docs/zh-cn/protocol/callback-schema.md`（及多语言对应文档）
   - 说明：增加 `VisionDump` 说明、字段定义、示例。

4. **集成示例层（可选）**
   - 增加一个最小示例：回调接收 `VisionDump` -> 输出 JSON 文件。

## 五、验证标准

满足以下条件即算 MVP 打通：

- 能稳定收到 `VisionDump` 事件（至少主界面）
- JSON 中包含可用点击点（`x`,`y`）
- 外部程序基于 JSON 决策并调用 `AsstAsyncClick` 可完成一次导航

## 六、后续增强

- 扩大场景覆盖（关卡选择、公招详情、基建页面）
- 增加语义标签（`action_type`, `priority`）
- 增加误触防护（置信度阈值、重复确认）
- 与 `MaaAI` 联动提升识别质量（可选）

## 七、当前仓库状态

- 分支：`feat/visiondump-scaffold`
- 本文档为二开路线草案，下一步可开始落地实现。
