# Android 字体加载与 Chromium 渲染引擎整合

## 字体加载在渲染流程中的位置

```
HTML/CSS 输入
    ↓
HTML 解析
    ├─ <p style="font-family: Roboto">Hello</p>
    └─ CSS font-family 提取: "Roboto"
    ↓
样式计算 (CSS resolution)
    ├─ 字体属性：
    │  ├─ font-family: "Roboto"
    │  ├─ font-size: 16px
    │  ├─ font-weight: 400
    │  ├─ font-style: normal
    │  └─ font-variant: normal
    │
    └─ 创建 ComputedStyle 对象
    ↓
布局 (Layout)
    ├─ 计算文本宽度、高度
    ├─ 需要字体指标
    │  ├─ ascent, descent
    │  ├─ line-height
    │  └─ baseline
    │
    ├─ [触发] 字体查询
    │  └─ CSSFontSelector::GetFontData()
    │     └─ 调用 SkFontMgr::matchFamilyStyle()
    │        └─ 返回 SkTypeface
    │
    └─ 创建 FontDescription 对象
    ↓
形状化 (Shaping with HarfBuzz)
    ├─ 输入：文本 + 字体
    │  ├─ "Hello"
    │  └─ Roboto-Regular
    │
    ├─ HarfBuzz 处理：
    │  ├─ Unicode → Glyph ID
    │  ├─ 应用 OpenType 特性
    │  │  ├─ ligatures (fi → ﬁ)
    │  │  ├─ kerning (Av → 更紧)
    │  │  └─ others
    │  │
    │  └─ 生成 ShapeResult
    │     ├─ [H(123), e(456), l(789), l(789), o(901)]
    │     └─ glyph_id, advance_width, offset
    │
    └─ 创建 LogicalRun 对象
    ↓
绘制准备 (Paint phase)
    ├─ 创建 SkTextBlob
    │  ├─ 搜集所有字形
    │  ├─ 搜集所有位置
    │  └─ 关联 SkFont/SkTypeface
    │
    ├─ 构建 PaintRecord
    │  └─ 包含 DrawTextBlobOp
    │
    └─ 准备渲染命令
    ↓
光栅化 (Rasterization)
    ├─ [延迟加载] 从磁盘读取字体文件
    │  └─ 仅在此刻真正加载完整数据
    │
    ├─ Skia 渲染：
    │  ├─ 获取字形轮廓 (SkTypeface::getPath)
    │  ├─ 应用变换 (位置、缩放)
    │  ├─ 光栅化 (扫描线算法)
    │  └─ 抗锯齿处理
    │
    └─ 输出像素到 GPU
    ↓
显示
```

---

## 关键类之间的交互

### 1. CSSFontSelector 和 SkFontMgr 的交互

**位置**: [third_party/blink/renderer/core/css/css_font_selector.cc]

```cpp
// CSSFontSelector 负责 CSS 层的字体选择
class CSSFontSelector {
 public:
  // 核心方法：获取字体数据
  FontData* GetFontData(const FontDescription& font_description) {
    // 1️⃣ 构建查询
    const AtomicString& family = font_description.Family();
    SkFontStyle skia_style(
        static_cast<int>(font_description.Weight()),
        SkFontStyle::kNormal_Width,
        font_description.IsItalic() ? SkFontStyle::kItalic_Slant
                                     : SkFontStyle::kUpright_Slant
    );
    
    // 2️⃣ 调用 SkFontMgr
    sk_sp<SkTypeface> typeface = 
        skia::DefaultFontMgr()->matchFamilyStyle(
            family.Utf8().data(),
            skia_style
        );
    
    if (!typeface) {
      // 字体未找到，使用备选字体
      typeface = skia::MakeTypefaceFromName("Roboto", skia_style);
    }
    
    // 3️⃣ 创建 FontPlatformData
    FontPlatformData platform_data(
        typeface,
        font_description.GetFontSelectionRequest().size,
        font_description.IsSyntheticBold(),
        font_description.IsSyntheticItalic()
    );
    
    // 4️⃣ 创建 SimpleFontData
    auto* font_data = 
        MakeGarbageCollected<SimpleFontData>(platform_data);
    
    // 5️⃣ 缓存并返回
    return font_data;
  }
  
 private:
  sk_sp<SkFontMgr> font_mgr_ = skia::DefaultFontMgr();
};
```

