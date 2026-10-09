# FlashSale 数据库、事务与并发实现设计

> 状态：V1.0 技术决策已确认，详细设计待代码实现与集成测试验证
> 适用范围：JDK 17、Spring Boot 3.5.x 兼容线、MySQL 8.x / InnoDB、MyBatis、Flyway
> 本文是实现规格，不代表 DDL、业务代码或测试已经实现。

## 1. 已确认的技术决策

| 项目 | 决策 |
|---|---|
| Java | JDK 17，编译目标 release=17 |
| Backend | Spring Boot 3.5.x 兼容线，锁定具体稳定补丁版本 |
| Database | MySQL 8.x，InnoDB，统一使用 UTC 保存时间 |
| Persistence | MyBatis + Flyway；不引入 JPA/Hibernate 作为第二套主方案 |
| Authentication | 短期 JWT access token + 数据库管理的 refresh token |
| Activity cancellation | 禁止新下单；关闭待支付订单并释放预占库存 |
| Purchase limit | 每用户每活动最多一件已支付商品；同时最多一笔待支付订单；关闭后可重新购买 |
| Mock payment | 仅本地、测试或受控演示环境开放 |
| Images | 开发环境本地存储；生产使用持久化目录或对象存储 |

### 1.1 依赖兼容性基线

- Spring Boot 3.5.x 要求 Java 17 或更高版本。
- MyBatis Spring Boot Starter 3.0.x 适用于 Spring Boot 3.2–3.5 和 Java 17+。不要把 Starter 4.x 与 Spring Boot 3.x 混用。
- 使用 Spring Boot dependency management 管理 Spring、Jackson、Validation、Security、JDBC 驱动等版本；不要随意覆盖其托管版本。
- MySQL 驱动使用 com.mysql:mysql-connector-j。
- Flyway 10+ 的 MySQL 支持需确认包含 org.flywaydb:flyway-mysql；Flyway 核心与数据库模块必须使用相同版本。
- 测试建议 JUnit 5、Spring Boot Test、Testcontainers MySQL。CI 必须用锁定版本执行构建和测试。
- Android 的 JDK/Gradle/Android Gradle Plugin 是独立兼容链，不能因后端使用 JDK 17 就推断 Android 工具链版本。

建议依赖类别：
- spring-boot-starter-web
- spring-boot-starter-validation
- spring-boot-starter-security
- spring-boot-starter-jdbc
- org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.x
- org.flywaydb:flyway-core + org.flywaydb:flyway-mysql（同版本）
- com.mysql:mysql-connector-j（运行时）
- 一个成熟 JWT 库，锁定版本，不自行实现 JWT 协议
- spring-boot-starter-test、Testcontainers MySQL（测试）

设计阶段不凭空锁定未经构建验证的所有补丁版本。Phase 0 应创建构建文件、执行依赖解析和 CI，再将具体版本写入构建文件与锁定配置。

## 2. 核心不变量

1. flash_sale_inventory 是活动库存权威计数；Redis（若未来加入）不能作为最终库存真相。
2. 下单、支付确认、订单关闭的库存变化必须与订单状态、购买资格、库存流水在同一个 MySQL 本地事务中提交。
3. 同一用户、同一活动最多一条购买资格记录；该行串行化待支付、已支付、已释放状态。
4. 幂等键只解决同一次请求重试；限购约束不同幂等键的竞争。
5. 所有扣减均使用条件 SQL 并检查 affected rows；不能只在 Java 中先读后改。
6. 重复操作必须安全；唯一约束是最后一道防线。
7. 数据库会话时区统一为 UTC；业务时间由服务端决定。
8. V1 每单数量固定为 1；支持多件时必须重新设计限购计数和库存语义。

库存等式：

    initial_quantity = available_quantity + reserved_quantity + sold_quantity

released_quantity 是累计审计指标，不参与上述等式。所有数量不得为负。

## 3. 表结构草案

以下为 MySQL 8.x DDL 设计草案。实际建表应通过 Flyway 迁移执行，并在 CI 中验证外键、CHECK、索引与 SQL 兼容性。

