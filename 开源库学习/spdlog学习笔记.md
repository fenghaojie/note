**说明**：所有内容翻译自spdlog的wiki，受英语水平所限，有所错误或失真在所难免，如果您有更好的建议，请在博文下留言。

# 线程安全
----
 `spdlog::` 命名空间下的是线程安全的，当*loggers*在不同的线程同时执行时，下述函数不应该被调用：
```c
spdlog::set_error_handler(log_err_handler); // or logger->set_error_handler(log_err_handler);
```
*logger*在其它线程执行过程中，添加或移除`sink`是线程不安全的
```c
logger->sinks().push_back(new_sink); // Don't do this if other thread is already using this logger
```
要创建线程安全的*loggers*，使用带 `_mt` 后缀的工厂函数，例如：
```c
auto logger = spdlog::basic_logger_mt(...);
```

要创建单线程的*loggers*，使用带 `_st` 后缀的工厂函数，例如：
```c
auto logger = spdlog::basic_logger_st(...);
```
对于`sinks`，以 `_mt` 后缀结尾的是线程安全的，比如：`daily_file_sink_mt`

以` _st` 后缀结尾的是非线程安全的，比如：`daily_file_sink_st`

# 快速开始
----
```c
#include "spdlog/spdlog.h"
 int main()
 {
  //Use the default logger (stdout, multi-threaded, colored)
  spdlog::info("Hello, {}!", "World");
 }
```
`spdlog`是个只有头文件的库，只需要将头文件拷贝到你的工程就可以使用了，编译器需要支持**C++11**
它使用一个类似python的格式API库fmt：
```c
logger->info("Hello {} {} !!", "param1", 123.4);
```
`spdlog`支持使用最小集的方式，意味着你只用包含你实际需要的头文件，而不是全部，比如说你只需要使用 `rotating logger`，那么你只需要 
```c
#include <spdlog/sinks/rotating_file_sink.h>
```
对于异步特性，你还需要
```c
#include <spdlog/asynch.h>
```
### 基本用法示例
```c
#include <iostream>
#include "spdlog/spdlog.h"
#include "spdlog/sinks/basic_file_sink.h" // support for basic file logging
#include "spdlog/sinks/rotating_file_sink.h" // support for rotating file logging

int main(int, char* [])
{
	try 
	{
		// Create basic file logger (not rotated)
		auto my_logger = spdlog::basic_logger_mt("basic_logger", "logs/basic.txt");
		
		// create a file rotating logger with 5mb size max and 3 rotated files
		auto file_logger = spdlog::rotating_logger_mt("file_logger", "myfilename", 1024 * 1024 * 5, 3);
	}
	catch (const spdlog::spdlog_ex& ex)
	{
		std::cout << "Log initialization failed: " << ex.what() << std::endl;
	}
}
```
### 使用工厂函数创建异步*logger*：
```c
#include "spdlog/async.h" //support for async logging.
#include "spdlog/sinks/basic_file_sink.h"

int main(int, char* [])
{
	try
	{        
		auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");
		for (int i = 1; i < 101; ++i)
		{
			async_file->info("Async message #{}", i);
		}
		// Under VisualStudio, this must be called before main finishes to workaround a known VS issue
		spdlog::drop_all(); 
	}
	catch (const spdlog::spdlog_ex& ex)
	{
		std::cout << "Log initialization failed: " << ex.what() << std::endl;
	}
}
```
### 创建异步*logger*并改变线程池设置
```c
#include "spdlog/async.h" //support for async logging.
#include "spdlog/sinks/basic_file_sink.h"
int main(int, char* [])
{
    try
    {                                        
        auto daily_sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>("logfile", 23, 59);
       // default thread pool settings can be modified *before* creating the async logger:
        spdlog::init_thread_pool(10000, 1); // queue with 10K items and 1 backing thread.
        auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");       
        spdlog::drop_all(); 
    }
    catch (const spdlog::spdlog_ex& ex)
    {
        std::cout << "Log initialization failed: " << ex.what() << std::endl;
    }
}
```
### 创建一个由多个*loggers*共享同一个输出文件的`sink`
```c
#include <iostream>
#include "spdlog/spdlog.h"
#include "spdlog/sinks/daily_file_sink.h"
int main(int, char* [])
{
    try
    {
        auto daily_sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>("logfile", 23, 59);
        // create synchronous  loggers
        auto net_logger = std::make_shared<spdlog::logger>("net", daily_sink);
        auto hw_logger  = std::make_shared<spdlog::logger>("hw",  daily_sink);
        auto db_logger  = std::make_shared<spdlog::logger>("db",  daily_sink);      

        net_logger->set_level(spdlog::level::critical); // independent levels
        hw_logger->set_level(spdlog::level::debug);
         
        // globally register the loggers so so the can be accessed using spdlog::get(logger_name)
        spdlog::register_logger(net_logger);
    }
    catch (const spdlog::spdlog_ex& ex)
    {
        std::cout << "Log initialization failed: " << ex.what() << std::endl;
    }
}
```
### 创建一个对应多个`sink`的*logger*，每一个`sink`都有独有的格式和日志级别
```c
//
// Logger with console and file output.
// the console will show only warnings or worse, while the file will log all messages.
// 
#include <iostream>
#include "spdlog/spdlog.h"
#include "spdlog/sinks/stdout_color_sinks.h" // or "../stdout_sinks.h" if no colors needed
#include "spdlog/sinks/basic_file_sink.h"
int main(int, char* [])
{
    try
    {
        auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
        console_sink->set_level(spdlog::level::warn);
        console_sink->set_pattern("[multi_sink_example] [%^%l%$] %v");

        auto file_sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>("logs/multisink.txt", true);
        file_sink->set_level(spdlog::level::trace);

        spdlog::logger logger("multi_sink", {console_sink, file_sink});
        logger.set_level(spdlog::level::debug);
        logger.warn("this should appear in both console and file");
        logger.info("this message should not appear in the console, only in the file");
    }
    catch (const spdlog::spdlog_ex& ex)
    {
        std::cout << "Log initialization failed: " << ex.what() << std::endl;
    }
}
```
### 日志宏定义

