# Android 版 Chromium 浏览器 - 自定义字体功能实现指南

## 📱 Android 版 Chromium 架构

### 与桌面端的关键区别

```
🖥️  Desktop Chrome                  📱 Android Chrome
┌─────────────────────┐            ┌──────────────────────┐
│ Chrome Settings UI  │            │ Chrome Settings App  │
│ (TypeScript/React)  │            │ (Java/Kotlin)        │
└──────────┬──────────┘            └──────────┬───────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐            ┌──────────────────────┐
│   PrefService       │            │   PrefService        │
│  (C++ 数据库)       │            │  (C++ 数据库)        │
└──────────┬──────────┘            └──────────┬───────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐            ┌──────────────────────┐
│ PrefsTabHelper      │            │ PrefsTabHelper       │
│ (C++ 观察者)        │            │ (C++ 观察者)         │
└──────────┬──────────┘            └──────────┬───────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐            ┌──────────────────────┐
│  WebPreferences     │            │  WebPreferences      │
│ (Mojo 序列化)       │            │ (Mojo 序列化)        │
└──────────┬──────────┘            └──────────┬───────────┘
           │                                 │
           ▼                                 ▼
        Renderer                          Renderer
```

## ✅ 好消息：Android 版 Chromium 已经支持！

### 现状分析

Android 版 Chromium 使用**完全相同的架构**：

```cpp
// prefs_tab_helper.cc 中的代码片段（所有平台共用）

void PrefsTabHelper::OnFontFamilyPrefChanged(const std::string& pref_name) {
  // 这个代码在所有平台（包括 Android）都运行
  std::string generic_family;
  std::string script;
  if (pref_names_util::ParseFontNamePrefPath(pref_name, &generic_family,
                                             &script)) {
    // ... 字体偏好处理 ...
    GetWebContents().SetWebPreferences(web_prefs);
    return;
  }
}
```

### 平台条件检查

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
// 行 50-89: Android 版 Chromium 的特定处理

#if BUILDFLAG(IS_ANDROID)
  // Android Chrome 包含字体设置支持
#include "chrome/browser/flags/android/chrome_feature_list.h"
#include "components/browser_ui/accessibility/android/font_size_prefs_android.h"
#else
  // 桌面端特定的代码
#endif

// 行 82: 字体族偏好注册
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  // 行 85-87: 注册字体大小
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
#endif
```

## 🎯 Android 版 Chromium 的现状

### 情况 1: 常规 Android Chrome

```
✅ 已支持的功能:
  • WebPreferences 结构（完整 7 个字体族）
  • PrefService 存储
  • PrefsTabHelper 观察者机制
  • FontFallbackIterator 字体选择
  • Mojo IPC 通信

❌ 受限制的功能:
  • Settings UI 中没有字体设置页面
    （条件: !IS_ANDROID 关闭了 UI）
  • 用户无法通过 Settings 改变字体
```

### 情况 2: 桌面 Android 扩展

```
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)

如果启用 ENABLE_DESKTOP_ANDROID_EXTENSIONS：
✅ 启用完整的字体族偏好注册
✅ 启用字体大小设置
✅ 启用多语言脚本支持

这用于开发或特殊构建
```

---

## 🔧 为 Android 版 Chromium 添加字体设置

### 路线 1: 启用现有功能 (推荐) ⭐⭐⭐

**目标**: 将桌面版的字体 Settings UI 移植到 Android

#### 第 1 步: 条件编译修改

**文件**: `chrome/browser/ui/prefs/prefs_tab_helper.cc`

```cpp
// 当前代码 (行 82)
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
#endif

// 改为（支持 Android Chrome）
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);

  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
  RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                            IDS_MINIMUM_LOGICAL_FONT_SIZE);
#endif
```

#### 第 2 步: 创建 Android Settings UI

**文件**: `chrome/android/java/src/org/chromium/chrome/browser/settings/FontSettingsFragment.java`

```java
// 新增 Android Settings Fragment

