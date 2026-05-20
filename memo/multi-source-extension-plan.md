# WeRSS 多渠道订阅扩展方案

> 目标：将现有微信公众号订阅系统，扩展为支持微博、雪球、Twitter/X 等多媒体渠道的统一订阅集合。

---

## 一、核心问题诊断

当前架构 **深度绑定微信**，体现在 4 层：

| 层次 | 微信耦合点 | 影响范围 |
|------|-----------|---------|
| **数据模型** | `feeds` 表字段 (`mp_name`, `faker_id`)，`articles` 表有 ~15 个微信专属字段 | 数据库 |
| **驱动层** | `driver/wx.py`, `driver/wxarticle.py` 直接写死微信逻辑 | 内容抓取 |
| **API 路由** | 路径前缀 `/api/v1/wx/`，`apis/mps.py` 内逻辑全部面向微信 | 接口层 |
| **前端/UI** | 术语 "公众号"、搜索流程针对微信 | 用户界面 |

---

## 二、设计思路：最小侵入 + 最大复用

**核心原则**：不重构现有表结构（向后兼容），通过 **新增抽象层** + **JSON 扩展字段** 实现多源支持。

---

## 三、数据库层改造

### 3.1 新增 `sources` 表（数据源/渠道定义）

```python
# core/models/source.py
class Source(Base):
    __tablename__ = "sources"

    id = Column(String(50), primary_key=True)        # "wechat", "weibo", "xueqiu", "twitter"
    name = Column(String(100), nullable=False)         # "微信公众号", "微博", "雪球", "Twitter/X"
    icon = Column(String(255))                         # 图标路径
    enabled = Column(Boolean, default=True)            # 是否启用
    auth_type = Column(String(50))                     # "qr_code", "cookie", "oauth", "api_key", "none"
    config_schema = Column(Text)                       # JSON: 该源的配置项 schema
    driver_class = Column(String(200))                 # "driver.wx.WxDriver" / "driver.weibo.WeiboDriver"
    search_enabled = Column(Boolean, default=False)    # 是否支持搜索订阅源
    description = Column(Text)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
```

预置数据：

```sql
INSERT INTO sources VALUES ('wechat',  '微信公众号', '/icons/wechat.png',  1, 'qr_code', '{}', 'driver.wx.WxDriver',       1, '...');
INSERT INTO sources VALUES ('weibo',   '微博',       '/icons/weibo.png',   1, 'cookie',  '{}', 'driver.weibo.WeiboDriver', 1, '...');
INSERT INTO sources VALUES ('xueqiu',  '雪球',       '/icons/xueqiu.png',  1, 'cookie',  '{}', 'driver.xueqiu.XueqiuDriver', 1, '...');
INSERT INTO sources VALUES ('twitter', 'X/Twitter',  '/icons/twitter.png', 1, 'api_key', '{}', 'driver.twitter.TwitterDriver', 1, '...');
```

### 3.2 改造 `feeds` 表（最小改动）

```python
# core/models/feed.py — 仅新增 2 个字段
class Feed(Base):
    __tablename__ = "feeds"

    # === 现有字段全部保留，不动 ===
    id = Column(String(255), primary_key=True)         # 保持 MP_WXS_xxx 格式兼容
    mp_name = Column(String(255))
    mp_cover = Column(String(255))
    mp_intro = Column(String(255))
    status = Column(Integer, default=1)
    sync_time = Column(Integer)
    update_time = Column(Integer)
    created_at = Column(DateTime, default=func.now())
    updated_at = Column(DateTime, default=func.now(), onupdate=func.now())
    faker_id = Column(String(255))

    # === 新增字段 ===
    source_id = Column(String(50), default="wechat")   # FK -> sources.id，向后兼容默认 "wechat"
    source_meta = Column(Text)                          # JSON: 平台专属元数据
```

**ID 生成规则扩展**：

- 微信（保持不变）：`MP_WXS_{base64_id}`
- 微博：`WB_{user_id}`
- 雪球：`XQ_{user_id}`
- Twitter：`TW_{user_id}`