### 2. SimpleFontData 和 SkTypeface 的交互

**位置**: [third_party/blink/renderer/platform/fonts/simple_font_data.h]

```cpp
// SimpleFontData 是 Chromium 平台字体数据的抽象
class SimpleFontData {
 private:
  // 关键成员
  FontPlatformData platform_data_;
  
  // platform_data_ 包含：
  // - SkTypeface* typeface
  // - float text_size
  // - bool synthetic_bold
  // - bool synthetic_italic
  // - CustomFontData* custom_data
  
 public:
  // 获取 Skia 字体对象
  SkFont GetSkFont() const {
    return platform_data_.CreateSkFont();
  }
  
  // 获取 Skia 字体属性
  SkTypeface* GetTypeface() const {
    return platform_data_.Typeface();
  }
  
  // 获取字形 ID
  uint16_t GlyphForCharacter(UChar32 c) const {
    return platform_data_.CharacterToGlyphID(c);
  }
};
```

### 3. FontPlatformData 的实现

**位置**: [third_party/blink/renderer/platform/fonts/font_platform_data.h]

```cpp
class FontPlatformData {
 private:
  sk_sp<SkTypeface> typeface_;  // ← SkFontMgr 返回的结果
  float text_size_;
  bool synthetic_bold_;
  bool synthetic_italic_;
  
 public:
  // 创建 SkFont 对象
  SkFont CreateSkFont() const {
    SkFont font;
    font.setTypeface(typeface_);        // 设置字体
    font.setSize(text_size_);           // 设置大小
    font.setSkewX(synthetic_italic_ ? kSkewX : 0);
    font.setScaleX(1.0f);
    
    // 其他设置...
    font.setHinting(SkFontHinting::kNormal);
    font.setEdging(SkFont::Edging::kSubpixelAntiAlias);
    
    return font;
  }
  
  // 获取字形 ID (Unicode → 字形映射)
  uint16_t CharacterToGlyphID(UChar32 c) const {
    return CreateSkFont().unicharToGlyph(c);
  }
};
```

---

## 完整渲染示例：从 HTML 到像素

### 输入 HTML

```html
<p style="font-family: Roboto, sans-serif; font-weight: bold;">
  Hello 世界
</p>
```

### 执行流程