package org.chromium.chrome.browser.settings;

import android.os.Bundle;
import androidx.preference.Preference;
import androidx.preference.PreferenceFragmentCompat;
import org.chromium.chrome.R;
import org.chromium.chrome.browser.preferences.ChromePreferenceKeys;
import org.chromium.base.shared_preferences.SharedPreferencesManager;

public class FontSettingsFragment extends PreferenceFragmentCompat {
    private static final String PREF_KEY_SERIF_FONT = 
        ChromePreferenceKeys.FONT_SERIF;
    private static final String PREF_KEY_SANS_SERIF_FONT = 
        ChromePreferenceKeys.FONT_SANS_SERIF;
    private static final String PREF_KEY_MONOSPACE_FONT = 
        ChromePreferenceKeys.FONT_MONOSPACE;
    private static final String PREF_KEY_FONT_SIZE = 
        ChromePreferenceKeys.FONT_SIZE;

    @Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        setPreferencesFromResource(R.xml.font_preferences, rootKey);
        
        // 创建字体选择下拉菜单
        createFontPreference("serif", R.string.serif_font_pref);
        createFontPreference("sansserif", R.string.sans_serif_font_pref);
        createFontPreference("fixed", R.string.monospace_font_pref);
    }

    private void createFontPreference(String fontType, int titleResId) {
        Preference fontPref = findPreference(fontType);
        if (fontPref != null) {
            fontPref.setTitle(titleResId);
            fontPref.setOnPreferenceChangeListener((pref, value) -> {
                // 保存到 PrefService
                saveFontPreference(fontType, (String) value);
                // 通知 WebContents 更新
                notifyFontPreferenceChanged(fontType, (String) value);
                return true;
            });
        }
    }

    private void saveFontPreference(String fontType, String fontName) {
        // 通过 Chrome 的 PrefService 保存
        String prefKey = "webkit.webprefs.fonts." + fontType + ".Zyyy";
        ChromePreferenceKeys.setString(prefKey, fontName);
    }

    private void notifyFontPreferenceChanged(String fontType, String fontName) {
        // 触发 PrefsTabHelper 的观察者
        // 自动调用 OnFontFamilyPrefChanged()
    }
}
```

#### 第 3 步: 集成到 Settings

**文件**: `chrome/android/java/src/org/chromium/chrome/browser/settings/SettingsActivity.java`

```java
// 添加字体设置菜单项

public class SettingsActivity {
    // 在 MainSettings 中添加字体设置
    private void setupFontSettings() {
        if (isAppearanceSettingsPage) {
            // 添加字体选项
            addPreference("font_settings", 
                         FontSettingsFragment.class,
                         R.string.font_settings_title);
        }
    }
}
```

#### 第 4 步: 资源文件

**文件**: `chrome/android/java/res/xml/font_preferences.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <PreferenceCategory
        android:title="@string/font_family_settings">

        <ListPreference
            android:key="webkit.webprefs.fonts.serif.Zyyy"
            android:title="@string/serif_font"
            android:entries="@array/font_names"
            android:entryValues="@array/font_values"
            android:defaultValue="Georgia" />

        <ListPreference
            android:key="webkit.webprefs.fonts.sansserif.Zyyy"
            android:title="@string/sans_serif_font"
            android:entries="@array/font_names"
            android:entryValues="@array/font_values"
            android:defaultValue="Arial" />

        <ListPreference
            android:key="webkit.webprefs.fonts.fixed.Zyyy"
            android:title="@string/monospace_font"
            android:entries="@array/font_names"
            android:entryValues="@array/font_values"
            android:defaultValue="Courier New" />
    </PreferenceCategory>

    <PreferenceCategory
        android:title="@string/font_size_settings">

        <SeekBarPreference
            android:key="webkit.webprefs.default_font_size"
            android:title="@string/default_font_size"
            android:min="8"
            android:max="32"
            android:defaultValue="16" />
    </PreferenceCategory>