在包含 *spdlog.h"之前，添加 `SPDLOG_ACTIVE_LEVEL` 宏定义可以设置期望的日志级别
```c
#define SPDLOG_ACTIVE_LEVEL SPDLOG_LEVEL_DEBUG
SPDLOG_LOGGER_TRACE(file_logger , "Some trace message that will not be evaluated.{} ,{}", 1, 3.23);
SPDLOG_LOGGER_DEBUG(file_logger , "Some Debug message that will be evaluated.. {} ,{}", 1, 3.23);
SPDLOG_DEBUG("Some debug message to default logger that will be evaluated");
```
### 记录用户定义的对象
```c
#include "spdlog/spdlog.h"
#include "spdlog/fmt/ostr.h" // must be included
#include "spdlog/sinks/stdout_sinks.h"

class some_class {};
std::ostream& operator<<(std::ostream& os, const some_class& c)
{ 
    return os << "some_class"; 
}

void custom_class_example()
{
    some_class c;
    auto console = spdlog::stdout_logger_mt("console");
    console->info("custom class with operator<<: {}..", c);
}
```
# 创建 logger
----
每一个*logger*中包含一个存有一个或多个 `std::shared_ptr<spdlog::sink>`的 vector

*logger*在记录每一条日志时（如果是有效的级别），将会调用每一个`std::shared_ptr<spdlog::sink>`中的`sink(log_msg)`函数

*spdlog*用`_mt`(多线程)和`_st`(单线程)后缀标识`sink`是否是线程安全

单线程的`sink`不可以在多线程中使用，它的速度会更快，因为没有锁竞争