### 3.1 用户与认证

~~~sql
CREATE TABLE users (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  login_name VARCHAR(100) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_users_login_name (login_name),
  CONSTRAINT chk_users_status CHECK (status IN ('ACTIVE', 'DISABLED'))
) ENGINE=InnoDB;

CREATE TABLE roles (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  role_code VARCHAR(50) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_roles_role_code (role_code)
) ENGINE=InnoDB;

CREATE TABLE user_roles (
  user_id BIGINT UNSIGNED NOT NULL,
  role_id BIGINT UNSIGNED NOT NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (user_id, role_id),
  CONSTRAINT fk_user_roles_user FOREIGN KEY (user_id) REFERENCES users(id),
  CONSTRAINT fk_user_roles_role FOREIGN KEY (role_id) REFERENCES roles(id)
) ENGINE=InnoDB;

CREATE TABLE refresh_tokens (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  token_hash CHAR(64) NOT NULL,
  expires_at DATETIME(3) NOT NULL,
  revoked_at DATETIME(3) NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_refresh_tokens_hash (token_hash),
  KEY idx_refresh_tokens_user_expiry (user_id, expires_at),
  CONSTRAINT fk_refresh_tokens_user FOREIGN KEY (user_id) REFERENCES users(id)
) ENGINE=InnoDB;
~~~

安全规则：
- 密码只存成熟算法生成的密码哈希。
- 刷新令牌使用高熵随机值，数据库只存令牌哈希；刷新时轮换旧令牌并撤销旧值。
- JWT access token 采用短有效期；签名密钥通过环境变量/密钥管理提供，禁止提交仓库。
- JWT 在过期前的即时撤销需通过版本号/黑名单等额外机制，否则以短有效期控制窗口。

### 3.2 商品、活动与库存

~~~sql
CREATE TABLE products (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  description TEXT NULL,
  image_ref VARCHAR(1000) NULL,
  status VARCHAR(20) NOT NULL,
  version BIGINT UNSIGNED NOT NULL DEFAULT 0,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  KEY idx_products_status_created (status, created_at),
  CONSTRAINT chk_products_status CHECK (status IN ('DRAFT', 'PUBLISHED', 'UNPUBLISHED'))
) ENGINE=InnoDB;

CREATE TABLE flash_sale_activities (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  product_id BIGINT UNSIGNED NOT NULL,
  sale_price DECIMAL(12,2) NOT NULL,
  starts_at DATETIME(3) NOT NULL,
  ends_at DATETIME(3) NOT NULL,
  status VARCHAR(20) NOT NULL,
  purchase_limit INT UNSIGNED NOT NULL DEFAULT 1,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  KEY idx_activity_listing (status, starts_at, ends_at, id),
  KEY idx_activity_product (product_id),
  CONSTRAINT fk_activity_product FOREIGN KEY (product_id) REFERENCES products(id),
  CONSTRAINT chk_activity_price CHECK (sale_price >= 0),
  CONSTRAINT chk_activity_time CHECK (ends_at > starts_at),
  CONSTRAINT chk_activity_limit CHECK (purchase_limit = 1),
  CONSTRAINT chk_activity_status CHECK (status IN
    ('DRAFT', 'SCHEDULED', 'PUBLISHED', 'CANCELLED', 'ENDED'))
) ENGINE=InnoDB;

CREATE TABLE flash_sale_inventory (
  activity_id BIGINT UNSIGNED NOT NULL,
  initial_quantity INT UNSIGNED NOT NULL,
  available_quantity INT UNSIGNED NOT NULL,
  reserved_quantity INT UNSIGNED NOT NULL DEFAULT 0,
  sold_quantity INT UNSIGNED NOT NULL DEFAULT 0,
  released_quantity BIGINT UNSIGNED NOT NULL DEFAULT 0,
  version BIGINT UNSIGNED NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (activity_id),
  CONSTRAINT fk_inventory_activity FOREIGN KEY (activity_id)
    REFERENCES flash_sale_activities(id),
  CONSTRAINT chk_inventory_nonnegative CHECK (
    available_quantity >= 0 AND reserved_quantity >= 0 AND sold_quantity >= 0
  ),
  CONSTRAINT chk_inventory_conservation CHECK (
    initial_quantity = available_quantity + reserved_quantity + sold_quantity
  )
) ENGINE=InnoDB;
~~~