</PreferenceScreen>
```

### 路线 2: 通过 Java 编程 API

**优点**: 应用可以编程控制字体

```java
// 应用开发者可以这样使用

WebView webview = ...;
Profile profile = Profile.getLastUsedRegularProfile();
PrefService prefService = profile.getPrefService();

// 设置 serif 字体为 "Georgia"
prefService.setString("webkit.webprefs.fonts.serif.Zyyy", "Georgia");

// 设置字体大小
prefService.setInteger("webkit.webprefs.default_font_size", 18);

// 刷新所有 WebContents
notifyWebPreferencesChanged(webview);
```

---

## 📋 核心代码文件位置

### Android 版 Chromium 的关键文件

| 文件 | 位置 | 作用 |
|------|------|------|
| PrefsTabHelper | `chrome/browser/ui/prefs/prefs_tab_helper.cc` | 字体偏好处理（所有平台共用） |
| PrefWatcher | `chrome/browser/ui/prefs/pref_watcher.cc` | 偏好观察者（所有平台共用） |
| WebPreferences | `third_party/blink/public/common/web_preferences/` | 跨进程数据结构（所有平台共用） |
| SettingsActivity | `chrome/android/java/src/org/chromium/chrome/browser/settings/SettingsActivity.java` | Android Settings UI |
| ChromePreferences | `chrome/android/java/src/org/chromium/chrome/browser/preferences/` | Android 偏好管理 |

### Android 特定的 Settings 路径

```
chrome/android/java/src/org/chromium/chrome/browser/settings/
├─ SettingsActivity.java              - Settings 主类
├─ MainSettings.java                  - Settings 菜单
├─ AppearanceSettingsFragment.java    - Appearance 分类
└─ FontSettingsFragment.java          - 字体设置页面 (需要创建)
```

---

## 🔗 与桌面端的代码复用

### 完全共用的代码

```cpp
// 这些代码在 Android 和桌面端都运行：

✅ PrefsTabHelper::OnFontFamilyPrefChanged()
✅ PrefsTabHelper::OverrideFontFamily()
✅ PrefWatcher 观察者机制
✅ WebPreferences 数据结构
✅ FontFallbackIterator 字体选择
✅ HarfBuzz 字形塑造
✅ Mojo IPC 通信

❌ 平台特定的代码：
❌ Settings UI (Java/TypeScript)
❌ 字体列表获取 (platform APIs)
❌ 偏好存储位置 (可能不同)
```

### 复用策略

```
┌─────────────────────────────────────────┐
│ 核心 C++ 逻辑（所有平台共用）          │
│ prefs_tab_helper.cc                    │
│ web_preferences.h                      │
│ font_fallback_iterator.cc              │
└─────────────┬───────────────────────────┘
              │
     ┌────────┴────────┐
     ▼                 ▼
┌──────────┐      ┌──────────┐
│ Desktop  │      │ Android  │
│  UI      │      │   UI     │
│(TypeScript)   │  (Java)   │
└──────────┘      └──────────┘
```

---

## 🎯 实现步骤（详细版）

### 第 1 阶段：理解现状（1 天）

```
1. 研究 prefs_tab_helper.cc 的条件编译
   └─ 理解 #if !BUILDFLAG(IS_ANDROID) 的含义
   
2. 查看桌面端的字体 Settings UI
   └─ chrome/browser/resources/settings/appearance_page/
   
3. 研究 Android Chrome 的 Settings 结构
   └─ chrome/android/java/src/org/chromium/chrome/browser/settings/
```

### 第 2 阶段：创建 Android Settings UI（3-5 天）

```
1. 创建 FontSettingsFragment.java
   └─ 使用 PreferenceFragmentCompat
   └─ 创建字体选择下拉菜单
   
2. 创建 font_preferences.xml
   └─ 定义首选项结构
   
3. 集成到 SettingsActivity
   └─ 添加菜单项
   └─ 导航到字体设置
   
4. 添加字符串资源
   └─ strings.xml 中的标签和描述