### 使用工厂函数创建*loggers*
```c
//Create and return a shared_ptr to a multithreaded console logger.
#include "spdlog/sinks/stdout_color_sinks.h"
auto console = spdlog::stdout_color_mt("some_unique_name");
```
这样就创建了一个*console logger*，并以"some_unique_name"作为它的id，将自身注册到`spdlog`，返回自身的智能指针

### 使用`spdlog::get("...")`访问*loggers*

*loggers*可以在任何地方使用线程安全的`spdlog::get("logger_name")`来进行访问，返回智能指针

**注意**：`spdlog::get`可能会拖慢你的程序，因为它内部维护了一把锁，所以要谨慎使用。比较推荐的用法是保存返回的`shared_ptr<spdlog::logger>`，直接使用它，至少在频繁访问的代码中。

一个很好的方法是建立一个`std::shared_ptr<spdlog::logger>`私有成员变量，并在构造函数中初始化：
```c
class MyClass
{
private:
   std::shared_ptr<spdlog::logger> _logger;
public:
   MyClass()
   {
     //set _logger to some existing logger
     _logger = spdlog::get("some_logger");
     //or create directly
     //_logger = spdlog::rotating_file_logger_mt("my_logger", ...);
   }
};
```
**注意2**：手动创建的*logger*不会被自动注册，并且也不会通过`get(...)`调用查找到

注册手动创建的*logger*对象需要使用`register_logger(...)`函数：
```c
spdlog::register_logger(my_logger);
...
auto the_same_logger = spdlog::get("mylogger");
```

### 创建 `rotating file logger`
```c
//Create rotating file multi-threaded logger
#include "spdlog/sinks/rotating_file_sink.h"
auto file_logger = spd::rotating_logger_mt("file_logger", "logs/mylogfile", 1048576 * 5, 3);
...
auto same_logger= spdlog::get("file_logger");
```
### 创建异步*logger*
```c
#include "spdlog/async.h"
void async_example()
{
    // default thread pool settings can be modified *before* creating the async logger:
    // spdlog::init_thread_pool(8192, 1); // queue with 8k items and 1 backing thread.
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");
    // alternatively:
    // auto async_file = spdlog::create_async<spdlog::sinks::basic_file_sink_mt>("async_file_logger", "logs/async_log.txt");
   
}
```

对于异步日志记录，spdlog使用具有专用消息队列的共享全局线程池。

为此，它在消息队列中创建固定数量的**预分配的插槽**（64位中每个插槽大约256个字节），并且可以使用`spdlog::init_thread_pool(queue_size，backing_threads_count)`进行修改。

当尝试记录一条日志时，并且队列已满，那么调用默认会被阻塞，并且默认直到一个插槽可用时，或者立即移除队列中最旧的日志信息，并追加最新的日志信息（如果*logger*以`async_overflow_policy==overrun_oldest`构造）

### 手动创建*loggers*
```c
auto sink = std::make_shared<spdlog::sinks::stdout_sink_mt>();
auto my_logger= std::make_shared<spdlog::logger>("mylogger", sink);
```

### 创建拥有多个`sink`的*logger*
```c
std::vector<spdlog::sink_ptr> sinks;
sinks.push_back(std::make_shared<spdlog::sinks::stdout_sink_st>());
sinks.push_back(std::make_shared<spdlog::sinks::daily_file_sink_st>("logfile", 23, 59));
auto combined_logger = std::make_shared<spdlog::logger>("name", begin(sinks), end(sinks));
//register it if you need to access it globally
spdlog::register_logger(combined_logger);
```

### 创建多个具有同一个输出文件的`file logger`
```c
auto sharedFileSink = std::make_shared<spdlog::sinks::basic_file_sink_mt>("fileName.txt");
auto firstLogger = std::make_shared<spdlog::logger>("firstLoggerName", sharedFileSink);
auto secondLogger = std::make_unique<spdlog::logger>("secondLoggerName", sharedFileSink);
```

# 自定义格式
----
每一个*logger*的`sink`都有一个格式化器，用来格式化消息为目标格式

