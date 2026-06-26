# SeaControll

一个面向 `ESP32 / ESP01S` 的通用 IoT 控制平台，当前按以下方向开发：

- 多账户设备管理
- 用户名密码注册与登录
- 登录后绑定 `Passkey`
- 设备专属链接码绑定
- 设备端无密码 `AP` 配网
- 设备能力驱动化
- GPIO 端口绑定
- 读写分离的设备模型
- 星期定时器
- 内置继电器 / 造浪泵 / 温度计模块
- 通用公式与规则执行引擎
- 后续共享驱动市场

## 当前架构

### 运行层

- `backend/`：`Go` 后端，负责认证、设备绑定、命令规划、MQTT、后续数据库接入
- `pages/`：控制前端，挂载到 `Pages` 风格静态服务
- `firmware/arduino/seacontroll_device/`：设备端固件

### 数据层

当前最终决定：

- 主业务存储：`PostgreSQL`
- 灵活配置存储：`JSONB`
- 共享市场：同一个 `PostgreSQL` 实例中独立表组

也就是说：

- 用户
- 设备
- 驱动定义
- 驱动实例
- GPIO 绑定
- 定时器
- 设备状态
- 市场包
- 市场版本

都进入同一个数据库体系。

## 当前开发阶段

当前仓库已经具备：

- Go 后端基础骨架
- 用户名密码认证骨架
- 登录后 Passkey 绑定骨架
- 控制器链接码绑定骨架
- 前端控制台骨架
- 设备端 AP 配网页骨架
- 后端 Docker 本地开发环境
- PostgreSQL 初始表结构
- 账户 / 设备 / 控制器链接码 / 设备所有权的 PostgreSQL 持久化
- 星期定时器 / 设备状态快照的 PostgreSQL 持久化
- 驱动定义查询 / 设备驱动实例与 GPIO 绑定接口骨架
- 驱动实例到运行时 capability 的映射
- GPIO 冲突校验与继电器默认上电状态校验
- 造浪泵模块的定速 / 线性变速 / 正弦波 / 贝塞尔波 / 随机波 / 脉冲波控制

## 公式引擎设计方向

当前产品方向已经确认：

- 不在 `ESP32 / ESP01S` 上执行任意 `JavaScript`
- 不把所有高级能力都写死成固定内置模式
- 设备端只内置**轻量执行引擎**
- 平台下发**可组合配置**
- 后端负责保存、鉴权、下发，不负责替设备执行实时控制算法

### 目标

用一套通用的配置模型同时支持：

- 造浪强度渐强 / 渐弱 / 循环
- 温度 `C/F` 转换
- 传感器线性校准
- 条件判断联动
- 多模块共享同一套基础计算能力

### 引擎边界

- `ESP32`：支持通用公式与规则执行引擎
- `ESP01S`：只保留轻量能力，不承担复杂图执行
- `树莓派` 这类高性能设备后续可单独作为另一种平台类型扩展

### 执行模型

设备端后续实现统一的 `tick` 执行器，每个周期执行一次配置图。

基础能力拆为四层：

- 输入：常量、时间、传感器值、模块状态、用户参数
- 运算：`+ - * /`、`min`、`max`、`clamp`、`map`、`abs`、`round`
- 逻辑：`if`、比较、上下限反向、区间判断、与或非
- 输出：PWM、继电器状态、传感器换算结果

### 设备端内置原则

后续内置的是：

- 基础变量系统
- 基础数学算子
- 基础逻辑判断
- 基础波形算子
- 状态保持与周期执行器

后续**不内置大量固定业务模式**，而是通过组合算子完成造浪、温控、校准等需求。

### 配置方向

后续优先采用受限 `JSON DSL`，而不是脚本语言。

这样可以保证：

- 可校验
- 可存储
- 可版本化
- 可在前端做可视化编辑
- 可在设备端稳定执行

关于账户设置的当前决策：

- 普通用户注册 / 登录只走用户名密码
- Passkey 不放在登录页做首要流程
- Passkey 作为登录后的账户设置能力处理
- 本轮先保留数据库和接口支撑，不提前开发设置页 UI

当前仍然是**架构推进阶段**，还没有完全把运行时数据全部切到 PostgreSQL。  
也就是说：