`source_meta` 示例：

```json
// 微博
{"uid": "1234567890", "verified": true, "verified_reason": "知名财经博主", "followers_count": 500000}

// 雪球
{"uid": "1234567890", "follower_count": 10000, "status_count": 3000, "description": "价值投资"}

// Twitter
{"user_id": "1234567890", "handle": "@elonmusk", "followers_count": 100000000, "verified": true}
```

### 3.3 改造 `articles` 表（最小改动）

```python
# core/models/article.py — 仅新增 1 个字段
class ArticleBase(Base):
    __tablename__ = "articles"

    # === 现有 ~35 个字段全部保留 ===
    # ...

    # === 新增字段 ===
    source_data = Column(Text)    # JSON: 平台专属数据，替代为新源加字段的方式
```

`source_data` 示例：

```json
// 微博
{"reposts_count": 500, "comments_count": 1200, "attitudes_count": 3000, "pic_urls": [...], "is_retweet": true, "retweeted_status": {...}}

// 雪球
{"reply_count": 50, "like_count": 200, "retweet_count": 80, "topics": ["#A股#", "#价值投资#"]}

// Twitter
{"retweet_count": 1000, "like_count": 5000, "reply_count": 300, "quote_count": 200, "media": [...], "lang": "en"}
```

**关键决策**：现有微信专属字段（`publish_type`, `show_type`, `art_type` 等 15 个字段）**不删除**，对新源直接忽略（值为 NULL）。新源的专属数据全部进 `source_data` JSON。零迁移成本。

---

## 四、驱动层抽象（Plugin 模式）

### 4.1 定义统一驱动接口

```python
# driver/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class FeedInfo:
    """统一的订阅源信息"""
    source_id: str           # "wechat" / "weibo" / ...
    account_id: str          # 平台内唯一 ID
    name: str                # 显示名称
    avatar: str              # 头像 URL
    description: str         # 简介
    source_meta: dict        # 平台专属元数据

@dataclass
class ArticleInfo:
    """统一的文章信息"""
    article_id: str          # 平台内唯一 ID
    title: str
    url: str                 # 原文链接
    description: str         # 摘要
    cover_url: str           # 封面图
    content: str             # 正文（HTML/文本）
    publish_time: int        # Unix timestamp
    source_data: dict        # 平台专属数据

class BaseDriver(ABC):
    """数据源驱动基类"""

    source_id: str = ""

    @abstractmethod
    async def authenticate(self, **kwargs) -> bool:
        """认证（扫码/Cookie/OAuth 等）"""
        ...

    @abstractmethod
    async def is_authenticated(self) -> bool:
        """检查认证是否有效"""
        ...

    @abstractmethod
    async def search(self, keyword: str, page: int = 1) -> list[FeedInfo]:
        """搜索订阅源"""
        ...

    @abstractmethod
    async def get_feed_info(self, account_id: str) -> FeedInfo:
        """获取订阅源信息"""
        ...

    @abstractmethod
    async def fetch_articles(self, account_id: str, since: int = 0,
                            limit: int = 20) -> list[ArticleInfo]:
        """获取文章列表"""
        ...

    @abstractmethod
    async def fetch_content(self, article_url: str) -> Optional[str]:
        """获取文章全文（可选，有些平台正文已在列表中）"""
        ...

    async def cleanup(self):
        """清理资源"""
        pass
```

### 4.2 现有微信驱动适配

```python
# driver/wx.py — 不改现有代码，包一层适配器
class WxDriver(BaseDriver):
    source_id = "wechat"

    def __init__(self):
        self._gather = WxGather()  # 复用现有 WxGather

    async def authenticate(self, **kwargs) -> bool:
        # 复用现有的 wxLogin QR 扫码逻辑
        return await self._gather.login()

    async def search(self, keyword: str, page: int = 1) -> list[FeedInfo]:
        results = self._gather.search_Biz(keyword)
        return [FeedInfo(
            source_id="wechat",
            account_id=r["fakeid"],
            name=r["nickname"],
            avatar=r["headimg"],
            description=r.get("signature", ""),
            source_meta={"fakeid": r["fakeid"], "alias": r.get("alias", "")}
        ) for r in results]

    # ... 其余方法适配现有 WxGather / WXArticleFetcher
```