`spdlog`默认的日志格式为：
```
[2019-04-18 13:31:59.678] [info] [my_loggername] Some message
```

有两种方式可以自定义*logger*的格式：
* 设置模式字符串（**推荐**）
```c
set_pattern(pattern_string);
```
* 或者实现自定义格式器，实现`formatter`接口，并调用
```c
set_formatter(std::make_shared<my_custom_formatter>());
```

### 使用```set_pattern(...)`自定义格式

格式应用到所有被注册的*logger*：
```c
spdlog::set_pattern("*** [%H:%M:%S %z] [thread %t] %v ***");
```
或者一个特定的`logger`对象：
```c
some_logger->set_pattern(">>>>>>>>> %H:%M:%S %z %v <<<<<<<<<");
```
或者一个特定的`sink`对象：
```c
some_logger->sinks()[0]->set_pattern(">>>>>>>>> %H:%M:%S %z %v <<<<<<<<<");
some_logger->sinks()[1]->set_pattern("..");
```

### 性能

无论何时用户调用`set_pattern(...)`，库都会将新模式设置为内部有效标识 - 这种方式使得即便在复杂模式下，性能仍然很高（每次记录日志时不会重新解析模式）

### 模式标记
| flag | meaning| example |
| :------ | :-------: | :-----: |
|`%v`|The actual text to log|"some user text"|
|`%t`|Thread id|"1232"|
|`%P`|Process id|"3456"|
|`%n`|Logger's name|"some logger name"
|`%l`|The log level of the message|"debug", "info", etc|
|`%L`|Short log level of the message|"D", "I", etc|
|`%a`|Abbreviated weekday name|"Thu"|
|`%A`|Full weekday name|"Thursday"|
|`%b`|Abbreviated month name|"Aug"|
|`%B`|Full month name|"August"|
|`%c`|Date and time representation|"Thu Aug 23 15:35:46 2014"|
|`%C`|Year in 2 digits|"14"|
|`%Y`|Year in 4 digits|"2014"|
|`%D` or `%x`|Short MM/DD/YY date|"08/23/14"|
|`%m`|Month 1-12|"11"|
|`%d`|Day of month 1-31|"29"|
|`%H`|Hours in 24 format  0-23|"23"|
|`%I`|Hours in 12 format  1-12|"11"|
|`%M`|Minutes 0-59|"59"|
|`%S`|Seconds 0-59|"58"|
|`%e`|Millisecond part of the current second 0-999|"678"|
|`%f`|Microsecond part of the current second 0-999999|"056789"|
|`%F`|Nanosecond part of the current second 0-999999999|"256789123"|
|`%p`|AM/PM|"AM"|
|`%r`|12 hour clock|"02:55:02 pm"|
|`%R`|24-hour HH:MM time, equivalent to %H:%M|"23:55"|
|`%T` or `%X`|ISO 8601 time format (HH:MM:SS), equivalent to %H:%M:%S|"23:55:59"|
|`%z`|ISO 8601 offset from UTC in timezone ([+/-]HH:MM)|"+02:00"|
|`%E`|Seconds since the epoch |"1528834770"|
|`%i`|Message sequence number (disabled by default - edit 'tweakme.h' to enable)|"1154"|
|`%%`|The % sign|"%"|
|`%+`|spdlog's default format|"[2014-10-31 23:46:59.678] [mylogger] [info] Some message"|
|`%^`|start color range|"[mylogger] [info(green)] Some message"|
|`%$`|end color range (for example %^[+++]%$ %v)|[+++] Some message|
|`%@`|Source file and line (use SPDLOG_TRACE(..),SPDLOG_INFO(...) etc.)|my_file.cpp:123|
|`%s`|Source file (use SPDLOG_TRACE(..),SPDLOG_INFO(...) etc.)|my_file.cpp|
|`%#`|Source line (use SPDLOG_TRACE(..),SPDLOG_INFO(...) etc.)|123|
|`%!`|Source function (use SPDLOG_TRACE(..),SPDLOG_INFO(...) etc. see tweakme for pretty-print)|my_func|

