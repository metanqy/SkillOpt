# SkillOpt 项目调查报告

## 1. 项目定位
SkillOpt 是一个“**不改模型权重、只优化技能文档（prompt/skill markdown）**”的训练框架。它把 Agent 技能优化类比为深度学习训练：rollout→reflect→aggregate→select→update→gate，并配套 epoch、learning rate、validation gate 等机制。

## 2. 代码结构总览
- `skillopt/engine/`：训练主循环（核心编排）。
- `skillopt/envs/`：多 benchmark 适配层（SearchQA、DocVQA、ALFWorld、SpreadsheetBench、OfficeQA、LiveMathematicianBench 等）。
- `skillopt/optimizer/`：技能更新策略、学习率调度、slow update、meta skill。
- `skillopt/gradient/`：patch 聚合与反思相关流程。
- `skillopt/model/`：多后端模型路由与封装（Azure OpenAI、Codex、Claude、Qwen）。
- `configs/`：分 benchmark 的配置文件。
- `scripts/train.py` 与 `scripts/eval_only.py`：训练/评估入口。
- `skillopt_webui/`：可视化监控界面。
- `docs/`：MkDocs 文档站点。

## 3. 运行机制（关键路径）
1. CLI 解析并加载 YAML 配置（支持 `_base_` 继承）。
2. 根据 `env` 注册表选择 benchmark adapter。
3. 训练器执行 ReflACT 六阶段循环：
   - Rollout：目标模型执行任务
   - Reflect：优化模型分析轨迹并提出 patch
   - Aggregate：合并 patch
   - Select：按预算选择 edits
   - Update：应用到 skill 文档
   - Gate：在验证集做接受/拒绝
4. 每 step 与 epoch 持久化历史、技能快照与比较日志。

## 4. 架构优点
- **EnvAdapter 抽象清晰**：环境逻辑与训练循环解耦，扩展新 benchmark 成本较低。
- **配置系统完整**：支持结构化配置、legacy flat key、命令行覆盖，且有 `_base_` 继承。
- **多模型后端设计合理**：通过 router 统一调用面，便于切换。
- **训练产物可追溯**：history、skill 快照、step 级 artifacts 完整。

## 5. 主要风险与问题清单

### 5.1 文档与代码存在不一致
- README 的“Supported Benchmarks”仅列出 6 个；但 `scripts/train.py` 中内置注册表还尝试加载 `babyvision/mmrb/mathverse/sealqa/swebench`。
- `docs/index.md` 中展示 `SWEBench` 配置路径 `configs/swebench/`，但当前仓库未见 `configs/swebench/default.yaml`。

**影响**：用户按文档运行可能遇到“配置不存在”或“依赖不满足”的混淆。

### 5.2 默认模型版本前瞻性较强
- 默认配置与 README 使用了 `gpt-5.5`。在部分环境中该部署名未必存在，用户首次运行容易因 deployment 命名不一致失败。

**建议**：文档明确“示例 deployment 名需替换为你的 Azure deployment 名”。

### 5.3 依赖拆分策略仍可优化
- 部分 benchmark 的依赖通过 optional extras 处理得较好（如 ALFWorld），但其他 env 的真实依赖可进一步模块化，降低首次安装阻力。

### 5.4 入口脚本参数面很大
- `scripts/train.py` 的兼容参数较多，虽增强灵活性，但对新用户理解成本高。

**建议**：增加“最小配置模板 + 场景化 presets”。

## 6. 可维护性评估
- **扩展性：高**（adapter + config 模式）。
- **可测试性：中**（目前仓库内未见成体系 pytest 用例目录，建议补齐核心单测）。
- **可运维性：中上**（输出结构化、可恢复；但外部依赖多，首跑对环境要求高）。
- **文档一致性：中**（存在少量“文档先行/代码未同步”痕迹）。

## 7. 建议的优先级改进路线

### P0（建议立即）
1. 对齐文档与代码中的 benchmark 列表与配置路径。
2. 在 README/Docs 增加“最小可运行命令（含必要环境变量）”与常见报错排查。
3. 明确模型 deployment 名与模型家族名的区别。

### P1（短期）
1. 补充核心模块单测：
   - config 继承/覆盖逻辑
   - adapter 注册与 fallback 行为
   - patch 规范化与选择策略
2. 增加 smoke test（不触发真实 API，mock backend）。

### P2（中期）
1. 建立 benchmark capability matrix（依赖、数据格式、是否需要工具执行）。
2. 增强 WebUI：展示 step 级 patch 质量与 gate 决策解释。

## 8. 结论
SkillOpt 架构方向正确，核心设计（训练循环 + adapter 抽象 + 可回溯产物）成熟度较高，适合研究/实验型场景。当前主要短板不在“核心算法主干”，而在**文档一致性、上手门槛、测试覆盖**。若按 P0/P1 方案补齐，项目可用性会明显提升。
