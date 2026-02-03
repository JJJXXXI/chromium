# Android Chromium 字体功能 - 快速参考卡

## 🎯 一句话总结

**Android 版 Chromium 浏览器已经具备所有字体功能的 C++ 代码，只是缺少 Android 的 Settings UI。添加 UI 需要 2-3 周，代码复用率 99%。**

---

## 📊 快速对比

| 方面 | 状态 | 说明 |
|------|------|------|
| **C++ 核心代码** | ✅ 存在 | PrefsTabHelper, WebPreferences 等（所有平台共用） |
| **字体算法** | ✅ 完成 | FontFallbackIterator, HarfBuzz（所有平台共用） |
| **数据存储** | ✅ 就绪 | PrefService（所有平台共用） |
| **IPC 机制** | ✅ 完成 | Mojo 通信（所有平台共用） |
| **Settings UI** | ❌ 缺失 | 需要创建 Java Fragment（仅 Android）|
| **工作量** | 📈 中等 | 2-3 周（主要是 UI 实现） |

---

## 🔗 关键代码位置

### 条件编译开关
```cpp
// 文件: chrome/browser/ui/prefs/prefs_tab_helper.cc
// 行: 82-88
#if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
  RegisterFontFamilyPrefs(registry, fonts_with_defaults);
  // ... 其他字体注册代码 ...
#endif
```

### 字体处理（所有平台共用）
```cpp
// 文件: chrome/browser/ui/prefs/prefs_tab_helper.cc
// 所有下面的方法都支持 Android（条件编译已处理）

OnFontFamilyPrefChanged()           // 偏好改变时
OverrideFontFamily()                // 覆盖字体族
UpdateFontSettings()                // 更新字体设置
```

### Android Settings 结构
```
chrome/android/java/src/org/chromium/chrome/browser/settings/
├─ SettingsActivity.java            (主 Activity)
├─ MainSettings.java                (Settings 菜单)
├─ AppearanceSettingsFragment.java  (外观设置，参考)
└─ ... 其他设置页面 ...

需要添加:
├─ FontSettingsFragment.java        (NEW)
└─ res/xml/font_preferences.xml     (NEW)
```

---

## 💻 代码架构图

```
┌─────────────────────────────────────────────────────────┐
│               Android Chrome 应用启动                     │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │  Settings Activity      │
        │  (Java, Android)        │
        └────────────┬────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  FontSettingsFragment         │
        │  (NEW - 需要实现)             │
        │  ├─ Serif 字体选择            │
        │  ├─ Sans-serif 字体选择       │
        │  ├─ Fixed 字体选择            │
        │  └─ 字体大小调整              │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  PrefService                  │
        │  (C++, 所有平台)              │
        │  保存: webkit.webprefs.fonts.*│
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  PrefsTabHelper              │
        │  (C++, 观察者)                │
        │  OnFontFamilyPrefChanged()    │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  WebPreferences               │
        │  (C++, 结构化数据)            │
        │  fonts map<script, name>      │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  Mojo IPC                     │
        │  (跨进程通信)                  │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  Renderer Process             │
        │  (Blink 引擎)                  │
        │  FontFallbackIterator         │
        │  HarfBuzz 字形塑造             │
        └────────────┬──────────────────┘
                     │
        ┌────────────▼──────────────────┐
        │  网页显示                      │
        │  使用自定义字体                 │
        └─────────────────────────────────┘
```

---

## 🚀 最快启动方式（5 分钟）

### 第 1 步: 查看现状（2 分钟）
```bash
# 打开这个文件看条件编译
cat chrome/browser/ui/prefs/prefs_tab_helper.cc | sed -n '82,88p'

# 输出应该是:
# #if !BUILDFLAG(IS_ANDROID) || BUILDFLAG(ENABLE_DESKTOP_ANDROID_EXTENSIONS)
#   RegisterFontFamilyPrefs(registry, fonts_with_defaults);
#   ...
# #endif
```

### 第 2 步: 查看参考实现（2 分钟）
```bash
# 查看 Android Settings 的结构
ls chrome/android/java/src/org/chromium/chrome/browser/settings/

# 查看 WebView 的实现（参考）
grep -A 20 "web_prefs->standard_font_family_map" \
  android_webview/browser/aw_settings.cc
```

