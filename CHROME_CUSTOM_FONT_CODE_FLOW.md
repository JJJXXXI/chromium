# Chrome 自定义字体功能 - 代码执行流详解

## 完整的代码执行链路

### Step 1: 用户在 Settings UI 中改变字体选择

**文件**: `chrome/browser/resources/settings/appearance_page/appearance_fonts_page.ts`

```typescript
// HTML 中的字体选择下拉菜单
<select id="serif-font-selector"
        data-pref="webkit.webprefs.fonts.serif.Latn"
        on-change="onFontChanged_">
  <option value="">Default</option>
  <option value="Times New Roman">Times New Roman</option>
  <option value="Georgia">Georgia</option>
  <option value="Garamond">Garamond</option>
</select>

// PolymerElement 中的处理
export class SettingsAppearanceFontsPageElement extends
    SettingsAppearanceFontsPageElementBase {
  
  // 使用 Polymer 的双向绑定
  // 用户选择 "Georgia" 时:
  // 1. DOM 更新: <select>.value = "Georgia"
  // 2. 触发 on-change 事件
  // 3. Polymer 绑定自动更新 prefs 对象
  // 4. prefs.webkit.webprefs.fonts.serif.Latn 变为 "Georgia"
  // 5. SettingsPrivate API 通知浏览器进程
}
```

---

### Step 2: 浏览器进程接收偏好变化

**文件**: `chrome/common/pref_names.h` 和 `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
// 启动时：注册所有字体偏好
void PrefsTabHelper::RegisterProfilePrefs(
    user_prefs::PrefRegistrySyncable* registry,
    const std::string& locale) {
  
  // 注册字体大小
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  
  // 注册字体族偏好
  // 对每个通用族 (standard, serif, fixed, sans-serif, etc.)
  // 和每个 ICU 脚本代码 (Latn, Hant, Cyrl, etc.)
  // 创建一个偏好条目
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  
  // 完整的偏好注册示例:
  // registry->RegisterStringPref(
  //   "webkit.webprefs.fonts.serif.Latn", 
  //   "Times New Roman");  // 默认值
  // registry->RegisterStringPref(
  //   "webkit.webprefs.fonts.serif.Hant",
  //   "宋体");
  // registry->RegisterStringPref(
  //   "webkit.webprefs.fonts.serif.Hans",
  //   "宋体");
  // 等等...
}

// 启动 PrefWatcher 以监听偏好变化
void PrefsTabHelper::GetServiceInstance() {
  PrefWatcherFactory::GetInstance();  // 创建全局 PrefWatcher
}

// PrefWatcher 是什么?
// 它是一个 Chrome 组件，监听所有标记为 SYNCABLE 的偏好
// 当偏好变化时，它调用观察者的 OnPreferenceChanged() 回调
```

---

### Step 3: PrefsTabHelper 观察偏好变化

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
// 当用户改变字体时的调用栈:
// UserChangesFont() 
//  → SettingsPrivate API → PrefService::SetString()
//  → 触发所有观察者

// PrefsTabHelper 是一个观察者:
class PrefsTabHelper : public ThemeServiceObserver,
                       public content::WebContentsUserData<PrefsTabHelper> {
  
  // 被 PrefWatcher 调用
  void OnWebPrefChanged(const std::string& pref_name) {
    // pref_name 例如: "webkit.webprefs.fonts.serif.Latn"
    
    // 使用 PostTask 延迟处理，给其他观察者（如 FontFamilyCache）反应的机会
    base::SingleThreadTaskRunner::GetCurrentDefault()->PostTask(
        FROM_HERE, base::BindOnce(&PrefsTabHelper::NotifyWebkitPreferencesChanged,
                                  weak_ptr_factory_.GetWeakPtr(), pref_name));
  }
};

