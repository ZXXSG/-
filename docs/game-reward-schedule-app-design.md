# 游戏奖励式日程计划 APP 设计文档

## 1. 产品概述
- **产品定位**：一款面向个人用户的安卓端日程计划应用，通过游戏化奖励机制（虚拟金币）激励用户养成按计划完成任务的习惯。
- **核心价值**：帮助用户建立执行计划的正反馈，鼓励其将虚拟奖励兑换为现实物品，从而强化目标感和自律能力。
- **目标用户**：需要通过自我激励管理日程的学生、职场人士或自由职业者。

## 2. 用户角色与使用场景
| 角色 | 典型行为 | 核心诉求 |
| --- | --- | --- |
| 普通用户 | 制定日程 → 完成任务 → 领取虚拟金币 → 记录现实兑换物品 | 直观的任务管理、可量化的激励、可视化进度 |
| 新用户 | 引导完成基础设置（目标、偏好），体验首日签到和奖励 | 快速上手、奖励机制易理解 |

主要使用场景：
1. 每日创建/查看日程，滑动或点击标记已完成的任务。
2. 任务完成后自动获取金币，累积到一定阈值后提醒兑换现实奖励。
3. 统计页面展示任务完成率、金币历史、现实奖励记录等。

## 3. 功能需求
### 3.1 必备功能
1. **账号系统**：手机号/邮箱+验证码登录，支持第三方登录预留扩展；存储用户基础信息、习惯设定。
2. **日程管理**：
   - 创建、编辑、删除日程任务，支持每日/每周重复。
   - 任务属性：标题、描述、优先级、预计时长、计划时间段、提醒方式。
   - 任务状态流转：待开始 → 进行中 → 已完成/已跳过。
3. **提醒与通知**：本地推送 + 可选云消息；任务开始前提醒，未完成任务的日终提醒。
4. **金币奖励系统**：
   - 完成任务自动发放金币，根据任务难度/优先级/连续完成天数调整奖励。
   - 支持设置兑换目标（现实物品），达到虚拟金额阈值后发出兑换提醒。
   - 支持金币消耗日志、奖励发放日志。
5. **数据统计**：
   - 每日/每周任务完成率、连续完成天数。
   - 金币余额趋势、历史兑换记录。
6. **激励玩法**（基础版）：
   - 签到系统：每日签到额外奖励，连续签到加成。
   - 成就系统：达成特定目标（例如完成 10 个高优先级任务）解锁勋章。

### 3.2 可选/扩展功能
- 社区激励（排行榜、好友互励）。
- AI 助手（根据行为数据智能推荐日程安排）。
- 多终端同步（iOS、小程序）。
- 离线模式与本地缓存。

## 4. 系统架构
- **客户端**：Android 原生（Kotlin）或 Flutter；负责 UI 展示、数据缓存、通知调用。
- **服务端**：Python (FastAPI / Django REST Framework)，遵循 RESTful API，使用 JWT 进行鉴权。
- **数据库**：PostgreSQL 或 MySQL 存储业务数据；Redis 用于缓存热数据、分布式锁、排行榜；
- **消息队列**（可选）：用于异步处理（任务提醒、统计计算）。
- **对象存储**（可选）：保存用户头像等静态资源。
- **部署**：Docker 容器化，Kubernetes 或云服务（如阿里云、AWS）部署；使用 CI/CD 管理发布。

```
Android App ⇄ API Gateway ⇄ Python Backend ⇄ Database / Cache / MQ
```

## 5. 数据模型概述
| 实体 | 关键字段 | 说明 |
| --- | --- | --- |
| User | id, email/phone, nickname, avatar_url, timezone, notification_settings, preferences | 用户基础信息与偏好 |
| Task | id, user_id, title, description, priority, scheduled_start, scheduled_end, repeat_rule, status, difficulty, reward_points | 日程任务 |
| TaskLog | id, task_id, status_before, status_after, timestamp, note | 任务状态变更日志 |
| RewardAccount | user_id, balance, total_earned, total_spent, last_reward_time | 金币账户 |
| RewardTransaction | id, user_id, task_id, type (earn/spend), points, source, created_at, metadata | 金币流水 |
| RedemptionGoal | id, user_id, name, target_amount, current_progress, deadline, status | 现实奖励目标设定 |
| Achievement | id, code, name, description, criteria | 成就定义 |
| UserAchievement | id, user_id, achievement_id, unlocked_at | 成就达成记录 |
| Streak | id, user_id, streak_type (signin/task), current_streak, longest_streak, last_updated | 连续完成记录 |
| Notification | id, user_id, type, payload, schedule_time, status | 消息推送记录 |

