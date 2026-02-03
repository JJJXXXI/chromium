# Android 端自定义字体功能实现指南

## 📱 Android 架构与桌面端的差异

### 关键差异

| 维度 | 桌面端 (Chrome) | Android (WebView) |
|------|-----------------|-------------------|
| **Settings 来源** | 浏览器设置 App | 应用 App (集成方) |
| **存储位置** | PrefService 本地数据库 | 应用端 SharedPreferences |
| **IPC 路径** | Browser Process ↔ Renderer | Java 层 ↔ C++ 层 (JNI) |
| **字体族支持** | 7 个 (包含脚本感知) | 6 个 (仅 CommonScript) |
| **脚本支持** | 150+ (多语言) | 1 个 (通用) |
| **更新机制** | PrefWatcher 观察者 | JNI 回调 |

### 最重要的差异

```
🖥️  桌面端:
  Chrome Settings UI (TypeScript)
    ↓
  PrefService (C++)
    ↓
  PrefsTabHelper (观察者)
    ↓
  WebPreferences 结构
    ↓
  Renderer 进程

📱 Android:
  应用 App (Java)
    ↓
  SharedPreferences (本地)
    ↓
  WebSettings API (Java)
    ↓
  AwSettings (JNI 桥接)
    ↓
  WebPreferences 结构
    ↓
  Renderer 进程
```

---

## 🔑 Android WebView 的当前实现

### 1. Java 层 API

**文件**: `android_webview/browser/aw_settings.java`

```java
// 当前 Android WebView 的 WebSettings API

public class WebSettings {
    // 获取字体
    public String getStandardFontFamily() { }
    public String getFixedFontFamily() { }
    public String getSansSerifFontFamily() { }
    public String getSerifFontFamily() { }
    public String getCursiveFontFamily() { }
    public String getFantasyFontFamily() { }
    
    // 设置字体
    public void setStandardFontFamily(String family) { }
    public void setFixedFontFamily(String family) { }
    public void setSansSerifFontFamily(String family) { }
    public void setSerifFontFamily(String family) { }
    public void setCursiveFontFamily(String family) { }
    public void setFantasyFontFamily(String family) { }
    
    // 字体大小
    public int getDefaultFontSize() { }
    public void setDefaultFontSize(int size) { }
    public int getDefaultFixedFontSize() { }
    public void setDefaultFixedFontSize(int size) { }
    public int getMinimumFontSize() { }
    public void setMinimumFontSize(int size) { }
}
```

### 2. C++ JNI 层 (AwSettings)

**文件**: `android_webview/browser/aw_settings.cc` (行 567-650)

```cpp
// 重要代码片段（当前实现）

void AwSettings::PopulateWebPreferencesLocked(JNIEnv* env,
                                              const JavaRef<jobject>& obj,
                                              jlong web_prefs_ptr) {
  WebPreferences* web_prefs = reinterpret_cast<WebPreferences*>(web_prefs_ptr);
  
  // ⚠️ 关键发现：只支持 CommonScript (通用脚本)！
  // 这是 Android WebView 与桌面端的主要区别
  
  // 设置字体族映射
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

  // 字体大小设置
  web_prefs->default_font_size =
      Java_AwSettings_getDefaultFontSizeLocked(env, obj);

  web_prefs->default_fixed_font_size =
      Java_AwSettings_getDefaultFixedFontSizeLocked(env, obj);

  web_prefs->minimum_font_size =
      Java_AwSettings_getMinimumFontSizeLocked(env, obj);

  web_prefs->minimum_logical_font_size =
      Java_AwSettings_getMinimumLogicalFontSizeLocked(env, obj);
}
```

---

## 🎯 Android 端实现路线

### 方案 A: 基础方案 (推荐) ⭐⭐⭐

**目标**: 为应用提供编程 API 让应用控制字体

**实现步骤**:

```
1️⃣  扩展 Java WebSettings API
    ├─ 添加 setStandardFontFamily(String family) API
    ├─ 对应桌面端的偏好系统
    └─ 应用调用这些 API 控制字体

2️⃣  C++ JNI 层扩展
    ├─ 添加 Java_WebSettings_setStandardFontFamilyLocked()
    ├─ 调用 PopulateWebPreferences()
    └─ 应用字体变更到 WebPreferences

3️⃣  字体变更通知
    ├─ 调用 web_contents()->OnWebPreferencesChanged()
    ├─ 通知渲染器进程
    └─ 页面重新渲染

4️⃣  测试和验证
    ├─ 单元测试
    ├─ 集成测试
    └─ 真机测试
```

**代码示例**:

```java
// Java 层 - 应用调用这个 API
WebView webview = ...;
WebSettings settings = webview.getSettings();

// 设置字体
settings.setSerifFontFamily("Georgia");
settings.setSansSerifFontFamily("Arial");
settings.setFixedFontFamily("Courier New");

// 字体大小
settings.setDefaultFontSize(16);
```

**优点**:
- ✅ 对标桌面端设计
- ✅ 应用完全可控
- ✅ 简单直接

**缺点**:
- ❌ 没有多语言支持 (仅 CommonScript)
- ❌ 应用需要自己存储偏好

---

### 方案 B: 增强方案 (中等难度)

**目标**: 支持多语言脚本的字体选择

**核心改动**: 支持多脚本的字体映射

```cpp
// C++ 层修改
void AwSettings::SetFontFamily(
    JNIEnv* env,
    const JavaRef<jobject>& obj,
    const std::string& generic_family,      // "serif"
    const std::string& script_code,         // "Hans"
    const std::string& font_family) {       // "宋体"
  
  WebPreferences* web_prefs = ...;
  
  // 类似桌面端的 OverrideFontFamily
  if (generic_family == "serif") {
    web_prefs->serif_font_family_map[script_code] = 
        ConvertUTF8ToUTF16(font_family);
  }
  // ... 其他通用族 ...
  
  web_contents()->OnWebPreferencesChanged();
}
```

**Java API**:
```java
// Java 层 API
settings.setFontFamily("serif", "Hans", "宋体");      // 简中 serif
settings.setFontFamily("serif", "Hant", "微软雅黑");  // 繁中 serif
settings.setFontFamily("serif", "Latn", "Georgia");    // 拉丁 serif
```

**优点**:
- ✅ 支持多语言
- ✅ 对齐桌面端功能
- ✅ 中文等 CJK 用户受益最大

**缺点**:
- ⚠️ 实现复杂度中等
- ⚠️ 需要 ICU 脚本代码支持
- ⚠️ 需要修改 Java API 签名

---

### 方案 C: 完整方案 (难度高)

**目标**: 集成 SharedPreferences 持久化存储

**架构**:

```
应用 App
  ↓ (SharedPreferences)
  ├─ "font_serif_Hans" = "宋体"
  ├─ "font_serif_Latn" = "Georgia"
  └─ "font_size_default" = "16"
  
  ↓ (读取)
  
AwSettings (JNI 桥接)
  ↓ (应用到)
  
WebPreferences
  ↓ (发送到)
  
Renderer 进程
```

**Java 层**:

```java
// 应用的 SharedPreferences
SharedPreferences prefs = context.getSharedPreferences("fonts", Context.MODE_PRIVATE);

// 保存用户选择
prefs.edit()
    .putString("font_serif_Hans", "宋体")
    .putString("font_serif_Latn", "Georgia")
    .putInt("font_size_default", 16)
    .apply();

// 初始化 WebView
WebView webview = ...;
WebSettings settings = webview.getSettings();

// 从存储恢复
String serifFontCN = prefs.getString("font_serif_Hans", "宋体");
settings.setFontFamily("serif", "Hans", serifFontCN);
```

**C++ 层**:

```cpp
// AwSettings 从 Java 端读取 SharedPreferences
void AwSettings::LoadFontsFromPreferences(JNIEnv* env,
                                          const JavaRef<jobject>& obj) {
  // 调用 Java 函数获取存储的字体偏好
  ScopedJavaLocalRef<jobject> prefs = 
      Java_AwSettings_loadFromSharedPreferences(env, obj);
  
  // 应用到 WebPreferences
  ApplyFontsFromPreferences(prefs);
}
```

**优点**:
- ✅ 完整的持久化存储
- ✅ 应用重启后保留设置
- ✅ 支持多语言
- ✅ 最接近桌面端体验

**缺点**:
- ⚠️ 实现复杂度高
- ⚠️ 需要权限管理
- ⚠️ 需要数据迁移考虑

---

## 📋 参考点清单

### 1. 从桌面端学习

#### 参考代码
```
桌面端                          Android WebView
─────────────────────────────────────────────────
prefs_tab_helper.cc         →  aw_settings.cc
web_preferences.h           →  web_preferences.h (共享)
PrefsTabHelper::OnWebPrefChanged → AwSettings::PopulateWebPreferences
OverrideFontFamily()        →  (需要实现)
GetWebContents().OnWebPreferencesChanged() → (现有支持)
```

#### 学习路径
1. 研究 `PrefsTabHelper::OverrideFontFamily()` 的逻辑
2. 在 Android 端复现相同的逻辑
3. 用 JNI 将 Java 调用桥接到 C++

### 2. 现有 Android WebView 代码

**关键文件**:
- `android_webview/browser/aw_settings.cc` - 主要实现
- `android_webview/browser/aw_settings.h` - 头文件
- `android_webview/glue/aw_contents.cc` - WebView 集成

**关键函数**:
```cpp
// 当前支持的 WebPreferences 设置
AwSettings::PopulateWebPreferencesLocked()

// 调用触发点
web_contents()->OnWebPreferencesChanged()
```

### 3. JNI 桥接模式

**模式分析**:

```cpp
// Java 层设置 → C++ 层应用的标准模式

// 1. Java 调用 JNI 函数
Java_AwSettings_setStandardFontFamily(env, obj, fontFamily);

// 2. C++ 端 JNI 函数接收
static void SetStandardFontFamily(JNIEnv* env,
                                   const JavaRef<jobject>& obj,
                                   const JavaRef<jstring>& family) {
  // 3. 转换 Java 字符串到 C++
  std::u16string font = ConvertJavaStringToUTF16(family);
  
  // 4. 应用到 WebPreferences
  WebPreferences* web_prefs = GetWebPreferences();
  web_prefs->standard_font_family_map[kCommonScript] = font;
  
  // 5. 通知渲染器
  web_contents()->OnWebPreferencesChanged();
}
```

### 4. 字体申请和权限

**Android 特定考虑**:

```java
// 访问系统字体列表
// 需要列出设备上可用的字体

public static List<String> getAvailableFonts(Context context) {
  // 方法 1: 遍历 /system/fonts/
  // 方法 2: 使用 Typeface.create() 检测
  // 方法 3: 使用系统 API (API 29+)
}
```

### 5. 测试策略

```java
// 单元测试
public class AwSettingsFontTest {
  @Test
  public void testSetSerifFont() {
    WebView webview = createTestWebView();
    WebSettings settings = webview.getSettings();
    
    settings.setSerifFontFamily("Georgia");
    
    // 验证 WebPreferences 已更新
    WebPreferences prefs = webview.getWebPreferences();
    assertEquals("Georgia", 
        prefs.serif_font_family_map.get("Zyyy"));
  }
}

// 集成测试 - 实际渲染
public class FontRenderingTest {
  @Test
  public void testSerifFontRenderingAndroid() {
    WebView webview = ...;
    webview.getSettings().setSerifFontFamily("Georgia");
    
    // 加载包含 CSS "font-family: serif" 的网页
    webview.loadUrl("file:///test_serif.html");
    
    // 捕获渲染结果
    // 使用 OCR 或图片对比验证字体
  }
}
```

---

## 🔧 Android 端的具体实现步骤

### 第 1 步: 修改 Java WebSettings API

