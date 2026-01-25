# Chromium 多线程和任务队列完全指南

> 从零到精通 Chromium 的 Threading、TaskQueue、Sequence 机制

---

## 概览：为什么需要复杂的线程管理？

### Chromium 的多进程架构

```
Browser Process (主进程)
├─ Browser Thread (UI 线程)
├─ IO Thread
├─ File Thread
├─ Worker Threads (ThreadPool)
└─ GPU Thread

Renderer Process (渲染进程)
├─ Main Thread (Blink)
├─ Compositor Thread
├─ Worker Threads
└─ Service Threads

Network Service Process
├─ Network IO Thread
└─ Worker Threads
```

### 核心问题

1. **线程不安全** - 多个线程同时访问数据会崩溃
2. **性能** - 需要合理分配任务到多个线程
3. **顺序保证** - 某些任务必须按顺序执行
4. **死锁** - 不小心会造成线程互相等待

**解决方案**: Chromium 的 TaskQueue + ThreadPool + Sequence 机制

---

## 核心概念

### 1. Task（任务）

```cpp
// base/task/task_traits.h
// 任务 = 一段代码 + 元数据

// 简单任务
base::OnceClosure task = base::BindOnce(&MyFunction, arg1, arg2);

// 任务元数据
base::TaskTraits traits = {
  base::TaskPriority::USER_BLOCKING,  // 优先级
  base::TaskShutdownBehavior::BLOCK_SHUTDOWN,  // 关闭行为
};
```

### 2. Sequence（序列）

```
Sequence = 任务队列
     ↓
保证队列中的任务按顺序执行

Example:
Sequence A: [Task1] → [Task2] → [Task3]
            (执行完)   (执行完)   (执行中)
            
多个序列可以在不同线程并行：
Sequence A: [Task1] → [Task2] (Thread 1)
Sequence B: [TaskA] → [TaskB] (Thread 2)  ← 并行执行
```

### 3. Thread（线程）

```
Thread = 操作系统线程

Chromium 中的标准线程：
├─ UI Thread (Browser process)
│   └─ 处理用户交互、窗口更新
│
├─ IO Thread (Browser process)
│   └─ 处理网络 I/O、文件 I/O
│
├─ Compositor Thread (Renderer process)
│   └─ 处理合成和 GPU 操作
│
└─ ThreadPool (所有 process)
    └─ 通用工作线程（可扩展）
```

### 4. TaskRunner（任务运行器）

```cpp
// base/task/task_runner.h
// TaskRunner = 在某个线程上执行任务的接口

// SingleThreadTaskRunner: 保证任务在单个线程执行
scoped_refptr<SingleThreadTaskRunner> runner = 
    base::CreateSingleThreadTaskRunner({...});

// ThreadPool: 在线程池中执行任务
base::ThreadPool::PostTask(FROM_HERE, task);
```

---

## 核心 API

### Part 1: 在 UI 线程执行

```cpp
// === 模式 1: 在当前线程执行 ===
GetUIThreadTaskRunner({})->PostTask(FROM_HERE, base::BindOnce([]() {
  LOG(INFO) << "在 UI 线程执行";
}));

// === 模式 2: 在主线程执行（Browser process） ===
content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE, 
    base::BindOnce(&MyClass::OnUIThread, weak_this));

// === 模式 3: 延迟执行 ===
GetUIThreadTaskRunner({})->PostDelayedTask(FROM_HERE,
    base::BindOnce(&MyClass::DoSomething, weak_this),
    base::Milliseconds(500));
```

### Part 2: 在 IO 线程执行

```cpp
// Browser process 中的 IO 线程
content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
    base::BindOnce(&NetworkOperation, url));
```

### Part 3: 在 ThreadPool 中执行

```cpp
// 最简单的方式 - 立即执行
base::ThreadPool::PostTask(FROM_HERE, base::BindOnce([]() {
  LOG(INFO) << "在线程池中执行";
}));

// 带优先级
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskPriority::USER_BLOCKING},  // 高优先级
    base::BindOnce(&ExpensiveWork));

// 返回 Future（等待结果）
base::ThreadPool::PostTaskAndReplyWithResult(
    FROM_HERE,
    base::BindOnce(&ComputeResult),
    base::BindOnce(&OnResultReady));
```