// 稍后在 UI 线程处理
void PrefsTabHelper::NotifyWebkitPreferencesChanged(
    const std::string& pref_name) {  // pref_name = "webkit.webprefs.fonts.serif.Latn"
  
#if !BUILDFLAG(IS_ANDROID)
  // 只在桌面平台处理字体偏好变化
  OnFontFamilyPrefChanged(pref_name);
#endif

  // 通知 WebContents WebPreferences 已变化
  // 这将导致渲染器进程收到更新
  GetWebContents().OnWebPreferencesChanged();
}
```

---

### Step 4: 特定的字体偏好处理

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
void PrefsTabHelper::OnFontFamilyPrefChanged(const std::string& pref_name) {
  // pref_name = "webkit.webprefs.fonts.serif.Latn"
  
  // 解析偏好名称以提取通用族和脚本
  std::string generic_family;  // 结果: "serif"
  std::string script;          // 结果: "Latn"
  
  if (pref_names_util::ParseFontNamePrefPath(pref_name, &generic_family,
                                             &script)) {
    // 解析成功
    
    // 从 PrefService 读取新的字体值
    PrefService* prefs = profile_->GetPrefs();
    std::string pref_value = prefs->GetString(pref_name);
    // pref_value = "Georgia"
    
    // 特殊处理：如果新值是空字符串（用户选择"默认"）
    if (pref_value.empty()) {
      // 获取当前 WebPreferences
      blink::web_pref::WebPreferences web_prefs =
          GetWebContents().GetOrCreateWebPreferences();
      
      // 应用空字符串（表示使用默认）
      OverrideFontFamily(&web_prefs, generic_family, script, std::string());
      
      // 更新 WebContents
      GetWebContents().SetWebPreferences(web_prefs);
      return;  // 提前返回，后续的 OnWebPreferencesChanged() 会处理
    }
  }
  
  // 对于非空的字体值，通常由主要的 OnWebPreferencesChanged 处理
}

// ParseFontNamePrefPath 如何工作
namespace pref_names_util {
  bool ParseFontNamePrefPath(const std::string& pref_path,
                             std::string* out_generic_family,
                             std::string* out_script) {
    // pref_path = "webkit.webprefs.fonts.serif.Latn"
    
    const std::string prefix = "webkit.webprefs.fonts.";
    if (!pref_path.starts_with(prefix)) {
      return false;  // 不是字体偏好
    }
    
    // 移除前缀: "serif.Latn"
    std::string remainder = pref_path.substr(prefix.length());
    
    // 找到最后一个点
    size_t last_dot = remainder.rfind('.');
    if (last_dot == std::string::npos) {
      return false;
    }
    
    // 分割成族和脚本
    *out_generic_family = remainder.substr(0, last_dot);      // "serif"
    *out_script = remainder.substr(last_dot + 1);             // "Latn"
    
    return true;
  }
}
```

---

### Step 5: 应用偏好到 WebPreferences 结构

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
void OverrideFontFamily(blink::web_pref::WebPreferences* prefs,
                        const std::string& generic_family,
                        const std::string& script,
                        const std::string& pref_value) {
  // 参数:
  // prefs = WebPreferences 结构指针
  // generic_family = "serif"
  // script = "Latn"
  // pref_value = "Georgia"
  
  // 根据通用族名称选择正确的映射
  blink::web_pref::ScriptFontFamilyMap* map = nullptr;
  
  if (generic_family == "standard") {
    map = &prefs->standard_font_family_map;      // 标准/无衬线
  } else if (generic_family == "serif") {
    map = &prefs->serif_font_family_map;         // 衬线 ✓ 匹配
  } else if (generic_family == "sansserif") {
    map = &prefs->sans_serif_font_family_map;    // 无衬线
  } else if (generic_family == "fixed") {
    map = &prefs->fixed_font_family_map;         // 等宽
  } else if (generic_family == "cursive") {
    map = &prefs->cursive_font_family_map;       // 草体
  } else if (generic_family == "fantasy") {
    map = &prefs->fantasy_font_family_map;       // 装饰
  } else if (generic_family == "math") {
    map = &prefs->math_font_family_map;          // 数学
  }
  
  if (map) {
    // map 现在指向 serif_font_family_map
    
    // 将用户选择的字体转换为 UTF-16 并存储
    // (*map)[script] 是对 map 中键为 script 的值的引用
    (*map)[script] = base::UTF8ToUTF16(pref_value);
    
    // 操作后的状态:
    // prefs->serif_font_family_map["Latn"] = u"Georgia"
    //                                        (UTF-16 编码)
  }
}