```

### 第 3 阶段：字体列表获取（2-3 天）

```
1. 实现 getAvailableFonts()
   ├─ 方法 A: 扫描 /system/fonts/
   ├─ 方法 B: Typeface 枚举
   └─ 方法 C: Android API 29+ 系统字体
   
2. 缓存字体列表
   └─ 避免重复扫描
   
3. 过滤可用字体
   └─ 移除不可用的字体
```

### 第 4 阶段：集成 PrefService（3-5 天）

```
1. 创建 PrefService 桥接
   ├─ JNI 调用读写偏好
   └─ 或通过 Profile 访问
   
2. 保存字体选择
   └─ prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", "Georgia")
   
3. 通知渲染器
   └─ OnWebPreferencesChanged()
   
4. 恢复用户设置
   └─ 应用启动时读取保存的字体
```

### 第 5 阶段：测试和优化（3-5 天）

```
1. 单元测试
   ├─ 字体选择测试
   ├─ 偏好保存/恢复测试
   └─ 多语言测试
   
2. 集成测试
   ├─ 渲染测试
   ├─ 性能测试
   └─ 设备兼容性测试
   
3. 真机测试
   ├─ 各种 Android 版本
   ├─ 各种屏幕尺寸
   └─ 性能监控
```

---

## 💻 代码示例

### 示例 1: 从 PrefService 读取字体偏好

```cpp
// C++ 端 - chrome/browser/prefs_helper_android.cc

#include "components/prefs/pref_service.h"
#include "chrome/common/pref_names.h"

std::u16string GetSerifFont(PrefService* prefs) {
  // 读取保存的 serif 字体
  return base::UTF8ToUTF16(
      prefs->GetString("webkit.webprefs.fonts.serif.Zyyy"));
}

void SetSerifFont(PrefService* prefs, const std::string& font_name) {
  // 保存 serif 字体选择
  prefs->SetString("webkit.webprefs.fonts.serif.Zyyy", font_name);
  
  // 触发 PrefsTabHelper 的观察者
  // （自动调用 OnFontFamilyPrefChanged）
}
```

### 示例 2: Java 层获取可用字体

```java
// Java 端 - android/java/src/org/chromium/chrome/browser/settings/FontUtils.java

public class FontUtils {
    /**
     * 获取系统中可用的字体列表
     */
    public static List<String> getAvailableFonts() {
        List<String> fonts = new ArrayList<>();
        
        // 方法 1: 扫描系统字体目录
        File fontDir = new File("/system/fonts/");
        if (fontDir.exists() && fontDir.isDirectory()) {
            File[] files = fontDir.listFiles((dir, name) -> 
                name.endsWith(".ttf") || name.endsWith(".otf"));
            if (files != null) {
                for (File file : files) {
                    // 提取字体名称（不含扩展名）
                    String fontName = file.getName();
                    fontName = fontName.substring(0, fontName.lastIndexOf('.'));
                    fonts.add(fontName);
                }
            }
        }
        
        // 常见的预定义字体
        if (fonts.isEmpty()) {
            fonts.add("Arial");
            fonts.add("Courier New");
            fonts.add("Georgia");
            fonts.add("Times New Roman");
            fonts.add("Verdana");
        }
        
        return fonts;
    }
}
```

### 示例 3: 字体更改时刷新网页

```java
// Java 端 - 字体偏好改变时的回调

