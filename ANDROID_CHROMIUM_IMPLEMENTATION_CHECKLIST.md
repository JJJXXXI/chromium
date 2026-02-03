# Android Chromium 字体功能实现 - 完整检查清单

## 🎯 项目概览

**目标**: 为 Android 版 Chromium 浏览器添加自定义字体功能
**范围**: Android Chrome 浏览器应用（不包括 WebView）
**预计时间**: 2-3 周
**代码复用率**: 99%（仅需要 Android UI 层实现）

---

## 📋 预备工作清单

### 1. 环境准备

- [ ] 已安装 Android SDK (API 25+)
- [ ] 已安装 Android NDK
- [ ] 已配置 Chromium 构建环境 (`gclient sync`, `autoninja`)
- [ ] 可以成功构建 Android Chrome (`autoninja -C out/Default chrome_apk`)
- [ ] 已安装 Java IDE (Android Studio) 或编辑器

### 2. 代码理解准备

- [ ] 已阅读 `ANDROID_CHROMIUM_CUSTOM_FONT_GUIDE.md`（本系列文档）
- [ ] 已查看 `chrome/browser/ui/prefs/prefs_tab_helper.cc` 第 50-100 行
- [ ] 已理解条件编译 `#if !BUILDFLAG(IS_ANDROID)` 的含义
- [ ] 已查看 `chrome/android/java/src/org/chromium/chrome/browser/settings/` 目录结构
- [ ] 已阅读 `android_webview/browser/aw_settings.cc` 作为参考（可选）

### 3. 创建功能分支

```bash
# 创建 git 分支
git checkout -b feature/android-custom-fonts

# 或者跟踪 GERRIT CL
gclient sync
```

- [ ] 已创建功能分支或 Gerrit CL

---

## 🔧 Phase 1: 理解现状（1-2 天）

### 1.1 分析条件编译

**任务**: 理解为什么 Android Chrome 目前没有字体支持

**检查清单**:
- [ ] 打开 `chrome/browser/ui/prefs/prefs_tab_helper.cc`
- [ ] 查看第 50-54 行的 Android 条件编译
- [ ] 查看第 82-88 行的字体注册守卫
- [ ] 理解 `#if !BUILDFLAG(IS_ANDROID)` 的含义
- [ ] 记录为什么这个守卫存在（代码注释在第 89 行）

**关键代码**:
```cpp
// 行 82-88
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFontSize, 16);
  registry->RegisterIntegerPref(prefs::kWebKitDefaultFixedFontSize, 13);
  registry->RegisterIntegerPref(prefs::kWebKitMinimumFontSize, 0);
  RegisterLocalizedFontPref(registry, prefs::kWebKitMinimumLogicalFontSize,
                            IDS_MINIMUM_LOGICAL_FONT_SIZE);
#endif
```

**完成标志**: 
- [ ] 能解释这行代码的含义
- [ ] 理解三个条件的区别（Android, Desktop Android Extensions）

### 1.2 查看桌面端的字体 Settings

**任务**: 理解桌面 Chrome 如何实现字体 Settings

**检查清单**:
- [ ] 查看 `chrome/browser/resources/settings/appearance_page/`
- [ ] 查看 `appearance_fonts_page.ts`（如果存在）
- [ ] 理解字体选择 UI 的结构
- [ ] 记录哪些字体被支持
- [ ] 记录字体大小范围

**关键文件**:
```
chrome/browser/resources/settings/appearance_page/
  ├─ appearance_page.ts
  ├─ appearance_fonts_page.ts (或类似名称)
  └─ ... 其他外观设置
```

**完成标志**:
- [ ] 能解释桌面端的字体 UI 如何工作
- [ ] 找到了字体列表的定义

### 1.3 查看 Android Settings 结构

**任务**: 理解 Android Chrome Settings 是如何组织的

