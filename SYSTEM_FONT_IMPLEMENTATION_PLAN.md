# Android 系统自定义字体读取实现方案

## 问题概述

目前 Chromium WebView 中的网页字体是硬编码的（"sans-serif", "serif", 等），不会跟随 Android 系统的自定义字体选择。

虽然 `Configuration.fontScale` 可以改变字体大小，但 **自定义字体名称无法被读取**。

---

## 寻找突破口

### 1. Android Configuration 对象支持什么？

**已验证可用**:
- `Configuration.fontScale` - 字体缩放因子 ✅
- `Configuration.orientation` - 屏幕方向
- `Configuration.keyboard` - 键盘类型
- `Configuration.screenWidthDp` - 屏幕宽度
- 等等

**关键问题**: `Configuration` 对象中没有直接的字体名称字段！

---

## 追踪 Chrome UI 的做法

### 查看代码位置

**Chrome Android UI 中的字体使用**:
```java
// ToolbarManager.java 等文件中使用了系统资源
if (mActivity.getResources().getConfiguration()...)
```

但搜索结果显示：**Chrome UI 本身也未实现自定义字体读取**。

这意味着：
1. Chrome UI 本身使用 Android Material Design 字体
2. Android 系统字体改变时，系统自动应用（通过 Resource Configuration）
3. **WebView 中的 Web 内容没有这种自动机制**

---

## 方案：如何获取系统自定义字体？

### 方案 A：TypefaceManager（API 29+）

```java
// Android 10+ 提供的新 API
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    Context context = ...;
    android.graphics.fonts.FontManager fm = 
        context.getSystemService(android.graphics.fonts.FontManager.class);
    
    // 获取系统字体集合
    android.graphics.fonts.Font[] fonts = fm.queryFont(...);
}
```

**问题**: 这是获取字体文件，不是字体名称。

### 方案 B：Settings 数据库查询

```java
// 某些 Android 版本在 Settings.System 中存储字体偏好
String customFont = Settings.System.getString(
    context.getContentResolver(),
    "font_family"  // 或其他键名
);
```

**问题**: 
- 键名因 Android 版本而异
- 不是标准 API，可能不存在

### 方案 C：ResourcesManager 和 Theme

```java
// 通过 Theme 资源查询
Resources resources = context.getResources();
android.content.res.Configuration config = resources.getConfiguration();

// 无直接 API，但可以尝试：
// config.getLocales()
// config.getColorMode()
// 等属性字段
```

**问题**: 没有字体名称字段

### 方案 D：FontsProvider（最接近）

一些 Android 定制版本（如 Samsung、MIUI）提供的系统字体切换功能存储在：
- `/system/fonts/` - 系统字体文件夹
- `/system/etc/fonts.xml` - 字体配置（已检查，无自定义字体信息）
- 系统 Settings 应用私有数据库

---

## 突破性发现：Android 9+ 的 Downloadable Fonts

```java
// 一种可能的标准方式
android.provider.FontsProvider  // Android 9+ 引入

// 通过 ContentProvider 查询可用字体
Cursor cursor = context.getContentResolver().query(
    FontsProvider.FONTS_URI,
    null, null, null, null
);
```

但这通常用于下载字体，不是系统字体。

---

## 现实方案：制造配置变化通知

既然无法直接读取系统自定义字体，我们可以采用 **观察者模式**：

### 步骤 1：监听系统配置变化

```java
// AwSettings.java 中添加

private ComponentCallbacks mComponentCallbacks = new ComponentCallbacks() {
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        // 系统配置改变时调用
        // 包括：语言、字体大小、方向、等
        
        // 尝试重新读取自定义字体
        updateSystemFontSettings();
    }
    
    @Override
    public void onLowMemory() {
    }
};

// 在构造或初始化时注册
mContext.registerComponentCallbacks(mComponentCallbacks);
```

### 步骤 2：添加系统字体读取函数

```java
private void updateSystemFontSettings() {
    // 方案 1：尝试 Samsung 的方式
    String samsungFont = getSamsungFont();
    
    // 方案 2：尝试 MIUI 的方式
    String miuiFont = getMiuiFont();
    
    // 方案 3：尝试设置数据库
    String settingsFont = getSettingsFont();
    
    // 选择第一个有效的
    String customFont = samsungFont != null ? samsungFont :
                        miuiFont != null ? miuiFont :
                        settingsFont != null ? settingsFont : null;
    
    if (customFont != null) {
        mStandardFontFamily = customFont;
        // 触发重新加载
        updateWebkitPreferencesLocked();
    }
}

private String getSamsungFont() {
    // Samsung One UI 存储字体偏好
    try {
        String result = Settings.System.getString(
            mContext.getContentResolver(),
            "sem_font_name"  // Samsung 特定键
        );
        return result;
    } catch (Exception e) {
        return null;
    }
}

private String getMiuiFont() {
    // MIUI 存储字体偏好
    try {
        String result = Settings.System.getString(
            mContext.getContentResolver(),
            "persist.sys.font_name"  // MIUI 特定键
        );
        return result;
    } catch (Exception e) {
        return null;
    }
}

private String getSettingsFont() {
    // 标准 Android 设置（可能存在）
    try {
        String result = Settings.System.getString(
            mContext.getContentResolver(),
            "font_name"  // 通用键
        );
        return result;
    } catch (Exception e) {
        return null;
    }
}
```

