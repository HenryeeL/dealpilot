# DealPilot

一个基于 Spring Boot + Redis 的本地生活点评后端：商户查询、优惠券秒杀、手机号验证码登录、探店笔记、附近商户和用户签到。项目重点在用 Redis 解决高并发场景下的常见问题，比如缓存穿透、缓存击穿、分布式锁、全局唯一 ID 和超卖。

## 核心功能

- **短信验证码登录（Redis 共享 Session）**
  - 验证码存 Redis（`login:code:{phone}`，有效期 2 分钟）。
  - 登录成功后生成 UUID token，用户信息用 Hash 存在 `login:token:{token}`。
  - 双拦截器：`RefreshTokenInterceptor` 拦截所有请求，刷新 token 有效期并把用户放进 `ThreadLocal`；`LoginInterceptor` 拦截需要登录的接口，未登录返回 401。
  - 手机号首次登录会自动注册。
- **商户查询缓存（`CacheClient` 通用缓存工具）**
  - 缓存穿透：DB 查不到时缓存空值（2 分钟）。
  - 缓存击穿：提供两种方案。
    - 互斥锁：`SETNX` 实现，`queryWithMutex`，当前商户详情用的是这个。
    - 逻辑过期 + 线程池异步重建：`queryWithLogicalExpire`。
  - 数据一致性：更新商户时先写数据库再删缓存，并加了 `@Transactional`。
- **优惠券秒杀 / 一人一单**
  - 下单前校验秒杀开始、结束时间和库存。
  - 用乐观锁防超卖：`UPDATE ... SET stock = stock - 1 WHERE voucher_id = ? AND stock > 0`。
  - 用分布式锁实现一人一单：`SimpleRedisLock`，`SETNX` 加锁，锁值为「UUID + 线程 ID」；释放锁时用 Lua 脚本（`unlock.lua`）先比对再删除，保证原子性。
  - 也集成了 Redisson（`RedissonConfig`），代码里保留了基于 `RLock` 的实现。
  - 全局唯一订单 ID（`RedisIdWorker`）：31 位时间戳 + 32 位每日自增序列（Redis `INCR`）。
  - `seckill.lua` 已写好：Lua 原子校验库存和重复下单，并把订单写入 Redis Stream `stream.orders`。对应的异步下单代码目前是注释状态，当前生效的是同步下单流程。
- **附近商户（Redis GEO）**：`/shop/of/type` 传入经纬度时，按 `GEOSEARCH` 查 5 km 内的同类商户，按距离排序、分页，并返回距离。
- **用户签到（Redis BitMap）**：按「用户 + 年月」一个 key，`SETBIT` 签到；`BITFIELD` 统计本月截至今天的连续签到天数。
- **探店笔记**：发布笔记、点赞（数据库计数）、查询热门笔记和我的笔记，支持图片上传和删除。
- **其他**
  - 店铺类型列表。
  - 按名称搜索商户。
  - 查询店铺的优惠券（普通券 + 秒杀券）。
  - MyBatis-Plus 分页。
  - 全局异常处理。

## 技术栈

| 类别 | 技术 |
| --- | --- |
| 语言 / 框架 | Java 8、Spring Boot 2.3.12 |
| 持久层 | MySQL 8、MyBatis-Plus 3.4.3 |
| 缓存 / 中间件 | Redis（Spring Data Redis 2.6.2 + Lettuce 6.1.6）、Redisson 3.13.6 |
| 工具 | Hutool 5.7.17、Lombok |
| 构建 | Maven |

## 项目结构

```
dealpilot
├── pom.xml
└── src
    ├── main
    │   ├── java/com/dp
    │   │   ├── DianPingApplication.java   # 启动类
    │   │   ├── config/        # MVC 拦截器注册、MyBatis-Plus 分页、Redisson、全局异常处理
    │   │   ├── controller/    # REST 接口
    │   │   ├── service/       # 业务接口及实现（impl/）
    │   │   ├── mapper/        # MyBatis-Plus Mapper
    │   │   ├── entity/        # 数据库实体（tb_*）
    │   │   ├── dto/           # Result、LoginFormDTO、UserDTO、ScrollResult
    │   │   └── utils/         # CacheClient、SimpleRedisLock、RedisIdWorker、拦截器、UserHolder 等
    │   └── resources
    │       ├── application.yaml
    │       ├── mapper/VoucherMapper.xml
    │       ├── seckill.lua    # 秒杀资格校验 + 写入 Stream
    │       └── unlock.lua     # 分布式锁安全释放
    └── test/java/com/dp
        ├── DianPingApplicationTests.java  # ID 生成并发测试、商户 GEO 数据预热、HyperLogLog 等
        └── RedissonTest.java              # Redisson 可重入锁测试
```

## 快速开始

### 环境要求

- JDK 8+
- Maven 3.x
- MySQL 8（数据库名 `hmdp`）
- Redis 6.2+（附近商户用到了 `GEOSEARCH`）

### 1. 准备数据库

