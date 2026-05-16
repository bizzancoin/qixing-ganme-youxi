# 七星跨平台地方棋牌源码游戏

> **文档说明**：本文档已与仓库当前结构、`Server-ALL` 后端入口及 `Server-ALL/common/lib/GlobalValues.js` 中 `GAME_TYPE` 枚举逐项核对（常量 **272** 项，其中数值型 `gameType` **271** 个互异：`2018095` 被两条常量共用）。修订日期：**2026-05-13**。

## 快速导航

| 章节 | 内容概要 |
|------|----------|
| [项目简介](#项目简介) | 仓库组成与后端定位 |
| [技术栈](#技术栈) | 框架、存储、中间件与部署 |
| [项目特性](#项目特性) | 各服务与业务能力要点 |
| [游戏列表](#游戏列表) | `GAME_TYPE` 全表与房间通用能力 |
| [项目目录结构](#项目目录结构) | 顶层与 `Server-ALL` 目录树 |
| [环境变量与配置](#环境变量与配置) | `NODE_ENV`、`NODE_API` 等 |
| [API 与路由结构](#api-与路由结构) | Express / Pomelo 路由入口 |
| [启动方式](#启动方式) | Pomelo 与各 `main.js` |
| [数据库与持久化](#数据库与持久化) | MySQL / Mongo / Redis / SQL 脚本 |
| [部署与运维说明](#部署与运维说明) | 进程、脚本、CI/CD 现状 |

## 项目简介

本仓库为棋牌类客户端与配套服务的完整工程集合。根目录 `说明.txt` 标明：包含 frameworks 打包、client 客户端、服务端、多套 UI、更新与数据库脚本等。

**后端主体**位于 `Server-ALL/`：以 **Pomelo 2.2.7** 分布式游戏服（`game-server`）为核心，配合多个独立 **Node.js + Express** 进程（运营后台 CMS、代理/会员端、商城、轻量 API、定时任务等），通过 **MySQL、MongoDB、Redis** 及 **微信相关配置** 完成业务与支付能力。仓库内还包含 `admin/` 下的统计、日志、监控等附属 Node 服务，以及 `Server-QX备份`、`Server-XL备份` 等历史备份目录。

适用场景：需要二次开发或交付验收的 **房卡/亲友圈/联盟类棋牌** 服务端与周边工具；新成员可从本文档定位入口与依赖，再结合具体环境配置联调。

## 技术栈

| 类别 | 实际代码依据 |
|------|----------------|
| **编程语言** | JavaScript（Node.js） |
| **游戏与实时通信框架** | Pomelo `2.2.7`（见 `Server-ALL/game-server/package-lock.json`），底层依赖含 `socket.io`、`ws` 等 |
| **HTTP 服务框架** | Express（`api/main.js`、`cms/main.js`、`member/main.js`、`mall/main.js`、`game-server/app/http/app.js` 等） |
| **数据库** | **MySQL**（`Server-ALL/common/mysqlUtil.js`）、**MongoDB**（官方 `mongodb` 驱动封装于 `Server-ALL/common/mongoUtil.js`） |
| **ORM / ODM** | 无统一 ORM；为自封装 `mysqlUtil` / `mongoUtil` 直接执行查询与更新 |
| **缓存** | **Redis**（`redis` 包，`Server-ALL/common/redisUtil.js`，含分布式锁等） |
| **消息队列** | **Kafka**（`kafka-node`，`Server-ALL/common/kafkaUtil.js`；仅在部分环境名如 `yueyang`、`shaoyang` 下初始化客户端） |
| **任务调度** | `node-schedule`（如 `game-server/init.js`、`member/init.js`）；定时业务集中在 `timer/main1.js` 等文件中显式调用各 `modules.*` |
| **鉴权与安全** | 各服务中 Express 中间件与业务校验（如 CMS：`modules.login.check`）；配置中含 RSA `privateKey`/`publicKey`（`game-server/config/index.js`、`api/config/index.js`）；游戏 HTTP 中 `modules.common.checkLocal` 等（如 `debug` 路由） |
| **文件上传** | `formidable`（CMS、Member 等 `init.js`） |
| **第三方与基础设施** | 微信支付等（`Server-ALL/common/payUtil.js` 调用微信商户 API）；**阿里云 OSS**（`ali-oss`，CMS/Member 等）；**钉钉**（`Server-ALL/common/dingdingUtil.js`）；邮件（`Server-ALL/common/mailUtil.js`）；短信（`Server-ALL/common/smsUtil.js`）；**ClickHouse**（`Server-ALL/common/clickhouseUtil.js`，按需使用） |
| **部署相关** | 游戏服提供 Bash 脚本 `game-server/run.sh`（`curl` 解散房间、`lsof`+`kill` 清端口、`pomelo start -e $NODE_ENV`）；仓库**未发现**项目根级自有 `Dockerfile` / `docker-compose`（仅第三方依赖内示例镜像） |

## 项目特性

以下内容均可在对应目录的 `init.js`、`main.js`、`modules/`、`routes/` 或 Pomelo `app/servers/*` 中找到实现或入口，而非泛泛描述。

- **分布式游戏服**：多进程类型 `pkmaster`、`pkcon`（前端 `clientPort`）、`pkplayer`、`pkclub`、`pkleague`、`pkroom`（`game-server/config/servers.json` 按 `NODE_ENV` 区分多套进程与端口）。
- **游戏房间与玩法逻辑**：`pkroom` 下含麻将/扑克等子目录（如 `app/servers/pkroom/games/`，配置引用如 `GameCfg-guizhou`）；**全部玩法 ID 与名称**见下文《游戏列表》章。
- **内嵌 HTTP 服务（游戏进程同仓）**：`game-server/app/http/` 提供静态资源、微信校验根路径、以及 `debug`、`user`、`club`、`league`、`recharge`、`status`、`third` 等路由模块。
- **运营 CMS**：`cms/` — Express + 静态 `web`、登录校验、多路由挂载；集成 MySQL 多实例、Redis、OSS、邮件等。
- **代理 / 会员 Web 服务**：`member/` — Express、`mPort` 监听；Mongo + MySQL + Redis；部分环境初始化排行榜等逻辑。
- **商城服务**：`mall/` — Express + 自维护 `mysqlUtil`/`redisUtil`/`s3Util` + OSS；独立 `config`。
- **对外轻量 API**：`api/` — 端口默认 `10000`（`api/config/index.js`），路由前缀挂载 `common`、`weixin`、`server`（如 `/server/info`、`/server/heartbeat`、`/weixin/miniprogram/info`）。
- **定时与批处理**：`timer/` — `main1.js` / `main2.js` / `main3.js` 分别拉起俱乐部统计、房卡欠款、红包、联盟排行、商务报备、订单归属等大量定时任务（见 `main1.js` 内注释时间点）。
- **公共库**：`Server-ALL/common/` — 配置模板、工具类、支付、消息、Kafka/Redis/MySQL/Mongo 封装等，被各子服务 `require('../common/...')` 引用。
- **扩展与场服变体**：`game-field`、`timer-field`、`cms-field` 与同名的 `game-server`、`timer`、`cms` 结构类似，用于分场/分环境部署（命名与 Pomelo 场常见做法一致）。
- **微信机器人 / 工具**：`wechaty/`（独立 Wechaty 相关工程）。
- **管理端与统计（admin）**：如 `admin/statis-main`、`admin/logger`、`admin/monitor` 等 Node 子项目（含各自 `package.json` 与业务脚本）。
- **数据库脚本**：根目录 `数据库/` 含 `数据库.sql`、`dev备份.sql`；`admin/statis-main/sql/` 等另有统计相关 SQL。

## 游戏列表

### 玩法标识与源码位置

- **全局玩法枚举**：`global.GAME_TYPE` 定义于 `Server-ALL/common/lib/GlobalValues.js`（由 `Server-ALL/common/static.js` 引入的 `./lib/GlobalValues` 加载），每个玩法对应**数值型 `gameType`** 与 **常量名**，客户端建桌、服务端路由与统计均依赖该 ID。
- **房间逻辑实现目录**：`Server-ALL/game-server/app/servers/pkroom/games/` 下按大类分子目录：`majiang`（麻将类）、`zipai`（字牌/跑胡子等）、`poker`（扑克类）、`doudizhu`（斗地主各地方变体）、`other`（其它）。具体玩法由 `gameType` 映射到对应 `GameCode*` / 规则脚本。
- **地区规则包**：同目录下多套 `GameCfg-<地区>.js`（如 `GameCfg-yueyang.js`、`GameCfg-guizhou.js` 等），与 `NODE_ENV` / 运营区域配合；游戏进程入口对 `GAME_CFG` 的引用见 `game-server/init.js`。

### 全部玩法一览（`GAME_TYPE`）

下表共 **272** 条数据行（表头除外）。对应源码中 **272** 个 `GAME_TYPE` 常量；其中 **271** 个不同的 `gameType` 数值（`KOU_DIAN_FENG_ZUI_ZI` 与 `LIN_FEN_KOU_DIAN_FENG_ZUI_ZI` 共用 `2018095`）。第三列为源码行尾注释原文。

| 玩法ID | 常量名 | 玩法说明（源码注释） |
| ---: | --- | --- |
| 102017036 | PAO_DE_KUAI_BW | 百万跑得快 |
| 0 | LIAN_YUN_GANG | 连云港 新浦麻将 |
| 10001 | MINI_GAME_DSQ | 小游戏 |
| 2017001 | SHU_YANG | 沭阳 |
| 2017002 | GUAN_YUN | 灌云 |
| 2017003 | DONG_HAI | 东海 |
| 2017004 | NIU_NIU | 牛牛 |
| 2017005 | NAN_JING | 南京 |
| 2017006 | SU_QIAN | 宿迁 |
| 2017007 | HUAI_AN | 淮安 楚州麻将 |
| 2017008 | HA_14DUN | 淮安十四墩 |
| 2017009 | HA_HONGZHONG | 淮安红中 自由麻将 |
| 2017010 | XU_ZHOU | 徐州 |
| 2017011 | TUAN_TUAN_ZHUAN | 掼蛋 |
| 2017012 | PAO_HU_ZI | 扯胡子三人场 扯胡子三人场 |
| 2017013 | SI_YANG | 泗阳 |
| 2017014 | SI_YANG_HH | 泗阳晃晃 |
| 2017015 | YAN_CHENG_HH | 盐城晃晃 |
| 2017016 | RU_GAO | 如皋 如皋长牌 |
| 2017017 | GAN_YU | 赣榆 |
| 2017018 | HUAI_AN_TTZ | 淮安团团转 团团转麻将 |
| 2017019 | HUAI_AN_CC | 淮安出铳 |
| 2017020 | RU_GAO_MJH | 如皋麻将胡 |
| 2017021 | GUAN_NAN | 灌南 |
| 2017022 | PAO_DE_KUAI | 跑得快 |
| 2017023 | XIN_PU_HZ | 红中麻将 |
| 2017024 | LUO_DI_SAO | 落地扫 |
| 2017025 | PAO_HU_ZI_SR | 扯胡子四人场 坐醒四人场 |
| 2017026 | DOU_DI_ZHU_NT | 南通斗地主 |
| 2017027 | NTHZ | 南通红中/十三张麻将 |
| 2017028 | ZP_LY_CHZ | 字牌耒阳扯胡子 耒阳字牌 |
| 2017029 | TONG_HUA | 通化麻将 |
| 2017030 | PAO_HU_ZI_King | 扯胡子四王三人场 四王扯胡子 |
| 2017031 | PAO_HU_ZI_SR_King | 扯胡子四王坐醒 |
| 2017032 | CHANG_SHA | 长沙麻将 |
| 2017033 | LIAN_SHUI | 淮安涟水麻将 |
| 2017034 | DOU_DI_ZHU_TY | 通用斗地主 |
| 2017035 | LEI_YANG_GMJ | 鬼麻将 |
| 2017036 | PAO_DE_KUAI_TY | 通用跑得快 |
| 2017037 | TY_HONGZHONG | 湖南通用红中麻将 |
| 2017038 | YZ_PAO_DE_KUAI_TY | 永州通用跑得快 |
| 2017039 | TY_ZHUANZHUAN | 通用转转 |
| 2017040 | HUAI_AN_ERZ | 淮安二人转 |
| 2017041 | PAO_HU_ZI_LR | 扯胡子单挑玩法 |
| 2017042 | SAN_DA_HA | 三打哈 |
| 2017043 | HY_LIU_HU_QIANG | 衡阳六胡抢 |
| 2017044 | HY_SHI_HU_KA | 衡阳十胡卡 |
| 2017045 | BAI_PU_LIN_ZI | 白蒲林梓长牌 |
| 2017046 | RU_GAO_SHUANG_JIANG | 双将长牌 |
| 2017047 | PAO_HU_ZI_LR_King | 四王单挑玩法 |
| 2017048 | HAI_AN_MJ | 海安麻将 |
| 2017049 | HAI_AN_BAI_DA | 海安白搭麻将 |
| 2017050 | JIN_ZHONG_MJ | 晋中麻将 |
| 2017051 | PAO_DE_KUAI_NT | 南通跑得快 |
| 2017052 | YONG_ZHOU_MJ | 永州麻将 |
| 2017053 | ML_HONGZHONG | 汨罗红中 |
| 2017054 | HZ_TUI_DAO_HU | 洪泽推到胡 |
| 2017055 | DOU_DI_ZHU_JZ | 晋中斗地主 |
| 2017056 | DOU_DI_ZHU_HA | 海安斗地主 |
| 2017057 | PAO_DE_KUAI_HA | 淮安跑得快 |
| 2017058 | PAO_DE_KUAI_JZ | 晋中跑得快 |
| 2017059 | JIANG_HUA_MJ | 江华麻将 |
| 2017060 | JIANG_YONG_15Z | 江永15张 |
| 2017061 | DAO_ZHOU_MJ | 道州麻将 |
| 2017062 | RU_DONG_SHUANG_JIANG | 如东双将 |
| 2017063 | PAO_DE_KUAI_LYG | 连云港跑得快 |
| 2017064 | PAO_DE_KUAI_XU_ZHOU | 徐州港跑得快 |
| 2017065 | JIN_ZHONG_KD | 晋中扣点 |
| 2018066 | PAO_DE_KUAI_HAIAN | 海安跑得快 |
| 2018067 | DA_TONG_ZI_SHAO_YANG | 邵阳打筒子 |
| 2018068 | RU_GAO_ER | 如皋长牌，二人玩法 |
| 2018069 | JIN_ZHONG_TUI_DAO_HU | 晋中推倒胡 |
| 2018070 | LING_SHI_BIAN_LONG | 灵石编龙 |
| 2018071 | LING_SHI_BAN_MO | 灵石半摸 |
| 2018072 | PING_YAO_MA_JIANG | 平遥麻将 |
| 2018073 | PING_YAO_KOU_DIAN | 平遥扣点 |
| 2018074 | SHAO_YANG_BO_PI | 邵阳剥皮 |
| 2018075 | JIE_XIU_1_DIAN_3 | 介休1点3 |
| 2018076 | JIE_XIU_KOU_DIAN | 介休扣点 |
| 2018077 | JIN_ZHONG_GUAI_SAN_JIAO | 晋中拐三角 |
| 2018078 | SHAO_YANG_ZI_PAI | 邵阳字牌 |
| 2018079 | XIANG_XIANG_GAO_HU_ZI | 湘乡告胡子 |
| 2018080 | LOU_DI_FANG_PAO_FA | 娄底放炮罚 |
| 2018081 | WANG_DIAO_MA_JIANG | 王钓麻将 |
| 2018082 | SHOU_YANG_QUE_KA | 寿阳缺卡 |
| 2018083 | LV_LIANG_KOU_DIAN | 吕梁扣点 |
| 2018084 | JIN_ZHONG_LI_SI | 晋中立四麻将 |
| 2018085 | HONG_TONG_WANG_PAI | 洪洞王牌 |
| 2018086 | DOU_DI_ZHU_LIN_FEN | 临汾斗地主 |
| 2018087 | LIN_FEN_YING_SAN_ZUI | 临汾硬三嘴 |
| 2018088 | LIN_FEN_YI_MEN_ZI | 临汾一门子 |
| 2018089 | SHAO_YANG_MA_JIANG | 邵阳麻将 |
| 2018090 | FEN_XI_YING_KOU | 汾西硬扣 |
| 2018091 | JI_XIAN_1928_JIA_ZHANG | 吉县1928夹张 (临汾) |
| 2018092 | ML_HONG_ZI | 汨罗红字 |
| 2018093 | LIN_FEN_XIANG_NING_SHUAI_JIN | 临汾乡宁摔金 |
| 2018094 | XIAO_YI_KOU_DIAN | 吕梁孝义扣点 |
| 2018095 | KOU_DIAN_FENG_ZUI_ZI | 扣点风嘴子 |
| 2018095 | LIN_FEN_KOU_DIAN_FENG_ZUI_ZI | 扣点风嘴子 |
| 2018096 | JIN_ZHONG_CAI_SHEN | 晋中财神 |
| 2018097 | HENG_YANG_SHIWUHUXI | 衡阳字牌15胡息 |
| 2018098 | HUAI_HUA_HONG_GUAI_WAN | 怀化红拐弯 |
| 2018099 | XU_ZHOU_PEI_XIAN | 徐州沛县麻将 |
| 2018100 | LV_LIANG_MA_JIANG | 吕梁麻将 |
| 2018101 | XIANG_YIN_TUI_DAO_HU | 湘阴推倒胡 |
| 2018102 | LV_LIANG_DA_QI | 吕梁打七 |
| 2018103 | DOU_DI_ZHU_LV_LIANG | 吕梁斗地主 |
| 2018104 | XIN_NING_MA_JIANG | 新宁麻将 |
| 2018105 | FAN_SHI_XIA_YU | 繁峙下雨 |
| 2018106 | DAI_XIAN_MA_JIANG | 代县麻将 |
| 2018107 | YUE_YANG_SAN_DA_HA | 岳阳三打哈 |
| 2018108 | YUE_YANG_HONG_ZHONG | 岳阳红中 |
| 2018109 | PAO_DE_KUAI_HUAIAN_NEW | 跑得快(淮安) 《淮安跑得快》移植过来 两者并存 |
| 2018110 | LONG_HUI_BA_ZHA_DAN | 隆回霸炸弹 |
| 2018111 | XIANG_XIANG_HONG_ZHONG | 红中麻将(湘乡红中) 《邵阳通中红中》移植过来 |
| 2018112 | XIANG_YIN_ZHUO_HONG_ZI | 湘阴捉红字 |
| 2018113 | DOU_DI_ZHU_XIN_ZHOU | 忻州斗地主 |
| 2018114 | HY_ER_PAO_HU_ZI | 衡阳二人跑胡子 |
| 2018115 | XIANG_XIANG_PAO_HU_ZI | 湘乡跑胡子 |
| 2018116 | SHAO_YANG_FANG_PAO_FA | 邵陽娄底放炮罚 |
| 2018117 | WU_TAI_KOU_DIAN | 五台扣点 |
| 2018118 | AN_HUA_PAO_HU_ZI | 安化跑胡子 |
| 2018119 | MJ_ZHUO_XIA_ZI | 岳阳捉虾子 |
| 2018120 | BAN_BIAN_TIAN_ZHA | 半边天炸 |
| 2018121 | LENG_SHUI_JIANG_SHI_HU_DAO | 冷水江十胡倒 |
| 2018122 | XIANG_TAN_PAO_HU_ZI | 湘潭跑胡子 |
| 2018123 | XIN_ZHOU_SAN_DA_ER | 忻州三打二 |
| 2018124 | HENG_YANG_SAN_DA_HA | 衡阳三打哈 |
| 2018125 | DA_NING_SHUAI_JIN | 大宁摔金 |
| 2018126 | YUE_YANG_WAI_HU_ZI | 岳阳歪胡子 |
| 2018127 | HENG_YANG_CHANG_SHA | 258麻将 |
| 2018128 | YUE_YANG_FU_LU_SHOU | 岳阳福禄寿 |
| 2018129 | XIANG_XIANG_SAN_DA_HA | 湘乡三打哈 |
| 2018130 | HUAI_HUA_MA_JIANG | 怀化麻将 |
| 2018131 | FEN_YANG_QUE_MEN | 汾阳缺门 |
| 2018132 | XUE_LIU | 血流 |
| 2018133 | XUE_ZHAN | 血战 |
| 2018134 | AN_HUA_MA_JIANG | 安化麻将 |
| 2018135 | JING_LE_KOU_DIAN | 静乐扣点 |
| 2018136 | HENG_YANG_FANG_PAO_FA | 衡阳放炮罚 |
| 2018137 | DA_TONG_GUAI_SAN_JIAO | 大同拐三角 |
| 2018138 | LUAN_GUA_FENG | 乱刮风 |
| 2018139 | YY_AN_HUA_PAO_HU_ZI | 岳阳安化跑胡子 |
| 2018140 | DOU_DI_ZHU_DA_TONG | 大同斗地主 |
| 2018141 | DA_TONG_ZHA_GU_ZI | 大同扎股子 |
| 2018142 | CHEN_ZHOU_ZI_PAI | 郴州字牌 |
| 2018143 | GUI_YANG_ZI_PAI | 桂阳字牌 |
| 2018144 | AN_HUA_MA_JIANG_SW | 安化四王麻将 |
| 2018145 | YUE_YANG_NIU_SHI_BIE | 岳阳牛十别 |
| 2018146 | NING_XIANG_PAO_HU_ZI | 宁乡跑胡子 |
| 2018147 | NING_XIANG_KAI_WANG | 宁乡开王麻将 |
| 2018148 | YUE_YANG_PENG_HU | 碰胡 |
| 2018149 | YI_YANG_WAI_HU_ZI | 益阳歪胡子 |
| 2018150 | TAO_JIANG_MA_JIANG | 桃江麻将 |
| 2018151 | YUE_YANG_YI_JIAO_LAI_YOU | 岳阳一脚赖油 |
| 2018152 | DIAN_TUO | 岳阳掂坨 |
| 2018153 | YUE_YANG_DA_ZHA_DAN | 岳阳打炸弹 |
| 2018154 | YI_YANG_MA_JIANG | 益阳麻将 |
| 2018155 | YUE_YANG_YUAN_JIANG_QIAN_FEN | 岳阳沅江千分 |
| 2018156 | CHEN_ZHOU | 郴州麻将 |
| 2018157 | YUAN_JIANG_MJ | 沅江麻将 |
| 2018158 | CHEN_ZHOU_MAO_HU_ZI | 郴州毛胡子 |
| 2018159 | ZHU_ZHOU_DA_MA_ZI | 株洲打码子 |
| 2018160 | NING_XIANG_MJ | 宁乡麻将 |
| 2018161 | XIANG_XI_2710 | 湘西2710 |
| 2018162 | PING_JIANG_ZHA_NIAO | 平江扎鸟 |
| 2018163 | FU_LU_SHOU_ER_SHI_ZHANG | 福禄寿20张 |
| 2018164 | CHANG_DE_PAO_HU_ZI | 常德跑胡子 |
| 2018165 | HAN_SHOU_PAO_HU_ZI | 汉寿跑胡子 |
| 2018166 | CHAO_GU_MJ | 岳阳炒股麻将 |
| 2018167 | NAN_XIAN_GUI_HU_ZI | 南县鬼胡子 |
| 2018168 | NAN_XIAN_MJ | 南县麻将 |
| 2018169 | YUN_CHENG_FENG_HAO_ZI | 运城风耗子 |
| 2018170 | YUN_CHENG_TIE_JIN | 运城贴金麻将 |
| 2019171 | JI_SHAN_NIU_YE_ZI | 稷山扭叶子 |
| 2018172 | YUAN_JIANG_GUI_HU_ZI | 沅江鬼胡子 |
| 2019173 | HE_JIN_KUN_JIN | 河津捆金 |
| 2019174 | DOU_DI_ZHU_ZERO | 斗地主单机版 |
| 2019175 | XIANG_SHUI_MJ | 响水麻将 |
| 2019176 | SHAO_YANG_SAN_DA_HA | 天天三打哈 改成 衡阳三打哈 |
| 2019177 | SHI_MEN_PAO_HU_ZI | 石门跑胡子 |
| 2019178 | QU_TANG_23_ZHANG | 曲塘23张 |
| 2019179 | XIANG_TAN_SAN_DA_HA | 湘潭三打哈 |
| 2019180 | AN_XIANG_WEI_MA_QUE | 安乡偎麻雀 |
| 2019181 | GUI_ZHOU_PU_DING_MJ | 贵州普定麻将 |
| 2019182 | GUI_ZHOU_AN_SHUN_MJ | 贵州安顺麻将 |
| 2019183 | GUI_ZHOU_SAN_DING_LIANG_FANG | 贵州三丁两房麻将 |
| 2019184 | PAO_DE_KUAI_ZERO | 跑得快单机版 |
| 2019185 | GUI_ZHOU_LIANG_DING_LIANG_FANG | 贵州两丁两房麻将 |
| 2019186 | HUAI_AN_DOU_DI_ZHU | 淮安斗地主 |
| 2019187 | SAN_DA_HA_NEW | 新三打哈 |
| 2019188 | GUI_ZHOU_SAN_DING_GUAI | 贵州三丁拐麻将 |
| 2019189 | GUI_ZHOU_ER_DING_GUAI | 贵州二丁拐麻将 |
| 2019190 | YUAN_LING_PAO_HU_ZI | 沅陵跑胡子 |
| 2019191 | DOU_DI_ZHU_GZ | 贵州斗地主 |
| 2019192 | GUI_ZHOU_XMY_GUI_YANG_ZHUO_JI | 贵州新麻友贵阳捉鸡 |
| 2019193 | GUI_ZHOU_MEN_HU_XUE_LIU | 贵州闷胡血流 |
| 2019194 | YONG_LI_SAN_DA_HA | 永利三打哈（衡阳三打哈） |
| 2019195 | GUI_ZHOU_GUI_YANG_ZHUO_JI | 贵州贵阳捉鸡 |
| 2019196 | POKER_96 | 96扑克 |
| 2019197 | GUI_ZHOU_DU_YUN_MJ | 贵州都匀麻将 |
| 2019198 | ML_HONGZHONG_ZERO | 单机版汨罗红中 |
| 2019199 | GUI_ZHOU_ZUN_YI_MJ | 贵州遵义麻将 |
| 2019200 | GUI_ZHOU_LIANG_DING_YI_FANG | 贵州两丁一房 |
| 2019201 | GUI_ZHOU_AN_LONG_MJ | 贵州安龙麻将 |
| 2019202 | GUI_ZHOU_XING_YI_MJ | 贵州兴义麻将 |
| 2019203 | GUI_ZHOU_WENG_AN_MJ | 贵州瓮安麻将 |
| 2019204 | GUI_ZHOU_KAI_LI_MJ | 贵州凯里麻将 |
| 2019205 | PAO_DE_KUAI_ELEVEN | 跑得快11张 |
| 2019206 | GUI_ZHOU_LI_PING_MJ | 贵州黎平麻将 |
| 2019207 | GUI_ZHOU_LIU_PAN_SHUI_MJ | 贵州六盘水麻将 |
| 2019208 | GUI_ZHOU_TIAN_ZHU_MJ | 贵州天柱麻将 |
| 2019209 | CHANG_SHA_ER_REN | 二人长沙麻将 二人缺门长麻 |
| 2019210 | GUI_ZHOU_RONG_JIANG_MJ | 贵州榕江麻将 |
| 2019211 | XU_PU_LAO_PAI | 淑浦老牌 邵阳(天天) |
| 2019212 | GUI_ZHOU_TONG_REN_MJ | 贵州铜仁麻将 |
| 2019213 | GUI_ZHOU_BI_JIE_MJ | 贵州毕节麻将 |
| 2019214 | GUI_ZHOU_REN_HUAI_MJ | 贵州仁怀麻将 |
| 2019215 | XU_PU_PAO_HU_ZI | 溆浦跑胡子 邵阳(天天) |
| 2019216 | ER_REN_YI_FANG_MJ | 二人一房麻将（急速8张麻将） |
| 2019217 | GUI_ZHOU_SUI_YANG_MJ | 贵州绥阳麻将 |
| 2019218 | YONG_ZHOU_BAO_PAI | 永州包牌 |
| 2019219 | GUI_ZHOU_JIN_SHA_MJ | 贵州金沙麻将 |
| 2019220 | GUI_ZHOU_XUE_ZHAN_DAO_DI_MJ | 贵州血战到底麻将 |
| 2019221 | DA_ZI_BO_PI | 大字剥皮 |
| 2019222 | XIN_SI_YANG | 新泗阳麻将 |
| 2019223 | ZHUO_HAO_ZI | 山西捉耗子 |
| 2019224 | PAO_DE_KUAI_XIANG_SHUI | 响水跑得快 |
| 2019225 | HONG_ZE_MA_JIANG | 洪泽麻将 |
| 2019226 | SHAN_XI_GAN_DENG_YAN | 干瞪眼 |
| 2019227 | YI_CHANG_XUE_LIU_MJ | 宜昌血流成河麻将 |
| 2019228 | PAO_DE_KUAI_HBTY | 湖北通用跑得快 |
| 2019232 | EN_SHI_MA_JIANG | 湖北: 恩施麻将 |
| 2019233 | WU_XUE_GE_BAN | 武穴隔板 |
| 2019234 | DANG_YANG_FAN_JING | 湖北: 当阳翻精 |
| 2019235 | DA_YE_ZI_PAI | 湖北: 大冶字牌 |
| 2019236 | HU_BEI_YI_JIAO_LAI_YOU | 湖北：一脚癞油 |
| 2019237 | CHONG_YANG_DA_GUN | 湖北：崇阳打滚 |
| 2019238 | JING_ZHOU_MA_JIANG | 邵阳：靖州麻将 |
| 2019239 | HU_BEI_JING_SHAN_MJ | 湖北京山麻将 |
| 2019240 | CHONG_YANG_MJ | 湖北:崇阳麻将 |
| 2019241 | TONG_CHENG_MJ | 湖北：通城麻将 |
| 2019242 | YI_CHANG_SHANG_DA_REN | 湖北：宜昌上大人 |
| 2019243 | HU_BEI_HUA_PAI | 湖北：湖北花牌 |
| 2019244 | DA_YE_KAI_KOU_FAN | 湖北：大冶开口番 |
| 2019245 | CHUO_XIA_ZI | 湖北：戳虾子（湖北一脚癞油改的） |
| 2019246 | YANG_XIN_MA_JIANG | 湖北：阳新麻将 |
| 2019247 | HONG_ZHONG_LAI_ZI_GANG | 湖北：红中癞子杠 |
| 2019248 | HUANG_SHI_HH_MJ | 湖北：黄石晃晃麻将 |
| 2019249 | TONG_SHAN_HH_MJ | 湖北：通山晃晃麻将 |
| 2019250 | DA_YE_510K | 湖北：大冶510K |
| 2019251 | JIANG_LING_HONG_ZHONG | 湖北：江陵红中 |
| 2019252 | QI_CHUN_HH_MJ | 湖北：蕲春晃晃 |
| 2019253 | QI_CHUN_GD_MJ | 湖北，蕲春广东麻将 |
| 2019254 | CHONG_YANG_HUA_QUAN_JIAO | 湖北：崇阳画圈脚 |
| 2019255 | GONG_AN_HUA_PAI | 湖北：公安花牌 |
| 2019256 | DOU_DI_ZHU_HBTY | 湖北通用斗地主 |
| 2019257 | DOU_DI_ZHU_QC | 蕲春斗地主 |
| 2019258 | SHI_SHOU_AI_HUANG | 湖北：石首捱晃麻将 |
| 2019259 | QIAN_JIANG_HH_MJ | 湖北：潜江晃晃麻将 |
| 2019260 | WU_XUE_MJ | 湖北：武穴麻将 |
| 2019261 | XIAO_GAN_KA_WU_XING | 湖北：孝感卡五星麻将 |
| 2019262 | QI_CHUN_HONG_ZHONG_GANG | 蕲春红中杠 |
| 2019263 | TONG_CHENG_GE_ZI_PAI | 通城个子牌 |
| 2019264 | TONG_SHAN_DA_GONG | 湖北：通山打拱 |
| 2019265 | QI_CHUN_DA_GONG | 湖北：蕲春打拱 |
| 2019266 | DA_YE_DA_GONG | 湖北：大冶打拱 |
| 2019267 | EN_SHI_SHAO_HU | 恩施绍胡 |
| 2019268 | WU_XUE_510K | 湖北：武穴510K |
| 2019269 | SUI_ZHOU_KA_WU_XING | 湖北：随州卡五星 |
| 2019270 | QIAN_JIANG_QIAN_FEN | 湖北：潜江千分 |
| 2019271 | YONG_ZHOU_LAO_CHUO | 永州老戳 |

### 游戏房间与对局通用能力（`pkroom`）

以下描述**房间组件与协议层**的共性能力；具体出牌、胡牌、计分随 `gameType` 在 `game-server/app/servers/pkroom/games/` 各子模块中实现。

| 能力 | 代码依据（摘要） |
|------|------------------|
| **房间生命周期** | `game-server/app/servers/pkroom/pkroomCode.js` 中 `Table`：`createParams` 含 `tableid`、`gameType`、`clubId` / `leagueId` 等；未满员超时解散（定时器 `1800 * 1000` ms，即 30 分钟）；`RoomEnd` 结束房间。 |
| **频道推送** | Pomelo `channelService.getChannel('ROOM_NUMBER_' + tableid)` 桌内广播。 |
| **快照与恢复** | `mjlog`、`createSnapshoot` 等；日志含「快照恢复」与 `gameType`。 |
| **加时赛 / 活动开关** | 启动读 MySQL `tb_activity_web`、`tb_activity_overtime_config`，维护红包加时赛、普通加时赛及按 `gameType` 开放的加时映射。 |
| **心跳与踢出** | 定时检查 `heartbeatTime`，超时标记离线；踢出间隔参考 `kickoutMinute`。 |
| **战绩与排行** | `gameTypeScoreRank` 等，用于玩法得分排行榜类逻辑。 |
| **与玩家服 RPC** | `game-server/app.js` 中 `NotifyUser` / `UpdatePlayer` / `RechargeSuccess` 等通过 `pkplayer` 的 `Rpc` 更新用户；解散等通过 `rpcInvoke`。 |
| **房间状态机** | `Server-ALL/common/lib/GlobalValues.js` 中 `TableState`：如 `waitJoin`、`waitReady`、`waitPut`、`waitEat`、`roundFinish`，以及三打哈阶段 `waitJiaoFen`、`waitKouDi`、血流类 `waitHuanPai` / `waitPuPai` 等。 |
| **胡牌结果类型** | 同文件 `WinType`：点炮、抢杠胡、自摸、杠开、杠后炮等。 |
| **协议分发** | `game-server/app/servers/pkroom/games/Protocol.js` 按 `tData.gameType` 分支到不同玩法的吃碰杠胡、过、托管等处理。 |
| **语音计费标记** | `alreadyButtonVoiceFee` 等与桌内语音消费相关字段。 |

> **说明**：单机类玩法在枚举中标为 `DOU_DI_ZHU_ZERO`、`PAO_DE_KUAI_ZERO` 等；小游戏为 `MINI_GAME_DSQ`。

## 项目目录结构

以下为仓库**顶层**与后端**核心目录**说明（省略各包内 `node_modules` 与大量客户端资源细节）。

```text
QIXing-BNBCODE/
├── Server-ALL/                 # 主服务端集合（Pomelo + 多 Express 服务）
│   ├── common/                 # 公共模块：config/{环境}/、lib/GlobalValues.js、mongoUtil、mysqlUtil、redisUtil 等
│   ├── game-server/            # Pomelo 游戏主进程：app.js、init.js、config/、app/servers/{pkcon,pkplayer,...}、modules/、app/http/
│   ├── game-field/             # 场服变体（结构近似 game-server）
│   ├── timer/                  # 定时任务：init.js、main1.js / main2.js / main3.js、modules/
│   ├── timer-field/
│   ├── api/                    # 轻量 API：init.js、main.js、routes/
│   ├── cms/                    # 运营后台：init.js、main.js、routes/、web/
│   ├── cms-field/
│   ├── member/                 # 会员/代理端 HTTP：init.js、main.js、routes/、web/
│   ├── mall/                   # 商城：init.js、main.js、routes/、config/
│   ├── agent/                  # 区域/代理相关 HTTP（Express）
│   ├── city/
│   ├── manager/、manager-nantong/、statis/、panda/  # 管理端、统计等（含 webSrc 等）
│   ├── wechaty/                # Wechaty 机器人相关
│   ├── myTest/                 # 测试与发版脚本、迁移 SQL 等
│   ├── image/、voice/、logs/   # 资源与日志
├── admin/                      # 后台周边：统计主服务、日志、监控、pomelo-test 等
├── client/                     # 游戏客户端（说明.txt 所指）
├── assets/、release/、各 *UI/  # UI 与资源工程
├── 数据库/                     # 根级 SQL：数据库.sql、dev备份.sql
├── 工具/                       # 调试与 CDN 等工具
├── Server-QX备份/、Server-XL备份/  # 历史备份服务端
├── 说明.txt                    # 简要模块说明与 Node 服务端提示
└── README.md                   # 本文件
```

**关键子目录职责简述**

- `Server-ALL/game-server/app/servers/`：按 Pomelo **serverType** 拆分逻辑（连接、玩家、房间、俱乐部、联盟、主控）。
- `Server-ALL/game-server/app/http/`：与游戏同机的 **Express** 子服务（端口见配置 `httpPort`，默认 `9990`）。
- `Server-ALL/*/config/index.js`：各服务入口配置，普遍通过 `../../common/config/${NODE_ENV}/` 加载 **mongo / mysql / redis / weixin / common** 等分文件配置。
- `Server-ALL/common/config/`：当前仓库中可见示例子目录包括 `dev-a`、`dev-b`、`field-dev`、`field-test`、`js3mj-test`、`yueyang-test` 等；**线上环境名需与运维实际目录一致**。

## 环境变量与配置

| 变量 / 机制 | 作用（代码出处） |
|-------------|------------------|
| `NODE_ENV` | 决定加载哪一套 `Server-ALL/common/config/<env>/` 及 Pomelo `servers.json` 中的环境键（如 `dev-a`、`yueyang-test`）；各服务 `init.js` 中均有 `process.env.NODE_ENV \|\| '默认值'`。 |
| `NODE_API` | 仅 `api/init.js`：取值 `local` / `common` 时改写转发到的游戏服 `host` 与 `port` 列表。 |
| 分环境业务参数 | `game-server/config/index.js` 根据 `env` 调整初始元宝、推荐奖励等（如 `yueyang`、`shaoyang`、`js3mj` 等分支）。 |

**注意**：部分 `init.js` 内存在 **OSS AccessKey 等硬编码**（如 `cms/init.js`、`member/init.js`），交付与公开展示前应改为环境变量或密钥管理系统；本 README 不摘录任何密钥内容。

## API 与路由结构

### 1. `api` 服务（Express）

- 入口：`api/main.js` — 全局 `app.use('/' + route, routes[route])`，即路由文件名为**一级路径前缀**。
- 已存在的路由文件：`routes/common.js`、`routes/weixin.js`、`routes/server.js`。
- 典型路径示例：
  - `GET/ALL /common/test` — 测试与访问日志
  - `ALL /weixin/miniprogram/info` — 小程序信息
  - `ALL /server/info`、`/server/heartbeat`、`/server/status` — 服务信息与探活

默认监听端口见 `api/config/index.js` 中 `port: 10000`（以该文件为准）。

### 2. `game-server` 内嵌 HTTP（Express）

- 入口：`game-server/app/http/app.js`，监听 `config.httpPort`（`game-server/config/index.js` 中为 `9990`）。
- 路由目录：`app/http/routes/` — `debug.js`、`user.js`、`club.js`、`league.js`、`recharge.js`、`status.js`、`third.js`。
- `run.sh` 中调用 `http://127.0.0.1:9990/debug/dissolveAll`，对应 `debug.js` 中 **`POST /debug/dissolveAll`**（并经过 `modules.common.checkLocal` 等本地校验）。

### 3. `cms` / `member` / `mall`

- 均采用 **`http.use('/' + route, routes[route])`** 模式，`routes/` 下多文件分模块（如 CMS 下含 `login.js`、`common.js`、`lottery.js` 等）；具体业务路径需查阅各 `routes/*.js` 内 `router` 定义。
- CMS 对非 API 路径会按 UA 重定向到 `web/mobile/index.html` 或 `web/html/index.html`（`cms/main.js`）。

### 4. Pomelo 客户端协议路由

- 游戏主协议为 Pomelo **RPC / 消息路由**，由 `app/servers/*/handler` 等注册；`game-server/app.js` 中示例：`pkroom.handler.tableMsg` 等。文档化需结合客户端协议或 handler 目录逐项整理（超出本 README 单表范围）。

## 启动方式

### Pomelo 游戏服（`game-server`）

1. 安装依赖：目录内存在 `package-lock.json`（以 lock 中声明版本为准；若缺失 `package.json` 需从锁文件或团队模板恢复安装入口）。
2. 设置 `NODE_ENV` 为 `servers.json` 中已有键名（如 `dev-a`）及 `Server-ALL/common/config` 下已有配置目录名。
3. 启动：`pomelo start -e $NODE_ENV`（与 `run.sh` 一致）。
4. Linux 下一键重启可参考 `game-server/run.sh`（含解散房间 HTTP、按端口段 kill、`svn up` 等——**需按实际环境修改**）。

### 各 Express 服务

在对应子目录执行 Node 入口（以仓库内文件名为准）：

| 目录 | 入口文件 |
|------|-----------|
| `api/` | `main.js` |
| `cms/` | `main.js` |
| `member/` | `main.js` |
| `mall/` | `main.js` |
| `timer/` | `main1.js` / `main2.js` / `main3.js`（按运维拆分进程） |

启动前同样需保证 `NODE_ENV` 与 `Server-ALL/common/config`（或 mall 自有 `config`）一致；`timer/init.js` 会打印 Redis、MySQL 等连接信息便于自检。

## 数据库与持久化

- **MySQL**：业务主数据；`timer` 等使用 master/slave/beta/mytest 多连接（见 `timer/init.js`）。
- **MongoDB**：游戏与会员等文档型数据（`mongoUtil`）。
- **Redis**：会话、锁、缓存等。
- **初始化脚本**：优先使用仓库根目录 `数据库/数据库.sql` 及环境备份；发版迁移示例见 `Server-ALL/myTest/joey/发版/` 下各 SQL。

## 部署与运维说明

- **进程模型**：游戏为 **多端口多进程**（`servers.json` 中 `pkcon` 前端口 `clientPort` 与后端 port 等）；运维需按表开放防火墙与安全组。
- **脚本环境**：`run.sh` 为 Bash，适用于 Linux；Windows 本地开发需自行等价处理端口与启动命令。
- **CI/CD**：仓库内**无**项目级 `.github/workflows` 或 GitLab CI 配置；依赖团队外部流水线。
- **容器化**：未发现本项目维护的镜像定义；若需 Docker 化需自行编写并挂载配置与数据卷。

## 其他说明

- 根目录 `说明.txt` 提及 **jtTool** 需 **Python 2.7** 与微软工具集，路径需自行修改——属客户端/工具链范畴，与 `Server-ALL` 后端并行存在。
- 备份目录 `Server-QX备份`、`Server-XL备份` 可能与主分支并存，部署时请以 `Server-ALL` 为现行主栈并做版本对齐。

---

*维护说明：新增服务、路由或玩法时，请同步更新本文件，并以 `main.js` / `app.js`、`routes/`、`game-server/config/servers.json` 及 `Server-ALL/common/lib/GlobalValues.js` 中的 `GAME_TYPE` 源码为准。*
