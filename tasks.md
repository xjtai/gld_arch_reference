---
specId: TASKS-参考组-001
type: tasks
parent_spec: spec.md
plan_ref: plan.md
domain: 客服
owner: 参考组 / 讲师
status: 已确认
confirmed: true
version: 1.0
updated_at: 2026-08-17
execution_mode: sequential
knowledge_refs: []
related_specs: [eval.md]
score: {}
---

> **参考实现**：任务划分随 `plan.md` 的选型而定，换个框架任务清单也会不同。**每项都有可执行的 verify 命令**这一点是通用的——别写"手工验证"。

# TASKS-参考组-001｜执行任务清单

## 执行期约束（每个 Task 执行前注入）

- 不允许修改 `plan.md` §6 不改清单里列出的内容。
- 代码一律写在 `src/` 下，五份文档留在仓库根目录。
- 遇到 `spec.md` / `plan.md` 没覆盖的情况，按最小合理假设处理，记在对应任务的"说明"里，同时补一行到本文件末尾的变更记录，不要擅自扩大范围。
- 每完成一个 Task，先跑该 Task 的 verify 命令，通过了再进入下一个。
- 密钥走 `.env`，不得写进代码或提交。

## 任务清单

### T-01　项目骨架初始化

- 依赖：无
- 负责人：王五
- 产出物：`src/` 可运行骨架（Flask + DeepSeek 客户端），`src/README.md` 写清依赖安装与启动
- 验收口径：别人 clone 下来照着 `src/README.md` 能启动并访问首页
- verify 命令：`cd src && pip install -r requirements.txt && python app.py` 后 `curl -s localhost:5001/health` 返回 `{"status":"ok"}`
- 状态：✅
- 说明：Flask 3.0 + openai SDK（走 DeepSeek 兼容接口）。加了 `/health` 便于 verify。

### T-02　实现路由智能体

- 依赖：T-01
- 负责人：王五
- 产出物：`src/router.py`，一次 LLM 调用输出 `{category, intent, order_id, need_context}`
- 验收口径：对照 `spec.md` US-02 的 6 个场景能正确分类；`need_context` 能正确识别"价格呢"这类省略句
- verify 命令：`cd src && python -m tests.test_router` —— 输出 6 个场景 + 2 个省略句的分类结果，全部命中才算过
- 状态：✅
- 说明：温度设 0；解析失败降级为 `category=闲聊`（兜底出口），不抛异常。

### T-03　实现售前咨询智能体（≥3 场景）

- 依赖：T-02
- 负责人：赵六
- 产出物：`src/agents/presale.py` + `tools/product.py` + `tools/policy.py`
- 验收口径：价格 / 库存 / 平台政策三个场景正确作答；查不到的商品不编造（`spec.md` §4）
- verify 命令：`cd src && python -m tests.test_presale` —— 覆盖 3 个场景 + 1 个不存在商品，断言回复含预期关键词
- 状态：✅
- 说明：价格等数据全部读 `data/products.json`，未写死在提示词里（`spec.md` §9 记的那处纠错）。

### T-04　实现售后咨询智能体（含退款 Golden Path）

- 依赖：T-02
- 负责人：赵六
- 产出物：`src/agents/aftersale.py` + `tools/order.py` + `tools/refund.py`
- 验收口径：对照 `spec.md` US-01 Golden Path 与 US-06（缺订单号先问）；退货/退款按 `spec.md` §2 区分处理
- verify 命令：`cd src && python -m tests.test_aftersale` —— 覆盖退款成功、缺订单号反问、非法订单号、订单不存在、重复退款五种情况
- 状态：✅
- 说明：重复退款拦截靠订单表的 `refund_status` 字段（`spec.md` §3 盲区表裁决的那条）。

### T-05　实现工具层、降级与可观测性

- 依赖：T-03, T-04
- 负责人：王五
- 产出物：`tools/__init__.py` 统一包装（10s 超时 + 重试 1 次 + 故障注入）、`src/trace.py` 事件记录、`static/index.html` 三栏 UI
- 验收口径：对照 `spec.md` US-03（降级）与 US-05（执行过程可解释）
- verify 命令：`cd src && FORCE_TOOL_FAILURE=true python -m tests.test_degradation` —— 断言返回兜底话术且进程不崩；再 `curl -s localhost:5001/stream` 确认 5 类事件都带时间戳
- 状态：✅
- 说明：**故障注入开关初版只包住了退款工具**，评审时要求覆盖全部工具（见变更记录）。