## 6. 核心业务流程
1. **任务完成发放奖励**：
   - App 将任务完成状态上报 → 服务端校验任务状态与重复约束 → 计算奖励金币（基础值 + 难度加成 + 连续完成加成）→ 写入 Task、RewardTransaction、更新 RewardAccount → 返回最新余额。
2. **签到与 streak**：
   - 用户触发签到 → 服务端校验是否已签到 → 发放签到奖励 → 更新 streak。
3. **兑换目标提醒**：
   - 用户设定兑换目标（金额、截止日期）→ 定时任务或触发式检查金币余额是否达到 → 推送提醒。
4. **数据统计**：
   - 使用定时任务聚合任务完成率、金币收入支出等数据，供客户端展示。

## 7. 算法与规则设计
### 7.1 奖励计算公式
```
reward_points = base_reward(priority, difficulty) * completion_multiplier(streak) * bonus_factor(time_efficiency)
```
- base_reward：按优先级/预计耗时设置基础金币（例如高优先级 30，中优先级 20，低优先级 10）。
- completion_multiplier：根据连续完成天数调整（例如 streak 1-3 → 1.0，4-6 → 1.1，7+ → 1.25）。
- time_efficiency（可选）：按实际完成时间与计划时间比，提前完成给予额外 5%-10% 奖励。
- 算法需要支持配置化，使用规则表或可扩展策略模式。

### 7.2 成就系统
- 分类：任务量成就、连续完成成就、兑换目标成就等。
- 触发方式：实时（任务完成时检查）+ 定时任务（每日复核）。
- 解锁后写入 UserAchievement 并发送通知。

## 8. 接口设计（RESTful）
所有接口统一前缀 `/api/v1`，使用 HTTPS。鉴权采用 JWT（`Authorization: Bearer <token>`）。

### 8.1 认证与用户
| 方法 | 路径 | 描述 | 请求参数 | 响应 |
| --- | --- | --- | --- | --- |
| POST | /auth/register | 注册账号 | {"email/phone", "password", "verification_code"} | 201, user 基本信息 + token |
| POST | /auth/login | 登录获取 token | {"email/phone", "password"} | 200, token + user |
| POST | /auth/token/refresh | 刷新 token | refresh_token | 新 access token |
| GET | /users/me | 获取当前用户信息 | - | 用户信息 |
| PATCH | /users/me | 更新个人资料/偏好 | 局部字段 | 更新后的用户信息 |

### 8.2 任务管理
| 方法 | 路径 | 描述 | 请求参数 | 备注 |
| --- | --- | --- | --- | --- |
| POST | /tasks | 创建任务 | {title, description, priority, scheduled_start, scheduled_end, repeat_rule, difficulty, reward_points_override?} | 返回 task |
| GET | /tasks | 查询任务列表 | 支持分页、状态过滤、日期范围 | - |
| GET | /tasks/{id} | 获取任务详情 | - | - |
| PATCH | /tasks/{id} | 更新任务 | 局部字段 | - |
| DELETE | /tasks/{id} | 删除任务 | - | 软删除或彻底删除配置化 |
| POST | /tasks/{id}/start | 标记任务开始 | {actual_start_time?} | 更新状态 → 进行中 |
| POST | /tasks/{id}/complete | 标记任务完成 | {actual_end_time?, note?} | 触发奖励计算 |
| POST | /tasks/{id}/skip | 跳过任务 | {reason?} | 更新状态，不发放奖励 |

### 8.3 奖励与金币
| 方法 | 路径 | 描述 | 请求参数 | 备注 |
| --- | --- | --- | --- | --- |
| GET | /rewards/account | 获取金币账户信息 | - | balance, total_earned, total_spent |
| GET | /rewards/transactions | 金币流水 | 分页、类型过滤 | - |
| POST | /rewards/redemption-goals | 创建兑换目标 | {name, target_amount, deadline, image?} | - |
| GET | /rewards/redemption-goals | 查询兑换目标列表 | 状态过滤 | - |
| PATCH | /rewards/redemption-goals/{id} | 更新兑换目标 | - | 支持调整目标金额、进度 |
| POST | /rewards/redemption-goals/{id}/complete | 标记已兑换 | {actual_cost?, note?} | 记录现实奖励 |

