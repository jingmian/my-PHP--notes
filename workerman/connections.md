### Worker

- connections
--此属性中存储了当前进程的所有的客户端连接对象，其中id为connection的id编号，详情见手册[TcpConnection的id属性](http://doc.workerman.net/tcp-connection/id.html)

新建一个`test.php`
```php
use Workerman\Worker;
use Workerman\Lib\Timer;
require_once __DIR__ . '/Workerman/Autoloader.php';

$worker = new Worker('text://0.0.0.0:2020');
$worker->count = 2;
// 这里开2两个worker进程,方便查看后面connections有客户端进来时,对比一下

// 进程启动时设置一个定时器，定时向所有客户端连接发送数据

$worker->onWorkerStart = function($worker)
{
    // 定时，每5秒一次
    Timer::add(5, function()use($worker)
    {
        // 遍历当前进程所有的客户端连接，发送当前服务器的时间
		var_dump($worker->connections )
        foreach($worker->connections as $connection)
        {
            $connection->send(time());
        }
    });
};
// 运行worker
Worker::runAll();
```
### A终端
```sh
root@aeb79d20f345:/var/www/html/site1/workman# php test.php start 
```

### B终端
- 同时开启3个命令窗口,执行以下命令

```sh
telnet 127.0.0.1 9501
```
效果如下:

![](https://github.com/jingmian/my-PHP--notes/blob/master/workerman/connections.png)
请观察最又边的workerman输出的窗口,红色标注的地方
```sh
int(3)
int(3)
int(0)
int(3)
int(0)
int(3)
int(0)
int(0)
int(3)
int(0)
int(3)
int(3)
int(0)
int(3)
int(0)
int(0)
int(3)
int(3)
int(0)
```
> 解释： 出现以上这种情况是因为,一开始定义了2个worker进程,一个woker进程里面有一个线程,,属于[多进程单线程模型](http://doc.workerman.net/faq/about-multi-thread.html).有一个worker进程接收3个客户端请求,有一个worker进程没有接收任何客户端请求,这个是系统调度的,你不能指明必须由哪个worker进程接收请求.所以出现3,0这种现象
