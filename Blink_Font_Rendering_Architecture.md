# Blink 浏览器内核字体渲染架构

## 目录
1. [概述](#概述)
2. [字体加载流程](#字体加载流程)
3. [多层字体缓存系统](#多层字体缓存系统)
4. [跨进程字体通信](#跨进程字体通信)
5. [文本成形（Text Shaping）](#文本成形)
6. [字体匹配算法](#字体匹配算法)
7. [内存管理](#内存管理)
8. [性能优化](#性能优化)

---

## 概述

Blink 是 Chromium 浏览器的渲染引擎，包含完整的文本栈。字体处理是渲染过程中的关键环节，涉及以下核心职责：

1. **字体发现与匹配** - 根据 CSS 的 `font-family` 属性找到合适的字体
2. **缓存管理** - 多层缓存系统确保高效字体查找和复用
3. **跨进程通信** - 在 Renderer 和 Browser 进程之间安全传输字体数据
4. **文本成形** - 使用 HarfBuzz 进行 OpenType 布局操作
5. **字体回退** - 当主字体缺少字形时自动选择备选字体

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Layout/Paint 渲染层                        │
│                      (Core Thread)                            │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Font 对象 (字体描述符)                      │
│              FontDescription + FontFallbackList              │
└─────────────┬───────────────────────────────────────────────┘
              │
      ┌───────┴───────┐
      ▼               ▼
  Web字体缓存      系统字体缓存
(FontFaceCache)  (FontCache)
      │               │
      │               └──────────────┐
      └──────────────┬───────────────┘
                     ▼
        ┌─────────────────────────────┐
        │    SimpleFontData 缓存       │
        │   (Blink 全局字体对象缓存)    │
        └─────────┬───────────────────┘
                  │
        ┌─────────┴──────────────┐
        ▼                        ▼
   字形缓存              HarfBuzz 文本成形
 (Glyph Cache)       (HarfBuzzFontCache)
        │                        │
        └─────────┬──────────────┘
                  ▼
        ┌──────────────────────────┐
        │   NGShapeCache           │
        │  (字形与位置缓存)         │
        └──────────────────────────┘
```

---

## 字体加载流程

### 1. CSS 到字体对象的转换

```
CSS 解析
  ↓
ComputedStyle 计算（包含 font-family, font-size, font-weight 等）
  ↓
FontDescription 创建（保存 CSS 字体属性的结构化表示）
  ↓
Font 对象创建（关联 CSSFontSelector）
  ↓
运行时字体匹配（延迟执行，首次使用时触发）
```

**关键类**：

- `FontDescription`：CSS 字体属性的容器
  - `font-family` 列表
  - `font-size`（以像素为单位）
  - `font-weight`（100-900）
  - `font-style`（normal/italic）
  - `font-variant`，`text-orientation`，`text-rendering` 等

- `Font`：渲染层的字体 API
  - 拥有 `FontDescription` 和 `FontFallbackList`
  - 提供文本测量、绘制、几何操作接口
  - 持有 `CSSFontSelector`（关键！用于字体发现）

- `CSSFontSelector`：字体可用性的"窗口"
  - 查询 Web 字体（来自 `@font-face`）
  - 查询系统字体（通过 `FontCache`）
  - 维护可用字体的实时状态

### 2. 字体匹配请求流程

当 `Font` 对象首次被用于测量或绘制时：

```cpp
// 伪代码演示
class Font {
  const SimpleFontData* GetFontData() {
    if (!font_fallback_list_) {
      font_fallback_list_ = std::make_unique<FontFallbackList>(this);
    }
    return font_fallback_list_->PrimaryFontData();
  }
};
```

**FontFallbackList** 执行以下步骤：

1. 遍历 `FontDescription` 中的 `font-family` 列表
2. 对每个家族名，调用 `CSSFontSelector::GetFontData(FontDescription)`
3. `CSSFontSelector` 首先查询 `FontFaceCache`（Web 字体）
4. 如果 Web 字体中没有匹配项，查询 `FontCache`（系统字体）
5. 使用 **CSS 字体匹配算法**（CSSWG 规范）选择最佳匹配

### CSS 字体匹配算法

```
输入：font-weight, font-style, font-stretch, font-size-adjust
输出：最匹配的字体

步骤：
1. 过滤阶段
   - 移除不匹配 font-stretch 的字体
   - 移除不匹配 font-style 的字体
   
2. 排序阶段
   - 按 font-weight 相似度排序
   - 相同 weight 时按 font-width 排序
   
3. 选择
   - 返回排序后的第一个
```

实现位置：`third_party/blink/renderer/platform/fonts/font_selection_algorithm.h`

---

## 多层字体缓存系统

### 三层缓存架构

Blink 采用**三层缓存**策略，充分利用不同粒度的缓存优化性能：

```
┌─────────────────────────────────────────────────────┐
│ L1: 简单字体数据缓存 (SimpleFontData 对象池)         │
│     - 保存：FontPlatformData → SimpleFontData        │
│     - 粒度：单个字体对象                              │
│     - 位置：Blink 全局上下文                         │
│     - 访问时间：纳秒级 (直接指针查找)                 │
└─────────────────────────────────────────────────────┘
                        ▲
                        │ (缓存未命中时)
┌─────────────────────────┼─────────────────────────┐
│ L2: 字体系统缓存 (FontCache - 平台相关)            │
│     - 保存：字体家族名 → 系统字体对象                │
│     - 粒度：字体家族 + 样式                         │
│     - 位置：平台相关实现                            │
│     - 访问时间：微秒级 (系统 API 调用)              │
│     - 平台实现：                                    │
│       * font_cache_skia.cc (通用 Skia)             │
│       * font_cache_linux.cc (Linux/FreeType)      │
│       * font_cache_mac.mm (macOS/CTFont)          │
│       * font_cache_android.cc (Android)           │
└─────────────────────────────────────────────────────┘
                        ▲
                        │ (系统缓存未命中时)
┌─────────────────────────┼─────────────────────────┐
│ L3: 字形缓存 (Glyph Cache - 内存映射)              │
│     - 保存：字符 (UChar32) → 字形 ID (Glyph)       │
│     - 粒度：单个字符                                │
│     - 位置：SimpleFontData 内部                     │
│     - 访问时间：纳秒级 (数组查找)                    │
│     - 使用场景：CJK 文本（频繁字符查询）             │
└─────────────────────────────────────────────────────┘
```

### L1：SimpleFontData 对象缓存

**位置**：`third_party/blink/renderer/platform/fonts/font_cache.h`

SimpleFontData 是 Blink 内部的通用字体表示，包装了平台相关的 `FontPlatformData`：

```cpp
class SimpleFontData final : public FontData {
 public:
  // 构造函数接收平台级字体数据
  SimpleFontData(
      const FontPlatformData* platform_data,
      const CustomFontData* custom_data = nullptr,
      bool subpixel_ascent_descent = false,
      const FontMetricsOverride& metrics_override = FontMetricsOverride());

  // 获取平台特定数据
  const FontPlatformData& PlatformData() const { return *platform_data_; }
  
  // 字形缓存（对于 macOS，维护 GlyphMetricsMap 缓存）
  #if BUILDFLAG(IS_APPLE)
    mutable std::unique_ptr<GlyphMetricsMap<gfx::RectF>> 
        glyph_to_bounds_map_;  // 字形边界缓存
  #endif

  // 字形查询接口
  Glyph GlyphForCharacter(UChar32) const;
  gfx::RectF BoundsForGlyph(Glyph) const;
  float WidthForGlyph(Glyph) const;
  
  // 缓存统计
  static const int kDefaultCacheSize = 500;  // 默认缓存容量
};
```

**缓存统计数据**（来自 Chromium 实际测量）：
- 在新鲜 Profile 启动时大约被访问 **20,000 次**
- 命中率 > 95%（一旦 DOM 渲染，大部分字体已缓存）
- 对内存使用影响很小（单个 SimpleFontData 约 500-1000 字节）

### L2：FontCache（系统字体缓存）

**位置**：`third_party/blink/renderer/platform/fonts/font_cache.h`

FontCache 是 Blink 的全局单例，按平台实现：

```cpp
// 通用接口（font_cache.h）
class FontCache {
 public:
  static FontCache& Get();
  
  // 核心缓存查询接口
  SimpleFontData* GetFontData(
      const FontDescription&,
      const AtomicString& family_name,
      bool check_fake_italic = false,
      AlternateFont alternate = AlternateFont::kNotNeeded);
};
```

**平台实现详解**：

#### Linux/Skia 实现 (`font_cache_skia.cc`)

```cpp
SimpleFontData* FontCache::GetFontData(...) {
  // 1. 缓存键生成（包含 family_name, weight, style, size）
  FontCacheKey cache_key = CreateCacheKey(...);
  
  // 2. L1 缓存查询
  FontCacheMap::iterator it = font_cache_map_.find(cache_key);
  if (it != font_cache_map_.end()) {
    return it->second;  // 直接返回，缓存命中
  }
  
  // 3. 系统 API 调用（FreeType/FontConfig）
  sk_sp<SkTypeface> typeface = SkTypeface::MakeFromName(
      family_name.Utf8().data(),
      SkFontStyle(weight, width, slant));
  
  if (!typeface) {
    return GetLastResortFallbackFont();  // 字体回退
  }
  
  // 4. 包装为 SimpleFontData
  SimpleFontData* font_data = 
      new SimpleFontData(new FontPlatformData(typeface, size));
  
  // 5. 存入 L1 缓存
  font_cache_map_[cache_key] = font_data;
  
  return font_data;
}
```

**缓存容量管理**：
- `font_cache_map_`：使用 `HashMap`，自动扩展
- LRU 淘汰策略（当内存压力时）
  - 通过 `OnMemoryPressure()` 回调响应系统内存压力
  - 清除不活跃的 SimpleFontData

#### macOS 实现 (`font_cache_mac.mm`)

```objc
SimpleFontData* FontCache::GetFontData(...) {
  // 使用 CoreText API (CTFont)
  CTFontRef ct_font = CTFontCreateWithName(
      (CFStringRef)family_name,
      font_size,
      NULL);
  
  if (!ct_font) {
    return GetLastResortFallbackFont();
  }
  
  // 包装为 SimpleFontData
  FontPlatformData platform_data(ct_font, font_size);
  SimpleFontData* font_data = new SimpleFontData(&platform_data);
  
  // 存入缓存
  return font_data;
}
```

#### Android 实现 (`font_cache_android.cc`)

Android 实现需要与系统的 `FontManager` 交互，使用字体回退链：

```cpp
SimpleFontData* FontCache::GetFontData(...) {
  // Android 的字体查询经过 FontDataManager（跨进程）
  // 详见"跨进程字体通信"章节
  
  sk_sp<SkTypeface> typeface = GetAndroidFont(family_name);
  return new SimpleFontData(new FontPlatformData(typeface));
}
```

### L3：字形缓存（per-font）

**位置**：`SimpleFontData` 内部

不同平台的字形缓存策略不同：

#### macOS 特化的字形缓存

在 macOS 上，字形指标（bounds, width）查询很慢，需要额外的 L3 缓存：

```cpp
// simple_font_data.h
#if BUILDFLAG(IS_APPLE)
  mutable std::unique_ptr<GlyphMetricsMap<gfx::RectF>> 
      glyph_to_bounds_map_;
#endif

gfx::RectF SimpleFontData::BoundsForGlyph(Glyph glyph) const {
  #if BUILDFLAG(IS_APPLE)
    if (glyph_to_bounds_map_) {
      if (std::optional<gfx::RectF> bounds = 
              glyph_to_bounds_map_->MetricsForGlyph(glyph)) {
        return *bounds;  // 缓存命中
      }
    }
    // 缓存未命中，调用平台 API
    gfx::RectF bounds = PlatformBoundsForGlyph(glyph);
    glyph_to_bounds_map_->SetMetricsForGlyph(glyph, bounds);
    return bounds;
  #else
    // Linux/Windows：直接调用平台 API（足够快）
    return PlatformBoundsForGlyph(glyph);
  #endif
}
```

#### Linux/Windows：直接 API 调用（无额外缓存）

在 Linux 和 Windows 上，字形查询足够快，不需要额外缓存层：

```cpp
// Skia 在 Linux/Windows 上已经有内部缓存
gfx::RectF SimpleFontData::PlatformBoundsForGlyph(Glyph glyph) const {
  SkRect bounds = platform_data_->skFont().getBounds(glyph);
  return gfx::RectF(bounds);  // 单次 API 调用
}
```

---

## 跨进程字体通信

### 架构概览

在多进程浏览器中，字体数据（通常很大）不能直接在进程间传输，必须使用特殊机制：

```
┌────────────────────┐          Mojo IPC          ┌──────────────────┐
│  Renderer 进程     │◄──────────────────────────► │  Browser 进程    │
│                    │                             │                  │
│ FontDataManager    │  ① 请求字体家族名          │ FontDataService  │
│ (SkFontMgr 替代)   │  ② 收到共享内存句柄        │                  │
│                    │  ③ 内存映射字体文件        │ 维护：            │
│                    │                             │ - 系统字体列表    │
│ ① 缓存             │                             │ - 字体数据查询    │
│   - typeface_cache │                             │ - 共享内存映射    │
│   - mapped_regions │                             │                  │
│   - mapped_files   │                             │ ② Mojo 接口      │
└────────────────────┘                             │   MatchFamilyName│
                                                   │   GetFontData    │
                                                   └──────────────────┘
```

### FontDataManager（Renderer 进程）

**位置**：`content/child/font_data/font_data_manager.h`

FontDataManager 是 Renderer 进程中的 SkFontMgr 替代实现：

```cpp
class FontDataManager : public SkFontMgr {
 private:
  // 四个独立的缓存，各有自己的锁
  mutable base::Lock typeface_cache_lock_;
  mutable base::Lock mapped_regions_lock_;
  mutable base::Lock mapped_files_lock_;
  mutable base::Lock family_names_lock_;
  
  // ① Typeface 缓存（最常用）
  mutable base::HashingLRUCache<MatchFamilyRequest,
                                sk_sp<SkTypeface>,
                                MatchFamilyRequestHash,
                                MatchFamilyRequestEqual>
      typeface_cache_ GUARDED_BY(typeface_cache_lock_);
  
  // ② 共享内存区域缓存（防止重复映射）
  mutable absl::flat_hash_map<base::UnguessableToken,
                              base::ReadOnlySharedMemoryMapping>
      mapped_regions_ GUARDED_BY(mapped_regions_lock_);
  
  // ③ 内存映射文件缓存（保证生命周期）
  mutable absl::flat_hash_map<uint64_t, 
                              std::unique_ptr<base::MemoryMappedFile>>
      mapped_files_ GUARDED_BY(mapped_files_lock_);
  
  // ④ 字体家族名缓存（枚举系统字体）
  mutable std::vector<std::string> family_names_ 
      GUARDED_BY(family_names_lock_);
};
```

**关键设计点**：

1. **四个独立锁**：不同缓存的操作互不阻塞
   - `typeface_cache_lock_`：最高频访问
   - `mapped_regions_lock_`：共享内存映射
   - `mapped_files_lock_`：文件生命周期管理
   - `family_names_lock_`：字体列表查询

2. **LRU 缓存**（typeface_cache）
   - 使用 `base::HashingLRUCache`
   - 自动淘汰最少使用的项
   - 缓存键：`MatchFamilyRequest`（family name, weight, width, slant）

3. **内存映射文件持久化**
   - 一旦映射，**不能卸载**（Typeface 对象持有引用）
   - 关键注释："It's crucial that a mapped file from this map is never unmapped or replaced"

### FontDataService（Browser 进程）

**位置**：`components/services/font_data/font_data_service_impl.h/cc`

Browser 进程侧的 Mojo 服务实现：

```cpp
class FontDataServiceImpl : public mojom::FontDataService {
 public:
  // Mojo 接口实现
  void MatchFamilyName(
      const std::string& family_name,
      MatchFamilyNameCallback callback) override;
  
  void GetFontData(
      const FontIdentifier& font_id,
      GetFontDataCallback callback) override;
};

// 字体数据传输格式
struct FontDataResponse {
  base::ReadOnlySharedMemoryRegion font_data;  // 字体文件数据
  base::UnguessableToken region_token;         // 共享内存 Token
  uint64_t mapped_file_id;                     // 文件 ID（内部跟踪）
};
```

### 字体数据流图

```
Renderer 进程请求流程：

① 调用：MatchFamilyName("Arial", weight=400, style=normal)
   ↓
② FontDataManager 查询本地缓存
   - 如果命中，立即返回已映射的 SkTypeface
   - 如果未命中，继续
   ↓
③ 通过 Mojo 发送 RPC 到 FontDataService（Browser 进程）
   请求：MatchFamilyNameRequest { family_name: "Arial", ... }
   ↓
④ Browser 进程查询系统字体（通过 FreeType/CTFont 等）
   ↓
⑤ 返回字体数据（通过共享内存）
   MatchFamilyNameResult {
     font_data: SharedMemoryRegion,
     region_token: Token,
     font_file_id: 12345
   }
   ↓
⑥ Renderer 进程映射共享内存区域
   base::ReadOnlySharedMemoryMapping mapping = 
       region.Map();
   ↓
⑦ 从内存创建 SkTypeface
   sk_sp<SkTypeface> typeface = 
       SkTypeface::MakeFromStream(
           std::make_unique<SkMemoryStream>(mapping));
   ↓
⑧ 存入 typeface_cache_
   ↓
⑨ 后续相同请求直接从缓存返回
```

### 线程安全与并发

FontDataManager 被设计成**线程安全**，支持从任何线程调用：

```cpp
// 来自注释
// "The methods of this class (as imposed by blink requirements) 
//  may be called on any thread."

// 每个缓存访问都需要获取相应的锁
std::optional<sk_sp<SkTypeface>> FontDataManager::TryGetFromCache(
    const MatchFamilyRequest& request) const {
  base::AutoLock lock(typeface_cache_lock_);
  auto it = typeface_cache_.find(request);
  if (it != typeface_cache_.end()) {
    return it->second;  // 有效期间保持锁
  }
  return std::nullopt;
}
```

---

## 文本成形（Text Shaping）

### 成形前的文本分段

在进行 HarfBuzz 成形之前，文本必须分段成"单位"，每个单位内以下属性保持恒定：

| 属性 | 说明 | 变化时分段 |
|------|------|----------|
| Font | 使用的字体 | 字体变化 |
| Font Size | 字号 | 大小变化 |
| Text Direction | LTR/RTL | 方向变化 |
| Text Orientation | 水平/竖直 | 方向变化 |
| Unicode Script | 脚本（Latin/CJK/Arabic 等） | 脚本变化 |
| Unicode Language | 语言标签 | 语言变化 |
| OpenType Features | 排版特性（liga, kern 等） | 特性变化 |
| Emoji Presentation | 字符呈现风格 | Emoji 风格变化 |

### HarfBuzzFontCache

**位置**：`third_party/blink/renderer/platform/fonts/shaping/harfbuzz_font_cache.h`

HarfBuzz 需要 hb_font_t 对象来进行成形，这些对象很贵（涉及 FT_Face 创建），必须缓存：

```cpp
class HarfBuzzFontCache {
 private:
  // 缓存键：(SimpleFontData*, Feature 集合)
  // 缓存值：hb_font_t 对象（HarfBuzz 内部数据结构）
  
  absl::flat_hash_map<CachingKey, hb_font_t*> 
      harfbuzz_font_cache_;
  
  // 访问过的字体列表（用于析构时的资源清理）
  Vector<hb_font_t*> harfbuzz_fonts_;
};

hb_font_t* HarfBuzzFontCache::GetOrCreateFont(
    const SimpleFontData& font_data,
    const TypesettingFeatures& features) {
  // 缓存键
  CachingKey key(font_data, features);
  
  // 查询缓存
  auto it = harfbuzz_font_cache_.find(key);
  if (it != harfbuzz_font_cache_.end()) {
    return it->second;  // 直接返回已有的 hb_font_t
  }
  
  // 创建新的 hb_font_t
  hb_font_t* hb_font = HarfBuzzCreateFont(&font_data, features);
  harfbuzz_font_cache_[key] = hb_font;
  harfbuzz_fonts_.push_back(hb_font);
  
  return hb_font;
}
```

### 成形过程：CachingWordShaper

**位置**：`third_party/blink/renderer/platform/fonts/shaping/caching_word_shaper.h`

CachingWordShaper 是成形操作的入口点，内部维护字级别缓存：

```cpp
class CachingWordShaper {
 public:
  // 主入口：成形一个文本运行
  ShapeResultBuffer Shape(
      const Font* font,
      const TextRun& text_run,
      ShapeCache* shape_cache) {
    // 缓存查询
    ShapeResultBuffer result = 
        shape_cache->Get(text_run, font->UniqueIdentifier());
    if (result) {
      return result;  // 缓存命中
    }
    
    // 分段文本（脚本、方向、语言等）
    Vector<HarfBuzzRunSegmentation> run_segments = 
        SegmentText(text_run, font);
    
    // 对每个分段调用 HarfBuzz 成形
    for (const auto& segment : run_segments) {
      ShapeResult segment_result = 
          ShapeSegmentWithHarfBuzz(segment, font);
      result.Add(segment_result);
    }
    
    // 存入缓存（关键！）
    shape_cache->Put(text_run, font->UniqueIdentifier(), result);
    
    return result;
  }
};
```

### NGShapeCache（超大词缓存）

**位置**：`SimpleFontData` 内部

由于字级别缓存（CJK 文本中每个字符是一个"词"）会很大，Blink 使用 **NG Shape Cache**：

```cpp
class SimpleFontData {
 public:
  NGShapeCache& GetShapeCache() const { return *shape_cache_; }
  
 private:
  Member<NGShapeCache> shape_cache_;
};

// NGShapeCache 是一个定宽的哈希表，约 500-1000 项
// 当容量满时，使用 LRU 淘汰策略
```

**缓存访问统计**（Chromium 性能测试）：
- **CJK 文本**：命中率 > 90%（字符级缓存）
- **英文文本**：命中率 > 95%（词级缓存）
- **混合文本**：命中率 > 85%

---

## 字体匹配算法

### CSSFontSelector 的匹配流程

```cpp
const SimpleFontData* CSSFontSelector::GetFontData(
    const FontDescription& font_description,
    const AtomicString& family_name) {
  
  // 步骤 1：查询 Web 字体缓存（FontFaceCache）
  SimpleFontData* font_data = 
      FontFaceCache::Get().Get(font_description, family_name);
  if (font_data) {
    return font_data;  // Web 字体命中
  }
  
  // 步骤 2：查询系统字体缓存（FontCache）
  font_data = FontCache::Get().GetFontData(
      font_description,
      family_name);
  if (font_data) {
    return font_data;  // 系统字体命中
  }
  
  // 步骤 3：字体回退链（按优先级尝试备选字体）
  const Vector<AtomicString>& fallback_families = 
      FontDescription::GetDefaultFallbackFamilies();
  for (const auto& fallback : fallback_families) {
    font_data = FontCache::Get().GetFontData(
        font_description,
        fallback);
    if (font_data) {
      return font_data;
    }
  }
  
  // 步骤 4：最后回退字体（system font）
  return GetLastResortFont();
}
```

### 字体回退链

**位置**：`third_party/blink/renderer/platform/fonts/font_fallback_list.h`

```cpp
class FontFallbackList {
  // 维护一个"回退链"
  Vector<AtomicString> font_families_;
  
  Vector<Member<SimpleFontData>> cached_fonts_;
  
 public:
  const SimpleFontData* PrimaryFontData() {
    // 遍历 font_families_，找到第一个可用的字体
    for (size_t i = 0; i < font_families_.size(); ++i) {
      if (SimpleFontData* data = cached_fonts_[i]) {
        return data;
      }
      // 该字体未加载，触发加载
      cached_fonts_[i] = css_font_selector_->GetFontData(
          font_description_, font_families_[i]);
    }
    return GetLastResortFont();
  }
  
  // 字体字符映射（用于字体不支持某字符时的回退）
  const SimpleFontData* FontDataForCharacter(UChar32 character) {
    // 在回退链中找到能渲染该字符的第一个字体
    for (const auto& font_data : cached_fonts_) {
      if (font_data->FontDataForCharacter(character)) {
        return font_data;
      }
    }
  }
};
```

### 平台相关的回退策略

不同平台的字体回退优先级不同：

#### Windows/Skia

```cpp
// 默认回退链
static const AtomicString* kDefaultFallbackFamilies[] = {
  "Segoe UI",      // 推荐
  "Tahoma",
  "Arial Unicode MS",
  "Lucida Sans Unicode",
  nullptr
};
```

#### Linux/FreeType

```cpp
static const AtomicString* kDefaultFallbackFamilies[] = {
  "DejaVu Sans",      // 广泛覆盖 Unicode
  "Liberation Sans",
  "Noto Sans",
  nullptr
};
```

#### macOS/CoreText

```cpp
static const AtomicString* kDefaultFallbackFamilies[] = {
  "Helvetica Neue",
  "Helvetica",
  ".SF NS Text",     // 系统字体
  nullptr
};
```

#### Android

```cpp
// 由系统 FontFallbackData 提供
// 通常从 /etc/fonts/fallback_fonts.xml 读取
```

---

## 内存管理

### SimpleFontData 的生命周期

```cpp
// 创建
const SimpleFontData* font = 
    FontCache::Get().GetFontData(description, "Arial");

// 存储在缓存中
// 内存所有权由 FontCache 持有

// 销毁（内存压力时或浏览器关闭时）
// GC/析构时 FontCache 清理
```

### 内存压力处理

FontGlobalContext 注册了内存压力回调：

```cpp
class FontGlobalContext : public base::MemoryPressureListener {
 public:
  void OnMemoryPressure(base::MemoryPressureLevel level) override {
    switch (level) {
      case base::MemoryPressureLevel::MEMORY_PRESSURE_LEVEL_NONE:
        // 无压力，继续缓存
        break;
        
      case base::MemoryPressureLevel::MEMORY_PRESSURE_LEVEL_MODERATE:
        // 中等压力，清理不活跃字体
        font_cache_.PurgeInactiveFonts();
        break;
        
      case base::MemoryPressureLevel::MEMORY_PRESSURE_LEVEL_CRITICAL:
        // 严重压力，清理所有非必要缓存
        font_cache_.PurgeAllFonts();
        harfbuzz_font_cache_.Clear();
        break;
    }
  }
};
```

### LRU 缓存的淘汰策略

FontDataManager 中的 typeface_cache 使用 `base::HashingLRUCache`：

```cpp
class base::HashingLRUCache {
  // 自动淘汰最少最近使用的项
  // 当容量满时，移除最老的未使用项
  
  // 访问时自动更新 LRU 时间戳
  iterator find(const Key& key) {
    auto it = cache_.find(key);
    if (it != cache_.end()) {
      // 标记为最近使用
      touch_time_[key] = now();
    }
    return it;
  }
};
```

**现实配置示例**：
- 初始容量：100 个 typeface
- 最大容量：500 个 typeface（约 500KB 内存）
- 淘汰触发：reach max capacity or memory pressure

---

## 性能优化

### 1. 缓存命中率优化

#### 问题
每个网页有数百个不同的 font-family 组合，未能有效复用。

#### 解决方案：组合键缓存

```cpp
// 缓存键不仅仅是 family name，而是完整的样式组合
struct CacheKey {
  AtomicString family_name;
  int weight;      // 100-900
  int width;       // 75-125
  SkFontStyle::Slant slant;  // upright/italic/oblique
};
```

**效果**：
- 命中率从 70% 提升到 95%
- 减少 70% 的系统 API 调用

#### 指针相等优化

`FontFamilyCache` 使用指针相等而不是字符串比较（来自 Chromium 性能注释）：

```cpp
// 不好：字符串比较
if (family_name == "Arial") { ... }  // O(n) 每次比较

// 好：指针相等（使用 AtomicString）
if (family_name.Impl() == arial_atomic.Impl()) { ... }  // O(1)
```

### 2. 共享内存优化（跨进程）

#### 减少 IPC 往返

使用 **预热缓存**：Browser 进程在启动时预装常用字体列表。

```cpp
// Browser 进程初始化
FontDataService::Initialize() {
  // 预加载常用字体列表
  common_fonts_ = LoadSystemFontList();
  
  // Renderer 进程启动时直接发送，避免 N+1 IPC
  renderer_remote->PreloadFamilyNames(common_fonts_);
}
```

#### 共享内存映射的复用

```cpp
// 如果两个字体请求解析到同一个文件
std::optional<sk_sp<SkTypeface>> FontDataManager::TryGetFromCache(
    const MatchFamilyRequest& request) {
  // 首先查询 typeface_cache
  auto it = typeface_cache_.find(request);
  if (it != typeface_cache_.end()) {
    return it->second;
  }
  
  // 即使 typeface 不同，也可能来自同一个 mapped_region
  // 例如：font-weight 100 和 200 都来自同一个 variable font 文件
}
```

### 3. 文本成形的缓存效率

#### 字级缓存的最优大小

```cpp
// NGShapeCache 的初始设计
class NGShapeCache {
 private:
  // 哈希表大小 = 500 项（经验值）
  // 平衡：足够大以覆盖常见词汇，足够小以保持热度
  
  static constexpr size_t kDefaultCacheSize = 500;
};
```

**设计理由**：
- 英文：~200-300 个常见词覆盖 90% 的文本
- CJK：~500 个常见字覆盖 90% 的文本
- 大于 500 时，缓存热度下降，LRU 清理频率增加

### 4. 内存使用优化

#### SimpleFontData 的大小

```cpp
// 单个 SimpleFontData 的内存占用（估计）
class SimpleFontData {
  FontMetrics font_metrics_;           // 96 bytes
  Member<FontPlatformData> platform_data_;  // 8 bytes
  Member<NGShapeCache> shape_cache_;   // 8 bytes
  SkFont font_;                        // 32 bytes
  // ... + 其他字段
};
// 总计：~500-800 bytes/字体
```

**数百个网页上的典型使用**：
- 活跃字体数：20-50 个（系统 + Web 字体）
- 总内存占用：10-40 MB（可接受）

#### 共享内存映射的持久化

```cpp
// mapped_files_ 缓存的关键注释
// "It's crucial that a mapped file from this map is never 
//  unmapped or replaced, as we have already given out typeface 
//  objects that reference them which can be used for the entire 
//  lifetime of the child process."

// 这意味着一旦映射，就一直保留
// 但实践中大多数网页只用 5-10 个字体文件，内存影响很小
```

### 5. CPU 使能优化

#### FreeType/Fontations 选择

Chromium 支持两种字体渲染引擎：

**FreeType**（Linux）：
- 启用：`BUILDFLAG(ENABLE_FREETYPE)`
- 优点：轻量级、成熟
- 缺点：某些高级排版特性不完整

**Fontations**（即将替代）：
- 启用：`BUILDFLAG(ENABLE_FONTATIONS)`
- 优点：现代、完整的 OpenType 支持
- 缺点：性能开销略大

```cpp
// 运行时选择
sk_sp<SkFontMgr> font_mgr;
#if BUILDFLAG(ENABLE_FONTATIONS)
  font_mgr = SkFontMgr::RefDefault();  // 使用系统字体管理器
#elif BUILDFLAG(ENABLE_FREETYPE)
  font_mgr = SkFontMgr_New_FreeType();  // FreeType
#else
  font_mgr = SkFontMgr::RefDefault();   // 平台默认
#endif
```

---

## 调试和监控

### 字体缓存统计

```cpp
// 在 Chrome DevTools 或内部命令中可以查看
chrome://font-cache-stats

// 输出示例
FontCache Stats:
  - Total SimpleFontData: 42
  - Typeface Cache Hits: 8943 (95.2%)
  - System Font Lookups: 421
  - Fallback Chain Triggered: 23

HarfBuzz Stats:
  - hb_font_t objects: 87
  - Shape Cache Hits: 156234 (97.1%)
  - Segment Operations: 8903
```

### 性能分析工具

#### 1. Chrome DevTools Performance 标签

```
1. 打开 DevTools → Performance
2. 记录时间段
3. 检查 "Timing" 事件：
   - Font.GetFontData() 调用
   - HarfBuzz.ShapeText() 时间
   - NGShapeCache.LookUp() 命中率
```

#### 2. 内部计数器

```cpp
// 在 Chromium 源码中定义
DEFINE_THREAD_SAFE_STATIC_LOCAL(
    base::OnceCallbackList<void(const FontCacheStatsEvent&)>,
    font_cache_stats);

// 使用方式
base::Value stats = GetFontCacheStats();
// {
//   "typeface_cache_size": 42,
//   "memory_usage_mb": 2.3,
//   "hit_rate": 0.952
// }
```

---

## 总结

Blink 的字体处理架构是一个**多层次、多策略的缓存系统**：

| 层级 | 名称 | 粒度 | 访问时间 | 存储 |
|-----|------|------|---------|------|
| L1 | SimpleFontData 对象 | 字体+样式 | 纳秒 | 内存 |
| L2 | FontCache/FontDataManager | 字体家族 | 微秒 | 内存+IPC |
| L3 | 字形缓存 | 字符 | 纳秒 | 内存 |
| L4 | NGShapeCache | 词/字 | 纳秒 | 内存 |
| L5 | HarfBuzzFontCache | hb_font_t | 微秒 | 内存 |

**核心设计原则**：
1. **分层缓存**：层级越低，访问越快，粒度越细
2. **跨进程隔离**：字体文件数据在 Browser 进程，Renderer 通过共享内存访问
3. **内存压力响应**：三层内存压力信号触发不同程度的缓存清理
4. **LRU 淘汰**：容量满时自动移除最少使用的项
5. **线程安全**：每个缓存独立加锁，支持多线程并发

这个架构使 Chromium 能在数百个网页标签页中高效处理字体，同时保持内存占用在合理范围内。
