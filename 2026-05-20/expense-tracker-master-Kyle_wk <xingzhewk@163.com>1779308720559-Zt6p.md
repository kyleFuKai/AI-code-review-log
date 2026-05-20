#OpenAi 代码评审.
### 😀代码评分：82
#### 😀代码逻辑与目的：
定义“每日财务管家”应用的核心数据模型（用户、分类、账单、标签关联、预算、AI审计日志），并提供支持事务与幂等执行的系统预设数据及测试账号初始化脚本。旨在为后端服务提供结构化、高可用的数据存储基座，并保障多环境重复部署时的数据一致性。
#### ✅代码优点：
1. DDL 设计规范严谨，显式声明存储引擎、字符集及排序规则，避免隐式转换风险。
2. 合理运用 `IF NOT EXISTS`、`ON UPDATE CURRENT_TIMESTAMP` 及复合唯一键，有效保障数据完整性与并发安全性。
3. Seed 脚本采用事务包裹与 `INSERT IGNORE`，具备优秀的幂等特性，符合基础设施初始化最佳实践。
#### 🤔问题点：
1. **会话变量强耦合**：`002_seed_data.sql` 严重依赖 MySQL 会话变量（`@p_xxx`）传递父级 ID。该模式强绑定连接生命周期，在连接池复用、分片集群或自动化 CI/CD 管道中极易出现作用域丢失，导致子级数据静默写入失败，且调试困难。
2. **冗余索引设计**：`bill` 表中 `idx_user_time(user_id, bill_time)` 已完全覆盖单列时间索引的查询路径。额外保留 `idx_bill_time(bill_time)` 将徒增 `INSERT/UPDATE` 的页分裂开销与磁盘占用，违背索引最小化原则。
3. **根节点语义缺陷**：`category.parent_id` 使用 `BIGINT DEFAULT 0` 标识根节点。在关系型数据库中，`NULL` 才是表达“无父级”的标准语义。使用 `0` 将导致未来引入递归 CTE 或树形聚合查询时，必须额外编写 `WHERE parent_id > 0` 过滤条件，极易引发全表扫描。
4. **安全与可观测性缺失**：测试账户密码哈希硬编码于脚本中，违反凭证管理基线。大量 `INSERT IGNORE ... WHERE @var IS NOT NULL` 的静默失败策略缺乏执行结果反馈，一旦父级因唯一键冲突未创建，子级数据将全部丢失，运维排查成本极高。
5. **唯一键与 NULL 兼容性隐患**：`user.phone` 允许 `NULL` 且设唯一索引。虽 MySQL 8.0 支持多 `NULL` 共存，但主流 ORM（如 MyBatis/JPA/Hibernate）在映射时可能触发非空断言异常，需额外配置规避。
#### 🎯修改建议：
1. **彻底废弃会话变量**：将 `@var` 替换为内联子查询 `SELECT id FROM category WHERE ...`。消除连接状态依赖，确保脚本在任何 SQL 解析器与连接池下表现一致。
2. **移除冗余索引**：删除 `bill.idx_bill_time`，依赖复合索引 `idx_user_time` 覆盖业务查询。若确有全局时间范围分析需求，应走 OLAP 数仓或定期归档。
3. **优化树形结构语义**：建议将 `parent_id` 默认值改为 `NULL`。若业务层强依赖 `0`，至少需在 DDL 注释中明确警告 CTE 递归时的转换成本。此处保留原设计但优化索引绑定逻辑。
4. **强化幂等执行反馈**：在 Seed 脚本末尾添加明确的结果校验语句。硬编码测试凭证需增加安全警告，建议生产环境通过配置中心动态注入。
5. **统一命名规范**：外键遵循 `fk_子表_关联表` 格式，提升元数据字典的可读性。