### 对齐
每一个模式标记可以通过预先添加一个宽度标记来对齐（上限128）

使用`-`（左对齐）或者`=`（中间对齐）去控制向哪边对齐：
| align | meaning| example | result|
| :------ | :-------: | :-----: |  :-----: |
|`%<width><flag>`|Align to the right|`%8l`|"&nbsp;&nbsp;&nbsp;&nbsp;info"|
|`%-<width><flag>`|Align to the left|`%-8l`|"info&nbsp;&nbsp;&nbsp;&nbsp;"|
|`%=<width><flag>`|Align to the center|`%=8l`|"&nbsp;&nbsp;info&nbsp;&nbsp;"|

# sink
----

`sink`是实际将日志写入目标位置的对象。每一个`sink`仅应负责写一个目标文件（比如 file，console，db），并且每一个`sink`有专属的私有格式化器`formatter`实例。

### 可用的sink

**rotating_file_sink**

达到最大文件大小时，关闭文件，重命名文件并创建新文件。 最大文件大小和最大文件数都可以在构造函数中配置。

**注意**：用户应该负责去创建任何他们需要的文件夹。*spdlog*除了文件不会尝试创建任何文件夹

```c
// create a thread safe sink which will keep its file size to a maximum of 5MB and a maximum of 3 rotated files.
#include "spdlog/sinks/rotating_file_sink.h"
...
auto file_logger = spdlog::rotating_logger_mt("file_logger", "logs/mylogfile", 1048576 * 5, 3);
```

或者手动创建`sink`并将它传递给*logger*：
```c
#include "spdlog/sinks/rotating_file_sink.h"
...
auto rotating = make_shared<spdlog::sinks::rotating_file_sink_mt> ("log_filename", "log", 1024*1024, 5, false);
auto file_logger = make_shared<spdlog::logger>("my_logger", rotating);
```

**daily_file_sink**

每天在一个特别的时间创建一个新的日志文件，并在文件名字上添加一个时间戳

**注意**：用户应该负责去创建任何他们需要的文件夹。*spdlog*除了文件不会尝试创建任何文件夹

```c
#include "spdlog/sinks/daily_file_sink.h"
..
auto daily_logger = spdlog::daily_logger_mt("daily_logger", "logs/daily", 14, 55);
```

将会在每天的14:55创建一个新的日志文件，并创建一个线程安全的`sink`

**simple_file_sink**

无任何限制的向一个日志文件中写入

**注意**：用户应该负责去创建任何他们需要的文件夹。*spdlog*除了文件不会尝试创建任何文件夹

```c
#include "spdlog/sinks/basic_file_sink.h"
...
auto logger = spdlog::basic_logger_mt("mylogger", "log.txt");
```

**stdout_sink/stderr_sink with colors**

```c
#include "spdlog/sinks/stdout_color_sinks.h"
...
auto console = spdlog::stdout_color_mt("console");
auto err_console = spdlog::color_logger_mt("console");
```

或者直接创建`sink`：
```c
auto sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
```

**ostream_sink**
```c
#include "spdlog/sinks/ostream_sink.h "
...
std::ostringstream oss;
auto ostream_sink = std::make_shared<spdlog::sinks::ostream_sink_mt> (oss);
auto logger = std::make_shared<spdlog::logger>("my_logger", ostream_sink);
```

**null_sink**

会丢弃所有到它的日志

```c
#include "spdlog/sinks/null_sink.h"
...
auto logger = spdlog::create<spdlog::sinks::null_sink_st>("null_logger");
```

**syslog_sink**

POSIX syslog(3) 发送日志到syslog
```c
#include "spdlog/sinks/syslog_sink.h"
...
auto syslog_logger = spdlog::syslog_logger("syslog", "my_ident");
```

**dist_sink**