### 步骤 3：处理更新触发

```java
// 在 EventHandler 中添加新方法
void updateSystemFontSettingsLocked() {
    runOnUiThreadBlockingAndLocked(
        AwSettings.this::updateSystemFontSettings);
}

// 当检测到配置改变时调用
void onConfigurationChanged(Configuration newConfig) {
    synchronized (mAwSettingsLock) {
        if (newConfig != mLastConfiguration) {
            updateSystemFontSettingsLocked();
            mLastConfiguration = newConfig;
        }
    }
}
```

---

## 完整实现流程

### 文件修改清单

1. **android_webview/java/src/org/chromium/android_webview/AwSettings.java**
   - 添加 `ComponentCallbacks` 实现
   - 添加系统字体读取方法
   - 在构造方法中初始化字体读取
   - 在 `onConfigurationChanged` 时重新读取

2. **android_webview/browser/aw_settings.cc**
   - 创建 native 方法用于触发字体更新
   - 已有 `PopulateWebPreferencesLocked`，无需修改

3. **android_webview/java/src/org/chromium/android_webview/AwSettings.java** (JNI 绑定)
   - 添加新的 JNI getter 方法（如果需要新字体类型）

---

## 关键代码修改建议

### 修改 1：AwSettings.java 构造方法

```java
public AwSettings(
        Context context,
        boolean isAccessFromFileUrlsGrantedByDefault,
        // ... 其他参数
) {
    mContext = context;
    // ... 现有代码
    
    synchronized (mAwSettingsLock) {
        // ... 现有初始化代码
        
        // 新增：读取系统字体
        updateSystemFontSettings();
        
        // 新增：注册配置改变监听
        mContext.registerComponentCallbacks(mComponentCallbacks);
    }
}
```

### 修改 2：添加成员变量

```java
private Configuration mLastConfiguration;

private ComponentCallbacks mComponentCallbacks = new ComponentCallbacks() {
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        synchronized (mAwSettingsLock) {
            if (!newConfig.equals(mLastConfiguration)) {
                mLastConfiguration = new Configuration(newConfig);
                updateSystemFontSettings();
                // 触发 WebKit 更新
                mEventHandler.updateWebkitPreferencesLocked();
            }
        }
    }
    
    @Override
    public void onLowMemory() {}
};
```

### 修改 3：添加字体读取方法

```java
@CalledByNative
private String getStandardFontFamilyLocked() {
    // 在现有 getter 方法中调用 updateSystemFontSettings
    return mStandardFontFamily;
}

private void updateSystemFontSettings() {
    // 尝试多种方式获取系统字体
    String font = tryGetSystemFont();
    if (font != null && !font.isEmpty()) {
        mStandardFontFamily = font;
        // 其他字体族也可能需要更新
    }
}

private String tryGetSystemFont() {
    // 优先级顺序尝试
    String[] fontKeys = {
        "sem_font_name",           // Samsung
        "persist.sys.font_name",   // MIUI
        "font_name",               // Generic
        "system_font",             // Other
    };
    
    for (String key : fontKeys) {
        try {
            String value = Settings.System.getString(
                mContext.getContentResolver(),
                key
            );
            if (value != null && !value.isEmpty()) {
                return value;
            }
        } catch (Exception e) {
            // 忽略异常，继续尝试下一个
        }
    }
    
    return null;
}
```

---

## 测试验证

### 测试场景

1. **初始化测试**
   - WebView 创建时应读取系统字体
   - 如果系统有自定义字体，应使用它

2. **配置更改测试**
   - 在 WebView 运行时改变系统字体
   - 应自动检测并更新网页字体

3. **兼容性测试**
   - 在无自定义字体的系统上：应回退到 "sans-serif"
   - 在不同 Android 版本上测试

4. **性能测试**
   - Settings 查询不应阻塞 UI
   - 配置改变频率不应过高

---

## 已知局限

1. **系统 API 碎片化**
   - 不同厂商（Samsung、MIUI、etc）使用不同的 Settings 键
   - 无法保证所有设备都支持

2. **权限要求**
   - 读取 Settings 通常需要 `READ_SETTINGS` 权限
   - 验证权限检查

3. **性能考虑**
   - Settings 查询可能涉及数据库访问
   - 应在后台线程进行

---

## 总结

**关键实现原理**：
1. 无法直接从 `Configuration` 获取自定义字体名称
2. 需要查询系统 Settings 数据库（因厂商而异）
3. 监听 `onConfigurationChanged` 来检测系统变化
4. 用"优雅降级"的方式处理不支持的系统

**代码改动范围**：
- 主要在 `android_webview/java/src/.../AwSettings.java`
- 添加约 100-150 行代码
- 无需修改 C++ 层（现有机制已支持）

**实施难度**：中等（需要处理多个系统 API）

**预期效果**：
- 支持 Samsung One UI、MIUI 等主流定制系统
- 对标准 Android 提供回退机制
- 用户改变系统字体时，网页内容自动更新
