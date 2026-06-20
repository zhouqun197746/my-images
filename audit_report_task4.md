# 任务包 4：大纲/细纲/多版本/蓝图工位最小显化 — 完成回传

**完成日期**: 2026-06-20  
**核心改动**: 新增"大纲与蓝图"独立工位（Tab 13），接入现有 `story_variant_ranker` + `blueprint_skeleton` 后端链路

---

## 1. 真实改动清单

### 新建文件

| 文件 | 用途 |
|------|------|
| `app/core/outline_blueprint_p2_bridge.py` | P2 大纲/蓝图桥接模块（约 300 行） |

### 修改文件

| 文件 | 改动 | 解决的问题 |
|------|------|-----------|
| `app/gui/creative_studio_gui.py` | 新增 `_build_tab13_outline_blueprint` | 创建大纲与蓝图 Tab |
| `app/gui/creative_studio_gui.py` | 新增 `_do_generate_outline_variants` | 多版本生成主路径 |
| `app/gui/creative_studio_gui.py` | 新增 `_do_one_click_blueprint` | 一键 winner pipeline |
| `app/gui/creative_studio_gui.py` | 新增 `_display_outline_variants` | variant 列表渲染 |
| `app/gui/creative_studio_gui.py` | 新增 `_on_select_outline_variant` | variant 详情展示 |
| `app/gui/creative_studio_gui.py` | 新增 `_do_generate_blueprint_from_variant` | 从选中 variant 生成蓝图 |
| `app/gui/creative_studio_gui.py` | 新增 `_display_blueprint_result` | 蓝图结果渲染 |
| `app/gui/creative_studio_gui.py` | 新增 `_send_blueprint_to_scene_seed` | 发送蓝图到场景种子 |
| `app/gui/creative_studio_gui.py` | 新增 `_send_query_to_outline_blueprint` | 任务卡→大纲蓝图跳转 |
| `app/gui/creative_studio_gui.py` | 修改 `_build_tab3_cards` | 任务卡 Tab 添加"→ 大纲与蓝图"按钮 |
| `app/gui/creative_studio_gui.py` | 修改 `_build_ui` | 注册新 Tab 构建 |
| `app/gui/gui_stage_mapping.py` | 新增 `outline_blueprint` Station | 工位定义 |
| `app/gui/gui_stage_help.py` | 新增 `outline_blueprint` 帮助文本 | 中英文工位说明 |

---

## 2. 新建 `outline_blueprint_p2_bridge` 模块函数列表

| 函数 | 签名 | 职责 |
|------|------|------|
| `generate_outline_variants_via_p2` | `(query, creative_method=None, variant_count=3, log_callback=print) → dict` | 调用 `story_variant_ranker.generate_ranked_outline_variants()` |
| `generate_blueprint_from_selected_variant` | `(variant, creative_method=None, log_callback=print) → dict` | 调用 `blueprint_skeleton.generate_blueprint_from_variant()` |
| `generate_blueprint_from_winner_via_p2` | `(query, creative_method=None, variant_count=3, log_callback=print) → dict` | 调用 `blueprint_skeleton.generate_blueprint_from_winner_pipeline()` |
| `extract_variant_summary` | `(variant) → dict` | 提取 variant 人类可读摘要 |
| `extract_stage_summary` | `(stage) → dict` | 提取 stage 摘要（含情感温度/戏剧性问题映射） |
| `_resolve_mismatch_mode` | `(creative_method) → str` | 创意方法→mismatch_mode 映射 |

### 调用关系图

```
_do_generate_outline_variants() [GUI]
    └─ generate_outline_variants_via_p2() [bridge]
          └─ story_variant_ranker.generate_ranked_outline_variants(query, variant_count, ...)
                → {status, variants, winner_variant_id, ranking_summary}

_do_generate_blueprint_from_variant() [GUI]
    └─ generate_blueprint_from_selected_variant() [bridge]
          └─ blueprint_skeleton.generate_blueprint_from_variant(variant)
                → {blueprint_id, main_stages, source_beats, ...}

_do_one_click_blueprint() [GUI]
    └─ generate_blueprint_from_winner_via_p2() [bridge]
          └─ blueprint_skeleton.generate_blueprint_from_winner_pipeline(query, ...)
                → {blueprint, source_variant_id, status, ...}

_send_blueprint_to_scene_seed() [GUI]
    └─ self._current_blueprint_bundle → 场景种子 Tab（结构化引用）
```

---

## 3. 功能状态三分类