// WebPreferences 结构在内存中现在看起来是这样的:
struct WebPreferences {
  // ... 其他字段 ...
  
  // serif_font_family_map 内容:
  // {
  //   "Latn" → u"Georgia"        ✓ 刚刚设置的
  //   "Hant" → u"微软雅黑"       (之前的值)
  //   "Hans" → u"宋体"           (之前的值)
  //   "Cyrl" → u"Times New Roman" (之前的值)
  //   "Jpan" → u"游明朝"        (之前的值)
  // }
  
  // 其他字体族映射保持不变
  // ...
};
```

---

### Step 6: 通知 WebContents 更新

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc` 和 `content/browser/web_contents/`

```cpp
// 回到 NotifyWebkitPreferencesChanged
void PrefsTabHelper::NotifyWebkitPreferencesChanged(
    const std::string& pref_name) {
  
#if !BUILDFLAG(IS_ANDROID)
  OnFontFamilyPrefChanged(pref_name);  // 刚才调用过
#endif

  // 通知 WebContents（包含 RenderViewHost）
  GetWebContents().OnWebPreferencesChanged();
  //                ↑
  //                这是关键调用，触发向渲染器发送新的 WebPreferences
}

// WebContents::OnWebPreferencesChanged() 做什么?
// (文件: content/browser/web_contents/web_contents_impl.cc)

void WebContentsImpl::OnWebPreferencesChanged() {
  // 遍历所有 RenderFrameHost（每个 frame 一个）
  for (auto* frame_host : frame_tree_.Frames()) {
    // 获取或创建当前的 WebPreferences
    auto web_prefs = GetOrCreateWebPreferences();
    
    // 通过 Mojo IPC 发送到渲染器进程
    // frame_host->GetAssociatedLocalFrame()->UpdateWebPreferences(web_prefs);
  }
}

// GetOrCreateWebPreferences() 做什么?
// (文件: chrome/browser/chrome_content_browser_client.cc)

blink::web_pref::WebPreferences 
ChromeContentBrowserClient::GetWebPreferencesForWebContents(
    content::WebContents* web_contents) {
  
  // 创建新的 WebPreferences 结构
  blink::web_pref::WebPreferences web_prefs;
  
  // 从 PrefService 读取所有相关的偏好并填充结构
  
  // 对每个注册的字体偏好，应用用户设置
  // 这包括我们刚才改变的 "webkit.webprefs.fonts.serif.Latn"
  
  PrefsTabHelper* prefs_tab_helper = 
      PrefsTabHelper::FromWebContents(web_contents);
  
  if (prefs_tab_helper) {
    // PrefsTabHelper 已经通过 OverrideFontFamily() 更新了 web_prefs
    // 所以 web_prefs 现在包含最新的字体映射
  }
  
  return web_prefs;
}
```

---

### Step 7: 通过 Mojo IPC 发送到渲染器

**文件**: 多个文件，涉及 Mojo 序列化

```cpp
// 浏览器进程 (browser_process/)
void RenderProcessHost::UpdateWebPreferences(
    const blink::web_pref::WebPreferences& prefs) {
  
  // 通过 Mojo 接口发送
  for (const auto& agent : render_frame_host_) {
    // Mojo 会自动序列化 WebPreferences 结构
    // 序列化包括:
    // - 7 个 ScriptFontFamilyMap
    // - 每个 map 中的所有条目
    // - 所有字体大小设置
    // - 其他 100+ 个设置
    
    agent->renderer_host->blink_preferences().SetWebPreferences(prefs);
  }
}

// 声明在: third_party/blink/public/mojom/webpreferences/web_preferences.mojom

module blink.mojom;

struct ScriptFontFamilyMap {
  map<string, string16> entries;
};

struct WebPreferences {
  ScriptFontFamilyMap standard_font_family_map;
  ScriptFontFamilyMap serif_font_family_map;
  ScriptFontFamilyMap fixed_font_family_map;
  ScriptFontFamilyMap sans_serif_font_family_map;
  ScriptFontFamilyMap cursive_font_family_map;
  ScriptFontFamilyMap fantasy_font_family_map;
  ScriptFontFamilyMap math_font_family_map;
  
  int32 default_font_size;
  int32 default_fixed_font_size;
  int32 minimum_font_size;
  int32 minimum_logical_font_size;
  // ... 其他 100+ 字段 ...
};

// Mojo 生成的序列化代码会将数据转换为二进制格式:
// [1] WebPreferences 类型标识
// [2] 7 个 map 的序列化数据
//     ├─ map 1: standard
//     ├─ map 2: serif
//     │   ├─ "Latn" → "Georgia"     ✓ 我们的新值
//     │   ├─ "Hant" → "微软雅黑"
//     │   └─ ...
//     ├─ map 3-7: 其他族
//     └─ ...
// [3] 整数字段
// [4] ... 其他字段
```