```
阶段 1: HTML 解析
  ├─ 文本内容: "Hello 世界"
  ├─ 字体家族: ["Roboto", "sans-serif"]
  ├─ 字体权重: bold (700)
  └─ 字体样式: normal (0)

阶段 2: 样式计算
  ├─ 当前 DOM 元素 <p>
  ├─ 计算后的样式：
  │  ├─ font-family: Roboto (或回退)
  │  ├─ font-size: 16px (假设)
  │  ├─ font-weight: 700
  │  └─ font-style: normal
  │
  └─ 创建 ComputedStyle

阶段 3: 布局计算
  │
  ├─ 获取字体指标
  │  ├─ [调用] CSSFontSelector::GetFontData()
  │  │  └─ FontDescription {
  │  │      family: "Roboto",
  │  │      size: 16,
  │  │      weight: 700,
  │  │      style: normal
  │  │     }
  │  │
  │  ├─ [查询] skia::DefaultFontMgr()->matchFamilyStyle(
  │  │     "Roboto", 
  │  │     SkFontStyle(700, 100, 0)
  │  │  )
  │  │
  │  ├─ SkFontMgr_Android::matchFamilyStyle()
  │  │  ├─ 在 fFamilies 找到 "Roboto" 家族
  │  │  ├─ 查询 StyleSet 中权重最接近 700 的字体
  │  │  └─ 返回: Roboto-Bold ✓
  │  │
  │  ├─ 获得 SkTypeface
  │  │  └─ sk_sp<SkTypeface> = SkTypeface_Android {
  │  │      path: "/system/fonts/Roboto-Bold.ttf",
  │  │      ttcIndex: 0,
  │  │      ...
  │  │     }
  │  │
  │  ├─ 创建 FontPlatformData
  │  │  └─ FontPlatformData {
  │  │      typeface_: sk_sp<SkTypeface>,
  │  │      text_size_: 16.0,
  │  │      synthetic_bold_: false,
  │  │      synthetic_italic_: false
  │  │     }
  │  │
  │  ├─ 创建 SimpleFontData
  │  │  └─ SimpleFontData {
  │  │      platform_data_: FontPlatformData,
  │  │      ...
  │  │     }
  │  │
  │  └─ 获得字体指标 (ascent, descent, etc.)
  │
  └─ 计算文本布局
     ├─ "Hello" 占用宽度: ~40px
     ├─ "世界" 占用宽度: ~40px
     └─ 总宽度: 80px

阶段 4: 文本形状化 (Shaping)
  │
  ├─ 对 "Hello"
  │  ├─ 字体: Roboto-Bold
  │  ├─ Unicode → Glyph: [H(123) → 123, e(101) → 456, ...]
  │  ├─ HarfBuzz 处理：
  │  │  └─ 无特殊连字或 kerning
  │  │
  │  └─ 输出: ShapeResult
  │     ├─ H: glyph=123, advance=10.2
  │     ├─ e: glyph=456, advance=9.8
  │     ├─ l: glyph=789, advance=4.5
  │     ├─ l: glyph=789, advance=4.5
  │     └─ o: glyph=901, advance=10.0
  │
  ├─ 对 "世界" (字体回退)
  │  ├─ Roboto-Bold 不支持 CJK
  │  │  └─ 触发字体回退机制
  │  │
  │  ├─ 查询: matchFamilyStyleCharacter(
  │  │     "sans-serif",
  │  │     SkFontStyle(700, 100, 0),
  │  │     bcp47=["zh"]
  │  │  )
  │  │
  │  ├─ SkFontMgr_Android 搜索：
  │  │  ├─ 检查 Roboto 家族的 Unicode 覆盖范围
  │  │  │  └─ 不包含 CJK (U+4E00-U+9FFF)
  │  │  │
  │  │  ├─ 检查 "Noto Sans" 家族
  │  │  │  └─ 也不包含 CJK
  │  │  │
  │  │  ├─ 检查 "Noto Sans CJK" 家族 ✓
  │  │  │  └─ 包含 Chinese Unicode 范围
  │  │  │
  │  │  └─ 返回最佳匹配：Noto Sans CJK
  │  │
  │  ├─ 字体: Noto Sans CJK
  │  ├─ Unicode → Glyph: [世(U+4E16) → 9999, 界(U+754C) → 8888]
  │  │
  │  └─ 输出: ShapeResult
  │     ├─ 世: glyph=9999, advance=20.0
  │     └─ 界: glyph=8888, advance=20.0
  │
  └─ 完整 ShapeResult
     ├─ 字形序列: [123, 456, 789, 789, 901, 9999, 8888]
     ├─ 位置序列: [(0,0), (10.2,0), (20.0,0), (24.5,0), (29.0,0), (39.0,0), (59.0,0)]
     ├─ 字体映射: [Roboto-Bold ×5, NotoSansCJK ×2]
     └─ 总宽度: 79.0px

阶段 5: 绘制准备
  │
  ├─ 创建 PaintRecord
  │  ├─ 包含所有绘制操作
  │  └─ 主要操作: DrawTextBlobOp
  │
  ├─ 构建 SkTextBlob
  │  │
  │  ├─ 为 Roboto-Bold 部分创建 Run
  │  │  ├─ glyphs: [123, 456, 789, 789, 901]
  │  │  ├─ positions: [(0,0), (10.2,0), (20.0,0), (24.5,0), (29.0,0)]
  │  │  └─ font: SkFont(Roboto-Bold, size=16)
  │  │
  │  └─ 为 NotoSansCJK 部分创建 Run
  │     ├─ glyphs: [9999, 8888]
  │     ├─ positions: [(39.0,0), (59.0,0)]
  │     └─ font: SkFont(NotoSansCJK, size=16)
  │
  └─ DrawTextBlobOp {
      blob: sk_sp<SkTextBlob>,
      x: 0,
      y: 0,
      flags: {
        color: black,
        antiAlias: true,
        ...
      }
     }

阶段 6: 光栅化 (Rasterization)
  │
  ├─ [第一次加载字体文件]
  │  ├─ 打开 /system/fonts/Roboto-Bold.ttf
  │  │  └─ 读取 ~1MB 字体数据到内存
  │  │
  │  └─ 打开 /system/fonts/NotoSansCJK.ttc
  │     ├─ 定位到正确的 ttcIndex (中文)
  │     └─ 读取 ~10MB 字体数据到内存
  │
  ├─ 对 Roboto-Bold Run：
  │  ├─ 对每个字形 (123, 456, 789, 789, 901)
  │  │  ├─ typeface->getPath(glyph_id, &path)
  │  │  │  └─ 从字体文件获取轮廓
  │  │  │
  │  │  ├─ path.transform(position, scale)
  │  │  │  └─ 应用位置和大小变换
  │  │  │
  │  │  ├─ canvas->drawPath(path, paint)
  │  │  │  └─ 光栅化轮廓为像素
  │  │  │
  │  │  └─ 抗锯齿处理
  │  │     └─ 边界像素混合
  │
  ├─ 对 NotoSansCJK Run：
  │  ├─ 对每个字形 (9999, 8888)
  │  │  ├─ typeface->getPath(glyph_id, &path)
  │  │  ├─ path.transform(position, scale)
  │  │  ├─ canvas->drawPath(path, paint)
  │  │  └─ 抗锯齿处理
  │
  └─ 输出
     ├─ 文本被光栅化为像素
     └─ 存储在帧缓冲中

阶段 7: 组合和显示
  ├─ 将所有图层合成
  ├─ 应用变换、滤镜等
  ├─ 输出到 GPU
  └─ 显示在屏幕上: "Hello 世界"
```

