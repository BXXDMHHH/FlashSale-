# FlashSale

FlashSale 秒杀商城项目，目标技术路线：

- Android 用户端：Kotlin + Jetpack Compose
- 后端：JDK 17 + Spring Boot 3.5.x 兼容线 + MyBatis
- 管理端：Vue 3 + TypeScript
- 架构：DDD 思路下的模块化单体，REST API，关系型数据库

## 开发文档

- [生产开发设计文档（V1.0 设计基线）](docs/PRODUCTION_DEVELOPMENT_PLAN.md)
- [数据库、事务与并发实现设计](docs/DATABASE_TRANSACTION_DESIGN.md)

生产开发文档包含项目范围、系统架构、业务规则、DDD 聚合边界、REST API 草案、权限安全、测试验收和开发里程碑。数据库专项文档包含已确认技术决策、MySQL DDL 草案、索引/唯一约束、下单/支付/关闭事务 SQL、异常回滚规则和并发测试计划。

## 当前状态

仓库当前提交的是设计文档，不代表 Android、Spring Boot 或 Vue 功能代码已经实现。技术决策已确认；接下来按数据库专项设计实现迁移脚本与事务用例，并通过真实 MySQL 并发测试验证。文档仍是设计规格，业务代码尚未实现。
