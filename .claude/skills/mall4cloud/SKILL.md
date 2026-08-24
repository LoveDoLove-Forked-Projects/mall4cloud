---
name: mall4cloud
description: "Mall4cloud 开源版现有功能副驾驶：Nacos/网关/多服务启动、平台端/商家端/uni-app、商品 SKU 库存、购物车、下单支付、订单发货、RBAC、搜索 Canal、Feign/Seata、部署排查。适用于下载 mall4cloud 后按现有微服务功能使用或二次开发；不要套用 mall4j 单体，不要编造开源版没有的能力。"
---

# Mall4cloud 开源版现有功能副驾驶

给已下载本仓库的人用：按**现有功能点**说明怎么用、改哪。细节以 `doc/` 和当前源码为准。路径相对仓库根目录。

这是 Spring Cloud **B2B2C 微服务**（Spring Boot 4、JDK 17、包名 `com.mall4j.cloud.*`）。使用通用 `SKILL.md`（Agent Skills），Cursor、Codex、Claude Code 均可加载。

## 每轮怎么做

1. 先判断：平台端 / 商家端 / C 端，以及启动、排查、部署，还是某个现有功能。
2. 只读下方路由该行的 1–3 篇 `doc/`，不要整目录读。
3. 问「哪个服务、哪张表、哪个 Controller、哪个端口」时，再读 [references/feature-map.md](references/feature-map.md)。Nacos 连不上、验证码失败仍先读排查文档，不要用地图代替日志。
4. 先说当前实现，再给基于现有文件的改法。跨服务改动要同时想到 Feign、库表和（如有）MQ/Seata。

## 快速路由

| 场景 | 最少阅读 |
| --- | --- |
| 第一次启动、登录后台 | `doc/2-环境搭建/1-环境要求.md`、`doc/2-环境搭建/2-数据库初始化.md`、`doc/2-环境搭建/3-后端启动.md` |
| 后台前端、图片、uni-app 地址 | `doc/2-环境搭建/4-管理后台启动.md`、`doc/2-环境搭建/5-移动端接口地址.md` |
| 中间件 Compose | `doc/8-部署运维/2-中间件-docker-compose/README.md` |
| Nacos/登录/401/验证码/503/搜索不到 | `doc/9-故障排查/1-启动登录接口常见问题.md` |
| 模块怎么分、改哪个服务 | `doc/1-项目概览/2-模块职责地图.md` |
| 分层、Feign、动态路由 | `doc/3-技术框架/1-后端分层与模块.md`、`doc/3-技术框架/2-前端工程与动态路由.md` |
| Token、菜单、按钮、接口权限 | `doc/4-技术实现/1-登录认证与权限链路.md` |
| 分页/校验/HTTP 200 业务码/Seata | `doc/4-技术实现/3-分页验证异常与事务.md` |
| 上传、MinIO、富文本 | `doc/4-技术实现/2-文件上传与富文本安全.md` |
| 商品 / 订单 / 库存 / 支付 | `doc/6-核心业务/1-商品订单库存阅读路线.md` |
| 消息、ES、Canal、库存锁定 | `doc/4-技术实现/4-消息搜索与库存链路.md` |
| 表在哪个库 | `doc/7-数据模型/2-权限商品订单核心表.md` |
| 改现有后台页、菜单不显示 | `doc/4-技术实现/1-登录认证与权限链路.md`、`doc/5-二次开发/2-命名约定与验收清单.md` |
| 部署上线、已知坑 | `doc/8-部署运维/1-部署路径与上线检查.md`、`doc/8-部署运维/3-已知风险与待核对项.md` |

## 固定口径

### 多服务、多端口、一个网关

后端是多个独立进程，各有端口并注册 Nacos（gateway `8000`，leaf `9100`，auth `9101`，rbac `9102`，multishop `9103`，product `9104`，user `9105`，order `9106`，search `9108`，platform `9112`，payment `9113`，biz `9000`）。完整表见 `feature-map.md`。

浏览器和 uni-app **只配网关 `8000`**。`VITE_APP_BASE_API` 不要写成 `9101` 等业务端口。网关按 Nacos 中 `mall4cloud-gateway.yml` 的前缀转发，例如 `/mall4cloud_rbac/**` → rbac。服务之间走 `mall4cloud-api-*` 的 Feign，不要拼对方 HTTP 地址。

这不是 mall4j 单体（`yami-shop-admin`/`yami-shop-api` 两个后端）。Mall4cloud 的「多端口」是微服务正常形态。

### 配置在 Nacos，库按服务拆

业务配置主要在 Nacos：公共 `application.yml` + 各服务 `mall4cloud-xxx.yml`。本地 `bootstrap.yml` / 网关 `application.yml` 只负责连 Nacos。`NACOS_HOST`/`NACOS_PORT` 以**当前源码默认值**为准（示例 IP 会变），和 Compose 初始化 SQL 里的地址必须一致。Compose 里 Nacos 控制台常见 `8080`，客户端 `8848`。初始化账号可能是 `nacos / 80jpnH4.r5g`，源码默认常写 `nacos/nacos`，启用鉴权时两边统一。不要无脑全仓替换某个示例 IP。