创建数据库 `hmdp`，并建好以下表。仓库里**没有附带建表 SQL**，需要自行准备：

`tb_user`、`tb_user_info`、`tb_shop`、`tb_shop_type`、`tb_voucher`、`tb_seckill_voucher`、`tb_voucher_order`、`tb_blog`、`tb_blog_comments`、`tb_follow`

### 2. 修改配置

`src/main/resources/application.yaml`：

```yaml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/hmdp?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: 123
  redis:
    host: 127.0.0.1
    port: 6379
    password: 123321
```

另外还有两处硬编码，需要按本机环境修改：

- `config/RedissonConfig.java`：Redisson 连接地址 `redis://127.0.0.1:6379`，密码 `123321`。
- `utils/SystemConstants.java`：图片上传目录 `IMAGE_UPLOAD_DIR`（默认是 `D:\lesson\nginx-1.18.0\html\hmdp\imgs\`）。

### 3. 运行

```bash
mvn spring-boot:run
# 或
mvn clean package
java -jar target/hm-dianping-0.0.1-SNAPSHOT.jar
```

启动后服务监听 `http://localhost:8081`。

### 4. 预热数据（可选）

- 附近商户功能依赖 Redis GEO 数据：运行测试 `DianPingApplicationTests#loadShopData`，把商户坐标按类型写入 `shop:geo:{typeId}`。
- `DianPingApplicationTests#testSaveShop` 会以逻辑过期方式写入商户缓存，用于测试逻辑过期方案。

## API 概览

统一返回格式：`{ "success": true, "errorMsg": null, "data": ..., "total": ... }`。
需要登录的接口要在请求头带上 `authorization: <token>`。

| 方法 | 路径 | 说明 | 需登录 |
| --- | --- | --- | --- |
| POST | `/user/code?phone=` | 发送验证码（打印在日志里，没有接短信服务） | 否 |
| POST | `/user/login` | 验证码登录，body：`{"phone","code"}`，返回 token | 否 |
| GET | `/user/me` | 当前登录用户 | 是 |
| GET | `/user/{id}` / `/user/info/{id}` | 用户信息 / 用户详情 | 是 |
| POST | `/user/sign` | 今日签到 | 是 |
| GET | `/user/sign/count` | 本月连续签到天数 | 是 |
| GET | `/shop/{id}` | 商户详情（走缓存） | 否 |
| POST / PUT | `/shop` | 新增 / 更新商户（更新时删缓存） | 否 |
| GET | `/shop/of/type?typeId=&current=&x=&y=` | 按类型分页查询；传 x、y 时按距离查附近商户 | 否 |
| GET | `/shop/of/name?name=&current=` | 按名称搜索商户 | 否 |
| GET | `/shop-type/list` | 店铺类型列表 | 否 |
| GET | `/voucher/list/{shopId}` | 店铺优惠券列表 | 否 |
| POST | `/voucher` | 新增普通券 | 否 |
| POST | `/voucher/seckill` | 新增秒杀券（同时写入秒杀表） | 否 |
| POST | `/voucher-order/seckill/{id}` | 秒杀下单，返回订单 ID | 是 |
| POST | `/blog` | 发布探店笔记 | 是 |
| PUT | `/blog/like/{id}` | 点赞 | 是 |
| GET | `/blog/hot?current=` | 热门笔记 | 否 |
| GET | `/blog/of/me?current=` | 我的笔记 | 是 |
| POST | `/upload/blog` | 上传笔记图片 | 否 |
| GET | `/upload/blog/delete?name=` | 删除笔记图片 | 否 |

## 测试

```bash
mvn test
```

测试类都是 `@SpringBootTest`，需要本地 MySQL 和 Redis 都已启动：

- `DianPingApplicationTests`
  - `testIdWorker`：300 个线程并发生成 3 万个订单 ID，统计耗时。
  - `loadShopData`：预热商户 GEO 数据。
  - `testHyperLogLog`：用 HyperLogLog 统计 100 万条数据的 UV。
- `RedissonTest`：验证 Redisson 锁的可重入性。

## 已知问题 / 待完善

- `VoucherMapper.xml` 的 `namespace` 仍是 `com.hmdp.mapper.VoucherMapper`，`application.yaml` 中的 `type-aliases-package` 和日志包名也还是 `com.hmdp`。包名已经改成 `com.dp`，所以要同步修改，否则 `/voucher/list/{shopId}` 会找不到对应的 SQL。
- `seckillVoucher` 在同一个类里直接调用 `@Transactional` 的 `createVoucherOrder`，事务不会通过代理生效。需要改成通过代理对象调用（例如 `AopContext.currentProxy()`）。
- 基于 Lua + Redis Stream 的异步秒杀（`seckill.lua`）还没有启用，Stream 消费者代码也没有实现。
- `/user/logout` 还没实现；关注（`/follow`）和评论（`/blog-comments`）只有空的 Controller。
- 仓库里没有建表 SQL 和前端资源，也没有 Docker / CI 配置。