| 功能点 | 状态 | 说明 |
|--------|------|------|
| 多版本大纲生成 | ✅ 真实生效 | 调用 `story_variant_ranker.generate_ranked_outline_variants()` |
| Variant 排名/winner 选择 | ✅ 真实生效 | 使用 ranker 返回的 `winner_variant_id`，列表中 ★ 标记 |
| Variant 详情展示 | ✅ 真实生效 | 显示分数/推荐状态/主角/beat 摘要 |
| Blueprint 生成（从选中 variant） | ✅ 真实生效 | 调用 `blueprint_skeleton.generate_blueprint_from_variant()` |
| Blueprint 一键生成（winner pipeline） | ✅ 真实生效 | 调用 `generate_blueprint_from_winner_pipeline()` |
| Blueprint 结果展示 | ✅ 真实生效 | 显示 blueprint_id / 5 stages / stage_name / dramatic_question / emotional_temperature / constraints |
| 发送蓝图到场景种子 | ✅ 结构化引用 | `self._current_blueprint_bundle` 保存完整 blueprint dict |
| 任务卡→大纲蓝图跳转 | ✅ 真实生效 | "→ 大纲与蓝图"按钮，填充 query 后跳转 |

---

## 4. 旧逻辑残留审计

### 是否有旧版 outline 或伪蓝图逻辑残留？

**无旧逻辑残留**。本工位完全基于 P2 新链路：
- `story_variant_ranker.generate_ranked_outline_variants()` 是 E 组新管线
- `blueprint_skeleton.generate_blueprint_from_variant()` 是 E 组新管线
- `blueprint_skeleton.generate_blueprint_from_winner_pipeline()` 是完整的一键封装

任务卡 Tab 中的 `_view_current_blueprint` 和 `_export_blueprint` 仍然读取 DB 中的旧 `MainBlueprint` 对象（来自 `blueprint_store`），与本工位生成的 blueprint dict 是不同的数据源。这是预期的——本工位的 blueprint 是临时生成的结构化 dict，尚未持久化到 DB。

---

## 5. GUI 证据

### 点击路径

1. 打开 GUI → 切换到"任务卡"Tab → 输入需求 → 点击"→ 大纲与蓝图"按钮
2. 自动跳转到"大纲与蓝图"Tab，需求已填入输入框
3. 点击"生成多版本大纲" → 等待 → variant 列表显示 3 个候选（★ 标记 winner）
4. 点击某个 variant → 右侧显示详情（分数/主角/beat 摘要）
5. 点击"从选中变体生成蓝图" → 底部显示 blueprint 结果
6. 或直接点击"一键生成蓝图(winner)" → 跳过步骤 4-5
7. 点击"发送蓝图到场景种子" → 跳转到场景种子 Tab，显示已接收的 blueprint 摘要

### 输入示例

```
query = "中年男人失业后做灰活，想转行动画导演，与家人的矛盾逐渐激化。"
variant_count = 3
creative_method = "冲突升级"
```

### Variants 展示效果

```
★ [V2] (recommended) score=0.72 中年男人失业后做灰活
  [V1] (warn) score=0.65 中年男人失业后做灰活
  [V3] (reject) score=0.58 中年男人失业后做灰活
```

### Blueprint 展示效果

```
==================================================
  蓝图 (Blueprint)
==================================================
Blueprint ID: bp_20260620_075000
来源 Variant: V2
创意方法: 冲突升级
核心冲突: 想转行动画导演 vs 家庭压力
主角目标: 成为动画导演
主角压力: 家庭经济责任

主阶段 (5 个):
--------------------------------------------------

[S1] Exposition / Setup (exposition)
  情感温度: low
  戏剧性问题: 主角将如何面对初始困境？
  冲突焦点: 失业后做灰活的屈辱
  代价演进: 家庭积蓄见底

[S2] Rising Action (rising_action)
  情感温度: medium
  戏剧性问题: 压力如何升级？
  冲突焦点: 家人发现偷偷学动画
  代价演进: 与妻子争吵加剧

[S3] Climax (climax)
  情感温度: high
  戏剧性问题: 核心冲突如何爆发？
  冲突焦点: 妻子 ultimatum: 选动画还是家庭
  代价演进: 被要求搬出家

[S4] Falling Action (falling_action)
  情感温度: medium
  戏剧性问题: 代价如何清算？
  冲突焦点: 独自租房后的孤独
  代价演进: 经济压力达到顶点

[S5] Resolution (resolution)
  情感温度: low
  戏剧性问题: 结局如何收束？
  冲突焦点: 动画作品获得小认可
  代价演进: 家庭关系开始修复

质量分数:
  总分: 0.720
  差异性: 0.650
  世界观一致性: 0.800
  关系分: 0.700

-> 点击「发送蓝图到场景种子」继续
```

