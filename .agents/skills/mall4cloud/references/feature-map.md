# Mall4cloud 开源版现有功能点地图

启动失败、Nacos/验证码问题先读 `doc/2`、`doc/9`。问哪个服务、哪张表、哪个端口、哪个 Controller 时再读本文件。

路径相对 Mall4cloud 仓库根目录。跨服务契约在 `mall4cloud-api/`，公共能力在 `mall4cloud-common/`。表结构以 `db/*.sql` 为准。配置以 Nacos 的 `application.yml` + `mall4cloud-*.yml` 为准。

## 服务与端口

| 模块 | 端口 | 职责 |
| --- | --- | --- |
| `mall4cloud-gateway` | `8000` | 唯一前端入口，Nacos 路由 `/mall4cloud_xxx/**` |
| `mall4cloud-leaf` | `9100` | 分布式 ID |
| `mall4cloud-auth` | `9101` | 登录、Token、验证码 |
| `mall4cloud-rbac` | `9102` | 菜单、角色、按钮/接口权限 |
| `mall4cloud-multishop` | `9103` | 店铺、商家用户、热搜、轮播 |
| `mall4cloud-product` | `9104` | 商品、分类、品牌、SKU、库存、购物车 |
| `mall4cloud-user` | `9105` | C 端用户、地址、地区 |
| `mall4cloud-order` | `9106` | 确认/提交订单、我的订单、商家发货 |
| `mall4cloud-search` | `9108` | ES 搜索、Canal 消费 |
| `mall4cloud-platform` | `9112` | 平台用户、系统配置 |
| `mall4cloud-payment` | `9113` | 支付与支付通知 |
| `mall4cloud-biz` | `9000` | 附件、MinIO 上传 |

最小平台端：gateway + auth + rbac + platform + biz。最小商家端：gateway + auth + rbac + multishop + biz；商品页再加 product。下单再加 user、order、payment、leaf；搜索再加 search + ES + RocketMQ + Canal。

网关前缀示例：`/mall4cloud_auth`、`/mall4cloud_rbac`、`/mall4cloud_product`、`/mall4cloud_order`、`/mall4cloud_payment`、`/mall4cloud_biz`。Controller 常见 `/a/` C 端、`/m/` 商家、`/p/` 平台或个别 C 端接口。`/ua/**` 免登录。

## 前端

| 工程 | 角色 | 页面根 | 接口 |
| --- | --- | --- | --- |
| `front-end/mall4cloud-platform` | 平台后台 | `src/views/modules/` | `.env.*` → 网关 `8000` |
| `front-end/mall4cloud-multishop` | 商家后台 | `src/views/modules/` | 同上 |
| `front-end/mall4cloud-uniapp` | 用户端 | 先核对 `.env.*` | `VITE_APP_BASE_API` → 网关 |

动态菜单：`menu.component` → `src/views/modules/<component>/index.vue`。按钮：`v-permission` 对 `menu_permission.permission`。

## 登录与权限

| 能力 | 入口 | 库/表 |
| --- | --- | --- |
| 登录 / Token | `LoginController /ua/login`、`TokenStore`、`CaptchaController` | `mall4cloud_auth.auth_account` |
| 菜单路由 | `MenuController`：`/menu/route`、`/menu/list_menu_ids` | `mall4cloud_rbac.menu` |
| 按钮权限 | `MenuPermissionController` | `menu_permission` |
| 角色 | `RoleController` | `role`、`role_menu`、`user_role` |
| 平台用户 | `SysUserController` | `mall4cloud_platform.sys_user` |
| 商家用户 | `ShopUserController` | `mall4cloud_multishop.shop_user` |

`sys_type`/`biz_type`：`1` 商家，`2` 平台。C 端请求不做 RBAC。文档：`doc/4-技术实现/1-登录认证与权限链路.md`。

平台后台页：`rbac/menu`、`rbac/role`、`rbac/sys-user`、`rbac/menu-permission`、`rbac/shop-menu`。商家后台页：`rbac/role`、`rbac/shop-user`。

## 商品、购物车、库存

服务：`mall4cloud-product`。库：`mall4cloud_product`。