**文件**: `android_webview/public/android_webview_java.gni` 和相关 Java 文件

```java
// 添加新的 setter 方法（如果不存在）
public void setFontFamilyByScript(String genericFamily, 
                                   String script, 
                                   String fontFamily) {
  // JNI 调用
  mAwSettings.setFontFamilyByScript(genericFamily, script, fontFamily);
}
```

### 第 2 步: 添加 JNI 实现

**文件**: `android_webview/browser/aw_settings.cc`

```cpp
// 在 AwSettings 类中添加新的公共方法

void AwSettings::SetFontFamilyByScript(
    const std::string& generic_family,
    const std::string& script,
    const std::u16string& font_family) {
  
  // 获取 WebPreferences
  WebPreferences* web_prefs = GetWebPreferences();
  if (!web_prefs) return;
  
  // 类似桌面端的 OverrideFontFamily 逻辑
  ScriptFontFamilyMap* map = nullptr;
  
  if (generic_family == "serif") {
    map = &web_prefs->serif_font_family_map;
  } else if (generic_family == "sansserif") {
    map = &web_prefs->sans_serif_font_family_map;
  } else if (generic_family == "fixed") {
    map = &web_prefs->fixed_font_family_map;
  }
  // ... 其他族 ...
  
  if (map) {
    (*map)[script] = font_family;
  }
  
  // 通知 WebContents
  web_contents()->OnWebPreferencesChanged();
}
```

### 第 3 步: 注册 JNI 函数

**文件**: `android_webview/browser/aw_settings.h`

```cpp
// 在头文件中声明

// static 方法用于 JNI 回调
static void SetFontFamilyByScriptJNI(JNIEnv* env,
                                      const JavaRef<jobject>& obj,
                                      const JavaRef<jstring>& generic_family,
                                      const JavaRef<jstring>& script,
                                      const JavaRef<jstring>& font_family);
```

### 第 4 步: 编译和测试

```bash
# 编译 Android WebView
autoninja -C out/android android_webview_apk

# 运行测试
autoninja -C out/android android_webview_unittests
```

---

## 📊 Android vs 桌面端功能对比

| 功能 | 桌面端 | Android (当前) | Android (增强) |
|------|--------|----------------|----------------|
| 通用字体族数 | 7 | 6 | 6-7 |
| 脚本支持 | 150+ (ICU) | 1 (CommonScript) | 150+ (ICU) |
| 存储位置 | PrefService | SharedPreferences | SharedPreferences |
| 多进程支持 | ✅ Browser + Renderer | ✅ 应用 + Renderer | ✅ 应用 + Renderer |
| 热更新 | ✅ | ✅ | ✅ |
| 用户 UI | Chrome Settings | 应用自己实现 | 应用自己实现 |
| 多语言 | ✅ | ⚠️ 有限 | ✅ |

---

## 🎯 推荐实现方案

### 最小可行版本 (MVP)

**优先级**:
1. 方案 A 的基础部分 (编程 API)
2. 支持桌面 Chromium 已有的 6 个字体族
3. 仅支持 CommonScript (通用脚本)

**工作量**: ~2-3 天

**代码示例**:
```java
// 应用可以这样使用
WebSettings settings = webView.getSettings();
settings.setSerifFontFamily("Georgia");
settings.setSansSerifFontFamily("Arial");
```

### 增强版本 (推荐后续)

**增加**:
1. 多脚本支持 (需要 ICU 脚本代码)
2. SharedPreferences 持久化
3. 字体变更通知 listener

**工作量**: ~1-2 周

**完整的多语言支持**:
```java
// 中文用户
settings.setFontFamily("serif", "Hans", "宋体");

// 日本用户
settings.setFontFamily("serif", "Jpan", "游明朝");

// 英文用户
settings.setFontFamily("serif", "Latn", "Georgia");
```

---

## 🚀 立即开始

### 参考的具体源文件

