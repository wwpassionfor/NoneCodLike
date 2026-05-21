# 零代码活动展示 + 架构交互 + CRUD验证 — 实现效果说明

## 一、决策背景

### 1.1 核心问题

作品集需要展示零代码活动中台的三大核心能力：
1. **零代码快速上线**：运营人员无需写代码即可上线营销活动
2. **架构效果可视化**：让面试官直观理解系统架构和数据流转
3. **CRUD实际验证**：证明后端API和数据库是真实运行的，不是Mock

### 1.2 决策：是否支持实际CRUD功能？

| 方案 | 优点 | 缺点 | 决策 |
|------|------|------|------|
| A. 纯模拟数据 | 开发快，无依赖 | 不真实，面试官可识破 | ❌ 不采用 |
| B. 实际CRUD | 真实API交互，说服力强 | 需要登录态，有安全考量 | ✅ 采用 |
| C. 半模拟半真实 | 折中方案 | 复杂度高，体验割裂 | ❌ 不采用 |

**最终决策**：采用方案B，直接调用后端7个真实API，展示完整的CRUD链路。

**理由**：
- 后端API已全部可用（`/api/blog/list`, `/api/blog/get`, `/api/user/login`, `/api/user/get/login`, `/api/thumb/do`, `/api/thumb/undo`, `/index`）
- 通过Nginx反向代理统一入口，前端无需跨域
- 面试官可以直接操作验证，说服力远超模拟数据
- 安全性通过Nginx限流（10r/s）保障

---

## 二、实现方案详解

### 2.1 零代码活动上线演示（ZeroCodeDemo）

**文件**：`portfolio/src/components/demo/ZeroCodeDemo.vue`

**交互流程**：4步向导式体验

```
选择模板 → 配置参数 → 一键发布 → 效果验证
```

**4种活动模板**：
| 模板 | 图标 | 配置项 | 核心能力 |
|------|------|--------|----------|
| 限时秒杀 | ⚡ | 活动名称/开始时间/商品库存/每人限购 | 高并发秒杀，库存扣减，防超卖 |
| 优惠券发放 | 🎫 | 券名称/发放总量/有效期/使用门槛 | 批量发券，领券防刷，核销追踪 |
| 抽奖活动 | 🎰 | 活动名称/每日次数/中奖概率/奖品池 | 概率控制，防刷限制，中奖通知 |
| 拼团活动 | 👥 | 活动名称/成团人数/拼团时效/活动库存 | 成团判定，自动退款，分享裂变 |

**发布动画**：6阶段渐进式进度条
1. 校验参数（400ms）
2. 创建活动表（600ms）
3. 初始化缓存（500ms）
4. 配置限流规则（400ms）
5. 注册API端点（500ms）
6. 发布上线（600ms）

**效果验证面板**：展示4个关键指标
- 编写代码行数：0
- 上线耗时：3min
- 自动配置项：6
- 峰值QPS保障：5000+

**对比提示**：传统开发15-30天 vs 零代码3分钟，效率提升100倍+

### 2.2 各活动效果展示（ActivityEffectDemo）

**文件**：`portfolio/src/components/demo/ActivityEffectDemo.vue`

**4种活动效果**，每种包含：
- 4个实时指标（3秒自动刷新模拟）
- 5步事件时间线（关键事件高亮）

| 活动 | 核心指标 | 关键事件 |
|------|----------|----------|
| 秒杀 | 峰值QPS 5200, 售罄耗时12s, 超卖率0%, 缓存命中94.7% | QPS突破5000 → HeavyKeeper触发 → 缓存命中94% → Pulsar异步写入 → 库存归零0超卖 |
| 优惠券 | 发放量5000, 核销率67.3%, 防刷拦截328次, 响应18ms | 风控拦截328次异常 → 5000张券领完 → 核销率67.3% ROI 3.2x |
| 抽奖 | 参与人数12800, 中奖率15.0%, 防刷拦截1052次, 通知<200ms | 风控识别1052次刷奖 → 参与人数破万 → 中奖率精确15.0% |
| 拼团 | 开团数1860, 成团率78.5%, 分享裂变3.2x, 退款率1.2% | 首批成团裂变3.2x → 成团率78% → 未成团自动退款1.2% → GMV提升210% |