### Part 4: 创建自己的 Sequence

```cpp
// 文件: my_worker.h
class MyWorker {
 public:
  MyWorker() {
    sequence_ = base::ThreadPool::CreateSequencedTaskRunner({});
  }
  
  void DoWork() {
    // 这些任务会在同一个 Sequence 上执行（保证顺序）
    sequence_->PostTask(FROM_HERE, base::BindOnce(&MyWorker::Task1, this));
    sequence_->PostTask(FROM_HERE, base::BindOnce(&MyWorker::Task2, this));
    sequence_->PostTask(FROM_HERE, base::BindOnce(&MyWorker::Task3, this));
  }
  
 private:
  scoped_refptr<SequencedTaskRunner> sequence_;
};

// 执行顺序保证：
// Task1 → Task2 → Task3
// (顺序执行，可能在不同线程)
```

---

## 实战场景 1: 网络请求 + UI 更新

### 问题
```
网络请求很慢，如果在 UI 线程做会卡死界面
需要：
1. 在后台线程做网络请求
2. 结果返回后在 UI 线程更新界面
```

### 错误的做法 ❌

```cpp
class NetworkFetcher {
 public:
  void Fetch() {
    // ❌ 直接在 UI 线程做网络操作
    std::string data = FetchFromNetwork();  // 阻塞 UI！
    UpdateUI(data);
  }
};
```

### 正确的做法 ✅

```cpp
class NetworkFetcher : public base::RefCounted<NetworkFetcher> {
 public:
  void Fetch() {
    // 1. 在线程池中做网络请求
    base::ThreadPool::PostTaskAndReplyWithResult(
        FROM_HERE,
        {base::TaskPriority::USER_VISIBLE},
        base::BindOnce(&NetworkFetcher::FetchInBackground, this),
        base::BindOnce(&NetworkFetcher::OnFetchComplete, this));
  }
  
 private:
  // 在后台线程执行
  std::string FetchInBackground() {
    return FetchFromNetwork();  // 可以阻塞，不在 UI 线程
  }
  
  // 在 UI 线程执行
  void OnFetchComplete(std::string result) {
    UpdateUI(result);  // 安全地更新 UI
  }
};
```

### 完整例子：网络请求 + 数据处理 + UI 更新

```cpp
// file: network_manager.cc

class NetworkManager : public base::RefCounted<NetworkManager> {
 public:
  void FetchUserProfile(const std::string& user_id) {
    // 任务 1: 网络请求（在 ThreadPool 中）
    base::ThreadPool::PostTaskAndReplyWithResult(
        FROM_HERE,
        {base::TaskPriority::USER_VISIBLE},
        base::BindOnce(&NetworkManager::FetchFromServer, this, user_id),
        base::BindOnce(&NetworkManager::OnNetworkDataReady, this));
  }
  
 private:
  // ========== ThreadPool 执行 ==========
  std::string FetchFromServer(const std::string& user_id) {
    // 这可能需要 1-5 秒
    // 在 ThreadPool 中执行，不会阻塞 UI
    DLOG(INFO) << "正在从服务器获取数据...";
    
    std::string raw_data = DoHttpRequest(
        "https://api.example.com/user/" + user_id);
    return raw_data;  // 返回原始数据
  }
  
  // ========== ThreadPool 执行 ==========
  // 如果需要多步处理，使用 Sequence 保证顺序
  void ParseData(const std::string& raw_data) {
    // 耗时的 JSON 解析
    parsed_profile_ = ParseJSON(raw_data);
  }
  
  void ValidateData() {
    // 数据验证
    if (!IsValidProfile(parsed_profile_)) {
      DLOG(ERROR) << "数据无效";
      return;
    }
  }
  
  // ========== UI 线程执行 ==========
  void OnNetworkDataReady(std::string raw_data) {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    
    // 数据已经准备好，现在可以安全地更新 UI
    UpdateUIWithProfile(raw_data);
    
    // 通知其他 UI 组件
    NotifyListeners();
  }
};
```

