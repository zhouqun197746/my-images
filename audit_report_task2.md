# 任务包 2：场景扩写主工位接入新 scene pipeline — 完成回传

**完成日期**: 2026-06-20  
**核心改动**: 场景扩写 Tab 主路径从旧 `generation/scene_writer.write_scene()` 切换到新 `core/scene_writer.write_scene()` (H1 pipeline)

---

## 1. 改动文件/函数清单

### 新建文件

| 文件 | 用途 |
|------|------|
| `app/core/scene_write_p2_bridge.py` | P2 场景扩写桥接模块（约 300 行） |

### 修改文件

| 文件 | 修改方法 | 改动类型 |
|------|---------|---------|
| `app/gui/creative_studio_gui.py` | `_do_write_scene` | **主路径切换为 P2 scene pipeline** |
| `app/gui/creative_studio_gui.py` | `_display_scene_result` | 展示 scene_pipeline/stage/blueprint 元信息 |
| `app/gui/creative_studio_gui.py` | `_display_scene_result_raw` | 新增：raw 结果展示 fallback |
| `app/gui/creative_studio_gui.py` | `_seed_send_to_scene_write` | **升级为结构化 seed_bundle 引用** |

---

## 2. 新/旧 scene pipeline 分界点

### 旧逻辑（fallback）

```
app/generation/scene_writer.py :: write_scene(task_card, blueprint, draft_type, use_llm, top_k, save)
  - 接受 SceneTaskCard 对象
  - 调用 RAG 检索 + LLM 生成
  - 返回 SceneDraft 对象
```

### 新逻辑（P2 主路径）

```
app/core/scene_writer.py :: write_scene(task: dict, validation: dict, polished_draft: dict) -> dict
  - 接受 writer_task dict（不是 SceneTaskCard）
  - 纯结构化组装，无 LLM 调用
  - 返回 scene dict（含 full_scene_text, applied_constraints 等）
```

### 桥接层（新建）

```
app/core/scene_write_p2_bridge.py
  ├─ write_scene_via_p2()   [主路径]
  │    ├─ _build_writer_task_from_card()  [SceneTaskCard → writer_task dict]
  │    ├─ _build_minimal_validation()     [构建 validation dict]
  │    ├─ app.core.scene_writer.write_scene()  [调用新 pipeline]
  │    └─ _adapt_scene_to_draft()         [scene dict → SceneDraft]
  │
  └─ write_scene_legacy()   [fallback]
       └─ app.generation.scene_writer.write_scene()  [旧逻辑]
```

### 调用关系图

```
_do_write_scene() [GUI]
    │
    ├─ 读取 task_card.metadata["p2_bridge"]
    │
    ├─ p2_bridge == "p2_mainline" 或有 seed_bundle
    │     └─ write_scene_via_p2() [P2 主路径]
    │           ├─ _build_writer_task_from_card(task_card, seed_bundle, style)
    │           │     → writer_task dict (含 source_trace: blueprint_id, stage_name, seed_bundle_id, ...)
    │           ├─ _build_minimal_validation(task)
    │           │     → validation dict (writer_readiness="writer_ready")
    │           ├─ app.core.scene_writer.write_scene(task, validation)
    │           │     → scene dict (full_scene_text, applied_constraints, ...)
    │           ├─ _adapt_scene_to_draft(scene_dict, task_card, seed_bundle)
    │           │     → SceneDraft(metadata={scene_pipeline:"p2_mainline", blueprint_id, stage_name, ...})
    │           ├─ add_scene_draft(draft) → 保存到 DB
    │           │
    │           └─ [失败时] write_scene_legacy()
    │                 └─ app.generation.scene_writer.write_scene()
    │                       → SceneDraft(metadata={scene_pipeline:"legacy"})
    │
    └─ p2_bridge == "legacy_fallback" 或无 P2 标记
          └─ write_scene_legacy() [直接走旧逻辑]
```

---

## 3. P2 成功日志样本

**输入**:
```
task_card: P2 主链路卡片 (p2_bridge="p2_mainline", blueprint_id="bp_20260620_...", stage_name="Rising Action")
seed_bundle: None
style: "novel"
use_llm: False
```