---

## 缓存层次

### 第一层：字体管理器缓存

```
全局单例: skia::DefaultFontMgr()
  ├─ 创建：初始化时
  ├─ 大小：~130 KB
  ├─ 内容：字体家族索引
  ├─ 生命周期：进程生命期
  └─ 访问：O(1) 查询
```

### 第二层：SkTypeface 缓存

```
sk_sp<SkFontMgr> 内部缓存
  ├─ 创建：matchFamilyStyle() 调用时
  ├─ 大小：~100 MB (取决于应用使用的字体)
  ├─ 内容：SkTypeface 对象
  │  ├─ SkTypeface_Android {
  │  │  path: "/system/fonts/Roboto-Bold.ttf",
  │  │  ttcIndex: 0,
  │  │  weight: 700,
  │  │  ...
  │  ├─ ...
  │  └─ (完整的元数据，但不是实际字体数据)
  │
  ├─ 生命周期：引用计数自动管理
  └─ 访问：引用计数 get() 操作 O(1)
```

### 第三层：字体文件缓存

```
Skia 字体文件缓存
  ├─ 创建：getPath() 调用时
  ├─ 大小：按需加载
  │  ├─ Roboto-Bold: ~1 MB
  │  ├─ NotoSansCJK: ~10 MB
  │  └─ 其他字体: ~1-2 MB each
  │
  ├─ 内容：完整字体数据
  │  ├─ glyf 表 (字形轮廓)
  │  ├─ cmap 表 (字符映射)
  │  └─ 其他表
  │
  ├─ 生命周期：应用驱动的持久缓存
  └─ 优化：LRU 策略淘汰旧字体
```

### 第四层：字形光栅化缓存

```
GPU 字形缓存
  ├─ 创建：drawPath() 调用时
  ├─ 大小：~50-200 MB (GPU 显存)
  ├─ 内容：光栅化的字形图像
  │  ├─ 高频字形：保留
  │  ├─ 低频字形：按需生成
  │  └─ 不同大小的变体
  │
  ├─ 生命周期：帧驱动
  └─ 优化：SkStrikeCache (Skia 字形缓存)
```

### 缓存效能数据

```
查询类型           首次 (ms)   缓存命中 (ms)   改进倍数
─────────────────────────────────────────────────
字体管理器查询      5-20       <0.1           50-200×
SkTypeface 获取     0.1-1      0.01           10×
字形光栅化          1-10       0.01           100-1000×
─────────────────────────────────────────────────
```

---

## 性能优化建议

### 1. 预加载常用字体

```cpp
// 在应用启动时提前加载常用字体
void PreloadCommonFonts() {
  auto* font_mgr = skia::DefaultFontMgr();
  
  // 预加载这些字体以避免首次查询延迟
  std::vector<std::string> common_families = {
    "Roboto",
    "Noto Sans",
    "Noto Sans CJK"
  };
  
  for (const auto& family : common_families) {
    font_mgr->matchFamilyStyle(family.c_str(), SkFontStyle());
  }
}
```

### 2. 减少字体切换