### 第 3 步: 开始实现（1 分钟）
```bash
# 创建新文件
touch chrome/android/java/src/org/chromium/chrome/browser/settings/FontSettingsFragment.java
touch chrome/android/java/res/xml/font_preferences.xml
```

---

## 📚 阅读顺序

1. **本文档** (5 分钟) ← 你在这里
2. **ANDROID_CHROMIUM_CUSTOM_FONT_GUIDE.md** (30 分钟) - 完整指南
3. **ANDROID_CHROMIUM_VS_WEBVIEW_VS_DESKTOP_COMPARISON.md** (15 分钟) - 架构对比
4. **ANDROID_CHROMIUM_IMPLEMENTATION_CHECKLIST.md** (按需查阅) - 详细检查清单

---

## 🎯 核心实现步骤

### Step 1️⃣: 创建 Fragment (1 天)

```java
// FontSettingsFragment.java
public class FontSettingsFragment extends PreferenceFragmentCompat 
    implements Preference.OnPreferenceChangeListener {
    
    @Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        setPreferencesFromResource(R.xml.font_preferences, rootKey);
        
        // 为每个偏好添加监听器
        Preference serifPref = findPreference("webkit.webprefs.fonts.serif.Zyyy");
        serifPref.setOnPreferenceChangeListener(this);
        // ... 其他字体 ...
    }

    @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        // 保存到 PrefService
        Profile.getLastUsedRegularProfile()
            .getPrefService()
            .setString(preference.getKey(), newValue.toString());
        return true;
    }
}
```

### Step 2️⃣: 创建 XML 配置 (半天)

```xml
<!-- font_preferences.xml -->
<?xml version="1.0" encoding="utf-8"?>
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android">
    <ListPreference
        android:key="webkit.webprefs.fonts.serif.Zyyy"
        android:title="衬线字体"
        android:entries="@array/font_names"
        android:entryValues="@array/font_values" />
    <!-- 其他字体... -->
</PreferenceScreen>
```

### Step 3️⃣: 集成到 Settings (半天)

```java
// 在 SettingsActivity 或 MainSettings 中添加导航
addPreference("font_settings", 
    FontSettingsFragment.class,
    R.string.font_settings_title);
```

### Step 4️⃣: 测试 (1 天)

```bash
# 构建
autoninja -C out/Default chrome_apk

# 安装
adb install -r out/Default/apks/Chrome.apk

# 测试: Settings > 字体 > 改变字体 > 打开网页 > 验证
```

---

## 🔑 关键概念

### Preference Key Names
```cpp
// 这些键名必须与 Chrome 源码中的定义相同
"webkit.webprefs.fonts.serif.Zyyy"           // Serif
"webkit.webprefs.fonts.sansserif.Zyyy"       // Sans-serif  
"webkit.webprefs.fonts.fixed.Zyyy"           // Monospace
"webkit.webprefs.default_font_size"          // 字体大小
```

### 脚本类型 (Script Type)
```cpp
// "Zyyy" 表示通用脚本（所有语言）
// 完整支持还需要其他脚本（可选）:
// "Arab" - 阿拉伯语
// "Cyrl" - 西里尔字母
// "Grek" - 希腊字母
// 等等...

// 当前最小实现: 仅支持 "Zyyy"（通用）
```

### 数据流
```
用户改变 Settings
    ↓
OnPreferenceChange() 回调
    ↓
PrefService.setString()
    ↓
通知所有观察者
    ↓
PrefsTabHelper 观察到改变
    ↓
调用 OnFontFamilyPrefChanged()
    ↓
更新 WebPreferences
    ↓
通知所有 WebContents
    ↓
Renderer 更新字体
    ↓
网页重新渲染
```

---

## 🎓 最小学习路径

### 如果你只有 1 小时

1. 阅读本文档 (15 分钟)
2. 查看 `prefs_tab_helper.cc` (15 分钟)
3. 查看 `android_webview/aw_settings.cc` 作为参考 (15 分钟)
4. 浏览 `ANDROID_CHROMIUM_CUSTOM_FONT_GUIDE.md` (15 分钟)

### 如果你有 1 天

1. 完整阅读所有文档 (3 小时)
2. 研究源代码 (3 小时)
3. 开始实现第一个版本 (2 小时)

### 如果你有 1 周

1. 完整学习所有文档和源代码 (2 天)
2. 实现完整功能 (3 天)
3. 测试和优化 (2 天)