将日志消息分发到其他接收器列表
```c
#include "spdlog/sinks/syslog_sink.h"
...
auto dist_sink = make_shared<spdlog::sinks::dist_sink_st>();
auto sink1 = make_shared<spdlog::sinks::stdout_sink_st>();
auto sink2 = make_shared<spdlog::sinks::simple_file_sink_st>("mylog.log");

dist_sink->add_sink(sink1);
dist_sink->add_sink(sink2);
```

**msvc_sink**

Windows debug sink (使用OutputDebugStringA窗口输出日志)

```c
#include "spdlog/sinks/msvc_sink.h"
auto sink = std::make_shared<spdlog::sinks::msvc_sink_mt>();
auto logger = std::make_shared<spdlog::logger>("msvc_logger", sink);
```

### 实现自己的`sink`

实现自己的`sink`，你需要实现`sink`类的接口

一种推荐的方式是继承自`base_sink`类

该类已经处理了线程锁，使得实现一个线程安全的`sink`非常容易

```c
#include "spdlog/sinks/base_sink.h"

template<typename Mutex>
class my_sink : public spdlog::sinks::base_sink <Mutex>
{
...
protected:
    void sink_it_(const spdlog::details::log_msg& msg) override
    {

    // log_msg is a struct containing the log entry info like level, timestamp, thread id etc.
    // msg.raw contains pre formatted log

    // If needed (very likely but not mandatory), the sink formats the message before sending it to its final destination:
    fmt::memory_buffer formatted;
    sink::formatter_->format(msg, formatted);
    std::cout << fmt::to_string(formatted);
    }

    void flush_() override 
    {
       std::cout << std::flush;
    }
};

#include "spdlog/details/null_mutex.h"
#include <mutex>
using my_sink_mt = my_sink<std::mutex>;
using my_sink_st = my_sink<spdlog::details::null_mutex>;
```

### 创建`sink`后，将其添加至`logger`

由于在spdlog v1.x 版本中有一个函数返回一个非常引用的*skins vector*，它允许你手动的向*skins*中添加。对于这个*skins vector*没有锁保护，因此它不是线程安全的。

```c
inline std::vector<spdlog::sink_ptr> &spdlog::logger::sinks()
{
    return sinks_;
}
```

# logger及注册
----

spdlog包含一个进程内全局的注册机制，对于本进程内所有创建的loggers

目的是在项目的任何地方轻松访问loggers，而不会传递它们。

```c
spdlog::get("logger1")->info("hello");
.. 
.. 
some other source file..
..
auto l = spdlog::get("logger1");
l->info("hello again");
```

如果未找到logger，会返回一个空智能指针。你应该判断智能指针的有效性

### 注册新的*loggers*

一般情况下没必要去注册loggers，因为它们已经自动注册了

手动创建的loggers，需要自己去注册，使用 `register_logger(std::shared_ptr<logger>)` 函数：
```c
spdlog::register_logger(some_logger);
```

将使用*some_logger*的name来注册它自己

### 注册冲突

当尝试注册一个名字已经被注册过的logger时，spdlog会抛出一个 `spdlog::spdlog_ex`异常

### 从注册器中移除*loggers*

`drop()`函数可以用来从注册器中移除一个*logger*

如果logger智能指针没有其它引用时，该logger将会被关闭并且释放它相关的资源

```c
spdlog::drop("logger_name");
//or remove them all
spdlog::drop_all()
```

# 异步日志
----

创建异步logger有多种方式。你需要 `#include "spdlog/async.h`

### 使用 `<spdlog::async_logger>`模板参数

```c
#include "spdlog/async.h"
void async_example()
{
    // default thread pool settings can be modified *before* creating the async logger:
    // spdlog::init_thread_pool(8192, 1); // queue with 8k items and 1 backing thread.
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>("async_file_logger", "logs/async_log.txt");
}
```

### 使用 `spdlog::create_async<sink>`

```c
auto async_file = spdlog::create_async<spdlog::sinks::basic_file_sink_mt>("async_file_logger", "logs/async_log.txt");
```

### 使用 `spdlog::create_async_nb<sink>`