说明：
- activity_id 是库存表主键，确保每个活动只有一个库存池。
- CHECK 用于防止负数和破坏库存等式；并发正确性仍由事务 SQL 与行锁保证。
- 商品下架后服务端必须拒绝新下单。活动创建后不允许直接重写初始库存；库存调整必须通过受审计用例。

### 3.3 订单、购买资格与幂等记录

~~~sql
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  order_no VARCHAR(40) NOT NULL,
  user_id BIGINT UNSIGNED NOT NULL,
  activity_id BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED NOT NULL,
  product_name_snapshot VARCHAR(200) NOT NULL,
  unit_price DECIMAL(12,2) NOT NULL,
  quantity INT UNSIGNED NOT NULL DEFAULT 1,
  total_amount DECIMAL(12,2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  payment_deadline DATETIME(3) NOT NULL,
  paid_at DATETIME(3) NULL,
  closed_at DATETIME(3) NULL,
  close_reason VARCHAR(30) NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_orders_order_no (order_no),
  KEY idx_orders_user_created (user_id, created_at, id),
  KEY idx_orders_activity_user_status (activity_id, user_id, status),
  KEY idx_orders_timeout (status, payment_deadline, id),
  CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id),
  CONSTRAINT fk_orders_activity FOREIGN KEY (activity_id) REFERENCES flash_sale_activities(id),
  CONSTRAINT fk_orders_product FOREIGN KEY (product_id) REFERENCES products(id),
  CONSTRAINT chk_orders_amount CHECK (unit_price >= 0 AND total_amount >= 0),
  CONSTRAINT chk_orders_quantity CHECK (quantity = 1),
  CONSTRAINT chk_orders_status CHECK (status IN ('PENDING_PAYMENT', 'PAID', 'CLOSED'))
) ENGINE=InnoDB;

CREATE TABLE purchase_claims (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  activity_id BIGINT UNSIGNED NOT NULL,
  current_order_id BIGINT UNSIGNED NULL,
  state VARCHAR(20) NOT NULL DEFAULT 'RELEASED',
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_purchase_claim_user_activity (user_id, activity_id),
  KEY idx_purchase_claim_current_order (current_order_id),
  CONSTRAINT fk_purchase_claim_user FOREIGN KEY (user_id) REFERENCES users(id),
  CONSTRAINT fk_purchase_claim_activity FOREIGN KEY (activity_id)
    REFERENCES flash_sale_activities(id),
  CONSTRAINT fk_purchase_claim_order FOREIGN KEY (current_order_id) REFERENCES orders(id),
  CONSTRAINT chk_purchase_claim_state CHECK (state IN ('RESERVED', 'PURCHASED', 'RELEASED'))
) ENGINE=InnoDB;

CREATE TABLE idempotency_records (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  operation VARCHAR(64) NOT NULL,
  idempotency_key VARCHAR(128) NOT NULL,
  request_hash CHAR(64) NOT NULL,
  status VARCHAR(20) NOT NULL,
  resource_id BIGINT UNSIGNED NULL,
  response_code VARCHAR(64) NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  completed_at DATETIME(3) NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_idempotency_scope (user_id, operation, idempotency_key),
  KEY idx_idempotency_created (created_at),
  CONSTRAINT fk_idempotency_user FOREIGN KEY (user_id) REFERENCES users(id),
  CONSTRAINT chk_idempotency_status CHECK (status IN ('PROCESSING', 'SUCCEEDED'))
) ENGINE=InnoDB;
~~~