### T-06　跑完全部验证

- 依赖：T-01…T-05
- 负责人：全组
- 产出物：`eval.md` 已回填实测结果与判定结论
- 验收口径：`eval.md` 全部门禁通过，或明确记录未通过项
- verify 命令：`cd src && python -m tests.run_eval` —— 跑 `eval.md` §1.1 的 22 条用例，输出准确率数字；再 `hey -z 60s -c 20 -m POST -d '{"msg":"手机多少钱"}' localhost:5001/chat` 压性能
- 状态：✅
- 说明：**首轮准确率 81.8%（18/22）未达标**，定位并修复后复测 95.5%（21/22）。详见 `eval.md` §2 与 `learnings.md` L-04。

### T-07　部署发布与结项回写

- 依赖：T-06
- 负责人：张三
- 产出物：可被现场访问的 demo、`learnings.md`、本文件全部状态更新
- 验收口径：照着 `src/README.md` 能把 demo 跑起来并演示；`learnings.md` 由 Agent 读四份变更记录生成并经人评审
- verify 命令：**换一台机器重新 clone，只照着 `src/README.md` 走一遍**，能启动并跑通 Golden Path。组内那台"能跑"不算数。
- 状态：✅
- 说明：换机验证时发现 `src/README.md` 漏写了 `PORT` 环境变量，已补（见 `learnings.md` L-05）。

## DoD（整体完成定义）

- [x] T-01 ~ T-07 全部状态为 ✅
- [x] `eval.md` 的判定结论（§7）不是"不通过"
- [x] demo 已部署发布，可被现场访问
- [x] `learnings.md` 已由 Coding Agent 生成并经人评审确认，至少有 L-01 ~ L-03

## 变更记录

| 时间 | 改动 | 人 |
|---|---|---|
| 08-17 09:50 | Coding Agent 基于确认版 plan.md 起草初稿 | Agent |
| 08-17 09:56 | **纠错**：T-03/T-04 的 verify 命令原写"手工验证对话效果"，要求改成可执行的测试脚本——否则"做完了"没法自证 | 张三 |
| 08-17 09:58 | **补充**：T-06 原只跑准确率，补上性能压测命令（`spec.md` §6 定了 10 QPS 口径，不测等于漏一条验收） | 李四 |
| 08-17 10:00 | **补充**：T-07 的 verify 从"能跑起来"改成"换一台机器 clone 后照 README 跑通"——组内机器有残留环境变量，测不出遗漏 | 张三 |
| 08-17 10:01 | 确认，status 改为已确认，进入编码 | 张三 |
| 08-17 11:05 | 编码期：`FORCE_TOOL_FAILURE` 初版只作用于退款工具，人工 review 发现后要求包住全部工具，否则 `eval.md` §3 只能测到一半 | 王五 |
| 08-17 11:40 | 编码期最小合理假设：`spec.md` 未定义"已签收订单能否改地址"，按"已发货即拒绝"处理（已签收属于已发货之后） | Agent |

---

### 给 Coding Agent 的执行说明

```
请阅读 spec.md、plan.md、tasks.md、eval.md 四份文件（均为已确认版）。严格按
tasks.md 的 T-01 到 T-06 顺序实现，每完成一项跑一下对应的 verify 命令，通过后
把状态改成 ✅ 并在"说明"里简述实现方式。

代码一律写在 src/ 目录下（结构按 plan.md §3 定的来），五份产出文档留在仓库
根目录不要动；T-01 要同时把依赖安装和启动命令写进 src/README.md。密钥统一走
根目录的 .env（已在 .gitignore 里），不要把真实 Key 写进代码或提交进仓库。

遇到 spec.md / plan.md 没覆盖的情况，按最小合理假设处理并说明假设，记进变更
记录，不要擅自扩大范围，且不允许改动 plan.md §6"不改清单"里列出的内容。
T-06 完成后把测试结果回填进 eval.md 并给出判定结论。
```