创建即便队列满也永远不会阻塞的logger

```c
auto async_file = spdlog::create_async_nb<spdlog::sinks::basic_file_sink_mt>("async_file_logger", "logs/async_log.txt");
```

### 直接构造，并使用全局线程池
```c
auto logger = std::make_shared<spdlog::async_logger>("as", some_sink, spdlog::thread_pool(), async_overflow_policy::block);
```

### 直接构造，使用自定义线程池

```c
spdlog::init_thread_pool(queue_size, n_threads);
auto logger = std::make_shared<spdlog::async_logger>("as", some_sink, spdlog::thread_pool(), async_overflow_policy::block);
```

### 直接构造，使用自定义线程池

```c
auto tp = std::make_shared<details::thread_pool>(queue_size, n_threads);
auto logger = std::make_shared<spdlog::async_logger>("as", some_sink, tp, async_overflow_policy::block);
```

**注意**：上例中的tp对象的生命期一定要长于logger对象，因为logger需要一个tp对象的weak_ptr

## 队列满时的决策

当队列满了的时候有两种可选方式：
* 阻塞调用直到有空间可用（默认行为）
* 移除并替换队列中最旧的信息，不用等待可用空间

使用 `create_async_nb` 工厂函数或者 在logger构造时使用`spdlog::async_overflow_policy`

```c
auto logger = spdlog::create_async_nb<spdlog::sinks::basic_file_sink_mt>("async_file_logger", "logs/async_log.txt");
// or directly:
 auto logger = std::make_shared<async_logger>("as", test_sink, spdlog::thread_pool(), spdlog::async_overflow_policy::overrun_oldest);
```

## spdlog的线程池

默认情况下，spdlog创建一个全局的线程池，队列大小为8192，一个工作线程服务于所有的`async loggers`

这意味着创建和销毁 `async loggers`非常廉价，因为它们不拥有或者创建任何后台线程或队列--它们被共享的线程池对象创建和管理

队列中所有的槽都是在线程池构造时预分配的（64位系统中每个槽占256字节）

线程池的大小和线程可以被重置：
```c
spdlog::init_thread_pool(queue_size, n_threads);
```

**注意**：这将会销毁之前的全局线程池对象tp，并创建一个新的线程池--这也意味着所有使用旧的线程池tp的loggers都将停止工作，因此建议在任何`async loggers`被创建之前调用该函数

如果不同的loggers必须要使用不同的队列，那么可以创建不同的线程池，并传递给loggers：
```c
auto tp = std::make_shared<details::thread_pool>(128, 1);
 auto logger = std::make_shared<async_logger>("as", some_sink, tp, async_overflow_policy::overrun_oldest);

auto tp2 = std::make_shared<details::thread_pool>(1024, 4);  // create pool with queue of 1024 slots and 4 backing threads
auto logger2 = std::make_shared<async_logger>("as2", some_sink, tp2, async_overflow_policy::block);
```

### Windows上的问题
在VS运行时存在一个bug，在退出时会导致应用程序死锁。如果你使用异步日志记录，一定要确保在main()函数退出时调用`spdlog::shutdown()`函数

# Flush策略
----

默认情况下，spdlog允许底层libc在它认为合适时进行刷新，以获得良好的性能。 您可以使用以下选项覆盖它：

### 手动flush

使用`logger->flush()`函数让logger去flush它的内容，logger将依次调用包含的每一个`sink`上的`flush()`函数

**注意**：如果使用`async logger`，`logger->flush()`会发送一个消息到队列，请求flush操作，因此函数会立即返回。这跟一些旧版本的spdlog是有区别的（老版本会同步等待直到flush完成，并接收到消息）

### 基于严重性的flush

可以设置一个最小日志等级来触发自动flush

下例中，只要error或者更严重的日志被记录时就会触发flush：
```c
my_logger->flush_on(spdlog::level::err);
```

### 基于间隔的flush

spdlog支持设置flush间隔。由一个单独的工作线程定期调用每一个logger的flush()实现