private void onFontPreferenceChanged(String fontType, String fontName) {
    // 1. 保存到 PrefService
    ProfileManager.getLastUsedRegularProfile()
        .getPrefService()
        .setString("webkit.webprefs.fonts." + fontType + ".Zyyy", fontName);
    
    // 2. 通知所有 WebContents 更新
    for (Activity activity : getAllChromeActivities()) {
        Tab tab = getCurrentTab(activity);
        if (tab != null) {
            tab.getWebContents()
                .onWebPreferencesChanged();  // 或通过 JNI 调用
        }
    }
}
```

---

## 📊 Android vs 桌面端完整对比

| 功能 | 桌面 Chrome | Android Chrome |
|------|------------|----------------|
| **字体族数** | 7 (+ math) | 7 (相同) |
| **脚本支持** | 150+ (ICU) | 150+ (相同) |
| **PrefService** | ✅ 完全支持 | ✅ 完全支持 |
| **PrefsTabHelper** | ✅ 内置 | ✅ 内置 |
| **Settings UI** | ✅ TypeScript | ❌ 需要创建 (Java) |
| **多进程** | ✅ Browser+Renderer | ✅ Browser+Renderer |
| **性能开销** | 低 | 低 |
| **工作量** | ✅ 已完成 | ⚠️ 需要 UI 实现 |

---

## ⏱️ 总体工作量估计

```
1. 理解现状               - 1 天
2. Android Settings UI    - 3-5 天
3. 字体列表管理           - 2-3 天
4. PrefService 集成       - 3-5 天
5. 测试和优化             - 3-5 天
─────────────────────────────────
总计                      - 12-22 天 (2-3 周)
```

---

## 🎯 推荐实现顺序

### MVP (最小可行版本) - 第 1 周

```
1. 创建基础 Settings UI
   ├─ 使用系统预定义字体列表
   └─ 支持 serif, sans-serif, monospace
   
2. 集成 PrefService
   ├─ 保存和恢复用户选择
   └─ 触发 OnWebPreferencesChanged()
   
3. 基础测试
   ├─ 单个设备
   └─ 基本功能验证
```

### 完整版 - 第 2-3 周

```
1. 增强 Settings UI
   ├─ 字体预览
   ├─ 字体大小调整
   └─ 更好的 UX
   
2. 高级功能
   ├─ 多语言脚本支持
   ├─ 字体列表缓存
   └─ 性能优化
   
3. 完整测试
   ├─ 多个 Android 版本
   ├─ 多个设备
   └─ 性能监控
```

---

## ✨ 关键优势

### 代码复用率极高

```
✅ C++ 核心代码: 100% 复用 (PrefsTabHelper 等)
✅ 数据结构: 100% 复用 (WebPreferences)
✅ 底层机制: 100% 复用 (font selection, IPC)

❌ UI 代码: 0% 复用 (需要重新实现 Java)
```

### 与桌面端功能对等

```
Android Chrome = Desktop Chrome
              (除了 UI 和字体列表获取)
```

### 最小修改

```
✅ 不需要修改 Blink 渲染器
✅ 不需要修改 IPC 通信
✅ 不需要修改 WebPreferences 结构
✅ 只需要添加 Settings UI
```

---

## 🚀 立即开始

### 参考的具体源文件

```
了解当前状态:
  chrome/browser/ui/prefs/prefs_tab_helper.cc
    └─ 行 50-89: Android 条件编译

参考字体处理逻辑:
  chrome/browser/ui/prefs/prefs_tab_helper.cc
    └─ OverrideFontFamily() 函数
    └─ 这个代码在 Android 上也运行！

参考桌面端 Settings UI:
  chrome/browser/resources/settings/appearance_page/
    └─ appearance_fonts_page.ts
    └─ 参考 UI 设计

参考 Android Settings 结构:
  chrome/android/java/src/org/chromium/chrome/browser/settings/
    └─ SettingsActivity.java
    └─ AppearanceSettingsFragment.java
```

---

## 💡 总结

**Android 版 Chromium 的字体功能现状**:

1. ✅ **核心功能已存在**: PrefsTabHelper, WebPreferences, 字体选择都已有
2. ✅ **架构完全相同**: 与桌面端共用 C++ 代码
3. ❌ **缺少 Settings UI**: 没有 Android 的字体设置页面
4. ✅ **实现难度不高**: 主要是创建 Java Settings UI
5. ✅ **代码复用极高**: 99% 的逻辑可复用

**建议方案**: 为 Android Chrome 创建字体 Settings UI，估计 2-3 周完成

这比从零开始实现容易得多，因为所有底层机制都已存在！
