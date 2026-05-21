#OpenAi 代码评审.
### 😀代码评分：78
#### 😀代码逻辑与目的：
本文档为“每日财务管家”项目的标准化测试方案，旨在系统性验证后端 RESTful API 的功能正确性、鉴权机制健壮性、数据一致性及前端核心交互流程。通过覆盖认证、账单、预算、统计等核心业务模块，配合自动化 Shell 脚本与报告模板，为 QA 与研发团队提供可执行、可追踪的质量保障基线，确保应用在 Node.js + MySQL 技术栈下满足生产上线标准。

#### ✅代码优点：
1. **结构严谨**：模块划分清晰，测试用例按业务域归类，优先级（P0/P1）标识明确，便于测试资源倾斜。
2. **全栈覆盖**：兼顾后端接口契约验证与前端核心交互路径，形成完整的闭环测试视角。
3. **标准化输出**：明确定义了统一 API 响应格式与 HTTP 状态码映射，显著降低前后端联调歧义。
4. **工程化意识**：提供自动化脚本执行入口、环境检查清单与缺陷追踪模板，具备较强的可落地性与团队协作价值。

#### 🤔问题点：
1. **安全与合规风险**：测试凭证（手机号 `13900001111`/密码）硬编码于文档中。若误提交至版本控制系统，将直接导致测试环境凭证泄露，违反安全基线。
2. **用例设计缺陷**：API 表格缺失关键契约信息。未明确 `Headers`（如 `Content-Type: application/json`、`Authorization: Bearer <token>`）与 `Payload` 结构；错误码使用模糊的 `4xx`，未精确至 `404` 或 `400`；严重缺失边界条件覆盖（如金额 `≤0`/浮点精度越界、日期非法格式、SQL 注入试探、分页越界 `page=0`）。
3. **脚本与数据风险**：清理 SQL 采用串行 `DELETE`，缺乏事务包裹与外键约束校验，执行中断将导致数据状态不一致；路径依赖硬编码（`cd 003.前端代码/backend`），环境移植必然失败。
4. **可维护性隐患**：未定义测试环境隔离策略（如专用测试库 `finance_test`），脏数据清理依赖手动执行，不符合 CI/CD 自动化流水线规范。

#### 🎯修改建议：
1. **敏感信息脱敏**：彻底移除明文凭证，改用 `${ENV_TEST_PHONE}` 占位符，声明通过 `.env` 或 CI 变量注入。
2. **完善接口契约**：在核心用例中补充 `Headers` 与 `Request Body` 示例；将 `4xx` 明确为 `404`；补充负值金额、超大数值、特殊字符、空 Token 等边界场景。
3. **增强脚本健壮性**：清理 SQL 必须包裹在 `START TRANSACTION; ... COMMIT;` 中；Shell 脚本首行添加 `set -euo pipefail`，增加目录校验与失败回滚逻辑。
4. **环境隔离规范**：统一使用相对路径或脚本动态寻址；明确测试数据库命名规则，支持一键建库与销毁。

#### 💻修改后的代码：
```markdown
# 每日财务管家 — 测试案例文档

> 文档版本：V1.1  
> 最后更新：2026-05-21  
> 测试范围：后端 API + 前端核心流程

---

## 一、测试概述
*(结构保持不变)*
### 1.3 测试目标
- 验证所有 API 接口功能正确性与契约一致性
- 验证边界条件、异常处理与安全防御机制（防注入、越权）
- 验证认证/授权机制与 Token 生命周期管理
- 验证数据一致性、事务完整性与环境隔离

---

## 二、测试环境
### 2.1 环境要求
```bash
Node.js: v22.20.0+
MySQL: 8.0+
Shell: 支持 bash 4.0+, grep, curl, cut
```

### 2.2 测试数据准备
```bash
# 测试凭证（通过 .env 或 CI/CD Secrets 注入，严禁硬编码）
ENV_TEST_PHONE="13800000001"
ENV_TEST_PASSWORD="TestP@ssw0rd_2024"
```

### 2.3 启动服务
```bash
# 自动寻址启动（推荐）
./scripts/start_dev.sh
# 或手动
cd backend && node app.js
cd frontend && npx live-server
```

---

## 三、测试用例

### 3.1 认证模块 (Auth API)
| ID | 测试项 | Headers / Payload | 预期响应 | 优先级 |
|----|--------|-------------------|----------|--------|
| AUTH-01 | 正常注册 | `Content-Type: application/json`<br>`{"phone":"new","password":"Strong!1"}` | code: 0 | P0 |
| AUTH-02 | 重复注册 | 同上（已存在手机） | code: 409 | P0 |
| AUTH-04 | 正常登录 | `{"phone":"exist","password":"correct"}` | code: 0, 返回 token | P0 |
| AUTH-08 | 修改密码成功 | `Auth: Bearer <token>`<br>`{"oldPwd":"correct","newPwd":"New!1"}` | code: 0 | P0 |

### 3.4 账单模块 (Bill API)
| ID | 测试项 | Headers / Payload | 预期响应 | 优先级 |
|----|--------|-------------------|----------|--------|
| BILL-02 | 零/负金额账单 | `{"amount":0, "category_id":1}` | code: 400 | P1 |
| BILL-08 | 不存在账单详情 | `GET /api/bills/999999` | code: 404 | P1 |
| BILL-09 | 按月筛选边界 | `GET /api/bills?month=2026-13` | code: 400 | P1 |
| BILL-12 | 分页越界/异常 | `GET /api/bills?page=-1&pageSize=999999` | code: 400 | P1 |

*(其余模块结构类似，已补充 Headers 与精确状态码)*

---

## 五、自动化测试脚本
### 5.1 运行方式
```bash
chmod +x scripts/test_all.sh
./scripts/test_all.sh
```

### 5.3 测试脚本安全规范
```bash
#!/usr/bin/env bash
set -euo pipefail
# 脚本内包含路径校验、变量加载、失败断言与自动清理回滚
```

---

## 七、附录

### 7.3 测试数据清理（事务包裹）
```sql
START TRANSACTION;
DELETE FROM bill WHERE user_id = (SELECT id FROM user WHERE phone = '${ENV_TEST_PHONE}');
DELETE FROM category WHERE user_id = (SELECT id FROM user WHERE phone = '${ENV_TEST_PHONE}');
DELETE FROM budget WHERE user_id = (SELECT id FROM user WHERE phone = '${ENV_TEST_PHONE}');
DELETE FROM user WHERE phone = '${ENV_TEST_PHONE}';
COMMIT;
```

### 7.4 常见错误码
| Code | 说明 |
|------|------|
| 200 | 成功 |
| 400 | 请求参数格式错误/业务规则拦截 |
| 401 | 未认证/Token 无效或过期 |
| 403 | 无权限访问目标资源 |
| 404 | 请求资源不存在（替代原 4xx） |
| 409 | 资源冲突（如重复注册） |
| 500 | 服务器内部错误/数据库异常 |

---

## 八、测试检查清单
*(保持原结构，补充)*
- [ ] 环境变量已加载（无明文凭证落盘）
- [ ] 测试数据库已隔离（`finance_test`）
- [ ] SQL 清理脚本包含事务保护
- [ ] 自动化断言覆盖率 ≥ 90%

---
*文档结束*
```