---

## 实战场景 2: 文件 I/O 不能阻塞任何线程

### 问题
```
文件 I/O 很慢，但需要保证操作顺序
（比如：写入文件 → 读取文件 → 验证）
```

### 解决方案：使用 Sequence

```cpp
class FileManager : public base::RefCounted<FileManager> {
 public:
  FileManager() {
    // 创建一个序列（保证顺序）
    file_sequence_ = base::ThreadPool::CreateSequencedTaskRunner({});
  }
  
  void SaveAndVerify(const std::string& filename, 
                     const std::string& content) {
    // 任务 1: 写入文件（在 file_sequence 上）
    file_sequence_->PostTask(
        FROM_HERE,
        base::BindOnce(&FileManager::WriteFile, this, filename, content));
    
    // 任务 2: 读取文件（在 file_sequence 上）
    // 保证在 WriteFile 完成后执行
    file_sequence_->PostTask(
        FROM_HERE,
        base::BindOnce(&FileManager::ReadFile, this, filename));
    
    // 任务 3: 验证（在 file_sequence 上）
    // 保证在 ReadFile 完成后执行
    file_sequence_->PostTask(
        FROM_HERE,
        base::BindOnce(&FileManager::Verify, this));
  }
  
 private:
  scoped_refptr<SequencedTaskRunner> file_sequence_;
  
  // 这些都在同一个序列中执行，保证顺序
  void WriteFile(const std::string& filename, const std::string& content) {
    DLOG(INFO) << "写入文件: " << filename;
    base::WriteFile(base::FilePath(filename), content.data(), 
                    content.size());
  }
  
  void ReadFile(const std::string& filename) {
    DLOG(INFO) << "读取文件: " << filename;
    cached_content_ = base::ReadFileToString(
        base::FilePath(filename));
  }
  
  void Verify() {
    DLOG(INFO) << "验证: " << cached_content_.size() << " 字节";
  }
  
  std::string cached_content_;
};
```

### 执行流程

```
Timeline:
T=0ms    WriteFile(task)          [ ]
         ReadFile(task)           [ ]
         Verify(task)             [ ]

T=10ms   WriteFile execute        [████████]
         (文件写入中...)

T=100ms  WriteFile complete       [████████] ✓
         ReadFile execute         ────[████] (等待中...)
         
T=120ms  ReadFile execute         [████████]
         Verify execute           ────────────[██]

T=130ms  ReadFile complete        [████████] ✓
         Verify execute           [████]

T=140ms  Verify complete          [████████] ✓
         
保证顺序：Write → Read → Verify （不会乱序）
```

---

## 实战场景 3: Browser Thread 间通信

### 问题
```
Browser 中有多个线程：UI、IO、File...
需要在不同线程之间安全通信
```

### 场景：UI 线程发送请求到 IO 线程

```cpp
// ============ UI 线程上的代码 ============
class ProfileManager {
 public:
  void OnUserClicked() {
    // 在 UI 线程中，需要在 IO 线程做网络操作
    content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
        base::BindOnce(&NetworkService::SendRequest,
                       base::Unretained(network_service_.get()),
                       url));
  }
  
 private:
  std::unique_ptr<NetworkService> network_service_;
};

// ============ IO 线程上的代码 ============
class NetworkService {
 public:
  void SendRequest(const std::string& url) {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::IO));
    
    // 做网络操作...
    std::string response = PerformNetworkIO(url);
    
    // 结果返回给 UI 线程
    content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE,
        base::BindOnce(&ProfileManager::OnNetworkResponse,
                       base::Unretained(ui_manager_.get()),
                       response));
  }
  
 private:
  ProfileManager* ui_manager_;
};
```

### 关键点：base::Unretained vs weak_ptr

