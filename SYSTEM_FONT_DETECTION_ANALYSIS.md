# Chromium UI 系统字体识别机制分析

## 关键发现

Chromium UI 识别系统字体的机制是通过一个 **多层次的管道** 实现的。以下是完整的代码调用链：

---

## 1. 字体初始化源头 - AwSettings.java

**文件**: `android_webview/java/src/org/chromium/android_webview/AwSettings.java`

**初始化代码** (第 186-191 行):
```java
private String mStandardFontFamily = "sans-serif";
private String mFixedFontFamily = "monospace";
private String mSansSerifFontFamily = "sans-serif";
private String mSerifFontFamily = "serif";
private String mCursiveFontFamily = "cursive";
private String mFantasyFontFamily = "fantasy";
```

**构造方法** (第 423 行):
```java
// By default, scale the text size by the system font scale factor. Embedders
// may override this by invoking setTextZoom().
mTextSizePercent = (int) (mTextSizePercent * context.getResources().getConfiguration().fontScale);
```

**关键点**:
- 字体名称初始化为 **generic family names**（"sans-serif", "serif", 等）
- 字体大小是通过 `context.getResources().getConfiguration().fontScale` 从系统读取的
- **目前没有从系统读取自定义字体名称**

---

## 2. 系统配置读取 - Configuration.fontScale

**模式**:
```java
context.getResources().getConfiguration().fontScale
```

这个方法读取 Android 系统设置中的字体缩放因子。这是 Chromium UI 目前唯一从系统获取的字体相关配置。

**启示**: 我们应该找到 Android 系统存储**自定义字体名称**的地方，方式类似于 `fontScale`。

---

## 3. Java 层到 Native 层的桥接 - PopulateWebPreferences

**文件**: `android_webview/browser/aw_settings.cc`

**PopulateWebPreferencesLocked 函数** (第 578-626 行):

```cpp
// 从 Java 层获取字体信息
web_prefs->standard_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getStandardFontFamilyLocked(env, obj));

web_prefs->fixed_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getFixedFontFamilyLocked(env, obj));

web_prefs->sans_serif_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getSansSerifFontFamilyLocked(env, obj));

web_prefs->serif_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getSerifFontFamilyLocked(env, obj));

web_prefs->cursive_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getCursiveFontFamilyLocked(env, obj));

web_prefs->fantasy_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getFantasyFontFamilyLocked(env, obj));
```

**调用链**:
1. Native C++ 代码调用 Java 方法获取字体信息
2. 通过 JNI 转换 Java String 到 C++ UTF16
3. 将结果写入 WebPreferences 的字体族映射表

---

## 4. WebPreferences 使用位置

**文件**: `android_webview/browser/aw_content_browser_client.cc`

**OverrideWebPreferences 函数** (第 655-668 行):

```cpp
void AwContentBrowserClient::OverrideWebPreferences(
    content::WebContents* web_contents,
    content::SiteInstance& main_frame_site,
    blink::web_pref::WebPreferences* web_prefs) {
  AwSettings* aw_settings = AwSettings::FromWebContents(web_contents);
  if (aw_settings) {
    aw_settings->PopulateWebPreferences(web_prefs);  // <-- 字体被注入到 WebPreferences
  }
  // ...
}
```

**调用位置**: `content/browser/web_contents/web_contents_impl.cc` (第 3903 行)

```cpp
GetContentClient()->browser()->OverrideWebPreferences(
    this, *main_frame->GetSiteInstance(), &prefs);
```

---

## 5. 完整的字体设置管道

```
┌─────────────────────────────────────────────┐
│ Android System Configuration                │
│ (system/etc/fonts.xml + custom fonts)       │
└─────────────────────────────────────────────┘
                    ↓
         ┌──────────────────────────┐
         │ Java AwSettings          │
         │ - mStandardFontFamily    │
         │ - mFixedFontFamily       │  ← 硬编码为 generic names
         │ - mSerifFontFamily       │
         │ - mSansSerifFontFamily   │
         └──────────────────────────┘
                    ↓ (JNI)
         ┌──────────────────────────┐
         │ C++ AwSettings           │
         │ PopulateWebPreferences() │
         └──────────────────────────┘
                    ↓
         ┌──────────────────────────┐
         │ WebPreferences struct    │
         │ - standard_font_family_map
         │ - serif_font_family_map  │  ← 最终在这里使用
         │ - sans_serif_font_family │
         └──────────────────────────┘
                    ↓
         ┌──────────────────────────┐
         │ Blink FontSelector       │
         │ FontCache (Android)      │
         │ CJK Hack                 │  ← 字体查询
         └──────────────────────────┘
                    ↓
         ┌──────────────────────────┐
         │ SkFontMgr_New_Android    │
         │ (读取 /system/etc/fonts) │
         └──────────────────────────┘
```