关键语义：
- purchase_claims 是当前购买资格，不是订单历史。PURCHASED 表示已成功购买，不得重新释放；RELEASED 可重新占用；RESERVED 表示当前订单待支付。
- 资格行在事务中通过 INSERT ... ON DUPLICATE KEY UPDATE 确保存在，再 SELECT ... FOR UPDATE 锁定。
- current_order_id 只能由当前资格持有订单更新；关闭旧订单时必须带 current_order_id = oldOrderId 条件，防止迟到任务释放新订单资格。
- 幂等表只记录已成功提交的下单结果；业务失败整体回滚，因此同键重试会重新评估业务条件。若未来要求失败结果也永久幂等，需另行定义失败结果持久化策略。
- request_hash 基于规范化后的业务请求字段计算，不应包含追踪字段。

### 3.4 支付尝试、库存流水与审计

~~~sql
CREATE TABLE payment_attempts (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  attempt_no VARCHAR(40) NOT NULL,
  order_id BIGINT UNSIGNED NOT NULL,
  attempt_number INT UNSIGNED NOT NULL,
  status VARCHAR(20) NOT NULL,
  result_code VARCHAR(64) NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  confirmed_at DATETIME(3) NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_payment_attempt_no (attempt_no),
  UNIQUE KEY uk_payment_order_attempt (order_id, attempt_number),
  KEY idx_payment_order_status (order_id, status),
  CONSTRAINT fk_payment_attempt_order FOREIGN KEY (order_id) REFERENCES orders(id),
  CONSTRAINT chk_payment_attempt_status CHECK (status IN
    ('CREATED', 'CONFIRMED', 'REJECTED', 'EXPIRED'))
) ENGINE=InnoDB;

CREATE TABLE inventory_transactions (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  activity_id BIGINT UNSIGNED NOT NULL,
  order_id BIGINT UNSIGNED NULL,
  operation_key VARCHAR(128) NOT NULL,
  transaction_type VARCHAR(20) NOT NULL,
  available_delta INT NOT NULL,
  reserved_delta INT NOT NULL,
  sold_delta INT NOT NULL,
  available_after INT UNSIGNED NOT NULL,
  reserved_after INT UNSIGNED NOT NULL,
  sold_after INT UNSIGNED NOT NULL,
  reason VARCHAR(64) NOT NULL,
  created_by BIGINT UNSIGNED NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  UNIQUE KEY uk_inventory_operation (activity_id, operation_key),
  KEY idx_inventory_order (order_id, created_at),
  KEY idx_inventory_activity_time (activity_id, created_at, id),
  CONSTRAINT fk_inventory_tx_activity FOREIGN KEY (activity_id)
    REFERENCES flash_sale_activities(id),
  CONSTRAINT fk_inventory_tx_order FOREIGN KEY (order_id) REFERENCES orders(id),
  CONSTRAINT chk_inventory_tx_type CHECK (transaction_type IN
    ('RESERVE', 'CONFIRM_SALE', 'RELEASE', 'ADMIN_ADJUST'))
) ENGINE=InnoDB;

CREATE TABLE audit_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  actor_user_id BIGINT UNSIGNED NULL,
  action VARCHAR(100) NOT NULL,
  target_type VARCHAR(64) NOT NULL,
  target_id VARCHAR(100) NOT NULL,
  result VARCHAR(20) NOT NULL,
  reason VARCHAR(500) NULL,
  correlation_id VARCHAR(128) NULL,
  before_state JSON NULL,
  after_state JSON NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (id),
  KEY idx_audit_target_time (target_type, target_id, created_at),
  KEY idx_audit_actor_time (actor_user_id, created_at),
  KEY idx_audit_correlation (correlation_id)
) ENGINE=InnoDB;
~~~

库存流水唯一操作键：
- 预占：RESERVE:<orderId>
- 支付确认：SALE:<orderId>
- 释放：RELEASE:<orderId>
- 管理调整：独立且唯一的调整单号

唯一键阻止同一业务操作重复记账；应用仍须先取得合法订单状态转换权。

### 3.5 索引设计

