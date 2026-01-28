# 字体系统调用链快速参考

## 源代码追踪路径

### 用户设置系统字体
```
Android Settings App
    ↓ (存储到系统 Settings 数据库)
Settings.System.getString("sem_font_name")  // Samsung
Settings.System.getString("persist.sys.font_name")  // MIUI
Settings.System.getString("font_name")  // Generic
```

### Chromium 获取系统字体

**Layer 1: Java (AwSettings)**
```
File: android_webview/java/src/org/chromium/android_webview/AwSettings.java
Line: 186-191 (初始化)
      423 (fontScale 读取)
      
Members:
  - private String mStandardFontFamily = "sans-serif"
  - private String mSerifFontFamily = "serif"
  - private String mSansSerifFontFamily = "sans-serif"
  - private String mFixedFontFamily = "monospace"
  - private String mCursiveFontFamily = "cursive"
  - private String mFantasyFontFamily = "fantasy"
```

**Layer 2: JNI Bridge**
```
File: android_webview/java/src/org/chromium/android_webview/AwSettings.java
Method: @CalledByNative methods like:
  - getStandardFontFamilyLocked()
  - getSerifFontFamilyLocked()
  - etc.
```

**Layer 3: C++ (AwSettings)**
```
File: android_webview/browser/aw_settings.cc
Func: PopulateWebPreferencesLocked()
Line: 578-626

Code:
  web_prefs->standard_font_family_map[kCommonScript] = 
      ConvertJavaStringToUTF16(
          Java_AwSettings_getStandardFontFamilyLocked(env, obj));
```

**Layer 4: Browser Client (Override)**
```
File: android_webview/browser/aw_content_browser_client.cc
Func: OverrideWebPreferences()
Line: 655-668

Code:
  aw_settings->PopulateWebPreferences(web_prefs);
```

**Layer 5: WebContents**
```
File: content/browser/web_contents/web_contents_impl.cc
Func: ComputeWebPreferences() → OnWebPreferencesChanged()
Line: 3903

Code:
  GetContentClient()->browser()->OverrideWebPreferences(
      this, *main_frame->GetSiteInstance(), &prefs);
```

**Layer 6: Blink Font Selection**
```
File: third_party/blink/public/common/web_preferences/web_preferences.h
Type: WebPreferences struct with:
  - standard_font_family_map
  - serif_font_family_map
  - sans_serif_font_family_map
  - fixed_font_family_map
  - cursive_font_family_map
  - fantasy_font_family_map
```

**Layer 7: Font Cache (Android)**
```
File: third_party/blink/renderer/platform/fonts/android/font_cache_android.cc
Func: GetGenericFamilyNameForScript()
Line: 231-287

Logic:
  1. 检查是否为 CJK 脚本
  2. 如果是 CJK: 使用 matchFamilyStyleCharacter(nullptr, ..., 0x4E00)
  3. 如果不是: 返回 generic_family_name_fallback (如 "Times New Roman")
```

**Layer 8: SkFontMgr (Android)**
```
Location: //third_party/skia/src/ports/SkFontMgr_android.cpp

Process:
  1. 读取 /system/etc/fonts.xml
  2. 查询字体族名称
  3. 返回 SkTypeface 或 nullptr
```

---

## 完整调用流程图

```
┌─────────────────────────────────────────────────┐
│ 1. Android System Settings                      │
│    system_font = Settings.System.getString()   │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 2. Java: AwSettings                             │
│    - Constructor: initialize fonts              │
│    - onConfigurationChanged: update fonts       │
│    - getStandardFontFamilyLocked(): return font │
└────────────────┬────────────────────────────────┘
                 │ (JNI)
┌────────────────▼────────────────────────────────┐
│ 3. C++: AwSettings::PopulateWebPreferencesLocked│
│    web_prefs->standard_font_family_map = font  │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 4. C++: ContentBrowserClient::OverrideWebPrefs │
│    (Injects font info into WebPreferences)     │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 5. C++: WebContentsImpl::ComputeWebPreferences  │
│    Returns configured WebPreferences           │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 6. C++: Blink FontSelector                      │
│    Receives WebPreferences struct              │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 7. C++: FontCache (Android CJK Hack)            │
│    GetGenericFamilyNameForScript(family)       │
│    - Detects script type                       │
│    - Queries SkFontMgr for font               │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│ 8. C++: SkFontMgr_New_Android                   │
│    - Reads /system/etc/fonts.xml               │
│    - Returns SkTypeface or nullptr             │
│    - If nullptr: fallback to next method       │
└─────────────────────────────────────────────────┘
```