| 能力 | 后台 Controller | C 端 | 后台页 | 表 |
| --- | --- | --- | --- | --- |
| SPU | `admin/SpuController` | `app/SpuController` | 平台/商家 `product/list`；商家发布 `product/prod-info` | `spu`、`spu_detail` |
| SKU / 价格 | 随商品保存 | `SkuFeignController` 等 | `prod-info` | `sku`，价格用整数 `price_fee` |
| 库存 | `SkuStockLockController` | 下单 Feign 锁定 | — | `sku_stock`、`sku_stock_lock` |
| 分类 | `admin/CategoryController` | `app/CategoryController` | `product/category` | `category` |
| 品牌 | `admin`/`platform` `BrandController` | `app/BrandController` | 平台 `product/brand` | `brand` |
| 属性 | `admin/AttrController` | `app/AttrController` | `product/attr` | 属性相关表 |
| 购物车 | — | `app/ShopCartController` `/a/shop_cart` | — | `shop_cart_item` |

搜索列表还依赖 `mall4cloud-search` + ES（`es/product.md`）+ Canal + RocketMQ。MySQL 有商品不等于搜索有数据。

文档：`doc/6-核心业务/1-商品订单库存阅读路线.md`。

## 订单与支付

| 能力 | 入口 | 页面 | 库/表 |
| --- | --- | --- | --- |
| 确认订单 | `order` `app/OrderController` `POST /a/order/confirm` | uni-app 下单页 | Redis 确认缓存 |
| 提交订单 | `POST /a/order/submit` | 同上 | `order`、`order_item`、`order_addr` |
| 我的订单 | `app/MyOrderController` `/p/myOrder` | uni-app 订单 | 同上 |
| 商家订单/发货 | `multishop/OrderController` `/m/order` | 商家 `order/order` | 同上 |
| 支付 | `PayController` `/pay`，网关前缀 `/mall4cloud_payment/pay/order` | 支付结果 | `mall4cloud_payment.pay_info` |
| 支付通知 | `PayNoticeController` `/notice/pay` | — | 模拟成功后 MQ 通知订单/库存 |

订单状态 SQL 注释：`1` 待付款、`2` 待发货、`3` 待收货、`5` 成功、`6` 失败。提交后 `SkuStockLockFeignClient.lock`，并发 `ORDER_CANCEL_TOPIC` 延迟取消。当前支付直接调通知接口模拟成功；`PayController` 硬编码回调域名，不能当生产配置。

## 店铺与运营

| 能力 | 服务 | 后台页 | 说明 |
| --- | --- | --- | --- |
| 店铺 | `multishop` `ShopDetailController`（platform/multishop/app） | 平台 `platform/shop-manage` | `mall4cloud_multishop` |
| 热搜 | `admin`/`app` `HotSearchController` | 平台 `platform/hot-search`；商家 `multishop/hot-search` | |
| 轮播 | `IndexImgController` | `platform/index-img`、`multishop/index-img` | |
| 附件/OSS | `biz` `OssController`、`AttachFileController` | 各表单上传 | `mall4cloud_biz`；资源域名 `VITE_APP_RESOURCES_URL` 与 Nacos `biz.oss.resources-url` |

## 用户

`mall4cloud-user`：`app/UserController` `/a/user`、`UserRegisterController`、`UserAddrController`、`AreaController`；商家侧有 `multishop/UserController` `/m/user`。库 `mall4cloud_user`。

## 公共约定

- 分页：`PageDTO` + `PageUtil.doPage` → `PageVO`（`pages`、`total`、`list`），`pageSize` 上限 500。
- 失败多返回 HTTP 200，看业务 `code`；未授权常见 `A00004`。
- 跨服务写：已有 `@GlobalTransactional` 再套，不要只加本地 `@Transactional`。
- 上传：Nacos `biz.oss.*` 与前端 `VITE_APP_RESOURCES_URL` 对齐 MinIO `9000`/`mall4cloud`；控制台是 `9001`。

## 部署与已知坑

文档：`doc/8-部署运维/`、`doc/9-故障排查/1-启动登录接口常见问题.md`。

- 前端 404：`VITE_APP_BASE_API` 必须是网关 `8000`。
- 503：目标服务未注册 Nacos。
- Compose Nacos 账号与 bootstrap 默认值可能不一致。
- RocketMQ dashboard 无登录保护，不要公网暴露。
- 不要用错误 SQL `update mall4cloud_product set update_time = now()` 触发搜索同步。
---