1. **学习 PrefsTabHelper**
   ```
   chrome/browser/ui/prefs/prefs_tab_helper.cc (行 260-285)
   └─ OverrideFontFamily() 函数
   ```

2. **参考 AwSettings 现有实现**
   ```
   android_webview/browser/aw_settings.cc (行 567-650)
   └─ PopulateWebPreferencesLocked() 函数
   ```

3. **参考 WebPreferences 结构**
   ```
   third_party/blink/public/common/web_preferences/web_preferences.h
   └─ 7 个 ScriptFontFamilyMap 定义
   ```

4. **查看 JNI 桥接模式**
   ```
   android_webview/browser/aw_settings.cc (前 100 行)
   └─ Java_AwSettings_* JNI 函数模板
   ```

### 实现顺序

```
第 1 步: 研究阶段 (1 天)
  ├─ 阅读 PrefsTabHelper 的 OverrideFontFamily
  ├─ 研究 AwSettings 的 PopulateWebPreferencesLocked
  └─ 理解 JNI 函数的模板

第 2 步: 设计阶段 (1 天)
  ├─ 设计 Java API
  ├─ 设计 C++ 实现
  └─ 确定 JNI 函数签名

第 3 步: 实现阶段 (3-5 天)
  ├─ 实现 Java API
  ├─ 实现 JNI 函数
  ├─ 实现 C++ 逻辑
  └─ 链接所有部分

第 4 步: 测试阶段 (3-5 天)
  ├─ 单元测试
  ├─ 集成测试
  ├─ 真机测试
  └─ 性能测试

总计: 8-15 天
```

---

## 💡 关键洞察

### 1. Android WebView 已经有基础设施

✅ WebPreferences 结构已存在  
✅ PopulateWebPreferencesLocked() 已存在  
✅ OnWebPreferencesChanged() 已有支持  

**你需要做的**: 只是**扩展现有的 API 和逻辑**

### 2. 借鉴桌面端的 OverrideFontFamily()

```cpp
// 桌面端 (prefs_tab_helper.cc)
void OverrideFontFamily(...) {
  ScriptFontFamilyMap* map = nullptr;
  if (generic_family == "serif") {
    map = &prefs->serif_font_family_map;
  }
  if (map) {
    (*map)[script] = font_name;
  }
}

// Android 端 (你的实现)
void AwSettings::SetFontFamilyByScript(...) {
  // 完全相同的逻辑！
  ScriptFontFamilyMap* map = nullptr;
  if (generic_family == "serif") {
    map = &web_prefs->serif_font_family_map;
  }
  if (map) {
    (*map)[script] = font_name;
  }
  web_contents()->OnWebPreferencesChanged();
}
```

### 3. Java-C++ 通信很简单

Android WebView 已经使用 JNI 大量通信：

```cpp
// 模板: 从 Java 调用 C++

// 1. Java 层
settings.setSerifFontFamily("Georgia");

// 2. 触发 JNI 调用
Native.setSerifFontFamily("Georgia");

// 3. C++ 端接收
void Java_WebSettings_setSerifFontFamily(JNIEnv* env, ...) {
  // 4. 应用逻辑
  web_prefs->serif_font_family_map[kCommonScript] = "Georgia";
  web_contents()->OnWebPreferencesChanged();
}
```

---

## 🎓 总结

**可以参考的核心点**:

1. ✅ **架构参考**: 桌面端的多进程设计
2. ✅ **代码参考**: PrefsTabHelper 的 OverrideFontFamily 函数
3. ✅ **基础设施**: Android WebView 已有 WebPreferences 支持
4. ✅ **实现模式**: 现有的 JNI 函数模板
5. ✅ **集成点**: web_contents()->OnWebPreferencesChanged()

**推荐方案**: 
- 从方案 A (编程 API) 开始
- 逐步过渡到方案 B (多脚本支持)
- 完整的用户体验需要应用方实现 UI

**工作量**: 基础版 2-3 天，增强版 1-2 周

**难度**: ⭐⭐ (中等) - 主要是 JNI 和数据结构的理解