```cpp
// ❌ 不好：频繁切换字体
for (const auto& char : text) {
  if (IsArabic(char)) {
    UseFont("Noto Sans Arabic");
  } else if (IsCJK(char)) {
    UseFont("Noto Sans CJK");
  } else {
    UseFont("Roboto");
  }
}

// ✓ 好：批处理相同字体的字符
std::vector<TextRun> runs = ShapeTextWithFontFallback(text);
for (const auto& run : runs) {
  DrawTextRun(run);
}
```

### 3. 使用可变字体

```cpp
// 可变字体支持连续的权重/宽度/样式变化
// 比维护多个字体文件更高效

// ✓ 使用可变字体
sk_sp<SkTypeface> variable_roboto = 
    font_mgr->matchFamilyStyle("Roboto[wght]", SkFontStyle());

// 无需加载额外文件，直接渲染任意权重
SkFont font(variable_roboto);
font.setWeight(600);  // 中权重，自动插值生成
```

### 4. 字体预热

```cpp
// 预先渲染一些字形到 GPU 缓存
void WarmupGlyphCache() {
  for (const auto& font : important_fonts) {
    for (char c : "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789,.!?") {
      RenderGlyph(font, c);  // 触发光栅化和 GPU 缓存
    }
  }
}
```

---

## 调试和诊断

### 启用详细日志

```cpp
// content/renderer/renderer_main_platform_delegate_android.cc
void RendererMainPlatformDelegate::PlatformInitialize() {
  // 启用 Skia 字体日志
  setenv("SKIA_DEBUG_FONTMGR", "1", 1);
  setenv("SKIA_DEBUG_TEXT", "1", 1);
  
  // 初始化字体管理器
  auto mgr = skia::DefaultFontMgr();
  
  // 输出诊断信息
  VLOG(1) << "Loaded " << mgr->countFamilies() << " font families";
}
```

### 性能分析

```cpp
// 测量字体查询时间
base::TimeTicks start = base::TimeTicks::Now();

sk_sp<SkTypeface> tf = font_mgr->matchFamilyStyle("Roboto", style);

base::TimeDelta duration = base::TimeTicks::Now() - start;
VLOG(1) << "Font query took " << duration.InMilliseconds() << " ms";
```

### 内存分析

```cpp
// 估算内存占用
void AnalyzeFontMemory() {
  auto* font_mgr = skia::DefaultFontMgr();
  
  // 字体家族数量
  int families = font_mgr->countFamilies();
  LOG(INFO) << "Font families: " << families;
  
  // 遍历并输出统计
  for (int i = 0; i < families; ++i) {
    SkString name;
    font_mgr->getFamilyName(i, &name);
    auto* style_set = font_mgr->createStyleSet(i);
    LOG(INFO) << "Family " << name.c_str() << ": " 
              << style_set->count() << " styles";
  }
}
```

---

## 总结

```
Android 字体加载与渲染流程：

入口点：
  ├─ RendererMainPlatformDelegate::PlatformInitialize()
  └─ skia::DefaultFontMgr() [单例初始化]

初始化流程：
  ├─ fontmgr_factory() [选择 NDK 或传统 API]
  ├─ SkFontMgr_New_Android() [创建管理器]
  ├─ scanSystemFonts() [扫描 /system/fonts/ 等]
  ├─ Fontations [解析字体元数据]
  └─ buildFamilyMap() [组织索引]

查询流程：
  ├─ CSSFontSelector::GetFontData()
  ├─ skia::DefaultFontMgr()->matchFamilyStyle()
  ├─ SkFontMgr_Android::matchFamilyStyle()
  └─ 返回 sk_sp<SkTypeface>

渲染流程：
  ├─ FontPlatformData [保存 SkTypeface]
  ├─ SimpleFontData [高级字体抽象]
  ├─ HarfBuzz shaping [Unicode → Glyph]
  ├─ SkTextBlob [构建渲染命令]
  └─ Skia 光栅化 [绘制到屏幕]

性能特点：
  ├─ 初始化时间：50-100ms
  ├─ 查询延迟：<1ms (缓存)
  ├─ 内存占用：150MB+
  ├─ 首次渲染：可能需要从磁盘加载
  └─ 后续渲染：极快 (多层缓存)

最佳实践：
  ✓ 在沙箱前初始化字体
  ✓ 利用字体回退处理 CJK
  ✓ 使用可变字体减少文件
  ✓ 批处理文本渲染操作
  ✓ 监控和优化缓存大小
```