**检查清单**:
- [ ] 查看 `chrome/android/java/src/org/chromium/chrome/browser/settings/`
- [ ] 找到 `SettingsActivity.java`
- [ ] 找到 `MainSettings.java` 或主 Settings Fragment
- [ ] 找到其他 `*SettingsFragment.java` 文件作为参考
- [ ] 理解 Fragment 的通用模式

**关键文件**:
```
chrome/android/java/src/org/chromium/chrome/browser/settings/
  ├─ SettingsActivity.java
  ├─ MainSettings.java 或 MainSettingsFragment.java
  ├─ AppearanceSettingsFragment.java (或类似)
  ├─ AccessibilitySettingsFragment.java
  └─ ... 其他 Fragment
```

**完成标志**:
- [ ] 理解 Android Settings 使用 PreferenceFragmentCompat
- [ ] 理解 Fragment 如何注册和导航
- [ ] 找到了 xml 偏好文件（通常在 `res/xml/`）

### 1.4 查看 WebView 参考实现

**任务**: 理解 Android WebView 如何处理字体（作为参考）

**检查清单**:
- [ ] 查看 `android_webview/browser/aw_settings.cc`（C++ 侧）
- [ ] 查看 `android_webview/java/src/org/chromium/android_webview/AwSettings.java`
- [ ] 理解 JNI 桥接模式
- [ ] 记录支持的字体族
- [ ] 理解 `PopulateWebPreferencesLocked()` 的实现

**关键代码片段**:
```cpp
// android_webview/browser/aw_settings.cc 行 608-630
web_prefs->standard_font_family_map[blink::web_pref::kCommonScript] =
    ConvertJavaStringToUTF16(
        Java_AwSettings_getStandardFontFamilyLocked(env, obj));
```

**完成标志**:
- [ ] 理解 JNI 的字体处理方式
- [ ] 理解 WebView 的单进程模型与 Chrome 多进程模型的区别
- [ ] 理解为什么 WebView 的实现与 Chrome 不同

**预计完成时间**: 1-2 天

---

## 🏗️ Phase 2: 创建 Android Settings UI（3-5 天）

### 2.1 创建 FontSettingsFragment.java

**任务**: 创建字体设置的 Android Fragment

**检查清单**:
- [ ] 在 `chrome/android/java/src/org/chromium/chrome/browser/settings/` 中创建新文件
- [ ] 文件名: `FontSettingsFragment.java`
- [ ] 继承 `PreferenceFragmentCompat`
- [ ] 实现 `OnCreatePreferences()` 方法
- [ ] 加载字体偏好 XML 文件

**代码模板**:
```java
package org.chromium.chrome.browser.settings;

import android.os.Bundle;
import androidx.preference.PreferenceFragmentCompat;
import org.chromium.chrome.R;

public class FontSettingsFragment extends PreferenceFragmentCompat {
    @Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        setPreferencesFromResource(R.xml.font_preferences, rootKey);
    }
}
```

**完成标志**:
- [ ] 文件已创建
- [ ] 代码能编译通过
- [ ] Fragment 已正确设置

### 2.2 创建 font_preferences.xml

**任务**: 定义字体偏好的 XML 结构

**检查清单**:
- [ ] 在 `chrome/android/java/res/xml/` 中创建新文件
- [ ] 文件名: `font_preferences.xml`
- [ ] 定义字体族 ListPreference（serif, sans-serif, fixed）
- [ ] 定义字体大小 SeekBarPreference
- [ ] 使用正确的 preference key names

**关键 key names** (必须与 Chrome 偏好相匹配):
```xml
webkit.webprefs.fonts.serif.Zyyy          <!-- Serif 字体 -->
webkit.webprefs.fonts.sansserif.Zyyy      <!-- Sans-serif 字体 -->
webkit.webprefs.fonts.fixed.Zyyy          <!-- Monospace 字体 -->
webkit.webprefs.default_font_size         <!-- 默认字体大小 -->
webkit.webprefs.default_fixed_font_size   <!-- 固定字体大小 -->
webkit.webprefs.minimum_font_size         <!-- 最小字体大小 -->
```