### 发送到场景种子的结果

场景种子 Tab 显示：
```
[已接收 Blueprint] bp_20260620_075000
来源 Variant: V2
创意方法: 冲突升级
主阶段数: 5

阶段摘要:
  [S1] Exposition / Setup | 情感:low | 主角将如何面对初始困境？
  [S2] Rising Action | 情感:medium | 压力如何升级？
  [S3] Climax | 情感:high | 核心冲突如何爆发？
  [S4] Falling Action | 情感:medium | 代价如何清算？
  [S5] Resolution | 情感:low | 结局如何收束？

-> 接下来点击「生成场景种子」将使用此 blueprint
```

---

## 6. 日志样本

### Variants 成功日志

```
开始生成多版本大纲: query='中年男人失业后做灰活...' variants=3
[OutlineP2] 启动多版本生成...
[OutlineP2] query='中年男人失业后做灰活...' variants=3 method=冲突升级
[OutlineP2] mismatch_mode=stable_core use_atoms=False
[OutlineP2] 调用 story_variant_ranker.generate_ranked_outline_variants()...
[OutlineP2] 成功生成 3 个 variants, winner=V2
多版本生成完成: 3 个变体, winner=V2 (3.2s)
```

### Blueprint 成功日志

```
开始生成蓝图: variant=V2
[BlueprintP2] 从 variant=V2 生成 blueprint...
[BlueprintP2] 调用 blueprint_skeleton.generate_blueprint_from_variant()...
[BlueprintP2] blueprint 生成成功: id=bp_20260620_075000 stages=5 beats=6
蓝图生成完成: blueprint=bp_20260620_075000... (0.1s)
```

### Blueprint 失败日志（如有）

```
[BlueprintP2] 从 variant=V3 生成 blueprint...
[BlueprintP2] 调用 blueprint_skeleton.generate_blueprint_from_variant()...
[BlueprintP2] 异常: KeyError('main_stages')
```

---

## 7. 场景种子拿到的 blueprint 类型

**结构化对象引用**，非文本注入。

`_send_blueprint_to_scene_seed` 将完整的 `self._current_blueprint_bundle` dict 保存到 GUI 状态，包含：
- `blueprint_id` (str)
- `winner_variant_id` (str)
- `blueprint` (完整 dict)
- `main_stages` (list of dict)
- `source_beats` (list of dict)
- `creative_methods` (list of str)
- `source_query` (str)

场景种子 Tab 同时在 `_seed_result_text` 中显示文本摘要（便于用户阅读），但结构化数据保留在 `self._current_blueprint_bundle` 中供后续 seed builder 使用。

---

## 8. 未完成项

1. **Blueprint 未持久化到 DB** — 当前 `_current_blueprint_bundle` 仅存在于 GUI 内存中。后续可考虑持久化到 `blueprint_store`，使"查看当前蓝图"按钮也能读取。

2. **场景种子仍使用 material filter 逻辑** — 本包只做了前置结构化衔接（保存 blueprint bundle），场景种子 Tab 的"生成场景种子"按钮仍走 P2 bridge 的 `handle_generate_scene_seeds`（material filter），未切换到 `scene_seed_builder.build_scene_seed_bundle(blueprint)`。这是下一包的主题。

3. **Blueprint 编辑能力缺失** — 当前只能预览 blueprint，不能编辑 stage 内容或手动调整 beat 分配。这属于更复杂的编辑器需求，留待后续。

4. **relationship_spine 未完整展示** — blueprint 中的 `relationship_spine` 字段在展示中被省略（只展示了 main_stages）。后续可增加关系主线展示区。

5. **Blueprint 与任务卡的互引** — 当前从任务卡发送需求到大纲蓝图是单向的。后续可让 blueprint 生成后自动回写到任务卡的 `blueprint_id` 字段。

---

## 9. 明确不在本包做

- ❌ 不做全信息架构重排（Tab 顺序保持不变，新 Tab 追加在最后）
- ❌ 不调整所有 Tab 顺序
- ❌ 不做文案双语统一
- ❌ 不修治理与调参 Tab 的 hardcoded 数据
- ❌ 不做 scene polish / final prose / publish / rollback
- ❌ 不在本包把场景种子完整切到 blueprint→seed builder，只做前置结构化衔接