```cpp
// ❌ 危险：如果对象被删除会崩溃
content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
    base::BindOnce(&MyClass::Method, base::Unretained(this)));
// 如果 this 被删除，执行时会崩溃！

// ✅ 安全：如果对象被删除，任务不执行
content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE,
    base::BindOnce(&MyClass::Method, weak_this));
// weak_this 失效 → 任务自动取消
```

### 模式：使用 WeakPtr 做跨线程回调

```cpp
class UIUpdater {
 public:
  UIUpdater() : weak_factory_(this) {}
  
  void RequestDataFromIO() {
    // 方法 1: 立即发送任务到 IO 线程
    content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
        base::BindOnce(&IOWorker::FetchData,
                       base::Unretained(io_worker_.get())));
    
    // 方法 2: IO 线程完成后回调（使用 WeakPtr）
    // 如果 UIUpdater 被删除，回调不执行
    content::GetIOThreadTaskRunner({})->PostTaskAndReply(FROM_HERE,
        base::BindOnce(&IOWorker::FetchData,
                       base::Unretained(io_worker_.get())),
        base::BindOnce(&UIUpdater::OnDataReady,
                       weak_factory_.GetWeakPtr()));
  }
  
 private:
  void OnDataReady() {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    UpdateUI();
  }
  
  std::unique_ptr<IOWorker> io_worker_;
  base::WeakPtrFactory<UIUpdater> weak_factory_;
};
```

---

## 线程安全的数据访问

### 问题 1: 竞态条件（Race Condition）

```cpp
class BadCounter {
 private:
  int count_ = 0;
  
  void Increment() {
    count_++;  // ❌ 多个线程同时调用会出错
  }
};

// 示例：
// Thread 1: count_ = 0 → read 0 → increment → write 1
// Thread 2: count_ = 0 → read 0 → increment → write 1
// 结果: count_ = 1 (应该是 2) ❌
```

### 解决方案 1: 使用 Sequence（单线程化）

```cpp
class Counter {
 public:
  Counter() {
    counter_sequence_ = base::ThreadPool::CreateSequencedTaskRunner({});
  }
  
  void Increment() {
    // 所有 Increment 调用都在同一个序列上
    counter_sequence_->PostTask(FROM_HERE,
        base::BindOnce(&Counter::IncrementOnSequence, this));
  }
  
  void GetValue(base::OnceCallback<void(int)> callback) {
    counter_sequence_->PostTaskAndReplyWithResult(FROM_HERE,
        base::BindOnce(&Counter::GetValueOnSequence, this),
        std::move(callback));
  }
  
 private:
  void IncrementOnSequence() {
    // 这里是单线程，没有竞态条件
    count_++;
  }
  
  int GetValueOnSequence() {
    return count_;
  }
  
  int count_ = 0;
  scoped_refptr<SequencedTaskRunner> counter_sequence_;
};
```

### 解决方案 2: 使用 Lock（互斥锁）

```cpp
class ThreadSafeCounter {
 public:
  void Increment() {
    base::AutoLock lock(lock_);  // 获取锁
    count_++;
    // 自动释放锁
  }
  
  int GetValue() {
    base::AutoLock lock(lock_);
    return count_;
  }
  
 private:
  int count_ = 0;
  base::Lock lock_;
};
```

### 解决方案 3: Atomic（原子操作）

```cpp
class AtomicCounter {
 public:
  void Increment() {
    count_.fetch_add(1, std::memory_order_relaxed);
  }
  
  int GetValue() {
    return count_.load(std::memory_order_relaxed);
  }
  
 private:
  std::atomic<int> count_{0};
};
```

---

## 任务优先级

### TaskPriority 的选择

```cpp
enum class TaskPriority {
  // 1. USER_BLOCKING - 用户正在等待（最高优先级）
  // 用途: 页面加载、用户交互的直接响应
  base::TaskPriority::USER_BLOCKING,
  
  // 2. USER_VISIBLE - 用户能看到结果（中等优先级）
  // 用途: 页面内容渲染、数据更新
  base::TaskPriority::USER_VISIBLE,
  
  // 3. BEST_EFFORT - 最好尽快做（低优先级）
  // 用途: 日志、分析、清理
  base::TaskPriority::BEST_EFFORT,
};
```