**代码示例**:
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

        <!-- 其他字体 -->
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

**完成标志**:
- [ ] XML 文件已创建
- [ ] XML 结构正确
- [ ] 所有 key names 与 Chrome 偏好相匹配
- [ ] XML 能通过验证

### 2.3 创建字体数据数组

**任务**: 定义字体列表数组

**检查清单**:
- [ ] 在 `chrome/android/java/res/values/arrays.xml` 中添加
- [ ] 创建 `font_names` 数组（显示名称）
- [ ] 创建 `font_values` 数组（技术名称）
- [ ] 确保两个数组长度相同

**代码示例**:
```xml
<!-- 在 chrome/android/java/res/values/arrays.xml 中添加 -->

<array name="font_names">
    <item>Georgia</item>
    <item>Arial</item>
    <item>Times New Roman</item>
    <item>Courier New</item>
    <item>Verdana</item>
    <item>Comic Sans</item>
</array>

<array name="font_values">
    <item>Georgia</item>
    <item>Arial</item>
    <item>Times New Roman</item>
    <item>Courier New</item>
    <item>Verdana</item>
    <item>Comic Sans</item>
</array>
```

**完成标志**:
- [ ] 数组已添加到 arrays.xml
- [ ] 数组名称与 XML 中的引用相匹配
- [ ] 数组长度相同

### 2.4 添加字符串资源

**任务**: 为 UI 添加本地化字符串

**检查清单**:
- [ ] 打开 `chrome/android/java/res/values/strings.xml`
- [ ] 添加 `font_settings_title`
- [ ] 添加 `font_family_settings`
- [ ] 添加 `serif_font`
- [ ] 添加 `sans_serif_font`
- [ ] 添加 `monospace_font`
- [ ] 添加 `font_size_settings`
- [ ] 添加 `default_font_size`
- [ ] 添加中文版本（可选，取决于 locales 文件）

**代码示例**:
```xml
<!-- 在 chrome/android/java/res/values/strings.xml 中添加 -->

<string name="font_settings_title">字体</string>
<string name="font_family_settings">字体族</string>
<string name="serif_font">衬线字体</string>
<string name="sans_serif_font">无衬线字体</string>
<string name="monospace_font">等宽字体</string>
<string name="font_size_settings">字体大小</string>
<string name="default_font_size">默认字体大小</string>
```

**完成标志**:
- [ ] 所有字符串已添加
- [ ] 字符串引用在代码中都能找到
- [ ] 不存在拼写错误

### 2.5 集成到 SettingsActivity

**任务**: 将字体设置添加到 Settings 菜单

**检查清单**:
- [ ] 打开 `chrome/android/java/src/org/chromium/chrome/browser/settings/SettingsActivity.java`
- [ ] 找到外观设置的导航项
- [ ] 添加字体设置菜单项
- [ ] 设置正确的 Fragment 类和标题

**关键位置**:
```java
// 在 MainSettings.java 或 SettingsActivity.java 中添加

// 找到外观设置部分，添加：
Preference fontSettings = new Preference(context);
fontSettings.setTitle(R.string.font_settings_title);
fontSettings.setOnPreferenceClickListener(preference -> {
    navigateTo(FontSettingsFragment.class, 
              R.string.font_settings_title);
    return true;
});
```

**完成标志**:
- [ ] 菜单项已添加
- [ ] 导航正确
- [ ] 能够编译通过

**预计完成时间**: 3-5 天

---

## 💾 Phase 3: 集成 PrefService（3-5 天）

### 3.1 理解 PrefService 集成

**任务**: 理解如何将 Android UI 与 C++ PrefService 连接