- 活动列表：idx_activity_listing(status, starts_at, ends_at, id)，用 EXPLAIN 验证查询计划。
- 用户订单：idx_orders_user_created(user_id, created_at, id)。
- 超时扫描：idx_orders_timeout(status, payment_deadline, id)。
- 幂等：唯一索引(user_id, operation, idempotency_key)。
- 限购：唯一索引(user_id, activity_id)。
- 库存：activity_id 主键。
- 库存流水：idx_inventory_activity_time(activity_id, created_at, id) 与唯一(activity_id, operation_key)。
- 不要为每个字段盲目加索引；库存表应保持索引精简。

## 4. 事务实现约定

### 4.1 通用约定

- 事务由应用层用例方法控制，例如 @Transactional(rollbackFor = Exception.class)。
- 核心表必须使用 InnoDB。
- 不要捕获数据库异常后吞掉并继续提交；业务异常应触发回滚或在事务边界外映射。
- 每个条件更新都检查 affected rows；预期为 1 而实际为 0 时，不能继续写流水。
- 对死锁、锁等待超时可有限次数重试整笔事务；重试复用相同幂等键，不能只重跑某条 SQL。
- 不在事务中执行远程 HTTP 请求、上传图片或等待用户交互。
- 实现阶段要按实际 SQL 顺序复核锁图，保持一致锁顺序并用并发测试检查死锁。

### 4.2 PlaceOrder：下单事务

推荐步骤：
1. 通过幂等唯一键获取请求处理权；同键并发请求等待首个事务提交或回滚。
2. 读取活动和商品可售状态，并采用与活动取消兼容的锁协议，避免取消提交后仍创建订单。
3. 创建或获取(user_id, activity_id)购买资格行并 SELECT ... FOR UPDATE。
4. PURCHASED 表示已买过，拒绝限购；RESERVED 表示已有待支付订单，拒绝重复下单；只有 RELEASED 可以继续。
5. 条件更新库存；库存不足时立即终止事务。
6. 插入订单快照，状态 PENDING_PAYMENT，payment_deadline = created_at + 15 minutes。
7. 将资格更新为 RESERVED 并写 current_order_id。
8. 写 RESERVE:<orderId> 库存流水；写幂等成功结果并关联订单 ID。
9. 提交后才向客户端返回订单。

原子库存预占 SQL：

~~~sql
UPDATE flash_sale_inventory
SET available_quantity = available_quantity - 1,
    reserved_quantity = reserved_quantity + 1,
    version = version + 1
WHERE activity_id = :activityId
  AND available_quantity >= 1;
~~~

资格行准备和锁定示意：

~~~sql
INSERT INTO purchase_claims (user_id, activity_id, state)
VALUES (:userId, :activityId, 'RELEASED')
ON DUPLICATE KEY UPDATE user_id = VALUES(user_id);

SELECT id, current_order_id, state
FROM purchase_claims
WHERE user_id = :userId AND activity_id = :activityId
FOR UPDATE;
~~~

幂等获取采用唯一键串行化：事务中插入 PROCESSING 记录并以 ON DUPLICATE KEY UPDATE 处理已有唯一键，然后锁定记录检查摘要与状态。成功结果和订单同一事务提交。若首个事务失败并回滚，等待者可重新取得处理权。不要在捕获重复键异常后继续使用已被 Spring 标记回滚的事务。

### 4.3 ConfirmPayment：支付确认事务

1. 锁定订单，校验订单归属、状态、截止时间和支付尝试。
2. 若订单已 PAID 且结果一致，幂等返回，不修改库存。
3. 只有 PENDING_PAYMENT 且尚未超过截止时间才能确认；CLOSED 或过期订单拒绝。
4. 条件更新取得状态转换权：

~~~sql
UPDATE orders
SET status = 'PAID', paid_at = CURRENT_TIMESTAMP(3)
WHERE id = :orderId
  AND status = 'PENDING_PAYMENT'
  AND payment_deadline > CURRENT_TIMESTAMP(3);
~~~

5. 校验购买资格仍为 RESERVED 且 current_order_id = orderId。
6. 原子转换库存：