### 4.3 新源驱动示例（微博）

```python
# driver/weibo.py
class WeiboDriver(BaseDriver):
    source_id = "weibo"

    async def authenticate(self, cookie: str = None, **kwargs) -> bool:
        """通过 Cookie 认证"""
        self._session = httpx.AsyncClient(cookies=cookie, headers={...})
        return await self.is_authenticated()

    async def search(self, keyword: str, page: int = 1) -> list[FeedInfo]:
        # 调用微博搜索 API: https://m.weibo.cn/api/container/getIndex?containerid=100103type%3D3%26q%3D{keyword}
        ...

    async def fetch_articles(self, account_id: str, since: int = 0, limit: int = 20) -> list[ArticleInfo]:
        # 调用: https://m.weibo.cn/api/container/getIndex?containerid=107603{uid}
        # 解析微博卡片数据，统一映射为 ArticleInfo
        ...
```

### 4.4 驱动注册表

```python
# driver/registry.py
_DRIVER_REGISTRY: dict[str, type[BaseDriver]] = {}

def register_driver(source_id: str):
    def decorator(cls):
        _DRIVER_REGISTRY[source_id] = cls
        return cls
    return decorator

def get_driver(source_id: str) -> BaseDriver:
    cls = _DRIVER_REGISTRY.get(source_id)
    if not cls:
        raise ValueError(f"Unknown source: {source_id}")
    return cls()

# 启动时自动注册
import driver.wx       # @register_driver("wechat")
import driver.weibo    # @register_driver("weibo")
```

---

## 五、API 层改造

### 路由策略：渐进式，不破坏现有

```
现有路由（保持不变，向后兼容）:
  /api/v1/wx/mps/*          -> 等同于 source_id=wechat
  /api/v1/wx/articles/*     -> 等同于 source_id=wechat
  /api/v1/wx/auth/qr/*      -> 微信扫码认证

新增路由（多源通用）:
  /api/v1/sources                           -> GET: 列出所有数据源及状态
  /api/v1/sources/{source_id}/auth          -> POST: 触发认证（扫码/提交cookie等）
  /api/v1/sources/{source_id}/auth/status   -> GET: 认证状态
  /api/v1/feeds                             -> GET: 列出所有订阅（支持 ?source=weibo 过滤）
  /api/v1/feeds/{source_id}/search/{kw}     -> GET: 在指定源搜索
  /api/v1/feeds/{source_id}                 -> POST: 添加订阅
  /api/v1/feeds/{feed_id}                   -> PUT/DELETE: 更新/删除订阅
  /api/v1/feeds/{feed_id}/sync              -> POST: 手动同步
  /api/v1/content                           -> GET: 统一内容列表（支持 ?source=xueqiu&tag=xxx）
  /api/v1/content/{content_id}              -> GET: 内容详情
```

**关键**：`/api/v1/wx/*` 全部保留不动，前端可以渐进迁移。

---

## 六、Jobs/调度层改造

```python
# jobs/sync.py — 统一同步调度
async def sync_feed(feed: Feed):
    """统一的订阅同步入口"""
    driver = get_driver(feed.source_id)  # 根据 source_id 获取对应驱动

    if not await driver.is_authenticated():
        log.warning(f"Source {feed.source_id} not authenticated, skip {feed.id}")
        return

    articles = await driver.fetch_articles(
        account_id=feed.source_meta.get("account_id"),
        since=feed.sync_time or 0
    )

    for article_info in articles:
        # 统一入库逻辑，复用现有 DB.add_article() 的框架
        await save_article(feed, article_info)

    feed.sync_time = int(time.time())
```