**检查清单**:
- [ ] 查看现有的 Preference 观察者实现
- [ ] 了解如何从 Java 读写 PrefService
- [ ] 了解 JNI 桥接（如果需要）
- [ ] 或者了解是否有现成的 API（如 Profile 对象）

**关键实现选项**:

**选项 A: 通过 PrefChangeRegistrar（推荐）**
```java
// 获取 Profile
Profile profile = Profile.getLastUsedRegularProfile();

// 获取 PrefService
PrefService prefService = profile.getPrefService();

// 设置字体
prefService.setString("webkit.webprefs.fonts.serif.Zyyy", "Georgia");

// 读取字体
String font = prefService.getString("webkit.webprefs.fonts.serif.Zyyy");
```

**选项 B: 通过 SharedPreferences（如果 PrefService 不可用）**
```java
// 备选方案
SharedPreferences prefs = PreferenceManager.getDefaultSharedPreferences(context);
prefs.edit()
    .putString("webkit.webprefs.fonts.serif.Zyyy", "Georgia")
    .apply();
```

**完成标志**:
- [ ] 理解 PrefService 的 Android API
- [ ] 知道如何从 Java 读写偏好
- [ ] 选定实现方案（A 或 B）

### 3.2 创建 PreferenceObserver

**任务**: 监听 Settings UI 中的偏好变化

**检查清单**:
- [ ] 在 FontSettingsFragment 中实现 `OnPreferenceChangeListener`
- [ ] 实现 `onPreferenceChange()` 方法
- [ ] 每当用户改变字体时保存到 PrefService
- [ ] 通知所有 WebContents 更新

**代码示例**:
```java
public class FontSettingsFragment extends PreferenceFragmentCompat
    implements Preference.OnPreferenceChangeListener {

    @Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        setPreferencesFromResource(R.xml.font_preferences, rootKey);
        
        // 为所有偏好设置监听器
        Preference serifPref = findPreference("webkit.webprefs.fonts.serif.Zyyy");
        if (serifPref != null) {
            serifPref.setOnPreferenceChangeListener(this);
        }
        // ... 其他字体 ...
    }

    @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        String prefKey = preference.getKey();
        String value = newValue.toString();
        
        // 保存到 PrefService
        Profile profile = Profile.getLastUsedRegularProfile();
        PrefService prefService = profile.getPrefService();
        prefService.setString(prefKey, value);
        
        // 通知渲染器
        notifyWebPreferencesChanged();
        
        return true;
    }
    
    private void notifyWebPreferencesChanged() {
        // 触发 PrefsTabHelper 的观察者
        // 这会自动通知所有 WebContents 更新
    }
}
```

**完成标志**:
- [ ] 监听器已实现
- [ ] 每个字体偏好都有监听器
- [ ] 改变时能保存到 PrefService
- [ ] 代码能编译通过

### 3.3 触发 WebPreferences 更新

**任务**: 确保偏好改变时渲染器能收到通知

**检查清单**:
- [ ] PrefsTabHelper 已经监听偏好变化
- [ ] 确认当 PrefService 改变时会触发观察者
- [ ] 观察者会调用 `SetWebPreferences()`
- [ ] WebContents 会收到新的 WebPreferences

**验证方法**:
```cpp
// 验证在 prefs_tab_helper.cc 中
// 这行代码会在偏好改变时被调用：
GetWebContents().SetWebPreferences(web_prefs);

// 确认这是自动的，无需额外代码
```

**完成标志**:
- [ ] 理解 PrefsTabHelper 会自动处理
- [ ] 确认不需要额外的 C++ 代码修改
- [ ] 确认数据流是正确的

### 3.4 处理初始值

**任务**: 确保应用启动时正确加载保存的字体

**检查清单**:
- [ ] FontSettingsFragment 在 `onCreatePreferences()` 中加载保存的值
- [ ] 从 PrefService 读取初始值
- [ ] 在 UI 中显示已保存的字体

