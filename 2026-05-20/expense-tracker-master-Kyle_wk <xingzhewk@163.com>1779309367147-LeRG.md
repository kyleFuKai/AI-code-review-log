#OpenAi 代码评审.
### 😀代码评分：75
#### 😀代码逻辑与目的：
该段代码为“每日财务管家”系统的底层数据初始化脚本。`001_schema_ddl.sql` 负责构建用户、分类、账单、标签、预算及审计日志的核心数据模型，确立表结构、索引策略与引用关系；`002_seed_data.sql` 通过事务控制与变量动态绑定，以幂等方式注入系统预设分类与测试账户，旨在为业务逻辑提供基础运行环境与数据验证基准。
#### ✅代码优点：
1. 字符集与排序规则统一采用 `utf8mb4_0900_ai_ci`，完全兼容现代多语言及 Emoji 存储需求。
2. 核心业务表合理应用了复合唯一索引（如 `uk_user_type_parent_name`）与业务覆盖索引，有效避免数据冗余并提升查询命中率。
3. 种子数据采用 `START TRANSACTION` 包裹并配合 `INSERT IGNORE`，具备事务原子性与重复执行的安全边界。
4. 预算表针对性设计 `idx_budget_effective` 范围索引，精准命中周期拦截查询场景，体现业务思考。
#### 🤔问题点：
1. **致命语法兼容性**：`002_seed_data.sql` 中 `SELECT ... WHERE @p_x IS NOT NULL;` 缺失 `FROM` 数据源声明。该写法在 MySQL 严格模式（STRICT_TRANS_TABLES）或 8.0+ 环境中直接抛 `ERROR 1064`，属基础语法缺陷，将直接阻断初始化流程。
2. **严重安全违规**：硬编码测试账户的 Bcrypt 哈希值至版本控制系统。此行为严重违反安全基线，攻击者可利用公开 Hash 进行离线字典碰撞或凭证重放。测试凭证必须脱敏处理或动态注入。
3. **架构设计僵化**：物理外键（`RESTRICT`）与现代高并发/分布式架构天然冲突，极易引发死锁与写入性能瓶颈。且 `bill`、`user` 等核心表缺失 `is_deleted` 逻辑删除字段，无法满足企业级数据审计与误操作恢复要求。
4. **查询性能陷阱**：`category` 表采用纯邻接表模型（仅 `parent_id`），未提供路径冗余或层级深度字段。业务端执行多级树形渲染时将触发严重的递归风暴与 N+1 查询问题，数据量突破万级后将直接拖垮 CPU。
5. **边界条件模糊**：`uk_phone` 索引设计虽兼容 `NULL`，但 `phone` 字段未强制格式校验。缺乏输入边界拦截极易导致脏数据污染，且唯一索引在高频并发写入下可能产生死锁风险。
#### 🎯修改建议：
1. **修复语法缺陷**：为所有动态 `SELECT` 语句显式追加 `FROM DUAL`，确保跨版本 MySQL 及严格模式下的绝对兼容。
2. **消除安全隐患**：彻底移除硬编码密码 Hash，改为占位符并在部署阶段通过 CI/CD 环境变量动态生成。生产环境严禁保留任何测试凭证。
3. **解除物理外键与软删除**：移除 `CONSTRAINT` 声明，将引用完整性校验下沉至应用层/ORM。为高频操作表统一补充 `is_deleted` 字段及对应变体唯一索引，保障数据可追溯性。
4. **优化树形结构查询**：为 `category` 表引入 `path` 字段存储完整层级路径（如 `/0/1/5/`），利用 `LIKE 'path%'` 前缀匹配替代递归查询，彻底根除性能瓶颈。
5. **规范索引与空值语义**：明确区分 `0`（系统预设）与 `NULL`（未赋值）的业务含义；为高频联合查询创建包含 `is_deleted` 状态的覆盖索引，避免回表开销。
#### 💻修改后的代码：
```sql
-- ============================================================================
-- 每日财务管家 — V1.0 数据库建表脚本 (MySQL 8.0+)
-- 优化项：移除物理外键、增加软删除、补充层级路径、规范索引语义
-- ============================================================================

CREATE TABLE IF NOT EXISTS `user` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '用户 ID',
    `nickname`        VARCHAR(32)        NOT NULL DEFAULT '' COMMENT '昵称',
    `avatar_url`      VARCHAR(512)       NOT NULL DEFAULT '' COMMENT '头像 URL',
    `phone`           VARCHAR(20)        NULL DEFAULT NULL COMMENT '手机号 (业务层强制正则校验)',
    `country_code`    VARCHAR(6)         NOT NULL DEFAULT '+86' COMMENT '国家/地区区号',
    `password_hash`   VARCHAR(255)       NOT NULL COMMENT '密码哈希 (Bcrypt, 生产环境强制非空)',
    `currency`        CHAR(3)            NOT NULL DEFAULT 'CNY' COMMENT '默认货币单位',
    `theme`           VARCHAR(16)        NOT NULL DEFAULT 'light' COMMENT '主题模式',
    `status`          TINYINT UNSIGNED   NOT NULL DEFAULT 1 COMMENT '状态：0=禁用 1=正常',
    `is_deleted`      TINYINT UNSIGNED   NOT NULL DEFAULT 0 COMMENT '逻辑删除：0=正常 1=已删除',
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间',
    `updated_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_phone` (`phone`, `is_deleted`),
    KEY `idx_status_created` (`status`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='用户表';

CREATE TABLE IF NOT EXISTS `category` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '分类 ID',
    `name`            VARCHAR(32)        NOT NULL COMMENT '分类名称',
    `icon`            VARCHAR(64)        NOT NULL DEFAULT '' COMMENT 'Material Symbol 图标名',
    `type`            ENUM('EXPENSE','INCOME') NOT NULL DEFAULT 'EXPENSE',
    `parent_id`       BIGINT UNSIGNED    NOT NULL DEFAULT 0 COMMENT '父分类 ID (0代表根节点)',
    `path`            VARCHAR(255)       NOT NULL DEFAULT '/' COMMENT '层级路径 (如 /0/12/), 优化树形查询',
    `sort_order`      INT UNSIGNED       NOT NULL DEFAULT 0,
    `is_preset`       TINYINT UNSIGNED   NOT NULL DEFAULT 1 COMMENT '1=系统预设 0=用户自定义',
    `is_archived`     TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `user_id`         BIGINT UNSIGNED    NOT NULL DEFAULT 0 COMMENT '0 代表全局系统分类',
    `is_deleted`      TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_user_type_parent_name` (`user_id`, `type`, `parent_id`, `name`, `is_deleted`),
    KEY `idx_path` (`path`),
    KEY `idx_type_sort` (`type`, `is_preset`, `sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='收支分类表';

CREATE TABLE IF NOT EXISTS `bill` (
    `id`              BIGINT UNSIGNED    NOT NULL AUTO_INCREMENT,
    `user_id`         BIGINT UNSIGNED    NOT NULL,
    `type`            ENUM('EXPENSE','INCOME') NOT NULL DEFAULT 'EXPENSE',
    `amount`          DECIMAL(12,2)      NOT NULL COMMENT '金额 (正数入账, 负数退款)',
    `category_id`     BIGINT UNSIGNED    NOT NULL,
    `remark`          VARCHAR(200)       NOT NULL DEFAULT '',
    `bill_time`       DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `is_recurring`    TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `is_deleted`      TINYINT UNSIGNED   NOT NULL DEFAULT 0,
    `created_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at`      DATETIME           NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_user_time_del` (`user_id`, `bill_time`, `is_deleted`),
    KEY `idx_category_time` (`category_id`, `bill_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='账单记录表';

-- (bill_tag, bill_tag_rel, category_budget, review_log 结构同理需补充 is_deleted, 此处省略以聚焦核心)

-- ============================================================================
-- 预设分类数据 + 测试用户 (修复语法缺陷与安全合规)
-- ============================================================================

START TRANSACTION;

-- 一级分类注入 (补充 path 初始值)
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `path`, `sort_order`, `is_preset`, `user_id`) VALUES
('餐饮美食', 'restaurant', 'EXPENSE', 0, '/0/1/', 1, 1, 0),
('交通出行', 'directions_car', 'EXPENSE', 0, '/0/2/', 2, 1, 0),
('购物消费', 'shopping_bag', 'EXPENSE', 0, '/0/3/', 3, 1, 0),
('休闲娱乐', 'movie', 'EXPENSE', 0, '/0/4/', 4, 1, 0),
('理财收益', 'trending_up', 'INCOME', 0, '/0/5/', 5, 1, 0);

-- 动态绑定 (增加 FROM DUAL 确保严格模式兼容)
SET @p_food := (SELECT `id` FROM `category` WHERE `name`='餐饮美食' AND `type`='EXPENSE' AND `user_id`=0 AND `is_deleted`=0 LIMIT 1);
SET @p_transport := (SELECT `id` FROM `category` WHERE `name`='交通出行' AND `type`='EXPENSE' AND `user_id`=0 AND `is_deleted`=0 LIMIT 1);

-- 二级分类 (标准 FROM DUAL 语法, 自动计算 path)
INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `path`, `sort_order`, `is_preset`, `user_id`)
SELECT '早餐', 'bakery_dining', 'EXPENSE', @p_food, CONCAT('/0/1/', @p_food, '/'), 1, 1, 0 FROM DUAL WHERE @p_food IS NOT NULL;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `path`, `sort_order`, `is_preset`, `user_id`)
SELECT '午餐', 'lunch_dining', 'EXPENSE', @p_food, CONCAT('/0/1/', @p_food, '/'), 2, 1, 0 FROM DUAL WHERE @p_food IS NOT NULL;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `path`, `sort_order`, `is_preset`, `user_id`)
SELECT '公共交通', 'directions_bus', 'EXPENSE', @p_transport, CONCAT('/0/2/', @p_transport, '/'), 1, 1, 0 FROM DUAL WHERE @p_transport IS NOT NULL;

INSERT IGNORE INTO `category` (`name`, `icon`, `type`, `parent_id`, `path`, `sort_order`, `is_preset`, `user_id`)
SELECT '私家车', 'directions_car', 'EXPENSE', @p_transport, CONCAT('/0/2/', @p_transport, '/'), 2, 1, 0 FROM DUAL WHERE @p_transport IS NOT NULL;

-- 测试账号 (Hash 已移除，必须通过部署流程动态生成)
INSERT IGNORE INTO `user` (`nickname`, `phone`, `password_hash`, `country_code`, `status`) VALUES
('测试用户_小李', '13800138000', 'RUNTIME_HASH_PLACEHOLDER_SECURE', '+86', 1),
('测试用户_小王', '13900139000', 'RUNTIME_HASH_PLACEHOLDER_SECURE', '+86', 1);

COMMIT;

-- 验证
SELECT '数据初始化完成' AS status, 
       (SELECT COUNT(*) FROM `category` WHERE `is_preset`=1) AS category_count, 
       (SELECT COUNT(*) FROM `user` WHERE `is_deleted`=0) AS active_user_count;
```