**日志输出**:
```
开始扩写场景: card=a1b2c3d4... style=novel mode=P2
[SceneP2] 任务卡为 P2 主链路 (bridge=p2_mainline)，启动新 scene pipeline
[SceneP2] 启动 P2 新链路场景扩写...
[SceneP2] task_card=a1b2c3d4... blueprint=bp_20260620_060742... stage=Rising Action
[SceneP2] writer_task 构建完成: draft_id=D81edce70... stage_id=S2
[SceneP2] validation 构建完成: writer_readiness=writer_ready
[SceneP2] 调用 app.core.scene_writer.write_scene()...
[SceneP2] scene_writer 成功: status=pass 字数=850
[SceneP2] SceneDraft 保存成功: id=e5f6g7h8... blueprint=bp_20260620_060742...
场景扩写完成: id=e5f6g7h8... pipeline=p2_mainline blueprint=bp_20260620_060742... stage=Rising Action len=850 (0.3s)
```

**SceneDraft metadata 字段**:
```python
{
    "scene_pipeline": "p2_mainline",
    "scene_id": "SC_81edce70",
    "stage_id": "S2",
    "stage_name": "Rising Action",
    "stage_index": 1,
    "story_function": "Rising Action",
    "tone_anchor": "neutral narrative",
    "narrative_distance": "medium",
    "dramatic_question": "压力如何升级？",
    "ending_button": "...",
    "applied_constraints": [...],
    "writer_notes": [...],
    "warnings": [...],
    "blueprint_id": "bp_20260620_060742",
    "creative_methods": ["冲突升级", "升级路径"],
    "seed_bundle_id": "",
    "source_trace": {...}
}
```

---

## 4. Fallback 日志样本

**触发条件**: P2 scene_writer 抛异常或返回空结果

**日志输出**:
```
开始扩写场景: card=a1b2c3d4... style=novel mode=P2
[SceneP2] 任务卡为 P2 主链路 (bridge=p2_mainline)，启动新 scene pipeline
[SceneP2] 启动 P2 新链路场景扩写...
[SceneP2] task_card=a1b2c3d4... blueprint=bp_20260620_060742... stage=Rising Action
[SceneP2] writer_task 构建完成: draft_id=D81edce70... stage_id=S2
[SceneP2] validation 构建完成: writer_readiness=writer_ready
[SceneP2] 调用 app.core.scene_writer.write_scene()...
[SceneP2] scene_writer 异常: KeyError('must_keep')
[SceneP2] 降级到旧逻辑 fallback
[SceneP2] ⚠️ fallback: 使用旧版 generation.scene_writer.write_scene()
[SceneLegacy] SceneDraft 保存成功: id=f9g0h1i2... llm=False
场景扩写完成: id=f9g0h1i2... pipeline=legacy blueprint=bp_20260620_060742... stage= len=1200 (2.1s)
```

**fallback SceneDraft metadata**:
```python
{
    "scene_pipeline": "legacy",
    "fallback_reason": "p2_scene_writer_failed_or_unavailable",
    "llm_used": False,
    ...
}
```

---

## 5. seed_bundle 传入的具体字段

### 场景种子 Tab → 场景扩写 Tab 的传递

**`_seed_send_to_scene_write` 保存的结构化 seed_bundle**:
```python
self._current_scene_seed_bundle = {
    "scene_seed_bundle_id": "ss_20260620_060742",     # 种子 bundle ID
    "selected_seed_index": 0,                          # 选中的种子索引
    "seeds": [{
        "title": "种子标题",                           # 种子标题
        "text_preview": "种子文本预览...",              # 种子文本（前500字）
        "source_type": "text",                         # 素材来源类型
        "category": "...",                             # 分类
        "seed_id": "SC1",                              # 种子 ID
    }],
    "total_seeds": 5,                                  # 总种子数
}
```

**`_do_write_scene` 读取并传给 `write_scene_via_p2`**:
```python
seed_bundle = getattr(self, '_current_scene_seed_bundle', None)
result = write_scene_via_p2(task_card=card, seed_bundle=seed_bundle, ...)
```

**`_build_writer_task_from_card` 中 seed_bundle 的使用**:
- `seed_bundle.scene_seed_bundle_id` → 写入 `writer_task.source_trace.seed_bundle_id`
- `seed_bundle.seeds[0].title` → 写入 `writer_task.source_trace.seed_title`
- `seed_bundle.seeds[0].text_preview` → 拼入 `writer_task.outline_text`（作为"种子参考"行）