**代码示例**:
```java
@Override
public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
    setPreferencesFromResource(R.xml.font_preferences, rootKey);
    
    // 从 PrefService 加载初始值
    Profile profile = Profile.getLastUsedRegularProfile();
    PrefService prefService = profile.getPrefService();
    
    ListPreference serifPref = findPreference("webkit.webprefs.fonts.serif.Zyyy");
    if (serifPref != null) {
        String savedFont = prefService.getString(
            "webkit.webprefs.fonts.serif.Zyyy");
        serifPref.setValue(savedFont);
        
        // 为了美观，显示"显示名称"而不是"技术名称"
        String displayName = getDisplayNameForFont(savedFont);
        serifPref.setSummary(displayName);
    }
    
    // 添加监听器
    serifPref.setOnPreferenceChangeListener(this);
}
```

**完成标志**:
- [ ] 应用启动时能加载保存的字体
- [ ] 初始值在 UI 中正确显示
- [ ] 用户改变后能正确保存

**预计完成时间**: 3-5 天

---

## 🔤 Phase 4: 字体列表管理（2-3 天）

### 4.1 获取可用字体

**任务**: 动态获取系统中可用的字体

**检查清单**:
- [ ] 选择字体获取方法（见下文）
- [ ] 实现 `getAvailableFonts()` 方法
- [ ] 测试能否正确识别常见字体
- [ ] 缓存结果以提高性能

**字体获取方法**:

**方法 1: 扫描系统字体目录（推荐）**
```java
private List<String> getAvailableFonts() {
    List<String> fonts = new ArrayList<>();
    File fontDir = new File("/system/fonts/");
    
    if (fontDir.exists() && fontDir.isDirectory()) {
        File[] files = fontDir.listFiles((dir, name) -> 
            name.endsWith(".ttf") || name.endsWith(".otf"));
        
        if (files != null) {
            for (File file : files) {
                String fontName = file.getName()
                    .replaceAll("\\.(ttf|otf)$", "");
                fonts.add(fontName);
            }
        }
    }
    
    Collections.sort(fonts);
    return fonts;
}
```

**方法 2: 使用 Android API (API 29+)**
```java
private List<String> getAvailableFonts() {
    List<String> fonts = new ArrayList<>();
    
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        // Android 10+ 提供字体 API
        FontManager fontManager = context.getSystemService(FontManager.class);
        // ... 使用 FontManager 获取字体 ...
    }
    
    return fonts;
}
```

**方法 3: 预定义字体列表（备选）**
```java
private List<String> getAvailableFonts() {
    return Arrays.asList(
        "Georgia",
        "Arial",
        "Times New Roman",
        "Courier New",
        "Verdana",
        "Comic Sans MS"
    );
}
```

**完成标志**:
- [ ] 选定实现方法
- [ ] 方法已实现
- [ ] 测试能正确返回字体列表
- [ ] 性能可接受

### 4.2 缓存字体列表

**任务**: 缓存字体列表以提高性能

**检查清单**:
- [ ] 实现字体列表缓存
- [ ] 仅在首次调用时扫描文件系统
- [ ] 后续调用返回缓存结果
- [ ] 考虑缓存过期时间（可选）

**代码示例**:
```java
private static List<String> sCachedFonts = null;

private List<String> getAvailableFonts() {
    if (sCachedFonts != null) {
        return sCachedFonts;
    }
    
    // 首次调用时执行扫描
    sCachedFonts = scanSystemFonts();
    
    return sCachedFonts;
}

private List<String> scanSystemFonts() {
    // ... 字体扫描逻辑 ...
}
```

**完成标志**:
- [ ] 缓存已实现
- [ ] 首次调用时扫描
- [ ] 后续调用快速返回

### 4.3 处理不可用字体

**任务**: 优雅地处理用户选择不可用的字体的情况

**检查清单**:
- [ ] 验证字体是否真的存在于系统中
- [ ] 如果不存在，显示警告或使用备选字体
- [ ] 记录不可用的字体
- [ ] 提供回退机制