---

## 关键代码片段

### 初始化 (Java)
```java
// File: AwSettings.java:186-191
private String mStandardFontFamily = "sans-serif";
private String mFixedFontFamily = "monospace";
private String mSansSerifFontFamily = "sans-serif";
private String mSerifFontFamily = "serif";
private String mCursiveFontFamily = "cursive";
private String mFantasyFontFamily = "fantasy";
```

### fontScale 读取 (Java)
```java
// File: AwSettings.java:423
mTextSizePercent = (int) (mTextSizePercent * 
    context.getResources().getConfiguration().fontScale);
```

### 字体传递到 Native (C++)
```cpp
// File: aw_settings.cc:607-626
web_prefs->standard_font_family_map[kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getStandardFontFamilyLocked(env, obj));
```

### WebPreferences 注入 (C++)
```cpp
// File: aw_content_browser_client.cc:655-668
void AwContentBrowserClient::OverrideWebPreferences(
    content::WebContents* web_contents,
    content::SiteInstance& main_frame_site,
    blink::web_pref::WebPreferences* web_prefs) {
  AwSettings* aw_settings = AwSettings::FromWebContents(web_contents);
  if (aw_settings) {
    aw_settings->PopulateWebPreferences(web_prefs);
  }
}
```

### CJK Hack 字体查询 (C++)
```cpp
// File: font_cache_android.cc:231-287
AtomString FontCache::GetGenericFamilyNameForScript(
    const AtomString& generic_family_name,
    UScriptCode script) {
  // 检查 CJK
  if (script == USCRIPT_HIRAGANA || ...) {
    // CJK: 使用 exemplar-based matching
    SkTypeface* typeface = 
        SkFontMgr::RefDefault()->matchFamilyStyleCharacter(
            nullptr, SkFontStyle(), nullptr, 
            0x4E00, SkFontMgr::FontTableTag());
  } else {
    // Non-CJK: 返回 fallback
    return generic_family_name_fallback;
  }
}
```

---

## 问题诊断

### 现状
- ❌ 字体名称是硬编码的 ("sans-serif", "serif", etc)
- ✅ 字体大小缩放可以通过 `Configuration.fontScale` 获取
- ❌ 无法获取系统自定义字体名称
- ❌ CJK Hack 对非 CJK 内容无效

### 导致的现象
1. **CJK 内容**（中文、日文、韩文）：
   - ✅ 自动获取 NotoSansCJK（通过 exemplar matching）
   - ✅ 跟随系统字体设置

2. **Non-CJK 内容**（英文、其他）：
   - ❌ 回退到 "Times New Roman"
   - ❌ "Times New Roman" 在 Android 中不存在
   - ❌ 最终回退到系统默认
   - ❌ **无法跟随系统字体**

---

## 解决方案概览

### 短期修复（72 行代码）
在 `AwSettings.java` 中添加系统字体读取：
```java
private void updateSystemFontSettings() {
    String font = tryGetSystemFont();
    if (font != null) {
        mStandardFontFamily = font;
        mEventHandler.updateWebkitPreferencesLocked();
    }
}

private String tryGetSystemFont() {
    String[] keys = {
        "sem_font_name",           // Samsung
        "persist.sys.font_name",   // MIUI
        "font_name"                // Generic
    };
    for (String key : keys) {
        try {
            String val = Settings.System.getString(
                mContext.getContentResolver(), key);
            if (val != null && !val.isEmpty()) return val;
        } catch (Exception e) {}
    }
    return null;
}
```

### 长期方案
1. 创建 Android 系统 API 对接层
2. 支持字体改变通知（ConfigurationChanged）
3. 处理厂商定制字体存储差异

---

## 代码文件树