### 实例

```cpp
// 页面加载 - 最高优先级
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskPriority::USER_BLOCKING},
    base::BindOnce(&RenderEngine::RenderFrame, frame));

// 数据刷新 - 中等优先级
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskPriority::USER_VISIBLE},
    base::BindOnce(&DataSync::RefreshData, sync));

// 发送分析数据 - 低优先级
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskPriority::BEST_EFFORT},
    base::BindOnce(&Analytics::SendMetrics, analytics));
```

---

## 常见的坑

### 坑 1: 忘记检查线程

```cpp
// ❌ 错误：没有检查当前线程
void SaveUserData(const UserData& data) {
  database_->Save(data);  // 如果在 UI 线程会阻塞！
}

// ✅ 正确：检查并在正确线程执行
void SaveUserData(const UserData& data) {
  if (content::BrowserThread::CurrentlyOn(
          content::BrowserThread::UI)) {
    // 错误的线程，转移到 IO 线程
    content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
        base::BindOnce(&SaveUserDataImpl, data));
  } else {
    SaveUserDataImpl(data);
  }
}
```

### 坑 2: 对象在任务执行前被删除

```cpp
// ❌ 崩溃风险
class Worker {
  void DoWork() {
    base::ThreadPool::PostTask(FROM_HERE,
        base::BindOnce(&Worker::Work, base::Unretained(this)));
  }
};

Worker* w = new Worker();
w->DoWork();
delete w;  // ❌ 任务还没执行，对象已删除！

// ✅ 使用 WeakPtr
class Worker {
  void DoWork() {
    base::ThreadPool::PostTask(FROM_HERE,
        base::BindOnce(&Worker::Work, weak_factory_.GetWeakPtr()));
  }
  
  base::WeakPtrFactory<Worker> weak_factory_{this};
};
```

### 坑 3: 死锁

```cpp
// ❌ 死锁
void Function1() {
  base::Lock lock(mutex_);
  
  // 等待其他线程...
  event_.Wait();  // ← 等待中，但持有 mutex_
}

void Function2() {
  // 想要获取 mutex_，但 Function1 持有
  base::AutoLock lock(mutex_);  // ← 永远无法获得
}

// ✅ 释放锁再等待
void Function1() {
  {
    base::Lock lock(mutex_);
    // 做点事
  }  // ← 释放锁
  
  event_.Wait();  // ← 现在可以等待
}
```

### 坑 4: PostTask 没有立即执行

```cpp
// ❌ 期望立即执行
int result = 0;
base::ThreadPool::PostTask(FROM_HERE,
    base::BindOnce([&result]() { result = 42; }));
LOG(INFO) << result;  // 输出 0，不是 42！

// ✅ 使用 PostTaskAndReplyWithResult
int result = 0;
base::ThreadPool::PostTaskAndReplyWithResult(FROM_HERE,
    base::BindOnce([]() { return 42; }),
    base::BindOnce([&result](int value) { result = value; }));
LOG(INFO) << result;  // 输出 42
```

---

## TaskShutdownBehavior（关闭行为）

### 应用场景

```cpp
enum class TaskShutdownBehavior {
  // 1. CONTINUE_ON_SHUTDOWN - 继续执行（即使应用关闭）
  // 用途: 不重要的清理任务
  
  // 2. SKIP_ON_SHUTDOWN - 跳过（不执行）
  // 用途: 不重要的任务，关闭时可以丢弃
  
  // 3. BLOCK_SHUTDOWN - 等待完成再关闭（默认）
  // 用途: 重要任务，必须完成
};
```

### 使用

```cpp
// 保存关键数据 - 必须等待完成
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskShutdownBehavior::BLOCK_SHUTDOWN},
    base::BindOnce(&SaveCriticalData));

// 上报分析数据 - 应用关闭时可以跳过
base::ThreadPool::PostTask(
    FROM_HERE,
    {base::TaskShutdownBehavior::SKIP_ON_SHUTDOWN},
    base::BindOnce(&SendAnalytics));
```