现有 `jobs/mps.py` 中的微信专属逻辑逐步迁移到 `WxDriver` 内部，调度层只调统一接口。

---

## 七、RSS 层改造

**几乎不需要改**。当前 RSS 生成基于 `feeds` + `articles` 表查询，只要数据入了这两个表，RSS 自动生成。

唯一改动：RSS channel 的 `<title>` 和 `<link>` 需要根据 `source_id` 调整：

```python
# core/rss.py — 小改动
def _get_channel_info(self, feed: Feed) -> dict:
    source = get_source(feed.source_id)  # 从 sources 表查
    return {
        "title": f"{feed.mp_name} - {source.name}",
        "link": feed.source_meta.get("profile_url", ""),
        "description": feed.mp_intro,
    }
```

---

## 八、前端改造要点

1. **导航栏**：增加「数据源」管理页面，展示各源认证状态
2. **订阅管理**：搜索框增加源选择器（下拉选微信/微博/雪球/Twitter）
3. **内容列表**：增加源标识 icon（小图标区分来源）
4. **统一阅读体验**：不同源的文章用同一个阅读器展示（已有的 article detail 页面）

---

## 九、新增目录结构

```
driver/
  base.py              # BaseDriver 抽象基类 + FeedInfo/ArticleInfo 数据类
  registry.py          # 驱动注册表
  wx.py                # 现有微信驱动（包一层适配器）
  wxarticle.py         # 现有（不变）
  weibo.py             # 新增：微博驱动
  xueqiu.py            # 新增：雪球驱动
  twitter.py           # 新增：Twitter/X 驱动

apis/
  mps.py               # 现有（不变，向后兼容）
  sources.py           # 新增：数据源管理 API
  feeds.py             # 新增：统一订阅管理 API
  ...

core/models/
  source.py            # 新增：Source 模型
  feed.py              # 现有（新增 source_id, source_meta 字段）
  article.py           # 现有（新增 source_data 字段）
  ...
```

---

## 十、实施路线

### Phase 1 — 基础框架（1-2周）

- [ ] 新增 `sources` 表 + Source 模型
- [ ] `feeds` 表新增 `source_id` / `source_meta` 字段
- [ ] `articles` 表新增 `source_data` 字段
- [ ] 定义 `BaseDriver` 接口
- [ ] `WxDriver` 适配器（包装现有代码）
- [ ] 驱动注册机制
- [ ] 新增 `/api/v1/sources` API

### Phase 2 — 第二个源验证（1周）

- [ ] 实现 `WeiboDriver`（微博 Cookie 认证 + 内容抓取）
- [ ] 统一 sync 调度逻辑
- [ ] 前端增加源选择器
- [ ] 验证端到端流程（搜索 -> 订阅 -> 同步 -> RSS）

### Phase 3 — 扩展更多源（每源 3-5天）

- [ ] `XueqiuDriver`（雪球）
- [ ] `TwitterDriver`（需要 API key 或第三方方案）
- [ ] 前端多源 UI 完善

### Phase 4 — 增强

- [ ] 统一内容全文获取（不同源的正文提取策略）
- [ ] 跨源聚合 RSS（`/feed/all.xml`）
- [ ] 源级别的频率限制 / 代理配置

---

## 十一、方案优势总结

| 维度 | 做法 | 收益 |
|------|------|------|
| **向后兼容** | 现有表字段不删、API 不改 | 零迁移成本，现有用户无感 |
| **扩展性** | 新增源只需实现 `BaseDriver` + 插入 `sources` 表 | 不改框架，插件式扩展 |
| **复用性** | RSS 生成、Tag 管理、Webhook、Cascade 全部复用 | 最大程度利用现有代码 |
| **渐进式** | `/api/v1/wx/*` 保留，新 API 逐步上线 | 可以分阶段交付 |
| **数据模型** | JSON 扩展字段容纳各平台差异 | 不需要为每个源加列 |