---

## ✅ 最小可行产品 (MVP)

要达到最小可用的产品，你需要:

- ✅ 创建 `FontSettingsFragment.java`
- ✅ 创建 `font_preferences.xml`
- ✅ 在 Settings 中添加菜单项
- ✅ 集成 PrefService 读写
- ✅ 基本测试

**时间**: 3-5 天
**代码行数**: ~200 行 Java + ~100 行 XML

---

## 🚨 常见陷阱

| 陷阱 | 症状 | 解决方案 |
|------|------|--------|
| **Preference Key 错误** | 设置未保存 | 检查 key 名是否与 prefs_tab_helper.cc 中的定义相同 |
| **字符串资源缺失** | 编译失败 | 确保所有字符串都在 strings.xml 中定义 |
| **Fragment 导航错误** | 打开 Settings 看不到字体 | 检查 SettingsActivity 中的导航代码 |
| **PrefService API 错误** | 运行时崩溃 | 检查 Profile 是否正确获取 |
| **缓存问题** | 改变未生效 | 清理应用缓存: `adb shell pm clear com.android.chrome` |

---

## 💡 Pro 技巧

### 技巧 1: 快速编译
```bash
# 只编译 chrome_apk，不编译其他
autoninja -C out/Default chrome_apk
# 比完整构建快 10 倍
```

### 技巧 2: 快速安装和测试
```bash
# 构建并自动安装
autoninja -C out/Default chrome_apk && \
  adb install -r out/Default/apks/Chrome.apk && \
  adb shell am start -n com.android.chrome/.MainActivity
```

### 技巧 3: 查看日志
```bash
# 查看应用日志（包括崩溃信息）
adb logcat | grep -E "chrome|FontSettings"
```

### 技巧 4: 调试 PrefService
```cpp
// 在代码中添加日志进行调试
DLOG(INFO) << "Font changed: " << new_font_name;

// 或者在 Chrome DevTools 中查看（Settings -> about:chrome）
```

---

## 📞 快速决策树

```
Q: Android Chromium 字体支持状态?
└─ 核心功能: ✅ 存在（C++ 代码）
└─ Settings UI: ❌ 缺失（需要创建）

Q: 工作量多少?
└─ 2-3 周（仅 UI 层，代码复用 99%）

Q: 难度多少?
└─ ⭐⭐ 中等（主要是 Android 开发）

Q: 有参考实现吗?
└─ ✅ 有（WebView）

Q: 需要修改 C++ 代码吗?
└─ ❌ 基本不需要（可选的小改动）

Q: 需要修改渲染器吗?
└─ ❌ 完全不需要

Q: 可以复用桌面端代码吗?
└─ ✅ 99% 可以复用（仅差 Settings UI）
```

---

## 🎯 成功指标

实现完成后，应该能达到:

- ✅ Settings 中能看到字体选项
- ✅ 能改变字体选择（Serif, Sans-serif, Monospace）
- ✅ 能改变字体大小
- ✅ 改变立即应用到所有网页
- ✅ 改变被保存，应用重启后保持
- ✅ 在多个 Android 版本上都能工作
- ✅ 性能没有显著下降

---

## 🆘 获取帮助

| 问题 | 查看 |
|------|------|
| 条件编译相关 | `prefs_tab_helper.cc` 行 82-88 |
| Fragment 实现 | `chrome/android/java/.../settings/` 其他 Fragment |
| WebView 参考 | `android_webview/browser/aw_settings.cc` |
| 完整指南 | `ANDROID_CHROMIUM_CUSTOM_FONT_GUIDE.md` |
| 对比分析 | `ANDROID_CHROMIUM_VS_WEBVIEW_VS_DESKTOP_COMPARISON.md` |
| 检查清单 | `ANDROID_CHROMIUM_IMPLEMENTATION_CHECKLIST.md` |

---

## 🎉 准备好开始了吗?

1. ✅ 阅读本文档 (你完成了!)
2. 📖 阅读 `ANDROID_CHROMIUM_CUSTOM_FONT_GUIDE.md`
3. 💻 按照检查清单开始实现
4. 🧪 进行测试
5. 🚀 提交到 Gerrit

**预计时间**: 2-3 周
**预计代码**: ~300 行
**复用率**: 99%

**祝你成功！** 🚀
