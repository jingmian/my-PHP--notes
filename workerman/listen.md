### Worker

- [listen](http://doc.workerman.net/worker/listen.html)
--用于实例化Worker后执行监听。此方法主要用于在Worker进程启动后动态创建新的Worker实例，能够实现同一个进程监听多个端口，支持多种协议。需要注意的是用这种方法只是在当前进程增加监听，并不会动态创建新的进程，也不会触发onWorkerStart方法.既是同一个端口监听多种协议.

新建一个`test.php`,`php`版本必须大于`7.0`
```php
use Workerman\Worker;
require_once __DIR__ . '/Workerman/Autoloader.php';


// 初始化一个worker容器，监听9501端口
$worker = new Worker('websocket://0.0.0.0:9501');

/*
 * 注意这里进程数必须设置为1，否则会报端口占用错误
 * (php 7可以设置进程数大于1，前提是$inner_text_worker->reusePort=true)
 */
$worker->count = 1;
// worker进程启动后创建一个text Worker以便打开一个内部通讯端口
$worker->onWorkerStart = function($worker)
{
    // 开启一个内部端口，方便内部系统推送数据，Text协议格式 文本+换行符
    $inner_text_worker = new Worker('text://0.0.0.0:9501');

    $inner_text_worker->onMessage = function($connection, $buffer)
    {
var_dump($buffer);
        // $data数组格式，里面有uid，表示向那个uid的页面推送数据
        $data = json_decode($buffer, true);

        $uid = $data['uid'];
        // 通过workerman，向uid的页面推送数据
        $ret = sendMessageByUid($uid, $buffer);
        // 返回推送结果
        $connection->send($ret ? 'ok' : 'fail');
    };
    // ## 执行监听 ##
    $inner_text_worker->listen();
};
// 新增加一个属性，用来保存uid到connection的映射
$worker->uidConnections = array();
// 当有客户端发来消息时执行的回调函数
$worker->onMessage = function($connection, $data)
{
    global $worker;
var_dump(2222);
    // 判断当前客户端是否已经验证,既是否设置了uid
    if(!isset($connection->uid))
    {
        // 没验证的话把第一个包当做uid（这里为了方便演示，没做真正的验证）
        $connection->uid = $data;
        /* 保存uid到connection的映射，这样可以方便的通过uid查找connection，
         * 实现针对特定uid推送数据
         */
        $worker->uidConnections[$connection->uid] = $connection;
        return;
    }
};
$worker->onConnect = function($connection){
    echo "new connection from ip " . $connection->getRemoteIp() . "\n";
};
// 当有客户端连接断开时
$worker->onClose = function($connection)
{
    global $worker;
    if(isset($connection->uid))
    {
        // 连接断开时删除映射
        unset($worker->uidConnections[$connection->uid]);
    }

    echo "delel connection from ip " . $connection->getRemoteIp() . "\n";
};
$worker->onError = function($connection, $code, $msg)
{
    echo "error $code $msg\n";
};
// 向所有验证的用户推送数据
function broadcast($message)
{
    global $worker;
    foreach($worker->uidConnections as $connection)
    {
        $connection->send($message);
    }
}

// 针对uid推送数据
function sendMessageByUid($uid, $message)
{
    global $worker;
    if(isset($worker->uidConnections[$uid]))
    {
        $connection = $worker->uidConnections[$uid];
        $connection->send($message);
        return true;
    }
    return false;
}

// 运行所有的worker
Worker::runAll();
```
### A终端
```sh
root@aeb79d20f345:/var/www/html/site1/workman# php test.php start 
```

### B终端
- 同时开启1个命令窗口,执行以下命令

```sh
telnet 127.0.0.1 9501
```
效果如下:

```sh
$ telnet 127.0.0.1 9501
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
gdf
HTTP/1.1 200 Websocket
Server: workerman/4.0.15

<div style="text-align:center"><h1>Websocket</h1><hr>powered by <a href="https://www.workerman.net">workerman 4.0.15</a></div>Connection closed by foreign host.

```
###为何出现`Connection closed by foreign host.`
> 解释： `telnet`是用`tcp`连接的,而你的目的是连接`'text://0.0.0.0:9501`这个协议的,但是系统调用线程工作的时候被分配到`websocket://0.0.0.0:9501`这个协议上了,`websocket`协议需要客户端携带一个`Sec-WebSocket-Key`秘钥才行的,所以服务端匹配秘钥不正确,就拒绝连接,要是你尝试`telnet`几次,说不定系统调用分配到`text`协议上,从而让你有误觉一时可以一时不可以
