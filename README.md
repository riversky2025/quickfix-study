# quickfix-study
# 安装

## 平台依赖
|   平台   | 运行  | 编译                                     | 测试 |
| :------: | :---- | :--------------------------------------- | :--- |
|   win    |       | Microsoft Visual C++                     | ruby |
|  linux   | glibc | gcc gcc-c++ glibc-devel gnu make sstream | ruby |
| Solaris  | glibc | gcc gcc-c++ glibc-devel gnu make sstream | ruby |
| FreeBSD  |       | ruby                                     |
| Mac OS X |       | Xcode Tools                              | ruby |

> 可选依赖
> * mysql 
> * mssql
> * PostgreSQL
> * STLport

## 基本编译
[下载](https://github.com/quickfix/quickfix)

```
git clone https://github.com/quickfix/quickfix
```

平台选择X64(windows sdk 8.1-- 工具集v140)  静态编译/动态编译（多线程版）（debug与release）  
如果需要支持mysql或者odbc需要修改配置项后编译
```config_windows.h 
#define HAVE_STLPORT 1
Compile with stlport instead of standard visual c++ STL implementation.
#define HAVE_ODBC 1
Compiles ODBC support into QuickFIX.
#define HAVE_MYSQL 1
Compiles MySQL support into QuickFIX. If you enable this option, the mysql include and library directories must be in the Visual Studio search paths.
#define HAVE_POSTGRESQL 1
Compiles PostgreSQL support into QuickFIX. If you enable this option, the postgres include and library directories must be in the Visual Studio search paths.
```
| 目录  | 静态/动态 | debug/release | 64/32 |
| :---: | :-------- | :------------ | :---- |
| mtd64 | 静态      | debug         | 64    |
| mtd32 | 静态      | debug         | 32    |
| mt64  | 静态      | release       | 64    |
| mt32  | 静态      | release       | 32    |
**目前编译仅支持mysql57**
## 数据库初始化

sql文件位于/src/sql/mysql文件夹下

## 测试
 
 待测  
# 简单启动
## 项目设置
```基本配置

    C/C++ | Code Generation | Enable C++ Exceptions, should be Yes.
    C/C++ | Code Generation | Runtime Library, should be set to Multithreaded DLL or Debug Multithreaded DLL.
    C/C++ | General | Additional Include Directories add the root quickfix directory.
    Linker | Input | Additional Dependencies, must contain quickfix.lib and ws2_32.lib.
    Linker | General | Additional Library Directories, add the quickfix/lib directory.

```

## 应用生成

在C++中只需要实现Application接口就可以了。  
其接口如下： 
```
namespace FIX
{
/**
 * This interface must be implemented to define what your %FIX application
 * does.
 *
 * These methods notify your application about events that happen on
 * active %FIX sessions. There is no guarantee how many threads will be calling
 * these functions. If the application is sharing resources among multiple sessions,
 * you must synchronize those resources. You can also use the SynchronizedApplication
 * class to automatically synchronize all function calls into your application.
 * The various MessageCracker classes can be used to parse the generic message
 * structure into specific %FIX messages.
 */
class Application
{
public:
  virtual ~Application() {};
  /// Notification of a session begin created
  virtual void onCreate( const SessionID& ) = 0;
  /// Notification of a session successfully logging on
  virtual void onLogon( const SessionID& ) = 0;
  /// Notification of a session logging off or disconnecting
  virtual void onLogout( const SessionID& ) = 0;
  /// Notification of admin message being sent to target
  virtual void toAdmin( Message&, const SessionID& ) = 0;
  /// Notification of app message being sent to target
  virtual void toApp( Message&, const SessionID& )
  throw( DoNotSend ) = 0;
  /// Notification of admin message being received from target
  virtual void fromAdmin( const Message&, const SessionID& )
  throw( FieldNotFound, IncorrectDataFormat, IncorrectTagValue, RejectLogon ) = 0;
  /// Notification of app message being received from target
  virtual void fromApp( const Message&, const SessionID& )
  throw( FieldNotFound, IncorrectDataFormat, IncorrectTagValue, UnsupportedMessageType ) = 0;
};
```
其实就是Application声明周期中的监听或者过滤器。主要包含session创建，登录，登出，以及应用通信（发送与接收），会话（发送与接收）  
服务端demo:其中存储，日志使用了简单工厂的方法。
```
#include "quickfix/FileStore.h"
#include "quickfix/FileLog.h"
#include "quickfix/SocketAcceptor.h"
#include "quickfix/Session.h"
#include "quickfix/SessionSettings.h"
#include "quickfix/Application.h"

int main( int argc, char** argv )
{
  try
  {
    if(argc < 2) return 1;
    std::string fileName = argv[1];

    FIX::SessionSettings settings(fileName);

    MyApplication application;
    FIX::FileStoreFactory storeFactory(settings);
    FIX::FileLogFactory logFactory(settings);
    FIX::SocketAcceptor acceptor
      (application, storeFactory, settings, logFactory /*optional*/);
    acceptor.start();
    // while( condition == true ) { do something; }
    acceptor.stop();
    return 0;
  }
  catch(FIX::ConfigError& e)
  {
    std::cout << e.what();
    return 1;
  }
}
```
## 配置
一个quickfix 服务端或者客户端 可以管理多个FIX sessions.一个FIX session中需要指定版本号（BeginString），当前应用ID(SenderCompID),对方ID(TargetCompID).并且SessionQualifier 可以区分相同session之间的歧义。  
SessionSettings类的创建可以通过文件名或者输入流(c++)进行构造。   
配置文件采用ini风格的格式。主要有两种头控制（DEFAULT和SESSION），DEFAULT表示所有SESSION采用的默认配置，SESSION表示定义一个会话。（如果配置错误将会返回一个ConfigError异常）
配置文件demo
```
# default settings for sessions
[DEFAULT]
ConnectionType=initiator
ReconnectInterval=60
SenderCompID=TW

# session definition
[SESSION]
# inherit ConnectionType, ReconnectInterval and SenderCompID from default
BeginString=FIX.4.1
TargetCompID=ARCA
StartTime=12:30:00
EndTime=23:30:00
HeartBtInt=20
SocketConnectPort=9823
SocketConnectHost=123.123.123.123
DataDictionary=somewhere/FIX41.xml

[SESSION]
BeginString=FIX.4.0
TargetCompID=ISLD
StartTime=12:00:00
EndTime=23:00:00
HeartBtInt=30
SocketConnectPort=8323
SocketConnectHost=23.23.23.23
DataDictionary=somewhere/FIX40.xml

[SESSION]
BeginString=FIX.4.2
TargetCompID=INCA
StartTime=12:30:00
EndTime=21:30:00
# overide default setting for RecconnectInterval
ReconnectInterval=30
HeartBtInt=30
SocketConnectPort=6523
SocketConnectHost=3.3.3.3
# (optional) alternate connection ports and hosts to cycle through on failover
SocketConnectPort1=8392
SocketConnectHost1=8.8.8.8
SocketConnectPort2=2932
SocketConnectHost2=12.12.12.12
DataDictionary=somewhere/FIX42.xml
```
### 配置介绍
<table>
<tr><td colspan=4>SESSION</td></tr>
<tr><td>配置项</td><td>描述</td><td>域</td><td>默认值</td></tr>
<tr><td>BeginString</td><td>版本</td><td>FIXT1.1 FIX.4.4 FIX.4.3 FIX.4.2 FIX.4.1 FIX.4.0</td><td></td></tr>
<tr><td>SenderCompID</td><td>当前FIX sessionID</td><td>区分大小写的字母数字字符串</td><td></td></tr>
<tr><td>TargetCompID</td><td>连接对方SESSION</td><td>区分大小写的字母数字字符串</td><td></td></tr>
<tr><td>SessionQualifier</td><td>附加识别信息</td><td>区分大小写的字母数字字符串</td><td></td></tr>
<tr><td>DefaultApplVerID</td><td>仅在FIXT 1.1(及更新版本)中需要。</td><td>FIX.4.2 ...</td><td></td></tr>
<tr><td>ConnectionType</td><td>连接类型</td><td>initiator/acceptor</td><td></td></tr>
<tr><td>StartTime</td><td>会话激活时间</td><td>时间格式HH:MM:SS，时间用UTC表示</td><td></td></tr>
<tr><td>EndTime</td><td>会话释放时间</td><td>时间格式HH:MM:SS，时间用UTC表示</td><td></td></tr>
<tr><td>StartDay</td><td>周session开始时间配置</td><td>星期几英文缩写(即mo, mon, mond, monda, monday均有效)</td><td></td></tr>
<tr><td>EndDay</td><td>周session结束时间配置</td><td>星期几英文缩写(即mo, mon, mond, monda, monday均有效)</td><td></td></tr>
<tr><td>LogonTime</td><td>此会话登录的时间</td><td>时间格式HH:MM:SS，时间用UTC表示</td><td>SessionTime value</td></tr>
<tr><td>LogoutTime</td><td>此会话登出时间</td><td>时间格式HH:MM:SS，时间用UTC表示</td><td>EndTime value</td></tr>
<tr><td>LogonDay</td><td>周session开始时间配置</td><td>星期几英文缩写(即mo, mon, mond, monda, monday均有效)</td><td>StartDay value</td></tr>
<tr><td>LogoutDay</td><td>周session结束时间配置</td><td>星期几英文缩写(即mo, mon, mond, monda, monday均有效)</td><td>EndDay value</td></tr>
<tr><td>UseLocalTime</td><td>表示启动时间和结束时间以本地时间而不是UTC表示。消息中的时间仍然设置为UTC，因为这是FIX规范所要求的</td><td>Y/N</td><td>N</td></tr>
<tr><td>MillisecondsInTimeStamp</td><td>确定是否应该将毫秒添加到时间戳中。仅适用于修复。4.2及以上版本。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>TimestampPrecision</td><td>用于设置时间戳的小数部分。允许的值是0到9。如果设置，则重写毫秒时间戳。</td><td>0-9</td><td></td></tr>
<tr><td>SendRedundantResendRequests</td><td>如果设置为Y, QuickFIX将发送所有必要的重发请求，即使它们看起来是多余的。一些系统将不会认证引擎，除非它这样做。当设置为N时，QuickFIX将尝试最小化重发请求。这在大容量系统中特别有用。</td><td>Y/N</td><td>N</td></tr>
<tr><td>ResetOnLogon</td><td>重新登录时序号是否重置。仅Acceptors有效</td><td>Y/N</td><td>N</td></tr>
<tr><td>ResetOnLogout</td><td>正常注销登出后是否序号是否重置为1</td><td>Y/N</td><td>N</td></tr>
<tr><td>ResetOnDisconnect</td><td>异常终止后序号是否重置为1</td><td>Y/N</td><td>N</td></tr>
<tr><td>RefreshOnLogon</td><td>确定登录时是否应从持久层恢复会话状态。用于创建热故障转移会话。</td><td>Y/N</td><td>N</td></tr>
<tr><td colspan=4>Validation</td></tr>
<tr><td>UseDataDictionary</td><td>告诉会话是否需要数据字典。如果使用重复组，应该始终使用DataDictionary。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>DataDictionary</td><td>用于验证传入修复消息的XML定义文件。如果没有提供DataDictionary，则只执行基本的消息验证  此设置仅适用于FIX .1.1之前的FIX传输版本。</td><td>有效的XML数据字典文件，QuickFIX在spec目录中提供了以下默认设置<br/> FIX44.xml FIX43.xml FIX42.xml FIX41.xml FIX40.xml</td><td></td></tr>
<tr><td>AppDataDictionary</td><td>应用数据字典。验证应用程序消息的XML定义文件。此设置仅对fix .1.1(或更新的)会话有效。有关旧传输版本(FIX.4.0 - FIX.4.4)的更多信息，请参见DataDictionary。
此设置支持为每个会话创建自定义应用程序数据字典的可能性。此设置仅用于FIXT 1.1和新的传输协议。此设置可以用作前缀，为FIXT传输指定多个应用程序字典。例如:<br/>   
DefaultApplVerID=FIX.4.2 <br/> 
# For default application version ID<br/> 
AppDataDictionary=FIX42.xml<br/> 
# For nondefault application version ID<br/> 
# Use BeginString suffix for app version<br/> 
AppDataDictionary.FIX.4.4=FIX44.xml<br/> 
</td><td>有效的XML数据字典文件，QuickFIX在spec目录中提供了以下默认设置 FIX50SP2.xml
FIX50SP1.xml FIX50.xml FIX44.xml FIX43.xml FIX42.xml FIX41.xml  FIX40.xml<br/></td><td></td></tr>
<tr><td>ValidateLengthAndChecksum</td><td>如果设置为N，长度或校验和字段不正确的消息将不会被拒绝。您还可以使用它强制接受没有数据字典的重复组。在这个场景中，您将无法访问所有重复的组。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>ValidateFieldsOutOfOrder</td><td>如果设置为N，无序的字段(即header中的body字段，或body中的header字段)将不会被拒绝。用于连接没有正确排序字段的系统。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>ValidateFieldsHaveValues</td><td>如果设置为N，没有值(空)的字段将不会被拒绝。用于连接不正确发送空标记的系统。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>ValidateUserDefinedFields</td><td>如果将用户定义的字段设置为N，则不会拒绝数据字典中未定义的字段，或在不属于这些字段的消息中出现的字段。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>PreserveMessageFieldsOrder</td><td>是否应该保留主传出消息主体中字段的顺序(如配置文件中定义的那样)。默认值:只保留指定的组顺序。</td><td>Y/N</td><td>N</td></tr>
<tr><td>CheckCompID</td><td>如果设置为Y，则必须使用正确的SenderCompID和TargetCompID从对方接收消息。有些系统会根据设计发送不同的compid，因此必须将其设置为N。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>CheckLatency</td><td>如果设置为Y，则必须在定义的秒数内从对方接收消息(参见MaxLatency)。如果系统使用localtime而不是GMT作为时间戳，那么关闭这个选项是很有用的。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>MaxLatency</td><td>如果将CheckLatency设置为Y，则定义允许处理消息的秒延迟数。默认是120。</td><td>正整数</td><td>120</td></tr>
<tr><td colspan=4>其他配置</td></tr>
<tr><td>HttpAcceptPort</td><td>端口监听HTTP请求。将浏览器指向该端口将显示一个控制面板。必须在默认区域中。</td><td>正整数</td><td></td></tr>
<tr><td colspan=4>客户端Initiator</td></tr>
<tr><td>ReconnectInterval</td><td>重新连接尝试之间的时间，以秒为单位。仅用于客户端</td><td>正整数</td><td>30</td></tr>
<tr><td>HeartBtInt</td><td>以秒为单位的心跳间隔。仅用于启动器。</td><td>正整数</td><td></td></tr>
<tr><td>LogonTimeout</td><td>断开连接之前等待登录响应的秒数。</td><td>正整数</td><td>10</td></tr>
<tr><td>LogoutTimeout</td><td>断开连接之前等待注销响应的秒数。</td><td>正整数</td><td>2</td></tr>
<tr><td>SocketConnectPort</td><td>用于连接到会话的套接字端口。仅与SocketInitiator一起使用</td><td>正整数</td><td></td></tr>
<tr><td>SocketConnectHost</td><td>要连接的主机。仅与SocketInitiator一起使用</td><td>x.x.x.x格式的有效IP地址。x或域名</td><td></td></tr>
<tr><td>SocketConnectPort n</td><td>用于连接到故障转移会话的备用套接字端口，其中n是一个正整数。(即)。SocketConnectPort1 SocketConnectPort2…必须连续且具有匹配的SocketConnectHost[n]</td><td>正整数</td><td></td></tr>
<tr><td>SocketConnectHost n</td><td>用于连接到故障转移会话的备用套接字主机，其中n是一个正整数。(即)。SocketConnectHost1 SocketConnectHost2…必须连续且具有匹配的SocketConnectPort[n]</td><td>x.x.x.x格式的有效IP地址。x或域名</td><td></td></tr>
<tr><td colspan=4>服务端Acceptor</td></tr>
<tr><td>SocketAcceptPort</td><td>套接字端口，用于监听传入连接，仅与SocketAcceptor一起使用</td><td>正整数,有效的开放套接字端口。目前，这必须在[DEFAULT]部分中定义。</td><td></td></tr>
<tr><td>SocketReuseAddress</td><td>指示套接字应使用SO_REUSADDR创建，仅与SocketAcceptor一起使用</td><td>Y/N</td><td>Y</td></tr>
<tr><td>SocketNodelay</td><td>指示应使用TCP_NODELAY创建套接字。目前，这必须在[DEFAULT]部分中定义。</td><td>Y/N</td><td>N</td></tr>
<tr><td colspan=4>存储</td></tr>
<tr><td>PersistMessages</td><td>如果设置为N，则不会持久化任何消息。这将迫使QuickFIX总是发送空白，而不是重新发送消息。如果您知道您永远不想重新发送消息，请使用此功能。适用于行情数据流。</td><td>Y/N</td><td>N</td></tr>
<tr><td colspan=4>文件</td></tr>
<tr><td>FileStorePath</td><td>目录来存储序列号和消息文件。</td><td>用于存储文件的有效目录，必须具有写访问权限</td><td></td></tr>
<tr><td colspan=4>MYSQL</td></tr>
<tr><td>MySQLStoreDatabase</td><td>用于存储消息和会话状态的MySQL数据库访问的名称。</td><td>有效的数据库存储文件，必须有写访问和正确的数据库shema</td><td>quickfix</td></tr>
<tr><td>MySQLStoreUser</td><td>用户名</td><td></td><td>root</td></tr>
<tr><td>MySQLStorePassword</td><td>密码</td><td></td><td>空</td></tr>
<tr><td>MySQLStoreHost</td><td>地址</td><td>x.x.x格式的有效IP地址。x或域名</td><td>localhost</td></tr>
<tr><td>MySQLStorePort</td><td>端口</td><td>正整数</td><td>3306</td></tr>
<tr><td>MySQLStoreUseConnectionPool</td><td>使用数据库连接池。如果可能，会话将共享单个数据库连接。否则，每个会话都有自己的连接。</td><td>Y/N</td><td>N</td></tr>
<tr><td colspan=4>POSTGRESQL</td></tr>
<tr><td>PostgreSQLStoreDatabase</td><td>用于存储消息和会话状态的MySQL数据库访问的名称。</td><td>有效的数据库存储文件，必须有写访问和正确的数据库shema</td><td>quickfix</td></tr>
<tr><td>PostgreSQLStoreUser</td><td>用户名</td><td></td><td>postgres</td></tr>
<tr><td>PostgreSQLStorePassword</td><td>密码</td><td></td><td>空</td></tr>
<tr><td>PostgreSQLStoreHost</td><td>地址</td><td>x.x.x格式的有效IP地址。x或域名</td><td>localhost</td></tr>
<tr><td>PostgreSQLStorePort</td><td>端口</td><td>正整数</td><td>默认标准端口</td></tr>
<tr><td>PostgreSQLStoreUseConnectionPool</td><td>使用数据库连接池。如果可能，会话将共享单个数据库连接。否则，每个会话都有自己的连接。</td><td>Y/N</td><td>N</td></tr>
<tr><td colspan=4>ODBC</td></tr>
<tr><td>OdbcStoreUser</td><td>登录到ODBC数据库的用户名。</td><td>具有读写权限的</td><td>sa</td></tr>
<tr><td>OdbcStorePassword</td><td>密码</td><td>正确的ODBC密码。如果PWD在OdbcStoreConnectionString中，则忽略。</td><td>空</td></tr>
<tr><td>OdbcStoreConnectionString</td><td>用于数据库的ODBC连接字符串</td><td>有效ODBC连接字符串</td><td>DATABASE=quickfix;DRIVER={SQL Server};SERVER=(local);</td></tr>

<tr><td colspan=4>Logging日志</td></tr>
<tr><td colspan=4>File 文件</td></tr>
<tr><td>FileLogPath</td><td>存储日志的目录</td><td>用于存储文件的有效目录，必须具有写访问权限</td><td></td></tr>
<tr><td>FileLogBackupPath</td><td>存储备份日志的目录。</td><td>用于存储备份文件的有效目录，必须具有写访问权限</td><td></td></tr>
<tr><td colspan=4>SCREEN 控制台</td></tr>
<tr><td>ScreenLogShowIncoming</td><td>将传入的消息打印到标准输出。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>ScreenLogShowOutgoing</td><td>输出消息到标准输出。</td><td>Y/N</td><td>Y</td></tr>
<tr><td>ScreenLogShowEvents</td><td>将事件打印到标准输出。</td><td>Y/N</td><td>Y</td></tr>
<tr><td colspan=4>MYSQL</td></tr>
<tr><td>MySQLLogDatabase</td><td>用于存储日志的MySQL数据库访问的名称。</td><td>有效的数据库存储文件，必须有写访问和正确的数据库shema</td><td>quickfix</td></tr>
<tr><td>MySQLLogUser</td><td>用户名</td><td></td><td>root</td></tr>
<tr><td>MySQLLogPassword</td><td>密码</td><td></td><td>空</td></tr>
<tr><td>MySQLLogHost</td><td>地址</td><td>x.x.x格式的有效IP地址。x或域名</td><td>localhost</td></tr>
<tr><td>MySQLLogPort</td><td>端口</td><td>正整数</td><td>3306</td></tr>
<tr><td>MySQLLogUseConnectionPool</td><td>使用数据库连接池。如果可能，会话将共享单个数据库连接。否则，每个会话都有自己的连接。</td><td>Y/N</td><td>N</td></tr>
<tr><td>MySQLLogIncomingTable</td><td>记录传入消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>MySQLLogOutgoingTable</td><td>记录传出消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>MySQLLogEventTable</td><td>记录事件的表的名称。</td><td>具有正确模式的有效表。</td><td>event_log</td></tr>
<tr><td colspan=4>POSTGRESQL</td></tr>
<tr><td>PostgreSQLLogDatabase</td><td>用于存储日志的MySQL数据库访问的名称。</td><td>有效的数据库存储文件，必须有写访问和正确的数据库shema</td><td>quickfix</td></tr>
<tr><td>PostgreSQLLogUser</td><td>用户名</td><td></td><td>root</td></tr>
<tr><td>PostgreSQLLogPassword</td><td>密码</td><td></td><td>空</td></tr>
<tr><td>PostgreSQLLogHost</td><td>地址</td><td>x.x.x格式的有效IP地址。x或域名</td><td>localhost</td></tr>
<tr><td>PostgreSQLLogPort</td><td>端口</td><td>正整数</td><td>3306</td></tr>
<tr><td>PostgreSQLLogUseConnectionPool</td><td>使用数据库连接池。如果可能，会话将共享单个数据库连接。否则，每个会话都有自己的连接。</td><td>Y/N</td><td>N</td></tr>
<tr><td>PostgresSQLLogIncomingTable</td><td>记录传入消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>PostgresSQLLogOutgoingTable</td><td>记录传出消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>PostgresSQLLogEventTable</td><td>记录事件的表的名称。</td><td>具有正确模式的有效表。</td><td>event_log</td></tr>
<tr><td colspan=4>ODBC</td></tr>
<tr><td>OdbcLogUser</td><td>用户名</td><td></td><td>sa</td></tr>
<tr><td>OdbcLogPassword</td><td>密码</td><td></td><td>空</td></tr>
<tr><td>OdbcLogConnectionString</td><td>连接信息</td><td></td><td>DATABASE=quickfix;DRIVER={SQL Server};SERVER=(local);</td></tr>
<tr><td>OdbcLogIncomingTable</td><td>记录传入消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>OdbcLogOutgoingTable</td><td>记录传出消息的表的名称</td><td>具有正确模式的有效表</td><td>messages_log</td></tr>
<tr><td>OdbcLogEventTable</td><td>记录事件的表的名称。</td><td>具有正确模式的有效表。</td><td>event_log</td></tr>
<tr><td colspan=4>SSL(DEFAULT section)</td></tr>
<tr><td>SSLProtocol</td><td>主要指定ssl的实现 SSLv2 SSLv3 TLSv1 ...</td><td></td><td>all -SSLv2</td></tr>
<tr><td>SSLCipherSuite</td><td>cipher-spec中的SSL密码规范由4个主要属性和一些额外的次要属性组成。<br/>Key Exchange <br/>Algorithm:<br/>
RSA or Diffie-Hellman variants.<br/>
Authentication Algorithm:<br/>
RSA, Diffie-Hellman, DSS or none.<br/>
Cipher/Encryption Algorithm:<br/>
DES, Triple-DES, RC4, RC2, IDEA or none.<br/>
MAC Digest Algorithm:<br/>
MD5, SHA or SHA1.<br/>
For more details refer to mod_ssl documentation.</td><td></td><td>HIGH:!RC4</td></tr>
<tr><td>CertificationAuthoritiesFile</td><td>该指令设置了一个一体化文件，您可以在其中组装与之打交道的客户端的证书颁发机构(CA)的证书。</td><td></td><td></td></tr>
<tr><td>CertificationAuthoritiesDirectory</td><td>该指令设置保存与之打交道的客户端的证书颁发机构(CAs)证书的目录。</td><td></td><td></td></tr>
<tr><td colspan=4>服务端ACCEPTOR</td></tr>
<tr><td>ServerCertificateFile</td><td>该配置指向pems编码的证书文件，也可以选择指向对应的RSA或DSA私钥文件(包含在同一个文件中)。</td><td></td><td></td></tr>
<tr><td>ServerCertificateKeyFile</td><td>该配置指向pems编码的私钥文件。如果私钥没有与服务器证书文件中的证书结合，则使用此附加指令指向具有独立私钥的文件。</td><td></td><td></td></tr>
<tr><td>CertificateVerifyLevel</td><td>该配置设置证书验证级别。它适用于在建立连接时在标准SSL握手中使用的身份验证过程。0表示不验证。1意味着验证。</td><td></td><td></td></tr>
<tr><td>CertificateRevocationListFile</td><td>该配置设置了一个一体化文件，您可以在其中组装与之打交道的客户端的证书颁发机构(CA)的证书撤销列表(CRL)。</td><td></td><td></td></tr>
<tr><td>CertificateRevocationListDirectory</td><td>此配置设置保存与之打交道的证书颁发机构(CAs)的证书撤销列表(CRL)的目录。</td><td></td><td></td></tr>
<tr><td colspan=4>客户端INITIATOR</td></tr>
<tr><td>ClientCertificateFile</td><td>该配置指向pems编码的证书文件，也可以选择指向对应的RSA或DSA私钥文件(包含在同一个文件中)。</td><td></td><td></td></tr>
<tr><td>ClientCertificateKeyFile</td><td>该指令指向pems编码的私钥文件。如果私钥没有与服务器证书文件中的证书结合，则使用此附加指令指向具有独立私钥的文件。</td><td></td><td></td></tr>
</table>

### 校验
QuickFIX将在消息到达应用程序之前验证并拒绝任何格式不佳的消息。XML文件定义会话支持的消息、字段和值。  
spec目录中包含几个标准文件。  
定义文件的骨架是这样的。  
```  
<fix type="FIX" major="4" minor="1">
  <header>
    <field name="BeginString" required="Y"/>
    ...
  </header>
  <trailer>
    <field name="CheckSum" required="Y"/>
    ...
  </trailer>
  <messages>
    <message name="Heartbeat" msgtype="0" msgcat="admin">
      <field name="TestReqID" required="N"/>
    </message>
    ...
    <message name="NewOrderSingle" msgtype="D" msgcat="app">
      <field name="ClOrdID" required="Y"/>
      ...
    </message>
    ...
  </messages>
  <fields>
    <field number="1" name="Account" type="CHAR" />
    ...
    <field number="4" name="AdvSide" type="CHAR">
     <value enum="B" description="BUY" />
     <value enum="S" description="SELL" />
     <value enum="X" description="CROSS" />
     <value enum="T" description="TRADE" />
   </field>
   ...
  </fields>
</fix>
``` 
# 利用Message进行工作
## 接受消息
您感兴趣的大多数消息将到达您的应用程序重载的fromApp函数中。所有消息都有一个标题和一个尾部。如果想查看这些字段，必须在消息上调用getHeader()或getTrailer()来访问它们。    
### 类型安全的消息和字段
QuickFIX为标准规范中定义的所有消息提供了一个类。  
```
void fromApp(const FIX::Message& message,const FIX::SessionID& sessionID) throw(FIX::FieldNotFound&,FIX::IncorrectDataFormat&,FIX::IncorrectTagValue&,FIX::UnsupportedMessageType&){
    crack(messsage,sessionID);
}
void onMessage(const FIX42::NewOrderSingle& message,const FIX::SessionID&){
    FIX::ClOrdID clOrdID;
    message.get(clOrdID);
    FIX::ClearingAccount clearingAccount;
    message.get(clearingAccount);
}
void onMessage(const FIX41::NewOrderSingle& message,const FIX::SessionID&){
    FIX::ClOrdID clOrdID;
    message.get(clOrdID);

    // compile time error!! field not defined in FIX41
    FIX::ClearingAccount clearingAccount;
    message.get(clearingAccount);
}
void onMessage( const FIX42::OrderCancelRequest& message, const FIX::SessionID& ){
    FIX::ClOrdID clOrdID;
     message.get(clOrdID);

    // compile time error!! field not defined for OrderCancelRequest
      FIX::Price price;
     message.get(price);
}
```
为了处理方便，需要实现MessageCracker接口，那么针对已经处理的类型就可以调用重载的方法。对于没有重载的的消息类型会抛出UnsupportedMessageType 异常
使用方式如下：  
``` 
#include "quickfix/Application.h"
#include "quickfix/MessageCracker.h"

class MyApplication
: public FIX::Application,
  public FIX::MessageCracker
```
### 仅使用类型安全字段
使用getField方法从任何消息中获取任何字段。
```
void fromApp( const FIX::Message& message, const FIX::SessionID& sessionID )
  throw( FIX::FieldNotFound&, FIX::IncorrectDataFormat&, FIX::IncorrectTagValue&, FIX::UnsupportedMessageType& )
{
  // retrieve value into field class
  FIX::Price price;
  message.getField(price);

  // another field...
  FIX::ClOrdID clOrdID;
  message.getField(clOrdID);
}
```
### 不适用使用类型安全字段
使用标记号创建您自己的字段。  
```
void fromApp( const FIX::Message& message, const FIX::SessionID& sessionID )
  throw( FIX::FieldNotFound&, FIX::IncorrectDataFormat&, FIX::IncorrectTagValue&, FIX::UnsupportedMessageType& )
{
  // retreive value into string with integer field ID
  std::string value;
  value = message.getField(44);

  // retrieve value into a field base with integer field ID
  FIX::FieldBase field(44, "");
  message.getField(field);

  // retreive value with an enumeration, a little better
  message.getField(FIX::FIELD::Price);
}
```
## 发送消息
可以使用静态Session::sendToTarget方法将消息发送给通信方。    
这些接口主要有  
```
// send a message that already contains a BeginString, SenderCompID, and a TargetCompID
static bool sendToTarget( Message&, const std::string& qualifier = "" )
  throw(SessionNotFound&);

// send a message based on the sessionID, convenient for use
// in fromApp since it provides a session ID for incoming
// messages
static bool sendToTarget( Message&, const SessionID& )
  throw(SessionNotFound&);

// append a SenderCompID and TargetCompID before sending
static bool sendToTarget( Message&, const SenderCompID&, const TargetCompID&, const std::string& qualifier = "" )
  throw(SessionNotFound&);

// pass SenderCompID and TargetCompID in as strings
static bool sendToTarget( Message&, const std::string&, const std::string&, const std::string& qualifier = "" )
  throw(SessionNotFound&)
```
**类型安全的消息和域**
消息构造函数接受所有必需的字段，并为您添加正确的MsgType和BeginString。使用set方法，编译器将不允许您添加不属于规范定义的消息的一部分的字段。  
```
void sendOrderCancelRequest(){
  FIX42::OrderCancelRequest message(FIX::OrigClOrdID("123"),fix::ClOrdID("321"),FIX::Symbol("LNUX"),FIX::Side(FIX::Side_Buy));
  message.set(FIX::Text("Cancel My Order!"));
  FIX::Session::sendToTarget(message,SenderCompID("TW"),TargetCompID("TARGET"));
}
```
**只使用类型安全的域**
setField方法允许您向任何消息添加任何字段。  
```
void sendOrderCancelRequest(){
  FIX::Message message;
  FIX::Header header& =message.getHeader();
  header.setField(FIX::BeginString("FIX4.2"));
  header.setField(FIX::SenderCompID(TW));
  header.setField(FIX::TargetCompID("TARGET"));
  header.setField(FIX::MsgType(FIX::MsgType_OrderCancelRequest));
  message.setField(FIX::OrigClOrdID("123"));
  message.setField(FIX::ClOrdID("321"));
  message.setField(FIX::Symbol("LNUX"));
  message.setField(FIX::Side(FIX::Side_BUY));
  message.setField(FIX::Text("Cancel My Order!"));

  FIX::Session::sendToTarget(message);
}
```
**非类型安全的**
你可以使用使用setField完成消息构建   
```
void sendOrderCancelRequest()
{
  FIX::Message message;
  // BeginString
  message.getHeader().setField(8, "FIX.4.2");
  // SenderCompID
  message.getHeader().setField(49, "TW");
  // TargetCompID, with enumeration
  message.getHeader().setField(FIX::FIELD::TargetCompID, "TARGET");
  // MsgType
  message.getHeader().setField(35, 'F');
  // OrigClOrdID
  message.setField(41, "123");
  // ClOrdID
  message.setField(11, "321");
  // Symbol
  message.setField(55, "LNUX");
  // Side, with value enumeration
  message.setField(54, FIX::Side_BUY);
  // Text
  message.setField(58, "Cancel My Order!");

  FIX::Session::sendToTarget(message);
}
```
## 重复组
QuickFIX能够发送包含重复组甚至递归重复组的消息。所有重复组都以一个字段开始，该字段指示在一个集合中有多少重复组。可以通过引用在父消息或组的范围内以该字段命名的类来创建组。    
**发送重复组**  
创建消息时，声明重复组数量的required字段被设置为零。QuickFIX会在添加组时自动增加字段。  
```
//行情消息
FIX42::MarketDataSnapshotFullRefresh message(FIX::Symbol("QF"));
//以MessageName::NoField的形式重复组
FIX42::MarketDataSnapshotFullRefresh::NoMDEntries group;
group.set(FIX::MDEntryType('0'));
group.set(FIX::MDEntryPx(12.32));
group.set(FIX::MDEntrySize(100));
group.set(FIX::OrderID("ORDERID"));
message.addGroup(group);
 // 对于重用的域不需要重新生成一个对象
    group.set(FIX::MDEntryType('1'));
    group.set(FIX::MDEntryPx(12.32));
    group.set(FIX::MDEntrySize(100));
    group.set(FIX::OrderID("ORDERID"));
    message.addGroup(group);

```
**接收重复组**  
从消息中提取组，需要提供希望提取的组对象。您应该首先检查条目字段的数量(该组以该字段命名)，以获得组的总数。上面创建的消息现在在这个示例中解析。  
```
    FIX::NoMDEntryies noMDEntries;
    message.get(noMDEntries);
    FIX42::MarketDataSnapshotFullRefresh::NoMDEntries group;
    FIX::MDEntryType MDEntryType;
    FIX::MDEntryPx MDEntryPx;
    FIX::MDEntrySize MDEntrySize;
    FIX::OrderID orderID;

    message.getGroup(1, group);
    group.get(MDEntryType);
    group.get(MDEntryPx);
    group.get(MDEntrySize);
    group.get(orderID);

    message.getGroup(2, group);
    group.get(MDEntryType);
    group.get(MDEntryPx);
    group.get(MDEntrySize);
    group.get(orderID);
```
## 用户定义域
FIX允许用户定义规范中未定义的字段。如何使用QuickFIX设置和获取用户定义的字段?一种方式是使用非类型安全集，得到这样的字段:  
```
message.setField(6123,"value");
message.setField(6123);
```
QuickFIX还提供了用于创建类型安全字段对象的宏。  
```
#include  "quickfix/Field.h"
namespace FIX{
  USER_DEFINE_STRING(MyStringField,6123);
  USER_DEFINE_PRICE(MyPriceField,8756)
}
```
必须在FIX命名空间中声明用户定义的字段。现在，在你的应用程序的其他地方，你可以这样写代码:  
```
MyStringField stringField("string");
MyPriceField priceField(14.54);
message.setField(stringField);
message.setField(priceField);

message.getField(stringField);
message.getField(priceField);
```
这些宏允许您定义所有受支持的修复类型的字段。请记住，您可以编写任何类型类型的字段，只要您提供一个新的宏和转换器，它可以将您的类型转换为字符串或从字符串转换为字符串。  
```
USER_DEFINE_STRING( NAME, NUM )
USER_DEFINE_CHAR( NAME, NUM )
USER_DEFINE_PRICE( NAME, NUM )
USER_DEFINE_INT( NAME, NUM )
USER_DEFINE_AMT( NAME, NUM )
USER_DEFINE_QTY( NAME, NUM )
USER_DEFINE_CURRENCY( NAME, NUM )
USER_DEFINE_MULTIPLEVALUESTRING( NAME, NUM )
USER_DEFINE_EXCHANGE( NAME, NUM )
USER_DEFINE_UTCTIMESTAMP( NAME, NUM )
USER_DEFINE_BOOLEAN( NAME, NUM )
USER_DEFINE_LOCALMKTDATE( NAME, NUM )
USER_DEFINE_DATA( NAME, NUM )
USER_DEFINE_FLOAT( NAME, NUM )
USER_DEFINE_PRICEOFFSET( NAME, NUM )
USER_DEFINE_MONTHYEAR( NAME, NUM )
USER_DEFINE_DAYOFMONTH( NAME, NUM )
USER_DEFINE_UTCDATE( NAME, NUM )
USER_DEFINE_UTCTIMEONLY( NAME, NUM )
USER_DEFINE_NUMINGROUP( NAME, NUM )
USER_DEFINE_SEQNUM( NAME, NUM )
USER_DEFINE_LENGTH( NAME, NUM )
USER_DEFINE_PERCENTAGE( NAME, NUM )
USER_DEFINE_COUNTRY( NAME, NUM )
```
## Demo例子
可以在源码中的examples目录中找到
ordermatch c++的服务端(可以撮合和执行限价单)
executor 服务端可以撮合所有的限价单
tradeclient c++客户端（基于控制台)
trideclientgui (java gui客户端)
# 测试
## 单元测试
## 接收测试