---

### Step 8: 渲染器进程接收并应用更新

**文件**: `third_party/blink/renderer/platform/fonts/generic_font_family_settings.cc`

```cpp
// Mojo 反序列化数据后，调用 SetWebPreferences

class RenderThreadImpl {
  void SetWebPreferences(
      const blink::web_pref::WebPreferences& web_prefs) {
    
    // 更新 Blink 的全局 GenericFontFamilySettings
    GetGenericFontFamilySettings().SetWebPreferences(web_prefs);
  }
};

// GenericFontFamilySettings 是什么?
namespace blink {

class GenericFontFamilySettings {
 private:
  // 7 个本地副本
  ScriptFontFamilyMap standard_font_family_map_;
  ScriptFontFamilyMap serif_font_family_map_;
  ScriptFontFamilyMap fixed_font_family_map_;
  ScriptFontFamilyMap sans_serif_font_family_map_;
  ScriptFontFamilyMap cursive_font_family_map_;
  ScriptFontFamilyMap fantasy_font_family_map_;
  ScriptFontFamilyMap math_font_family_map_;
  
 public:
  // 从 WebPreferences 复制数据的方法
  void UpdateFromWebPreferences(
      const blink::web_pref::WebPreferences& web_prefs) {
    
    // 复制 7 个映射
    standard_font_family_map_ = web_prefs.standard_font_family_map;
    serif_font_family_map_ = web_prefs.serif_font_family_map;
    //  ↑ 现在包含 ["Latn" → u"Georgia", "Hant" → ..., ...]
    fixed_font_family_map_ = web_prefs.fixed_font_family_map;
    // ... 其他 4 个映射 ...
    
    // 清除缓存（因为字体设置已变化）
    InvalidateFontCache();
  }
  
  // 检索特定脚本的字体
  const AtomicString& SerifFontFamily(UScriptCode script) const {
    // script = USCRIPT_LATIN (代表 "Latn")
    return GenericFontFamilyForScript(serif_font_family_map_, script);
  }
};

// GenericFontFamilyForScript 实现
const AtomicString& GenericFontFamilySettings::GenericFontFamilyForScript(
    const ScriptFontFamilyMap& font_map,
    UScriptCode script) const {
  
  // 将脚本代码转换为字符串键
  int script_code = static_cast<int>(script);
  // script_code = 25 (USCRIPT_LATIN)
  
  // 在映射中查找
  auto it = font_map.find(script_code);
  
  if (it != font_map.end()) {
    // 找到了用户设置的字体!
    return it->value;  // 返回 "Georgia" (作为 AtomicString)
  }
  
  // 如果没有该脚本的专用字体，返回空
  return g_null_atom;
}

} // namespace blink
```

---

### Step 9: 字体选择时使用自定义字体

**文件**: `third_party/blink/renderer/platform/fonts/font_fallback_iterator.cc`