**代码示例**:
```java
private boolean isFontAvailable(String fontName) {
    List<String> availableFonts = getAvailableFonts();
    return availableFonts.contains(fontName);
}

@Override
public boolean onPreferenceChange(Preference preference, Object newValue) {
    String fontName = newValue.toString();
    
    if (!isFontAvailable(fontName)) {
        Toast.makeText(context, 
            "字体 " + fontName + " 不可用，使用默认字体", 
            Toast.LENGTH_SHORT).show();
        return false; // 不接受改变
    }
    
    // 保存到 PrefService
    // ...
    return true;
}
```

**完成标志**:
- [ ] 验证逻辑已实现
- [ ] 不可用字体被正确处理
- [ ] 用户能看到清晰的反馈

**预计完成时间**: 2-3 天

---

## ✅ Phase 5: 测试和验证（3-5 天）

### 5.1 编译验证

**任务**: 确保代码能正确编译

**检查清单**:
- [ ] 创建的 Java 文件无编译错误
- [ ] XML 文件格式正确
- [ ] 字符串资源引用正确
- [ ] 构建 APK：`autoninja -C out/Default chrome_apk`
- [ ] 构建成功，没有警告

**命令**:
```bash
# 检查 Java 编译
autoninja -C out/Default chrome_apk

# 检查是否有错误
# 查看输出中是否有 "ERROR" 或 "FAILED"
```

**完成标志**:
- [ ] `autoninja` 成功完成
- [ ] APK 生成成功
- [ ] 没有编译错误或警告

### 5.2 单元测试

**任务**: 为字体功能编写单元测试

**检查清单**:
- [ ] 创建 `FontSettingsFragmentTest.java`
- [ ] 测试偏好改变时是否正确保存
- [ ] 测试初始值是否正确加载
- [ ] 测试字体列表获取功能
- [ ] 运行测试：`autoninja -C out/Default chrome_unittests`

**测试代码示例**:
```java
public class FontSettingsFragmentTest {
    @Test
    public void testSaveFont() {
        // 创建 Fragment
        FontSettingsFragment fragment = new FontSettingsFragment();
        
        // 模拟用户选择字体
        Preference pref = fragment.findPreference("webkit.webprefs.fonts.serif.Zyyy");
        pref.callChangeListener("Georgia");
        
        // 验证已保存
        // ...
    }
}
```

**完成标志**:
- [ ] 单元测试已创建
- [ ] 至少覆盖主要功能
- [ ] 测试通过

### 5.3 集成测试

**任务**: 测试完整的端-到-端功能

**检查清单**:
- [ ] 在真机或模拟器上安装应用
- [ ] 打开 Settings > 字体
- [ ] 改变字体选择
- [ ] 打开网页，验证字体已改变
- [ ] 关闭应用并重启，验证设置已保存
- [ ] 多次改变字体，验证都能正确应用

**测试步骤**:
```
1. 安装 APK：
   adb install -r out/Default/apks/Chrome.apk

2. 打开应用并进入 Settings

3. 找到字体设置（在 Appearance 下）

4. 改变 serif 字体为 "Georgia"

5. 打开网页（例如 google.com）

6. 验证网页使用了新字体

7. 重启应用

8. 验证字体设置被保存

9. 打开同一网页，验证仍然使用 Georgia 字体
```

**完成标志**:
- [ ] 能成功进入字体设置
- [ ] 能改变字体选择
- [ ] 改变会立即应用到网页
- [ ] 设置会被保存
- [ ] 应用重启后设置保持不变

### 5.4 兼容性测试

**任务**: 测试不同 Android 版本和设备的兼容性

**检查清单**:
- [ ] 测试 Android 6 (API 23) 以上的版本
- [ ] 测试不同的屏幕尺寸（手机、平板）
- [ ] 测试不同的 DPI (ldpi, mdpi, hdpi, xhdpi 等)
- [ ] 测试不同的语言设置
- [ ] 测试朝向改变（横屏 <-> 纵屏）