### 8.4 签到与成就
| 方法 | 路径 | 描述 | 请求参数 | 备注 |
| --- | --- | --- | --- | --- |
| POST | /gamification/sign-in | 每日签到 | - | 返回奖励信息 |
| GET | /gamification/streaks | 查询 streak 数据 | type 参数（signin/task） | - |
| GET | /gamification/achievements | 获取成就列表 | unlock_status 过滤 | - |
| POST | /gamification/achievements/{id}/claim | 领取成就奖励 | - | - |

### 8.5 统计与通知
| 方法 | 路径 | 描述 | 请求参数 | 备注 |
| --- | --- | --- | --- | --- |
| GET | /analytics/dashboard | 概览统计 | date_range, granularity | 返回任务完成率、金币走势等 |
| GET | /notifications | 获取通知列表 | 状态过滤 | - |
| POST | /notifications/{id}/ack | 标记通知已读 | - | - |

## 9. 非功能需求
- **安全性**：
  - 所有接口强制 HTTPS，密码加密存储（bcrypt）。
  - JWT 访问控制，支持刷新机制。敏感操作需要二次验证（如密码修改）。
  - 防刷机制：任务完成请求限流、签到每天一次检查。
- **性能**：
  - 支持 10 万日活用户；关键接口（任务查询、奖励发放）响应时间 < 300ms。
  - 使用缓存加速用户配置、任务列表；数据库建立合理索引。
- **可用性**：
  - 99.5% SLA；任务和奖励数据定期备份；提供错误监控与告警。
- **可维护性**：
  - 遵循分层架构（API 层、Service 层、Domain/Repository 层）。
  - 配置化奖励策略，支持热更新（存储于数据库或配置中心）。

## 10. 后端模块划分
1. **Auth Service**：负责注册、登录、JWT 签发、用户资料。
2. **Task Service**：任务 CRUD、状态机、提醒调度。
3. **Reward Service**：金币计算、账户管理、兑换目标。
4. **Gamification Service**：签到、成就、streak。
5. **Analytics Service**：统计计算、报表输出。
6. **Notification Service**：推送与消息管理。
7. **Scheduler/Worker**：使用 Celery/RQ 执行异步任务与定时任务。

## 11. 技术栈建议
- **后端框架**：FastAPI（高性能、类型提示友好）或 Django REST Framework（生态成熟）。
- **数据库层**：SQLAlchemy + Alembic 做迁移；使用 Pydantic 定义接口模型。
- **缓存与消息队列**：Redis，结合 Celery 处理异步任务；
- **监控**：Prometheus + Grafana；日志聚合使用 ELK 或 Loki。
- **测试**：pytest 做单元/集成测试，使用 factory-boy/pytest fixtures 构造数据。

## 12. 接口返回通用规范
- 使用统一响应结构：
```
{
  "code": 0,
  "message": "success",
  "data": {...}
}
```
- 错误码规范：
  - 1xxx：认证与权限相关错误。
  - 2xxx：业务校验错误（任务状态、奖励规则）。
  - 3xxx：系统内部错误或第三方失败。

## 13. 开发计划与里程碑（建议）
1. **MVP（4-6 周）**：账号、任务管理、金币奖励、签到、统计基础。
2. **Phase 2**：成就系统、兑换目标管理、通知完善。
3. **Phase 3**：AI 推荐、社区互动、跨平台支持。

## 14. 风险与对策
- 奖励机制不够吸引 → 收集用户数据迭代奖励策略；加入动态调整。
- 数据准确性风险 → 设计幂等接口、使用事务确保奖励发放和余额一致。
- 推送通知延迟 → 使用可靠消息队列，监控推送成功率。
- 用户隐私与安全 → 加强密码策略、敏感数据脱敏、权限审计。

## 15. 未来扩展方向
- 引入任务模板市场，与健康、学习等垂直领域合作。
- 支持 AR 展示虚拟奖励、游戏化进阶玩法（养成类角色）。
- 与可穿戴设备联动，自动识别运动类任务完成情况。