- **数据库表结构已经建立**
- **账户 / 设备 / 控制器链接码 / 设备所有权已经接入 PostgreSQL**
- **设备定时器 / 状态快照已经接入 PostgreSQL**
- **驱动实例 / GPIO 绑定已经接入 PostgreSQL**
- **WebAuthn 临时会话、命令缓存目前仍以内存为主**
- **下一阶段继续替换剩余内存存储**

这是刻意的，目的是先把模型定稳，再切持久化，避免反复推翻。

## 本地开发环境

### 依赖

- Docker
- Docker Compose

### 启动后端容器

```bash
cd backend
docker compose up --build
```

启动后访问：

- 后端：`http://localhost:8080`
- PostgreSQL：`localhost:5432`
- Adminer：`http://localhost:8081`

说明：

- 容器现在只负责 `backend`、`postgres`、`adminer`
- 管理端由 Go 后端直接托管，入口是 `http://localhost:8080/admin`
- `pages/` 只负责用户端页面，不放进容器
- 设备端当然也不进入容器

### 启动 Go 设备仿真端

仿真端不进容器，直接本地运行：

```bash
cd firmware/device-sim
go run .
```

说明：

- 运行目录会自动生成 `device-sim.json`
- 首次启动如果没有配置，会自动生成随机 `deviceId`
- 首次启动会交互询问后端地址与控制器链接码
- `platform` 默认使用 `esp32`，需要时可在控制台执行 `setup` 修改
- 仿真端会把 `deviceToken` 写回 `device-sim.json`
- 二次启动如果已有 `deviceToken`，会直接当作已绑定设备启动
- 仿真端会：
  - 拉取 bootstrap
  - 连接 MQTT
  - 订阅下行命令
  - 输出执行日志
  - 周期上报设备状态
- 启动后可在控制台手动输入：
  - `setup`
  - `status`
  - `timers`
  - `report`
  - `bootstrap`
  - `relay <targetId> <on|off|toggle>`
  - `pwm <targetId> <duty>`
  - `wave <targetId> <direct|linearRamp|sineWave|bezierWave|randomWave|pulseWave|stop>`
  - `temp <targetId> <value>`
  - `command <json>`
- 仿真端会自动读取设备当前定时器，并在命中时间时本地执行并打印 `[定时器]` 日志

## 访问控制与角色边界

当前已经明确以下规则，后续开发必须遵守：

### 平台管理端 `admin`

- `admin` 只允许平台管理员访问
- `admin` 的职责只包括：用户列表、用户详情、指定用户名下设备列表、必要的平台级运维操作
- `admin` 不直接承担普通用户的设备控制、驱动配置、定时配置、设备配网页面
- `admin` 不应直接把普通用户侧的所有业务接口全部列出或暴露在页面上
- `admin` 首页只作为登录入口和导航壳，不把所有后台功能堆在单页中
- 后续后台功能按模块拆分独立页面，例如：`/admin/users`、`/admin/users/{id}`、`/admin/users/{id}/devices`

当前后台页面骨架已经拆为：

- `/admin`：管理员登录入口页
- `/admin/users`：用户列表页骨架
- `/admin/users/{id}`：用户详情页骨架
- `/admin/users/{id}/devices`：指定用户设备列表页骨架

首次部署流程固定为：

1. 系统检测是否已有账户
2. 如果没有任何账户，只允许显示“初始化管理员”
3. 首个管理员使用系统固定标识注册 Passkey，不需要填写管理员用户名
4. 首个管理员注册成功后，页面切换为“管理员登录”
5. 只有系统已初始化后，才允许显示登录入口

当前管理员固定标识规则：

- 不在页面暴露管理员用户名输入
- 初始化接口使用系统保留账户名 `__platform_admin__`
- 这样可以避免管理员用户名重复或被错误配置

### 用户端 `pages`

- `pages` 只面向普通用户
- `pages` 登录后只允许看到当前登录用户自己的设备和配置
- `pages` 不允许查看其他用户信息
- `pages` 后续所有业务页面默认都要先做鉴权，再加载数据
- `pages` 也必须按页面职责拆分，不采用“所有功能一个页面”的组织方式
- 用户设置类能力单独收敛到登录后的设置页，不放在登录页
- 设置页后续承载：修改密码、绑定 Passkey 等账户操作

### 未鉴权访问规则

- `admin` 和 `pages` 都不能在未鉴权状态下直接展示业务数据
- 未鉴权状态只允许显示登录入口、注册入口、必要的引导信息
- 用户列表、设备列表、驱动配置、定时器配置、状态查询都必须在鉴权成功后访问