~~~sql
UPDATE flash_sale_inventory
SET reserved_quantity = reserved_quantity - 1,
    sold_quantity = sold_quantity + 1,
    version = version + 1
WHERE activity_id = :activityId
  AND reserved_quantity >= 1;
~~~

7. 将资格更新为 PURCHASED，支付尝试更新为 CONFIRMED，写 SALE:<orderId> 库存流水。
8. 任一步失败，整笔事务回滚，订单不能单独变成 PAID。

### 4.4 CloseOrder：用户取消与超时关闭

1. 锁定订单；已 CLOSED 则幂等返回；已 PAID 则拒绝普通关闭。
2. 超时任务必须确认 payment_deadline <= CURRENT_TIMESTAMP(3)；用户取消按产品规则处理待支付订单。
3. 条件更新 PENDING_PAYMENT -> CLOSED，并检查受影响行数。
4. 将购买资格由 RESERVED 改为 RELEASED，必须满足 current_order_id = orderId。
5. 释放库存并写 RELEASE:<orderId> 流水：

~~~sql
UPDATE flash_sale_inventory
SET available_quantity = available_quantity + 1,
    reserved_quantity = reserved_quantity - 1,
    released_quantity = released_quantity + 1,
    version = version + 1
WHERE activity_id = :activityId
  AND reserved_quantity >= 1;
~~~

6. 订单状态、资格、库存和流水同一事务提交；任一步失败都回滚。
7. 旧订单任务不得清除新订单资格，资格更新必须带 current_order_id 条件。

### 4.5 活动取消

- 取消操作必须先取得活动排他锁，再标记 CANCELLED，阻止新下单。
- V1 规模可控时，在同一事务中找出该活动所有 PENDING_PAYMENT 订单，按固定顺序锁定并关闭，逐笔释放库存、释放匹配资格、写库存流水。
- 需要与 PlaceOrder 采用同一活动锁协议；普通无锁 SELECT 后更新活动状态不足以解决竞争。
- 事务规模必须设置上限并测试时长。若规模增长到不适合单事务，应改为“活动先停止接单 + 可恢复分批关闭”，并明确取消处理中状态；不能宣称批处理仍是全量原子操作。

## 5. 幂等与异常回滚规则

| 场景 | 处理 |
|---|---|
| 同键、同摘要、首次成功 | 保存订单 ID 与结果；重试返回同一结果 |
| 同键、不同摘要 | IDEMPOTENCY_KEY_CONFLICT；不改变库存/订单 |
| 同键并发到达 | 唯一键竞争串行化；首事务回滚后等待者可接管 |
| 不同键、同用户同活动并发 | 购买资格唯一行与行锁保证限购 |
| 库存预占失败 | 整笔事务回滚，不建单、不留资格占用或成功幂等结果 |
| 插入订单失败 | 库存和资格写入一并回滚 |
| 库存流水插入失败 | 整笔事务回滚，订单状态不能独自提交 |
| 返回客户端前连接断开 | 已提交则重试返回原订单；未提交则重试重新执行 |
| 支付与关闭竞争 | 只有一个状态转换成功；另一方读取终态后幂等返回或拒绝 |
| 死锁/锁超时 | 回滚整笔事务；有限次数重试同一幂等请求 |
| 重复释放/重复确认 | 状态条件与流水唯一键双重防护 |

幂等记录的 PROCESSING 状态应与下单处于同一事务，因此失败回滚不会留下永久 PROCESSING。若未来改为事务外创建处理中记录，必须另行设计租约、接管与修复机制。

## 6. 并发与集成测试计划

必须在 MySQL 8.x / InnoDB 上运行；H2 或纯 Mock 不能替代。Testcontainers 固定 MySQL 镜像版本，CI 确保能拉取镜像。