---

## 6. 关键代码位置速查表

| 组件 | 文件路径 | 代码行 | 作用 |
|------|--------|------|------|
| Java 初始化 | `android_webview/java/.../AwSettings.java` | 186-191 | 字体初始值定义 |
| fontScale 读取 | `android_webview/java/.../AwSettings.java` | 423 | 从系统获取字体大小缩放 |
| JNI 获取 | `android_webview/browser/aw_settings.cc` | 578-626 | Java→Native 字体传递 |
| OverrideWebPrefs | `android_webview/browser/aw_content_browser_client.cc` | 655-668 | 字体注入到 WebPreferences |
| ComputeWebPrefs | `content/browser/web_contents/web_contents_impl.cc` | 3903 | 触发 OverrideWebPreferences |
| CJK Hack | `third_party/blink/renderer/platform/fonts/android/font_cache_android.cc` | 231-287 | 字体查询逻辑 |

---

## 7. 问题诊断

### 现有机制的局限性

1. **AwSettings.java 中的字体值是硬编码的**
   - 从不读取系统自定义字体信息
   - 只有 generic family names

2. **系统配置读取方式**
   - ✅ 可以读取 `Configuration.fontScale`（字体大小倍数）
   - ❌ 无法读取自定义字体名称（目前没有实现）

3. **fonts.xml 的限制**
   - 自定义系统字体未必在 `/system/etc/fonts.xml` 中
   - CJK Hack 会查询 fonts.xml，但找不到自定义字体
   - 导致回退到硬编码的 "Times New Roman" 等（不存在的字体）

---

## 8. 参考实现 - FontSizePrefs 模式

**文件**: `components/browser_ui/accessibility/android/java/src/.../FontSizePrefs.java`

这是一个范例，展示了如何从 Android 系统配置读取信息：

```java
// 读取系统字体大小缩放因子
fontScale = (int) (fontScale * context.getResources().getConfiguration().fontScale);
```

**启示**: 我们应该创建类似的机制来读取系统自定义字体。

---

## 9. 下一步方案

### 方案 A: 查询 Android TypefaceManager（API 29+）
```java
// 获取系统自定义字体
TypefaceManager tm = context.getSystemService(TypefaceManager.class);
// 查询可用的字体族
```

### 方案 B: 读取 Android Settings 数据库
```java
// 从 Settings 中读取自定义字体设置
String customFont = Settings.System.getString(context.getContentResolver(), "font");
```

### 方案 C: 通过资源配置查询
```java
// 通过 Theme 属性查询字体
android.R.attr.typeface
```

### 方案 D: 监听字体更改事件
```java
// 类似 fontScale，监听配置变化
// 当用户改变系统字体时收到通知
```

---

## 10. 集成步骤

为了让 Chromium 网页内容跟随系统自定义字体，需要：

1. **在 Java 层** (AwSettings.java)
   - 添加从 Android 系统读取自定义字体的代码
   - 保存到 `mStandardFontFamily` 等私有变量

2. **触发更新**
   - 在系统字体改变时重新读取（类似 fontScale）
   - 调用 `updateWebkitPreferencesLocked()`

3. **验证调用链**
   - AwSettings (Java) → aw_settings.cc (Native) → WebPreferences → FontSelector → SkFontMgr

4. **测试**
   - 在 Android 系统中改变字体
   - 验证 Chromium 网页内容是否跟随

---

## 11. 关键代码文件总结

**系统字体读取源头**（需要改动）:
- [AwSettings.java - 初始化](android_webview/java/src/org/chromium/android_webview/AwSettings.java#L186-L191)
- [AwSettings.java - 构造](android_webview/java/src/org/chromium/android_webview/AwSettings.java#L395-L453)

**字体数据流**（当前代码）:
- [aw_settings.cc - PopulateWebPreferencesLocked](android_webview/browser/aw_settings.cc#L578-L626)
- [aw_content_browser_client.cc - OverrideWebPreferences](android_webview/browser/aw_content_browser_client.cc#L655-L668)

**字体最终使用**:
- [web_contents_impl.cc - ComputeWebPreferences](content/browser/web_contents/web_contents_impl.cc#L3634)
- [font_cache_android.cc - CJK Hack 查询](third_party/blink/renderer/platform/fonts/android/font_cache_android.cc#L231-L287)

---

## 总结

Chromium UI 能够跟随系统字体的机制是：
1. 在 Java 层初始化时读取系统配置（如 fontScale）
2. 通过 JNI 传递到 C++ 层的 WebPreferences
3. 在页面加载时注入到 Blink 的字体选择器

**当前问题**：字体名称是硬编码的，从未尝试读取系统自定义字体信息。

**解决方向**：在步骤 1 中添加对 Android 系统自定义字体的查询，使其与 fontScale 的读取方式一致。