### 角色模型约束

- 平台管理员和普通用户是两套职责边界
- 管理员看到的是“用户”和“用户拥有的设备”
- 普通用户看到的是“自己”和“自己的设备”
- 后续如果增加代操作、封禁、审计，必须单独设计，不直接复用普通用户页面

### 数据库默认账号

- Database：`seacontroll`
- User：`seacontroll`
- Password：`seacontroll`

### Passkey 本地开发配置

Docker 默认环境变量：

- `WEBAUTHN_RP_ID=localhost`
- `WEBAUTHN_RP_ORIGIN=http://localhost:8080`
- `WEBAUTHN_RP_NAME=SeaControll`

这意味着本地测试 `Passkey` 时，前端应从：

```text
http://localhost:8080
```

访问，而不是任意其它域名。

## 目录说明

```text
backend/
  cmd/server/main.go
  internal/
  migrations/
pages/
  index.html
  devices.html
  device.html
  device-drivers.html
  market.html
  market-package.html
  scripts/
  styles.css
firmware/
  arduino/seacontroll_device/
docs/
  DEVELOPMENT_PLAN.md
```

后端容器编排文件位于：

- `backend/docker-compose.yml`

## 数据库表设计

初始化 SQL 在：

- `backend/migrations/001_init.sql`

当前已建立以下核心表：

### 账户与认证

- `accounts`
- `account_passkeys`
- `account_sessions`

### 设备绑定

- `devices`
- `device_ownerships`
- `device_link_codes`

### 驱动模型

- `driver_definitions`
- `device_driver_instances`
- `gpio_bindings`

### 业务运行数据

- `device_timers`
- `device_state_snapshots`
- `device_event_logs`

### 共享市场

- `market_packages`
- `market_package_versions`
- `market_package_assets`
- `market_package_installs`

## 驱动模型方向

系统后续将围绕两层模型演进：

### 1. 驱动定义

描述一个驱动“是什么”，例如：

- 内置继电器
- 内置 MOS PWM
- DS18B20 温度计

驱动定义会描述：

- 驱动类型
- 支持 `read/write` 哪种操作
- 配置 schema
- 读取 schema
- 写入 schema
- 默认参数
- 适用平台

### 2. 驱动实例

描述某个设备上实际挂载的一个能力实例，例如：

- `relay_main`
- `pump_left`
- `water_temp_sensor`

驱动实例会绑定：

- 设备
- 驱动定义
- GPIO
- 用户配置
- 是否启用

当前已经补了这组接口骨架：

- `GET /api/drivers`
- `GET /api/devices/{deviceId}/drivers`
- `POST /api/devices/{deviceId}/drivers`

注意：

- 这些接口属于设备业务域，不代表要直接暴露给平台管理端页面
- 平台管理端和用户端后续会按角色拆分接口与页面能力

当前系统行为已经推进到：

- 如果设备存在驱动实例，则 `bootstrap` 优先返回驱动实例映射出的能力
- `bootstrap` 会直接下发 `driverInstances`，固件优先按驱动实例初始化通道
- 设备首次绑定后默认是空控制器，不带任何功能端口
- 保存驱动实例时会校验 GPIO 冲突
- 保存继电器驱动实例时会校验 `defaultPowerOnState` 与 `activeLevel`
- 内置驱动定义已统一补齐 `configSchema` / `readSchema` / `writeSchema`
- 内置驱动定义已补充 `builtinFunctions`，前端后续可直接按函数装配命令
- 设备状态上报与命令下发已切换到标准协议 `seacontroll.command.v1`
- 当前不保留旧协议兼容逻辑，设备端与后端按新协议直接交互

## 设备端预期流程

当前设备端已经按这个思路设计：

1. 设备首次启动，没有 Wi‑Fi 配置
2. 自动开启无密码 AP：`SeaControll-Setup`
3. 用户连接 AP，访问 `http://192.168.4.1`
4. 填写：
   - Wi‑Fi 名称
   - Wi‑Fi 密码
   - 服务端地址
   - 设备 ID
   - 控制器链接代码