| ID | 测试操作 | 验收标准 |
|---|---|---|
| DB-01 | 库存 20，20 个不同用户成功下单 | 可售 0、预占 20、已售 0；20 个订单与预占流水 |
| DB-02 | 库存 1，100 个不同用户并发抢购 | 成功订单最多 1；库存非负且总量守恒 |
| DB-03 | 同用户、同活动、同键同请求 50 并发 | 只创建一单；成功响应订单 ID 一致 |
| DB-04 | 同用户、同活动、不同键并发 | 最多一笔待支付订单；库存只预占一次 |
| DB-05 | 同键但请求摘要不同 | 冲突请求拒绝；只有一个有效请求结果 |
| DB-06 | 预占后故意让订单插入失败 | 全回滚；库存、资格、订单、流水和幂等成功结果无残留 |
| DB-07 | 让库存流水写入失败 | 整笔回滚；不能存在缺少流水的已提交订单 |
| DB-08 | 支付成功后重复确认 20 次 | 只销售一次，只有一条销售流水 |
| DB-09 | 订单关闭后重复执行关闭任务 | 只释放一次，只有一条释放流水 |
| DB-10 | 同一订单同时支付与关闭 | 只有一个合法终态生效；库存与终态一致 |
| DB-11 | 旧订单关闭任务与新订单下单竞争 | 旧任务不能释放新订单资格 |
| DB-12 | 活动取消与并发下单竞争 | 取消提交后无新订单；原待支付订单关闭且预占释放 |
| DB-13 | 模拟支付超过截止时间 | 拒绝支付；关闭/释放可完成 |
| DB-14 | 用户查询他人订单 | 返回 403 或统一不可见响应，不泄露订单内容 |
| DB-15 | 库存对账 | 数量守恒；每笔预占最终有且仅有 SALE 或 RELEASE 终结流水 |
| DB-16 | 注入死锁或事务中断 | 完整回滚；重试无重复订单或库存变化 |

### 6.1 故障注入要求

- 通过测试专用 Mapper/测试钩子在库存更新后、订单插入后、流水写入时制造异常。
- 在事务代理调用边界外触发测试，确保验证真实 Spring 事务回滚。
- 用 CountDownLatch/屏障同步并发启动，不可用顺序执行伪装并发。
- 每个测试结束后直接查询数据库，不只断言 HTTP 状态码。
- 检查库存等式、非负约束、购买资格与订单状态、库存流水唯一性。
- 多轮重复运行并保留失败样例；一次本机通过不代表已证明性能目标。

## 7. 超时任务、监控和恢复

- 使用索引(status, payment_deadline, id) 分批扫描到期订单。
- 每个订单调用同一 CloseOrder 事务；任务重复执行必须安全。
- 单实例 V1 可使用 Spring 定时任务；多实例时可使用经验证的任务领取机制或 FOR UPDATE SKIP LOCKED，订单状态条件更新仍是最终保护。
- 日志记录 traceId、orderId、activityId、操作和错误码；禁止记录密码、JWT、刷新令牌。
- 提供库存对账报告：初始库存、可售、预占、已售、流水与异常订单。
- 数据修复必须通过受审计管理用例，不得无记录地手工改库存计数。

## 8. 开发顺序与完成条件

1. Phase 0：建立 JDK 17 / Spring Boot / MyBatis / Flyway 构建基线，锁定版本并使 CI 构建通过。
2. Phase 1：实现 Flyway 迁移与 Mapper，在 MySQL 测试 DDL。
3. Phase 2：实现库存条件更新与流水唯一键，完成 DB-01、DB-02。
4. Phase 3：实现购买资格与下单幂等，完成 DB-03 至 DB-07。
5. Phase 4：实现支付、关闭和活动取消事务，完成 DB-08 至 DB-13。
6. Phase 5：权限、对账、故障注入与端到端测试，完成 DB-14 至 DB-16。

每一阶段通过测试后再进入下一阶段。本文是设计规格，不代表业务代码已经完成。

## 9. 官方兼容性参考

- Spring Boot 3.5 系统要求：https://docs.spring.io/spring-boot/3.5/system-requirements.html
- MyBatis Starter 兼容矩阵：https://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/
- Flyway MySQL 支持：https://github.com/flyway/flyway/blob/main/documentation/Reference/Database%20Driver%20Reference/MySQL.md