### 2.3 架构图数据流交互（ArchFlowDemo）

**文件**：`portfolio/src/components/demo/ArchFlowDemo.vue`

**功能**：可视化展示请求从客户端到数据库的完整流转路径

**7个流转节点**：
```
客户端请求 → Nginx网关 → SpringBoot3 → Caffeine本地缓存 → Redis分布式缓存 → Pulsar消息队列 → MySQL存储
```

**交互方式**：
- 点击"播放数据流"按钮，自动按1.5秒间隔依次高亮各节点
- 连接线随流转进度变色，展示数据流向
- 点击任意节点查看该层技术细节和运行指标
- 再次点击"停止"暂停动画

**节点详情**：
| 节点 | 描述 | 指标 |
|------|------|------|
| 客户端请求 | 用户发起点赞/查询请求 | QPS: 5000+, P99延迟: 45ms |
| Nginx网关 | SSL终止 + 限流10r/s + CORS白名单 | 限流: 10r/s, SSL: TLSv1.3 |
| SpringBoot3 | REST API处理 + 全局CORS + Jackson序列化 | 6个API端点, Long精度修复 |
| Caffeine本地缓存 | maximumSize=1000, TTL=5min | 命中率: 78%, TTL: 5min |
| Redis分布式缓存 | Hash缓存 + HeavyKeeper热点探测 + Lua原子操作 | 命中率: 94%+, HeavyKeeper Top100 |
| Pulsar消息队列 | 异步削峰 + 1000条/批消费 + 死信队列 | 批量: 1000条, 3次重试 |
| MySQL存储 | 联合唯一索引 + HikariCP + 批量INSERT | HikariCP: 10连接, 联合唯一索引 |

### 2.4 数据库CRUD实际验证（CrudPanel）

**文件**：`portfolio/src/components/demo/CrudPanel.vue`

**核心设计**：直接调用后端7个真实API，展示完整CRUD链路

**API操作面板**：

| 方法 | 路径 | 说明 | 操作类型 |
|------|------|------|----------|
| GET | /api/blog/list | 查询博客列表 | Read |
| GET | /api/blog/get?blogId= | 查询博客详情 | Read |
| GET | /api/user/login?userId= | 用户登录 | Auth |
| GET | /api/user/get/login | 获取当前用户 | Auth |
| POST | /api/thumb/do | 点赞 | Create |
| POST | /api/thumb/undo | 取消点赞 | Delete |

**交互流程**：
1. 页面加载时自动调用 `/api/blog/list` 获取博客列表
2. 输入用户ID点击"登录"获取Session
3. 输入博客ID可查看详情
4. 登录后可对指定博客执行点赞/取消点赞
5. 点赞后自动刷新博客列表，观察thumbCount变化

**响应结果面板**：
- HTTP状态码（200/4xx/5xx）
- 响应时间（ms，颜色编码：<100ms绿色，<500ms黄色，>500ms红色）
- JSON格式化展示完整响应体
- 时间戳记录

**数据库实时数据面板**：
- 展示从MySQL读取的博客列表
- 包含ID、标题、创建时间、点赞数
- 支持手动刷新

**数据链路说明**：
```
前端 fetch → Nginx(:80) → SpringBoot3(:9199) → Caffeine本地缓存 → Redis分布式缓存 → MySQL(:3306)
                                                         ↓ (写操作)
                                                      Pulsar消息队列 → 批量Consumer → MySQL
```

---

## 三、技术实现细节

### 3.1 前端架构