---

## 调试多线程问题

### 1. 确认当前线程

```cpp
// 检查当前线程
if (content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI)) {
  DLOG(INFO) << "我在 UI 线程";
} else {
  DLOG(INFO) << "我不在 UI 线程";
}

// 获取当前线程 ID
DLOG(INFO) << "Thread ID: " << base::PlatformThread::CurrentId();
```

### 2. 添加线程 ID 到日志

```cpp
// base/logging.h
LOG(INFO) << "[Thread " << base::PlatformThread::CurrentId() 
          << "] Event occurred";

// 输出示例:
// [Thread 12345] Event occurred
// [Thread 12346] Another event
```

### 3. 使用 DCHECK 验证线程

```cpp
// 这个函数必须在 UI 线程调用
void UpdateUI() {
  DCHECK(content::BrowserThread::CurrentlyOn(
      content::BrowserThread::UI));
  // 如果不在 UI 线程，会立即崩溃并显示错误
}

// 生产环境中的检查（不会崩溃，只会 log）
void UpdateUI() {
  if (!content::BrowserThread::CurrentlyOn(
          content::BrowserThread::UI)) {
    LOG(ERROR) << "UpdateUI called from wrong thread!";
    return;
  }
}
```

### 4. Chrome DevTools 中看到的线程

```
在 Chrome 的 about:tracing 中：
├─ Browser Thread (UI)
│  └─ Task 1: OnClick()
│  └─ Task 2: UpdateUI()
│
├─ IO Thread
│  └─ Task A: NetworkRequest()
│
└─ ThreadPool Worker 0-7
   └─ Task X: ComputeHash()
   └─ Task Y: ParseJSON()
```

---

## 实战演习：完整的网页加载任务分解

### 假设场景
```
用户点击链接 → 网页加载 → DOM 解析 → 计算布局 → 绘制 → 显示
```

### 代码实现

```cpp
class PageLoader {
 public:
  void LoadPage(const std::string& url) {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    
    // 任务 1: 在 IO 线程获取网页
    DLOG(INFO) << "[UI] 开始加载页面: " << url;
    
    content::GetIOThreadTaskRunner({})->PostTaskAndReplyWithResult(
        FROM_HERE,
        {base::TaskPriority::USER_BLOCKING},
        base::BindOnce(&PageLoader::FetchHTMLOnIOThread, this, url),
        base::BindOnce(&PageLoader::OnHTMLReady, weak_factory_.GetWeakPtr()));
  }
  
 private:
  // ========== IO 线程 ==========
  std::string FetchHTMLOnIOThread(const std::string& url) {
    DLOG(INFO) << "[IO] 获取 HTML from " << url;
    // 模拟网络请求
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return "<html>...</html>";
  }
  
  // ========== UI 线程 ==========
  void OnHTMLReady(std::string html) {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    
    DLOG(INFO) << "[UI] HTML 已获取，大小: " << html.size();
    
    // 任务 2: 在 ThreadPool 中解析 DOM（耗时）
    base::ThreadPool::PostTaskAndReplyWithResult(
        FROM_HERE,
        {base::TaskPriority::USER_BLOCKING},
        base::BindOnce(&PageLoader::ParseDOMInThreadPool, this, html),
        base::BindOnce(&PageLoader::OnDOMReady, weak_factory_.GetWeakPtr()));
  }
  
  // ========== ThreadPool ==========
  std::vector<std::string> ParseDOMInThreadPool(const std::string& html) {
    DLOG(INFO) << "[Pool] 解析 DOM";
    // 模拟耗时的 DOM 解析
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    return {"<div>", "<p>", "</p>", "</div>"};
  }
  
  // ========== UI 线程 ==========
  void OnDOMReady(std::vector<std::string> dom_nodes) {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    
    DLOG(INFO) << "[UI] DOM 已解析，节点数: " << dom_nodes.size();
    
    // 任务 3: 计算布局（在 Compositor 线程）
    // 假设 content_renderer_client 可用
    if (auto runner = GetCompositorTaskRunner()) {
      runner->PostTaskAndReply(
          FROM_HERE,
          base::BindOnce(&PageLoader::ComputeLayoutOnCompositor, this),
          base::BindOnce(&PageLoader::OnLayoutReady, weak_factory_.GetWeakPtr()));
    }
  }
  
  // ========== Compositor 线程 ==========
  void ComputeLayoutOnCompositor() {
    DLOG(INFO) << "[Compositor] 计算布局";
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
  }
  
  // ========== UI 线程 ==========
  void OnLayoutReady() {
    DCHECK(content::BrowserThread::CurrentlyOn(
        content::BrowserThread::UI));
    
    DLOG(INFO) << "[UI] 布局已完成";
    
    // 任务 4: 最后在 UI 线程显示
    Render();
  }
  
  void Render() {
    DLOG(INFO) << "[UI] 显示页面";
  }
  
  scoped_refptr<TaskRunner> GetCompositorTaskRunner() {
    // 简化版本
    return nullptr;
  }
  
  base::WeakPtrFactory<PageLoader> weak_factory_{this};
};

// 执行日志输出顺序：
// [UI] 开始加载页面: https://example.com
// [IO] 获取 HTML from https://example.com   (1 秒后)
// [UI] HTML 已获取，大小: 5000
// [Pool] 解析 DOM                           (随后)
// [UI] DOM 已解析，节点数: 100              (500ms 后)
// [Compositor] 计算布局                    (随后)
// [UI] 布局已完成                          (200ms 后)
// [UI] 显示页面
```