每个业务服务对应自己的 MySQL 库（`db/mall4cloud_auth.sql` 等）。改表只动该服务的库和 `db/` 脚本；登录失败先确认 auth、rbac、platform 或 multishop 是同一套脚本，不要混版本。

### 按需启动，不要一次拉全套

中间件最小：MySQL、Redis、Nacos、MinIO。  
后台能登录：gateway + auth + rbac + biz +（platform 或 multishop）。  
看商品：再加 product。下单支付：再加 user、order、payment，通常还要 leaf。跨服务写还要 Seata。搜索/发布后列表：再加 Elasticsearch、RocketMQ、Canal、search。

IDE 里一次起太多服务很吃内存；优先最小链路。JDK 17。前端必须 **pnpm**（禁止 npm/yarn），环境变量只有 `VITE_APP_*`。平台端和商家端 Vite 默认都是 `9527`，不能同时占同一端口。

平台/商家默认账号 `admin` / `123456`。`sys_type`/`biz_type`：`1` 商家端，`2` 平台端。

### 请求前缀和权限

常见 Controller 前缀：`/a/` C 端，`/m/` 商家端，`/p/` 平台或部分 C 端（如 `/p/myOrder`）。网关外再加服务前缀 `/mall4cloud_order` 等。

`/ua/**`、Swagger、内部 Feign 默认不登录。登录走 auth，`Authorization` 带 Token；业务服务 `AuthFilter` Feign 调 `checkToken`。平台/商家请求还要 RBAC（`menu` + `menu_permission.uri/method`）；C 端不做 RBAC。只加前端 `v-permission` 不够。

菜单 `component` → `src/views/modules/<component>/index.vue`。按钮编码 `<域>:<功能>:<动作>`，如 `product:category:save`。

### 改功能落哪

| 要改的 | 落点 |
| --- | --- |
| 平台用户、系统配置 | `mall4cloud-platform` + 平台前端 |
| 店铺、商家用户、热搜、轮播 | `mall4cloud-multishop` + 对应前端 |
| 菜单/角色/按钮 | `mall4cloud-rbac`，两套后台都可能改页 |
| 商品/SKU/库存/购物车 | `mall4cloud-product` |
| 确认单、提交单、发货 | `mall4cloud-order` |
| 支付 | `mall4cloud-payment` |
| C 端用户、地址 | `mall4cloud-user` |
| 上传 | `mall4cloud-biz` |
| 搜索索引 | `mall4cloud-search` + `es/` |

入参 DTO、出参 VO、表 Model。列表用 `PageDTO`/`PageVO`（`pageSize` 最大 500）。多数业务失败仍 HTTP 200，看返回 `code`（未授权常见 `A00004`）。跨服务写参考已有 `@GlobalTransactional`，不要只加本地事务。主键需要发号时用 leaf。

### 交易、库存、支付、搜索

`/a/order/confirm` 写入 Redis 确认缓存 → `/a/order/submit` 落 `order`/`order_item`/`order_addr`，Feign `SkuStockLockFeignClient.lock`，并发 `ORDER_CANCEL_TOPIC` 延迟取消（代码里 30 分钟档）。库存是 `sku_stock` + `sku_stock_lock`，不是只减一个字段。金额如 `price_fee` 用整数。

支付 `/mall4cloud_payment/pay/order`：建 `pay_info` 后直接调 `PayNoticeController.submit` 模拟成功，再 MQ 通知订单和扣库存。`PayController` 仍可能硬编码回调前缀，Nacos 另有 `application.domainUrl`。这是本地跑通，不是生产微信/支付宝。

搜索 ≠ MySQL 查询。商品发布后搜不到：查 Canal binlog、RocketMQ `canal-topic`、search 消费、ES mapping（`es/product.md`、`es/order.md`）。不要用错误 SQL `update mall4cloud_product set update_time = now()`。

上传：`/mall4cloud_biz/oss/info` 预签名直传 MinIO，或 `/oss/upload_minio`。`VITE_APP_RESOURCES_URL` 与 Nacos `biz.oss.resources-url` 都指向 MinIO **9000** 桶 `mall4cloud`；**9001** 只是控制台。富文本用 `v-rich`，不要裸 `v-html`。

## 不要编

不要按 mall4j 单体改模块名和「两个后端端口」。Mall4cloud 多服务多端口是正常的；错误是前端绕过网关，或把 `yami-shop-*` 套过来。

开源版没有：SaaS/多租户/跨境/分销/营销中台、完整退款结算教程、生产三方支付。用户要这些时说明现状即可。

分不清平台端还是商家端、前端是否打网关、要起哪些服务/中间件、要把模拟支付当三方上线、或能力不在现有功能点里：先停下来问一句。

不重写订单状态机和库存锁定；不把密钥写入仓库或前端；不把 `doc/` 全文贴进回答；生产问题不定责（RocketMQ dashboard 当前无登录保护，不要公网暴露）。
---