**测试覆盖**:
- [ ] Android 6 (API 23)
- [ ] Android 8 (API 26)
- [ ] Android 10 (API 29)
- [ ] Android 12 (API 31)
- [ ] 最新的 Android 版本

**完成标志**:
- [ ] 测试了至少 4 个 Android 版本
- [ ] 在不同版本上功能都正常
- [ ] 没有 API 兼容性问题

### 5.5 性能测试

**任务**: 确保字体功能不会显著影响性能

**检查清单**:
- [ ] 启动时间没有明显增加
- [ ] 改变字体时渲染不卡顿
- [ ] 内存占用在可接受范围内
- [ ] 电池消耗没有显著增加

**性能指标**:
```
启动时间:     < 5 秒 (vs 原本)
改变字体响应时间: < 500 ms
内存开销:     < 5 MB
```

**完成标志**:
- [ ] 性能指标在可接受范围内
- [ ] 没有明显的卡顿或延迟
- [ ] 电池续航没有显著下降

**预计完成时间**: 3-5 天

---

## 🚀 Phase 6: 优化和增强（可选，2-3 天）

### 6.1 UI 优化

**任务**: 改进用户界面的可用性

**检查清单**:
- [ ] 添加字体预览
- [ ] 改进字体列表显示
- [ ] 添加"恢复默认"按钮
- [ ] 改进字体大小调整的 UX
- [ ] 添加帮助文本

### 6.2 性能优化

**任务**: 进一步优化性能

**检查清单**:
- [ ] 优化字体列表扫描
- [ ] 使用后台线程加载字体
- [ ] 改进缓存机制
- [ ] 减少不必要的重新渲染

### 6.3 功能增强

**任务**: 添加额外功能

**检查清单**:
- [ ] 支持多语言脚本（拉丁、中文、阿拉伯等）
- [ ] 添加字体大小预设（小、正常、大）
- [ ] 添加导出/导入设置功能
- [ ] 添加字体搜索功能

---

## 📝 Phase 7: 文档和提交（1-2 天）

### 7.1 编写文档

**任务**: 为实现编写清晰的文档

**检查清单**:
- [ ] 编写 IMPLEMENTATION.md 解释实现
- [ ] 编写测试文档
- [ ] 编写 API 文档（如果有新公开 API）
- [ ] 编写用户指南

### 7.2 代码审查准备

**任务**: 为代码审查做好准备

**检查清单**:
- [ ] 所有代码遵循 Chromium 代码风格指南
- [ ] 所有 Java 代码通过 checkstyle
- [ ] 所有 XML 文件格式正确
- [ ] 提交消息清晰说明改变
- [ ] 关联相关的 bug 或 issue

### 7.3 提交到 Gerrit

**任务**: 将代码提交到 Gerrit CR

**检查清单**:
- [ ] 运行预提交检查：`git cl presubmit`
- [ ] 没有预提交错误
- [ ] 上传到 Gerrit：`git cl upload`
- [ ] 添加适当的标签和评审者
- [ ] 在 CL 描述中解释改变

**命令**:
```bash
# 检查预提交
git cl presubmit

# 上传到 Gerrit
git cl upload

# 或者
git cl upload --re=reviewer@example.com
```

---

## 🎯 最终检查清单

### 代码质量

- [ ] 所有代码遵循 Chromium 代码风格
- [ ] 所有函数都有文档注释
- [ ] 没有 TODO 或 FIXME 注释（或有明确的计划）
- [ ] 没有调试代码或 `System.out.println()`
- [ ] 没有硬编码的魔数或字符串

### 功能完整性

- [ ] ✅ Settings UI 能正常工作
- [ ] ✅ 字体选择能保存和恢复
- [ ] ✅ 渲染器能收到新的字体偏好
- [ ] ✅ 网页显示正确的字体
- [ ] ✅ 多次改变字体都能正常工作