```cpp
// 当 Blink 需要为文本找字体时:

class FontFallbackIterator {
  // 处理 font-family 属性和通用族
  
  bool SetupFallbackFontList(UScriptCode script) {
    // script = USCRIPT_LATIN (因为文本是拉丁字母)
    
    // 遍历 CSS font-family 列表
    // 例如: font-family: "My Font", serif, monospace
    
    for (const auto& font_name : font_list) {
      if (font_name == "serif") {
        // 这是一个通用族！使用用户自定义的字体
        
        // 从 GenericFontFamilySettings 获取
        const AtomicString& serif_font = 
            GetGenericFontFamilySettings().SerifFontFamily(
                USCRIPT_LATIN);
        
        if (!serif_font.empty()) {
          // 返回用户选择的字体: "Georgia"
          current_font_family = serif_font;  // "Georgia"
          return true;
        }
      }
    }
  }
};

// FontCache 现在需要获取 "Georgia" 的字体数据

class FontCache {
  scoped_refptr<SimpleFontData> GetFontData(
      const FontDescription& font_description,
      const AtomicString& family_name) {
    // family_name = "Georgia"
    
    // 查找 "Georgia" 字体
    // 这可能触发:
    // - 平台 API 调用 (Windows: GetFontData, Mac: CTFontCreateWithName, 等)
    // - 从磁盘加载 TrueType/OpenType 字体文件
    // - 初始化字体缓存
    
    // 返回 SimpleFontData 对象，包含：
    // - 字形指标
    // - 字形数据
    // - 字间距信息
    // - 等等
  }
};
```

---

### Step 10: 渲染文本

**文件**: `third_party/blink/renderer/core/text/TextRun.cpp` 等

```cpp
// 现在 Blink 使用获得的字体数据来渲染文本

void HarfBuzzShaper::Shape(const FontData* primary_font,
                           const TextRun& run) {
  // primary_font 现在指向 "Georgia" 字体的数据
  
  // 调用 HarfBuzz 进行字形塑造
  // hb_shape(font_obj, buffer);
  
  // HarfBuzz 使用 "Georgia" 的字形和特性信息
  // 生成字形索引、位置、等等
}

// 最后，绘制系统使用生成的字形数据
// 调用平台的绘制 API (Skia, etc.)
// 在屏幕上绘制 "Georgia" 字体的字形
```

---

## 完整流程时间线

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        完整执行时间线                                    │
└─────────────────────────────────────────────────────────────────────────┘

T+0ms:   用户点击 Settings → Appearance → Fonts
T+1ms:   UI 下拉菜单打开，显示可用字体列表
T+50ms:  用户选择 "Georgia"
T+51ms:  SettingsPrivate API 接收变化
         → PrefService::SetString(
             "webkit.webprefs.fonts.serif.Latn", 
             "Georgia")

T+52ms:  PrefWatcher 检测到变化
         → 调用 PrefsTabHelper::OnWebPrefChanged()

T+53ms:  PostTask 延迟处理
         → 加入消息队列

T+60ms:  任务运行器执行 PostTask
         → PrefsTabHelper::NotifyWebkitPreferencesChanged()
         → PrefsTabHelper::OnFontFamilyPrefChanged()
         → 解析 "serif.Latn"
         → 调用 OverrideFontFamily()
         → 更新 WebPreferences 结构

T+61ms:  GetWebContents().OnWebPreferencesChanged()
         → 构建新的 WebPreferences 对象
         → 通过 Mojo 序列化

T+62ms:  Mojo 消息发送到渲染器进程
         (可能跨越进程边界)

T+100ms: 渲染器进程接收 Mojo 消息
         → 反序列化 WebPreferences
         → GenericFontFamilySettings::UpdateFromWebPreferences()
         → serif_font_family_map_[USCRIPT_LATIN] = "Georgia"

T+101ms: FontCache::InvalidateFontCache()
         → 清除旧的字体缓存

T+150ms: 网页内容需要重新渲染（用户滚动或其他操作）
         → FontFallbackIterator 查询 serif 字体
         → GenericFontFamilySettings::SerifFontFamily(USCRIPT_LATIN)
         → 返回 "Georgia"
         → FontCache::GetFontData("Georgia")
         → 加载 "Georgia" 字体数据