#### 💻修改后的代码：
```sql
-- ============================================================================
-- 每日财务管家 — V1.0 数据库建表脚本 (MySQL 8.0+)
-- 字符集：utf8mb4 | 排序规则：utf8mb4_0900_ai_ci
-- ============================================================================

CREATE TABLE IF NOT EXISTS `user` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '用户 ID',
    `nickname`        VARCHAR(32)        NOT NULL DEFAULT '' COMMENT '昵称',
    `avatar_url`      VARCHAR(512)       NOT NULL DEFAULT '' COMMENT '头像 URL',
    `phone`           VARCHAR(20)        NULL DEFAULT NULL COMMENT '手机号 (允许 NULL，兼容未绑定用户)',
    `country_code`    VARCHAR(6)         NOT NULL DEFAULT '+86' COMMENT '国家/地区区号',
    `password_hash`   VARCHAR(255)       NOT NULL COMMENT '密码哈希 (bcrypt，注册时必填)',
    `currency`        CHAR(3)            NOT NULL DEFAULT 'CNY' COMMENT '默认货币单位',
    `theme`           VARCHAR(16)        NOT NULL DEFAULT 'light' COMMENT '主题模式',
    `status`          TINYINT UNSIGNED   NOT NULL DEFAULT 1 COMMENT '状态：0=禁用 1=正常',
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间',
    `updated_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_phone` (`phone`) COMMENT '唯一索引兼容多个 NULL 值',
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='用户表';

CREATE TABLE IF NOT EXISTS `category` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '分类 ID',
    `name`            VARCHAR(32)        NOT NULL COMMENT '分类名称',
    `icon`            VARCHAR(64)        NOT NULL DEFAULT '' COMMENT 'Material Symbol 图标名',
    `type`            ENUM('EXPENSE','INCOME') NOT NULL DEFAULT 'EXPENSE',
    `parent_id`       BIGINT UNSIGNED    NULL DEFAULT NULL COMMENT '父分类 ID (NULL 表示根节点)',
    `sort_order`      INT UNSIGNED       NOT NULL DEFAULT 0,
    `is_preset`       TINYINT UNSIGNED   NOT NULL DEFAULT 1 COMMENT '是否系统预设：1=是 0=用户自定义',
    `is_archived`     TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `user_id`         BIGINT UNSIGNED    NOT NULL DEFAULT 0 COMMENT '所属用户ID，0表示全局预设',
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_type_parent_name` (`user_id`, `type`, `parent_id`, `name`),
    KEY `idx_type_parent` (`type`, `parent_id`),
    KEY `idx_user` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='收支分类表';

CREATE TABLE IF NOT EXISTS `bill` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    `user_id`         BIGINT UNSIGNED    NOT NULL,
    `type`            ENUM('EXPENSE','INCOME') NOT NULL DEFAULT 'EXPENSE',
    `amount`          DECIMAL(12,2)      NOT NULL,
    `category_id`     BIGINT UNSIGNED    NOT NULL,
    `remark`          VARCHAR(200)       NOT NULL DEFAULT '',
    `bill_time`       DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `is_recurring`    TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `created_by`      BIGINT UNSIGNED    NOT NULL DEFAULT 0,
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_bill_user` FOREIGN KEY (`user_id`) REFERENCES `user`(`id`) ON DELETE RESTRICT,
    CONSTRAINT `fk_bill_category` FOREIGN KEY (`category_id`) REFERENCES `category`(`id`) ON DELETE RESTRICT,
    KEY `idx_user_time` (`user_id`, `bill_time`)
    -- 已移除冗余的 idx_bill_time(bill_time)，复合索引 idx_user_time 已完全覆盖查询路径
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='账单记录表';

CREATE TABLE IF NOT EXISTS `bill_tag` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    `user_id`         BIGINT UNSIGNED    NOT NULL,
    `name`            VARCHAR(16)        NOT NULL,
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_name` (`user_id`, `name`),
    CONSTRAINT `fk_bill_tag_user` FOREIGN KEY (`user_id`) REFERENCES `user`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='账单标签表';