---

## 总结：何时使用什么

```cpp
场景 1: UI 更新
  → content::GetUIThreadTaskRunner({})->PostTask(...)

场景 2: 网络/文件 I/O
  → content::GetIOThreadTaskRunner({})->PostTask(...)
  或 base::ThreadPool::PostTask(...)

场景 3: 后台计算（任意顺序）
  → base::ThreadPool::PostTask(...)

场景 4: 后台操作（需要保证顺序）
  → base::ThreadPool::CreateSequencedTaskRunner()

场景 5: 需要等待结果
  → PostTaskAndReplyWithResult(...)

场景 6: 跨线程回调（不想崩溃）
  → 使用 WeakPtr

场景 7: 线程安全数据访问
  → 使用 Sequence（最简单）
  或 Lock（细粒度）
  或 Atomic（超级快）
```

---

## 常用代码片段

### 片段 1: 创建后台线程任务

```cpp
// 最简单
base::ThreadPool::PostTask(FROM_HERE, 
    base::BindOnce(&MyClass::BackgroundWork, this));

// 带 WeakPtr 防护
base::ThreadPool::PostTask(FROM_HERE,
    base::BindOnce(&MyClass::BackgroundWork, weak_factory_.GetWeakPtr()));

// 有返回值
base::ThreadPool::PostTaskAndReplyWithResult(FROM_HERE,
    base::BindOnce(&ComputeHash, data),
    base::BindOnce(&MyClass::OnHashReady, weak_factory_.GetWeakPtr()));
```

### 片段 2: 跨线程通信

```cpp
// UI → IO
content::GetIOThreadTaskRunner({})->PostTask(FROM_HERE,
    base::BindOnce(&IOClass::DoIO, io_instance));

// IO → UI
content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE,
    base::BindOnce(&UIClass::UpdateUI, weak_ui_ptr));

// Pool → UI
base::ThreadPool::PostTaskAndReply(FROM_HERE,
    base::BindOnce(&ExpensiveWork),
    base::BindOnce(&UIClass::OnWorkDone, weak_ui_ptr));
```

### 片段 3: 序列化操作

```cpp
auto sequence = base::ThreadPool::CreateSequencedTaskRunner({});

sequence->PostTask(FROM_HERE, base::BindOnce(&Step1));
sequence->PostTask(FROM_HERE, base::BindOnce(&Step2));
sequence->PostTask(FROM_HERE, base::BindOnce(&Step3));

// 保证执行顺序：Step1 → Step2 → Step3
```