### 测试覆盖

- [ ] ✅ 单元测试通过
- [ ] ✅ 集成测试通过
- [ ] ✅ 兼容性测试通过
- [ ] ✅ 性能测试通过

### 文档完整性

- [ ] ✅ 有实现文档
- [ ] ✅ 有测试文档
- [ ] ✅ 有用户指南
- [ ] ✅ Gerrit CL 有清晰的描述

---

## 📊 进度跟踪模板

使用这个模板跟踪你的进度：

```
Week 1:
  [ ] Day 1: Phase 1.1 - 理解条件编译
  [ ] Day 2: Phase 1.2-1.4 - 研究 Settings 结构
  [ ] Day 3-4: Phase 2.1-2.5 - 创建 Settings UI
  [ ] Day 5: Phase 3.1-3.2 - 集成 PrefService

Week 2:
  [ ] Day 1-2: Phase 3.3-3.4 - 完成 PrefService 集成
  [ ] Day 3-4: Phase 4 - 字体列表管理
  [ ] Day 5: Phase 5.1-5.2 - 编译和单元测试

Week 3:
  [ ] Day 1-2: Phase 5.3-5.5 - 集成测试和性能测试
  [ ] Day 3-4: Phase 6 - 优化（可选）
  [ ] Day 5: Phase 7 - 文档和提交
```

---

## 🆘 故障排除

### 问题 1: 字体设置没有保存

**症状**: 改变字体后，重启应用字体回到默认

**解决方案**:
1. 检查 `onPreferenceChange()` 是否被调用
2. 检查 PrefService 的 `setString()` 是否成功
3. 检查 PrefService 的键名是否正确
4. 验证 `prefsTabHelper.cc` 中的观察者是否工作

### 问题 2: 网页没有显示改变的字体

**症状**: Settings 中改变字体，但网页仍使用旧字体

**解决方案**:
1. 检查 `onPreferenceChange()` 后是否通知了 WebContents
2. 检查 PrefsTabHelper 是否在监听偏好改变
3. 检查 WebPreferences 中的字体值是否被正确设置
4. 在 `prefsTabHelper.cc` 中添加日志进行调试

### 问题 3: Settings UI 没有出现

**症状**: 打开 Settings 看不到字体选项

**解决方案**:
1. 检查 `FontSettingsFragment.java` 是否在正确的包中
2. 检查 `SettingsActivity.java` 中的导航代码
3. 检查 `font_preferences.xml` 是否在 `res/xml/` 中
4. 检查字符串资源是否都定义了
5. 运行 `autoninja chrome_apk` 确保编译成功

### 问题 4: 编译错误

**症状**: `autoninja chrome_apk` 报错

**解决方案**:
1. 读取完整的编译错误消息
2. 检查 XML 格式是否正确
3. 检查 Java 类是否正确导入
4. 检查字符串资源是否存在
5. 清理构建：`gn clean out/Default && gn gen out/Default`

---

## 📞 获取帮助

如果遇到问题，可以：

1. **查看 Chromium 文档**:
   - https://www.chromium.org/developers/
   - https://chromium.googlesource.com/chromium/src/+/main/docs/

2. **查看相关代码**:
   - `prefs_tab_helper.cc` - 字体处理
   - `android_webview/aw_settings.cc` - WebView 参考
   - `chrome/android/java/src/org/chromium/chrome/browser/settings/` - Settings 示例

3. **询问 Chromium 社区**:
   - 搜索相关的 bug 报告
   - 在 Chromium 论坛提问

---

## ✨ 祝你成功！

这是一个相对中等难度的项目，预计需要 2-3 周完成。核心优势是：

✅ 99% 的代码复用
✅ 最小的 C++ 改动
✅ 相对直接的实现路径
✅ 良好的参考实现（WebView）

祝你的实现顺利！