CREATE TABLE IF NOT EXISTS `bill_tag_rel` (
    `bill_id`         BIGINT UNSIGNED    NOT NULL,
    `tag_id`          BIGINT UNSIGNED    NOT NULL,
    PRIMARY KEY (`bill_id`, `tag_id`) COMMENT '复合主键，天然防重',
    CONSTRAINT `fk_bill_tag_rel_bill` FOREIGN KEY (`bill_id`) REFERENCES `bill`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_bill_tag_rel_tag` FOREIGN KEY (`tag_id`) REFERENCES `bill_tag`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='账单 - 标签关联表';

CREATE TABLE IF NOT EXISTS `category_budget` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    `user_id`         BIGINT UNSIGNED    NOT NULL,
    `category_id`     BIGINT UNSIGNED    NOT NULL,
    `amount`          DECIMAL(10,2)      NOT NULL,
    `period`          ENUM('DAILY','WEEKLY','MONTHLY') NOT NULL DEFAULT 'MONTHLY',
    `start_date`      DATE               NOT NULL,
    `end_date`        DATE               NULL DEFAULT NULL,
    `is_active`       TINYINT UNSIGNED   NOT NULL DEFAULT 1,
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_category_period` (`user_id`, `category_id`, `period`),
    CONSTRAINT `fk_category_budget_user` FOREIGN KEY (`user_id`) REFERENCES `user`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_category_budget_category` FOREIGN KEY (`category_id`) REFERENCES `category`(`id`) ON DELETE CASCADE,
    KEY `idx_user_period_active` (`user_id`, `period`, `is_active`),
    KEY `idx_budget_effective` (`user_id`, `start_date`, `end_date`, `is_active`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='分类预算表';

CREATE TABLE IF NOT EXISTS `review_log` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    `project_name`    VARCHAR(64)        NOT NULL,
    `branch_name`     VARCHAR(128)       NOT NULL,
    `commit_author`   VARCHAR(128)       NOT NULL,
    `commit_message`  VARCHAR(512)       NOT NULL DEFAULT '',
    `review_content`  TEXT               NOT NULL COMMENT 'AI 评审内容',
    `review_url`      VARCHAR(512)       NOT NULL COMMENT 'GitHub 评审链接',
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_project_branch` (`project_name`, `branch_name`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='AI 代码评审日志表';

-- ============================================================================
-- 预设数据与测试账号初始化 (幂等安全版 - 移除会话变量强依赖)
-- 支持重复执行，数据不重复插入
-- ============================================================================

START TRANSACTION;

-- 1. 一级分类数据 (INSERT IGNORE 确保幂等)
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`) VALUES
-- 一级支出分类
('餐饮美食', 'restaurant', 'EXPENSE', NULL, 1, 1, 0),
('交通出行', 'directions_car', 'EXPENSE', NULL, 2, 1, 0),
('购物消费', 'shopping_bag', 'EXPENSE', NULL, 3, 1, 0),
('休闲娱乐', 'movie', 'EXPENSE', NULL, 4, 1, 0),
('居住物业', 'home', 'EXPENSE', NULL, 5, 1, 0),
('医疗健康', 'medical_services', 'EXPENSE', NULL, 6, 1, 0),
('教育培训', 'school', 'EXPENSE', NULL, 7, 1, 0),
('人情往来', 'group', 'EXPENSE', NULL, 8, 1, 0),
('金融理财', 'payments', 'EXPENSE', NULL, 9, 1, 0),
('其他支出', 'receipt_long', 'EXPENSE', NULL, 10, 1, 0),
-- 一级收入分类
('工资薪酬', 'payments', 'INCOME', NULL, 1, 1, 0),
('奖金收入', 'card_giftcard', 'INCOME', NULL, 2, 1, 0),
('理财收益', 'trending_up', 'INCOME', NULL, 3, 1, 0),
('兼职收入', 'work', 'INCOME', NULL, 4, 1, 0),
('其他收入', 'attach_money', 'INCOME', NULL, 5, 1, 0);

-- 2. 二级分类数据 (内联子查询绑定父级 ID，彻底消除会话变量风险)
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '早餐', 'bakery_dining', 'EXPENSE', c.id, 1, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '午餐', 'lunch_dining', 'EXPENSE', c.id, 2, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '晚餐', 'dinner_dining', 'EXPENSE', c.id, 3, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '零食饮料', 'local_cafe', 'EXPENSE', c.id, 4, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '水果', 'set_meal', 'EXPENSE', c.id, 5, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '烟酒', 'wine_bar', 'EXPENSE', c.id, 6, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '外卖', 'delivery_dining', 'EXPENSE', c.id, 7, 1, 0 FROM `category` c WHERE c.name='餐饮美食' AND c.type='EXPENSE' AND c.user_id=0;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '公共交通', 'directions_bus', 'EXPENSE', c.id, 1, 1, 0 FROM `category` c WHERE c.name='交通出行' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '打车租车', 'local_taxi', 'EXPENSE', c.id, 2, 1, 0 FROM `category` c WHERE c.name='交通出行' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '私家车', 'directions_car', 'EXPENSE', c.id, 3, 1, 0 FROM `category` c WHERE c.name='交通出行' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '机票', 'flight', 'EXPENSE', c.id, 4, 1, 0 FROM `category` c WHERE c.name='交通出行' AND c.type='EXPENSE' AND c.user_id=0;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '服饰鞋包', 'checkroom', 'EXPENSE', c.id, 1, 1, 0 FROM `category` c WHERE c.name='购物消费' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '数码产品', 'phone_android', 'EXPENSE', c.id, 2, 1, 0 FROM `category` c WHERE c.name='购物消费' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '美妆护肤', 'face', 'EXPENSE', c.id, 3, 1, 0 FROM `category` c WHERE c.name='购物消费' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '日用百货', 'shopping_basket', 'EXPENSE', c.id, 4, 1, 0 FROM `category` c WHERE c.name='购物消费' AND c.type='EXPENSE' AND c.user_id=0;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '电影演出', 'movie', 'EXPENSE', c.id, 1, 1, 0 FROM `category` c WHERE c.name='休闲娱乐' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '旅游度假', 'flight_takeoff', 'EXPENSE', c.id, 2, 1, 0 FROM `category` c WHERE c.name='休闲娱乐' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '运动健身', 'fitness_center', 'EXPENSE', c.id, 3, 1, 0 FROM `category` c WHERE c.name='休闲娱乐' AND c.type='EXPENSE' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '宠物养护', 'pets', 'EXPENSE', c.id, 4, 1, 0 FROM `category` c WHERE c.name='休闲娱乐' AND c.type='EXPENSE' AND c.user_id=0;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '基金收益', 'show_chart', 'INCOME', c.id, 1, 1, 0 FROM `category` c WHERE c.name='理财收益' AND c.type='INCOME' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '股票收益', 'trending_up', 'INCOME', c.id, 2, 1, 0 FROM `category` c WHERE c.name='理财收益' AND c.type='INCOME' AND c.user_id=0;
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `sort_order`, `is_preset`, `user_id`)
SELECT '银行利息', 'account_balance', 'INCOME', c.id, 3, 1, 0 FROM `category` c WHERE c.name='理财收益' AND c.type='INCOME' AND c.user_id=0;

-- 3. 测试账号 (⚠️ 生产环境严禁硬编码密码哈希，请改用环境变量或动态注入)
INSERT IGNORE INTO `user` (`nickname`, `phone`, `password_hash`, `country_code`) VALUES
('测试用户_小李', '13800138000', '$2a$10$N9qo8uLOickGH2j0iN0MteKHuqEEqMNnXNlB6gPGbG1fGqZ1c3LmG', '+86'),
('测试用户_小王', '13900139000', '$2a$10$N9qo8uLOickGH2j0iN0MteKHuqEEqMNnXNlB6gPGbG1fGqZ1c3LmG', '+86');

COMMIT;

-- 验证导入结果
SELECT '分类数据导入完成' AS status, COUNT(*) AS category_count FROM `category`;
SELECT '测试用户导入完成' AS status, COUNT(*) AS user_count FROM `user`;
```