```
Chromium 字体系统架构
├── Android System Layer
│   └── Settings.System (system_font)
│
├── Java Layer (android_webview)
│   └── AwSettings.java
│       ├── mStandardFontFamily
│       ├── getStandardFontFamilyLocked() 
│       └── onConfigurationChanged()
│
├── JNI Bridge
│   └── Java_AwSettings_getStandardFontFamilyLocked()
│
├── C++ Browser Layer (android_webview/browser)
│   ├── aw_settings.cc
│   │   └── PopulateWebPreferencesLocked()
│   └── aw_content_browser_client.cc
│       └── OverrideWebPreferences()
│
├── Content Layer (content/browser)
│   └── web_contents_impl.cc
│       ├── ComputeWebPreferences()
│       └── OnWebPreferencesChanged()
│
├── WebPreferences Struct
│   └── web_preferences.h
│       ├── standard_font_family_map
│       ├── serif_font_family_map
│       ├── sans_serif_font_family_map
│       └── ... (5 more font family maps)
│
├── Blink Layer (third_party/blink)
│   ├── font_selector.cc (FamilyNameFromSettings)
│   └── platform/fonts/android/
│       └── font_cache_android.cc
│           └── GetGenericFamilyNameForScript()
│
└── Skia Layer (third_party/skia)
    └── SkFontMgr_android.cpp
        ├── Reads /system/etc/fonts.xml
        └── Returns SkTypeface or nullptr
```

---

## 修改检查清单

### 需要修改的文件
- [ ] `android_webview/java/src/org/chromium/android_webview/AwSettings.java`
  - [ ] 添加系统字体读取方法
  - [ ] 注册 ComponentCallbacks
  - [ ] 在 onConfigurationChanged 时更新

### 不需要修改的文件
- ✅ `android_webview/browser/aw_settings.cc` (已支持 PopulateWebPreferences)
- ✅ `android_webview/browser/aw_content_browser_client.cc` (已支持 OverrideWebPreferences)
- ✅ `content/browser/web_contents/web_contents_impl.cc` (已有正确流程)
- ✅ `third_party/blink/.../font_selector.cc` (使用 WebPreferences)
- ✅ `third_party/blink/.../font_cache_android.cc` (查询字体)

---

## 性能考虑

| 操作 | 频率 | 成本 | 优化 |
|------|------|------|------|
| 初始化读取 | 1次/WebView | 低 | 后台线程 |
| onConfigurationChanged | 用户改字体时 | 低 | 锁定读取 |
| Settings.getString | 每次查询 | 中 | 缓存结果 |
| updateWebkitPreferences | 配置改变时 | 中 | 批量更新 |
| 字体查询 (SkFontMgr) | 每个文本段 | 高 | 缓存字体对象 |

---

## 测试用例

```
Test Case 1: Initial Font Detection
├── Create WebView
├── Verify mStandardFontFamily is read from system
└── Assert: Font used in web content matches system setting

Test Case 2: Configuration Change
├── Change system font while WebView is running
├── Trigger onConfigurationChanged
└── Assert: Web content uses new font

Test Case 3: Fallback (No Custom Font)
├── Run on system without custom font support
├── Verify fallback to "sans-serif"
└── Assert: Web content still renderable

Test Case 4: Cross-Vendor Compatibility
├── Test on Samsung One UI (sem_font_name)
├── Test on MIUI (persist.sys.font_name)
├── Test on Stock Android
└── Assert: Works on all tested systems
```

---

## 相关链接

- [Android Configuration 文档](https://developer.android.com/reference/android/content/res/Configuration)
- [Android Settings.System 文档](https://developer.android.com/reference/android/provider/Settings.System)
- [Chromium WebPreferences](third_party/blink/public/common/web_preferences/web_preferences.h)
- [Blink FontSelector](third_party/blink/renderer/platform/fonts/font_selector.cc)
- [Android SkFontMgr](third_party/skia/src/ports/SkFontMgr_android.cpp)

---

## 总结表

| 层级 | 文件 | 关键代码 | 当前状态 | 需要改动 |
|-----|------|--------|--------|---------|
| 1 | AwSettings.java | mStandardFontFamily | 硬编码 | ✅ 读取系统 |
| 2 | aw_settings.cc | PopulateWebPreferences | 正常 | ❌ 无需改动 |
| 3 | aw_content_browser_client.cc | OverrideWebPreferences | 正常 | ❌ 无需改动 |
| 4 | web_contents_impl.cc | ComputeWebPreferences | 正常 | ❌ 无需改动 |
| 5 | font_selector.cc | FamilyNameFromSettings | 正常 | ❌ 无需改动 |
| 6 | font_cache_android.cc | CJK Hack | 有效但仅限 CJK | ⚠️ 可优化 |
| 7 | SkFontMgr_android.cpp | matchFamilyStyleCharacter | 正常 | ❌ 无需改动 |