T+151ms: HarfBuzz 使用 "Georgia" 进行字形塑造
T+152ms: Skia 绘制 "Georgia" 字形到屏幕
T+153ms: 用户看到使用 "Georgia" 字体渲染的文本 ✓

总耗时: 约 150-200 毫秒（大部分是异步的）
```

---

## 数据变换追踪

### 用户选择："Georgia"

```
String → UTF-8 → PrefService
  "Georgia" (JavaScript String)
    ↓
  "Georgia" (std::string, UTF-8)
    ↓
  PrefService::SetString("webkit.webprefs.fonts.serif.Latn", "Georgia")
    ↓
  存储在本地数据库（通常是 SQLite）
```

### 发送到渲染器：

```
UTF-8 String → Mojo 序列化 → Binary → Network → Mojo 反序列化 → UTF-16 String
  "Georgia" (std::string, UTF-8)
    ↓
  Mojo 序列化为二进制消息
    ↓
  通过 IPC 管道发送
    ↓
  渲染器进程接收二进制消息
    ↓
  Mojo 反序列化为 std::u16string (UTF-16)
    ↓
  存储在 GenericFontFamilySettings::serif_font_family_map_
```

### 最终使用：

```
UTF-16 String → 平台 API → 字体文件
  u"Georgia" (std::u16string, UTF-16)
    ↓
  Windows: GetFontData("Georgia", ...)
  Mac: CTFontCreateWithName(CFSTR("Georgia"), ...)
  Linux: FcFontMatch(..., "Georgia", ...)
    ↓
  加载 "Georgia.ttf" 或 "Georgia.otf"
    ↓
  HarfBuzz 处理字形
    ↓
  Skia 渲染到屏幕
```

---

## 关键的代码引用点

| 操作 | 文件 | 函数 | 行号 |
|------|------|------|-----|
| Settings UI 显示 | `appearance_fonts_page.ts` | `setFontsData_()` | ~106 |
| 启动 PrefWatcher | `prefs_tab_helper.cc` | `GetServiceInstance()` | ~508 |
| 收到偏好变化 | `prefs_tab_helper.cc` | `OnWebPrefChanged()` | ~522 |
| 处理字体变化 | `prefs_tab_helper.cc` | `OnFontFamilyPrefChanged()` | ~526 |
| 应用到 WebPreferences | `prefs_tab_helper.cc` | `OverrideFontFamily()` | ~260 |
| 通知渲染器 | `web_contents_impl.cc` | `OnWebPreferencesChanged()` | ✗ |
| 渲染器接收 | `render_process_host_impl.cc` | `UpdateWebPreferences()` | ✗ |
| Blink 应用 | `generic_font_family_settings.cc` | `UpdateFromWebPreferences()` | ~65 |
| 字体选择 | `font_fallback_iterator.cc` | `SetupFallbackFontList()` | ✗ |

---

## 关键的数据结构大小

```cpp
// WebPreferences 结构大小估计

struct WebPreferences {
  // 7 个字体族映射 (最大)
  ScriptFontFamilyMap standard_font_family_map;        // ~1KB (假设 10 种脚本)
  ScriptFontFamilyMap serif_font_family_map;          // ~1KB
  ScriptFontFamilyMap fixed_font_family_map;          // ~1KB
  ScriptFontFamilyMap sans_serif_font_family_map;     // ~1KB
  ScriptFontFamilyMap cursive_font_family_map;        // ~1KB
  ScriptFontFamilyMap fantasy_font_family_map;        // ~1KB
  ScriptFontFamilyMap math_font_family_map;           // ~1KB
                                                       // ────────
                                                       // 小计: ~7KB
  
  // 其他整数字段 (~50 个)
  int default_font_size;                              // 4 字节
  int default_fixed_font_size;                        // 4 字节
  // ... 等等
                                                       // 小计: ~200 字节
  
  // 其他 bool/enum 字段
  // ...
                                                       // 小计: ~500 字节
};

总大小: 约 7.7 KB per WebPreferences
IPC 消息大小: 约 8-10 KB (包括 Mojo 序列化开销)

Mojo 序列化开销: ~1-2 KB
最终 IPC 消息: 约 10 KB
```