下例中，对所有已注册的loggers定期5秒调用flush()：
```c
spdlog::flush_every(std::chrono::seconds(5));
```

**注意**：仅应在线程安全的loggers上使用该特性，因为定期flush是从不同的线程发生的

# 默认的logger
---

为方便起见，spdlog创建了一个默认的全局记录器（stdout，colors和multithreaded）。

通过直接调用 `spdlog::info(..), spdlog::debug(..), etc`

下例可以替换任何其它logger为默认logger：
```c
spdlog::set_default_logger(some_other_logger);
spdlog::info("Use the new default logger");
```

# 错误处理
----

spdlog在记录过程中不会抛出异常

在构造logger或sink时可能会抛出异常，因为它认为出了严重错误

如果在日志记录过程中发生了错误，spdlog会打印错误信息到stderr

为了避免满屏幕大量打印错误信息，限制速率为每个logger 1 条消息/分钟

该行为可以被改变，通过调用`spdlog::set_error_handler(new_handler_fun)` 或者 `logger->set_error_handler(new_handler_fun)`

**修改全局错误处理句柄**
```c
spdlog::set_error_handler([](const std::string& msg) {
        std::cerr << "my err handler: " << msg << std::endl;
    });
```

**对于特定的logger**
```c
critical_logger->set_error_handler([](const std::string& msg) {
        throw std::runtime_error(msg);
    });
```

**默认错误处理句柄**

`default_err_handler`会使用下面语法打印错误
```c
fmt::print(stderr, "[*** LOG ERROR ***] [{}] [{}] {}\n", date_buf, name(), msg);
```

# 如何在dll中使用spdlog
----

由于spdlog是仅有头文件的库，构建共享库和在主程序中使用它将不会在它们之间共享注册器信息

就是说调用类似于 `spdlog::set_level(spdlog::level::level_enum::info)` 将不会改变dll中的loggers

### 解决办法

在主程序和dll中都注册logger

```c
/*
 * Disclaimer:
 *   This was not compiled but extracted from documentation and some code.
 */

// mylibrary.h
// In library, we skip the symbol exporting part

#include <memory>
#include <vector>
#include <spdlog/logger.h>
#include <spdlog/sinks/stdout_color_sinks.h>

namespace library
{
static const std::string logger_name = "example";

std::shared_ptr<spdlog::logger> setup_logger(std::vector<spdlog::sink_ptr> sinks)
{
    auto logger = spdlog::get(logger_name);
    if(not logger)
    {
        if(sinks.size() > 0)
        {
            logger = std::make_shared<spdlog::logger>(logger_name,
                                                      std::begin(sinks),
                                                      std::end(sinks));
            spdlog::register_logger(logger);
        }
        else
        {
            logger = spdlog::stdout_color_mt(logger_name);
        }
    }

    return logger;
}

void test(std::string message)
{
    auto logger = spdlog::get(logger_name);
    if(logger)
    {
        logger->debug("{}::{}", __FUNCTION__, message);
    }
}

}
```

```c
// In the main program

#include <mylibrary.h>
#include <spdlog/logger.h>
#include <spdlog/sinks/daily_file_sink.h>
#include <spdlog/sinks/stdout_sinks.h>

int main()
{
    // We assume that we load the library here
    ...

    // Let's use the library
    std::vector<spdlog::sink_ptr> sinks;
    sinks.push_back(std::make_shared<spdlog::sinks::stdout_sink_st>());
    sinks.push_back(std::make_shared<spdlog::sinks::daily_file_sink_st>("logfile", 23, 59));

    auto logger = library::setup_logger(sinks);

    spdlog::set_level(spdlog::level::level_enum::debug); // No effect for the library.
    library::test("Hello World!"); // No logging

    spdlog::register_logger(logger);

    // Now this will also affect the library logger
    spdlog::set_level(spdlog::level::level_enum::debug);

    library::test("Hello World!"); // Hurray !

    return 0;
}
```