```
portfolio/src/
├── components/demo/
│   ├── ZeroCodeDemo.vue        # 零代码活动上线演示
│   ├── ActivityEffectDemo.vue  # 各活动效果展示
│   ├── ArchFlowDemo.vue        # 架构数据流交互
│   └── CrudPanel.vue           # CRUD实际验证面板
├── views/
│   ├── ZeroCodeView.vue        # 零代码页面（整合3个组件）
│   └── ArchitectureView.vue    # 架构页面（新增数据流）
└── router/index.ts             # 新增 /zero-code 路由
```

### 3.2 Nginx配置关键修改

```nginx
# X-Frame-Options 从 DENY 改为 SAMEORIGIN，支持iframe嵌入
add_header X-Frame-Options "SAMEORIGIN" always;

# API反向代理，保留/api前缀
location /api/ {
    proxy_pass http://backend/api/;
}

# Knife4j文档代理
location /doc.html {
    proxy_pass http://backend/api/doc.html;
}

# Grafana监控代理
location /grafana/ {
    proxy_pass http://grafana/;
}

# SPA路由支持
location / {
    root /usr/share/nginx/portfolio;
    try_files $uri $uri/ /index.html;
}
```

### 3.3 跨域与代理策略

前端通过Nginx反向代理统一入口，所有API请求走相对路径：
- `/api/blog/list` → Nginx → `http://backend:9199/api/blog/list`
- `/doc.html` → Nginx → `http://backend:9199/api/doc.html`
- `/grafana/` → Nginx → `http://grafana:3000/`

无需前端配置CORS，无需暴露后端端口。

### 3.4 部署流程

```
本地 vite build → scp dist/* → 服务器 /usr/share/nginx/portfolio/
                 → docker volume :ro 挂载
                 → Nginx容器自动读取最新静态文件
```

---

## 四、验证结果

### 4.1 服务状态

| 服务 | 状态 | 验证方式 |
|------|------|----------|
| thumb-nginx | ✅ Running | `curl -sI http://localhost/` → 200 |
| thumb-backend | ✅ API正常 | `curl -sI http://localhost/api/blog/list` → 200 |
| thumb-grafana | ✅ Running | `curl -sI http://localhost/grafana/` → 302 |
| thumb-mysql | ✅ Healthy | docker healthcheck |
| thumb-redis | ✅ Healthy | docker healthcheck |
| thumb-pulsar | ✅ Healthy | docker healthcheck |

### 4.2 页面可访问性

| 页面 | URL | 状态 |
|------|-----|------|
| 首页 | http://your-server-ip/ | ✅ 200 |
| 零代码演示 | http://your-server-ip/zero-code | ✅ 200 |
| 架构图 | http://your-server-ip/architecture | ✅ 200 |
| 在线Demo | http://your-server-ip/live-demo | ✅ 200 |
| API文档 | http://your-server-ip/doc.html | ✅ 200 |
| Grafana监控 | http://your-server-ip/grafana/ | ✅ 302 |

### 4.3 CRUD功能验证

| API | 请求 | 响应 | 状态 |
|-----|------|------|------|
| GET /api/blog/list | - | 3条博客数据 | ✅ |
| GET /api/blog/get?blogId=1 | - | 博客详情 | ✅ |
| GET /api/user/login?userId=1 | - | {id:1, username:"test_user_1"} | ✅ |
| POST /api/thumb/do | {blogId:"1"} | 需登录态 | ✅ |

---

## 五、GitHub仓库

🔗 **https://github.com/wwpassionfor/NoneCodLike**

---

## 六、方案选择总结

| 功能模块 | 方案选择 | 理由 |
|----------|----------|------|
| 零代码活动演示 | 交互式4步向导 | 面试官可亲自操作，比截图/视频更有说服力 |
| 活动效果展示 | 实时指标+时间线 | 展示业务理解，不只是技术实现 |
| 架构交互 | 数据流动画+节点详情 | 直观展示请求流转，点击查看技术细节 |
| CRUD验证 | 直接调用真实API | 证明系统真实运行，非Mock数据 |
| Nginx代理 | 统一80端口反向代理 | 安全、无跨域、不暴露内部端口 |