5. 设备上报自身平台信息（`esp32` / `esp01s`）并通过链接代码向后端换取 `deviceToken`
6. 后端先完成控制器入网，不强制要求预选模板
7. 用户登录后进入设备详情页，再为该控制器添加继电器 / MOS / 温度计等功能模块
8. 设备使用 `deviceToken` 拉取 bootstrap 配置
9. 设备把 `timerGroups` 一并下发到本地，并按 `UTC+0` 自行执行星期制定时器
10. 设备接入 MQTT 并开始状态上报 / 指令接收

注意：

- 后端不负责实际执行定时器
- 后端只保存定时器配置并通过 bootstrap 下发给设备
- 前端提交定时器时，需要同时提交 `timezoneOffsetMinutes`，后端先把用户本地时间换算成 `UTC+0`
- 真实固件与 Go 仿真端都在设备侧本地执行定时器

## 当前内置驱动方向

数据库初始化时已预置三类驱动定义：

- `relay_builtin`
- `mos_pwm_builtin`
- `ds18b20_builtin`

这只是**驱动定义层**，不是完整运行时驱动市场。

当前已经内置的可复用功能包括：

- `relay_builtin`
  - `turnOn`
  - `turnOff`
  - `toggle`
- `mos_pwm_builtin`
  - `stop`
  - `maxPower`
  - `softStart`
  - `softStop`
  - `pulse`
- 模板虚拟组能力
  - `alternatingWave`

其中：

- `softStart` / `softStop` 用于泵类平滑变速
- `pulse` 用于快速脉冲式输出
- `alternatingWave` 会由后端展开成双通道 `sequenceGroup`，适合造浪泵交替波形

## 共享市场当前进度

当前已经具备：

- 市场包创建
- 版本发布
- 包级发布 / 下架
- 安装记录保存
- 版本兼容性校验
- 管理员审核通过 / 驳回

当前接口包括：

- `POST /api/market/packages`
- `GET /api/market/packages`
- `POST /api/market/packages/{id}/versions`
- `GET /api/market/packages/{id}/versions`
- `POST /api/market/packages/{id}/publish`
- `POST /api/market/packages/{id}/unpublish`
- `POST /api/market/packages/{id}/installs`
- `GET /api/market/packages/{id}/installs`
- `POST /api/admin/market/packages/{packageId}/versions/{versionId}/review`

当前规则：

- 市场包版本提交时必须带 `compatibilityJson.platforms`
- 市场包版本默认进入 `pending` 审核状态
- 只有 `approved` 版本才允许安装
- 设备状态页进入时会先异步下发一次 `reportState` 查询，再等待设备回传最新状态

## 当前最重要的下一步

按当前计划，接下来优先做：

1. 把内存存储替换为 PostgreSQL
2. 正式落地“驱动定义 / 驱动实例 / GPIO 绑定”
3. 完成控制器空白入网后的驱动实例化配置
4. 加入只读传感器驱动示例

## 开发计划

详细开发计划见：

- `docs/DEVELOPMENT_PLAN.md`

## 当前验证方式

后端静态验证：

```bash
cd backend
GOCACHE=/tmp/seacontroll-gocache go test ./...
```

如果要显式本地运行后端并连接数据库，设置：

```bash
DATABASE_URL=postgres://seacontroll:seacontroll@localhost:5432/seacontroll?sslmode=disable
```

前端静态验证：

```bash
node --check pages/scripts/common.js
node --check pages/scripts/index.js
node --check pages/scripts/devices.js
node --check pages/scripts/device.js
```

前端本地运行示例：

```bash
cd pages
python3 -m http.server 4173
```

补充说明：

- `pages/` 现在已经拆为登录页、设备列表页、设备详情页、驱动配置页、市场页
- 管理端请直接访问 `http://localhost:8080/admin`
- `pages/devices.html` 与 `pages/device.html` 会先校验登录态，再加载业务数据
- `pages/device.html` 现在只负责设备概览与模块列表展示
- `pages/module.html` 负责单个模块的状态查看、快捷控制、高级控制和星期定时配置
- `pages/device-drivers.html` 用于按模块类型添加模块、配置端口绑定，并管理当前设备下的模块列表
- `pages/market.html` 与 `pages/market-package.html` 用于浏览市场包、发布版本和安装版本

## 说明

当前版本重点是把：

- 开发方向
- 数据模型
- 本地运行环境
- 数据库存储策略

先固定下来。

这一步比继续堆功能更重要。后面我们会在这个基础上继续把业务实现补实。