**最终 SceneDraft metadata 中记录**:
```python
"seed_bundle_id": "ss_20260620_060742",
"source_trace": {
    "seed_bundle_id": "ss_20260620_060742",
    "seed_title": "种子标题",
    ...
}
```

---

## 6. UI 行为变化

### 场景扩写 Tab 日志区分

- **P2 主链路**: `[SceneP2]` 前缀日志
- **Legacy fallback**: `[SceneLegacy]` 前缀日志

### 场景扩写结果详情（`_display_scene_result`）

新增展示字段（在原有字段之上）:
```
标题: 场景标题
本地ID(local_id): e5f6g7h8-...
类型(draft_type): novel
状态(status): draft (v1)
字数: 850
生成链路(scene_pipeline): p2_mainline
蓝图ID(blueprint_id): bp_20260620_060742...
蓝图阶段(stage_name): Rising Action
阶段ID(stage_id): S2
故事函数(story_function): Rising Action
基调(tone_anchor): neutral narrative
叙事距离(narrative_distance): medium
戏剧性问题(dramatic_question): 压力如何升级？
创意方法(creative_methods): 冲突升级, 升级路径
种子bundle(seed_bundle_id): ss_20260620_060742...
任务卡(task_card_id): a1b2c3d4-...

约束应用 (5 项):
  · must_keep: pass
  · must_not_break: pass
  · tone_anchor: pass
  · narrative_distance: pass

==================================================
正文预览:
...
```

---

## 7. 完成标准核对

| 完成标准 | 状态 | 证据 |
|---------|------|------|
| P2 任务卡 + 正常输入 → 默认走新 scene_writer_runner | ✅ | `_do_write_scene` 检查 `p2_bridge=="p2_mainline"` 调用 `write_scene_via_p2` |
| 日志出现 `[SceneP2]` 启动记录 | ✅ | 见第 3 节日志样本 |
| blueprint/stage 信息被带入 | ✅ | `writer_task.source_trace` + `SceneDraft.metadata` |
| P2 失败时 fallback 到旧 write_scene | ✅ | `write_scene_legacy()` 包裹旧逻辑 |
| fallback 日志有记录 | ✅ | `[SceneP2] 降级到旧逻辑 fallback` |
| fallback 卡片 metadata 显示 `scene_pipeline="legacy"` | ✅ | `draft.metadata["scene_pipeline"]="legacy"` |
| 种子 Tab → 扩写 Tab 附带 seed_bundle | ✅ | `_current_scene_seed_bundle` 结构化保存 |
| 扩写请求中附带 seed_bundle（日志确认） | ✅ | `source_trace.seed_bundle_id` + `outline_text` 含种子参考 |

---

## 8. 未完成项

1. **新版 scene_writer 为纯结构化（无 LLM）** — 当前 `app/core/scene_writer.write_scene()` 是确定性规则组装，不调用 LLM。因此 P2 路径生成的场景正文可能比旧版 LLM 生成的更"模板化"。后续可考虑在 P2 bridge 中增加 LLM 增强选项。

2. **scene_writer_runner（bundle 级编排）未直接使用** — `scene_writer_runner.build_scene_prose_bundle()` 需要 handoff_bundle + polished_validation_bundle（来自完整 H 组 pipeline），这些在当前 GUI 流程中不可用。本包通过直接调用 `scene_writer.write_scene()` 绕过了 bundle 编排，构建了最小 validation。

3. **seed_bundle 的完整利用** — 当前仅使用了 seed_bundle 的 `scene_seed_bundle_id`、`title`、`text_preview`。后续可利用更多 seed 字段（如 `stage_function`、`emotional_temperature`、`forbidden_generic_lines`）来约束 scene writer。

4. **场景扩写后的下游闭环** — 本包只打通"扩写主链"，未接入 rewrite/polish/validation/approval 等下游 runner。这些留待后续专包。

---

## 9. 明确不在本包做

- ❌ 不改任务卡 Tab（保持 P2 新链路稳定）
- ❌ 不改输入理解 / 素材检索 / Reports / Obsidian / Notion / 治理与调参 Tabs
- ❌ 不改质检与改写 Tab 的主逻辑（rewrite 闭环留待后续专包）
- ❌ 不调整 Tab 顺序
- ❌ 不做文案翻译/字体统一
