# Archer

[![Php Version](https://img.shields.io/badge/php-%3E=7.1-brightgreen.svg?maxAge=2592000)](https://secure.php.net/)
[![Swoole Version](https://img.shields.io/badge/swoole-%3E=4.2.8-brightgreen.svg?maxAge=2592000)](https://github.com/swoole/swoole-src)
[![Archer License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)](https://github.com/swlib/archer/blob/master/LICENSE)

## 简介

 协程Task弓兵, `Swoole人性化组件库`之PHP高性能Task队列, 基于Swoole原生协程, 底层提供无额外I/O的高性能解决方案, 让开发者专注于功能开发, 从繁琐的传统Task队列或协程并发旋涡中解放。

- 基于Swoole协程开发, 以单进程协程实现Swoole Task提供的所有功能
- 人性化使用风格, API简单易用, 符合传统同步代码开发逻辑习惯，支持psr风格操作
- 完备的Exception异常事件, 符合面向对象的基本思路, 避免陷入若类型陷阱
- 多种Task模式（伪异步、协程同步、Defer模式多任务集合）等，满足各种开发情景
- 轻松将任意协程代码变为Defer模式，不用刻意修改为defer()与recv()。
- 可以将任意协程代码并发执行而不改变原先设计模式。
- 基于协程实现的毫秒级计时器

------
<br>

## 安装

最好的安装方法是通过 [Composer](http://getcomposer.org/) 包管理器 :（然而现在暂时并不支持这么安装，正在与官方争取加入swlib）

```shell
composer require swlib/archer
```
或者下载代码，并在autoloader中手动注册Archer：
```php
$loader = include YOUR_BASE_PATH . '/vendor/autoload.php';
$loader->setPsr4('Swlib\\Archer\\', YOUR_PATH . '/src/');
$loader->addClassMap([
    'Swlib\\Archer' => YOUR_PATH . '/src/Archer.php'
]);
```

------

## 依赖

- **PHP 7.1** or later
- **Swoole 4.2.8** or later

------
<br>

## 协程调度

Swoole底层实现协程调度, **业务层无需感知**, 开发者可以无感知的**用同步的代码编写方式达到异步IO的效果和超高性能**，避免了传统异步回调所带来的离散的代码逻辑和陷入多层回调中导致代码无法维护。  
Task队列循环与各Task的执行都处于独立的协程中，**不会占用用户自己创建的协程**。可以将**任意协程变为Defer模式**，无需手动触发defer()与recv()。  
Archer运行于全协程的场景中，禁忌同步阻塞代码的出现，会影响队列的运行。  

需要在`onRequet`, `onReceive`, `onConnect`等事件回调函数中使用, 或是使用go关键字包裹 (`swoole.use_shortname`默认开启).

```php
\Swoole\Runtime::enableCoroutine();
go(function () {
    $archer = \Swlib\Archer::psr();
    $archer->setTaskCallback(function(string $method, ...$param) {
        $redis = new \Redis();
        $redis->connect('127.0.0.1', 6379);
        return $redis->{$method}(...$param);
    });
    $task1 = $archer->setParams('get', 'some_key')->deferExecute();
    $task2 = $archer->setParams('hget', 'a', 'b')->deferExecute();
    $task3 = $archer->setParams('lget', 'k1', 10)->deferExecute();
    var_dump($task1->recv());
    var_dump($task2->recv());
    var_dump($task3->recv());
    
    Archer::taskTimerAfter(1500, function (string $s1, string $s2) {
        echo "1.5s later:{$s1} {$s2}\n";
    }, ['hello', 'world']);
});
```


------

## 接口
所有模式的Task在执行时所处的协程与原协程不是同一个，所以**所有基于[Context](https://wiki.swoole.com/wiki/page/865.html)的变量传递与维护会失效**，务必注意这一点。  
`psr`风格的接口在[下文](https://github.com/fdreamsu/SwArcher#psr风格)
### 模式1：伪异步模式
```php
\Swlib\Archer::task(callable $task_callback, ?array $params = null, ?callable $finish_callback = null): int;
```
- `$task_callback` 待执行的函数，
- `$params` 传入`$task_callback`中的参数，可缺省
- `$finish_callback` Task执行完之后的回调，可缺省，格式如下：

```php
function (int $task_id, $task_return_value, ?\Throwable $e) {
    // $task_id 为\Swlib\Archer::task() 返回的Task id
    // $task_return_value 为Task闭包 $task_callback 的返回值，若没有返回值或抛出了异常，则该项为null
    // $e为Task闭包 $task_callback 中抛出的异常，正常情况下为null
}
```
| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 返回 Taskid | $task_callback与$finish_callback处于同一个协程，但与当前协程不处于同一个 | 通过第3个参数传递给$finish_callback，若缺省则会产生一个warnning |
### 模式2：协程同步返回模式
（该模式与直接执行协程代码的区别在于：会进入Task队列，受队列的size和最大并发影响；运行于不同的协程）
```php
\Swlib\Archer::taskWait(callable $task_callback, ?array $params = null, ?float $timeout = null): mixed;
```
- `$task_callback` 待执行的函数，
- `$params` 传入`$task_callback`中的参数，可缺省
- `$timeout` 超时时间，超时后函数会直接抛出`Swlib\Archer\Exception\TaskTimeoutException`。注意：超时返回后Task仍会继续执行，不会中断，不会移出队列。若缺省则表示不会超时

| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 当前协程挂起，直到Task执行完成并返回结果 | $task_callback与当前协程不是同一个 | 若Task抛出了任何异常，Archer会捕获后在这里抛出。 |
### 模式3：Defer模式
获取Task：
```php
/*定义*/ \Swlib\Archer::taskDefer(callable $task_callback, ?array $params = null): \Swlib\Archer\Task\Defer;
$task = \Swlib\Archer::taskDefer($task_callback, ['foo', 'bar']);
```

| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 返回Task对象 | $task_callback与当前协程不是同一个 | 若Task抛出了任何异常，Archer会捕获后在执行recv时抛出。 |

获取执行结果：
```php
/*定义*/ \Swlib\Archer\Task\Defer->recv(?float $timeout = null);
$task->recv(0.5);
```
- `$timeout` 超时时间，超时后函数会直接抛出`Swlib\Archer\Exception\TaskTimeoutException`。注意：超时返回后Task仍会继续执行，不会中断，不会移出队列。若缺省则表示不会超时

| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 若Task已执行完则直接返回结果。否则协程挂起，等待执行完毕后恢复并返回结果。 | $task_callback与当前协程不是同一个 | 若Task抛出了任何异常，Archer会捕获后会在此处抛出。 |
### 模式4：Task集模式
获取容器：
```php
$container = \Swlib\Archer::getMultiTask();
```
向队列投递Task并立即返回Task id。
```php
$container->addTask(callable $task_callback, ?array $params = null): int;
```
两种执行方式：
###### 等待全部结果：等待所有Task全部执行完。返回值为键值对，键为Taskid，值为其对应的返回值
```php
$container->waitForAll(?float $timeout = null): array;
```
- `$timeout` 超时时间，超时后函数会直接抛出`Swlib\Archer\Exception\TaskTimeoutException`。注意：超时返回后所有Task仍会继续执行，不会中断，不会移出队列。若缺省则表示不会超时

| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 若运行时所有Task已执行完，则会直接以键值对的形式返回所有Task的返回值。否则当前协程挂起。当所有Task执行完成后，会恢复投递的协程，并返回结果。 | 所有Task所处协程均不同 | 若某个Task抛出了任何异常，不会影响其他Task的执行，但在返回值中不会出现该Task id对应的项，需要通过`getError(int $taskid)`或`getErrorMap()`方法获取异常对象 |
###### 先完成先返回：各Task的执行结果会根据其完成的顺序，以键值对的形式yield出来
对于生成器(Generator)的定义：[查看](http://php.net/manual/zh/class.generator.php)
```php
$container->yieldEachOne(?float $timeout = null): \Generator;
```
- `$timeout` 超时时间，超时后函数会直接抛出`Swlib\Archer\Exception\TaskTimeoutException`（该时间表示花费在本方法内的时间，外界调用该方法处理每个返回值所耗费的时间不计入）。注意：超时返回后所有Task仍会继续执行，不会中断，不会移出队列。若缺省则表示不会超时
- 生成器遍历完成后，可以通过 `Generator->getReturn()` 方法获取返回值的键值对

| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 若运行时已经有些Task已执行完，则会按执行完毕的顺序将他们先yield出来。若这之后仍存在未执行完的Task，则当前协程将会挂起，每有一个Task执行完，当前协程将恢复且其结果就会以以键值对的方式yield出来，然后协程会挂起等待下一个执行完的Task。 | 所有Task所处协程均不同 | 若某个Task抛出了任何异常，不会影响其他Task的执行，但这个Task不会被`yield`出来，需要通过`getError(int $taskid)`或`getErrorMap()`方法获取异常对象 |

获取某Task抛出的异常（若Task未抛出异常则返回null）
```php
$container->getError(int $id): ?\Throwable;
```
获取所有异常Task与他们抛出的异常，返回值为键值对，键为Taskid，值为其抛出的异常
```php
$container->getErrorMap(): array;
```
### 模式5：一次性Timer模式
该模式的Task不受[队列的配置](https://github.com/swlib/archer#%E9%85%8D%E7%BD%AE)影响  
（该模式与直接使用co::sleep()执行协程代码的区别在于：不直接阻塞当前协程；底层经过算法优化，会减少并行sleep()的协程数量，节约内存；可以在执行之前清除掉计时器；运行于不同的协程）
```php
\Swlib\Archer::taskTimerAfter(int $after_time_ms, callable $task_callback, ?array $params = null): \Swlib\Archer\Task\Timer\Once;
```
- `$after_time_ms` 计时时间，单位为毫秒
| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 返回 Task对象 | $task_callback与当前协程不是同一个 | Archer会捕获异常，并产生一个warnning |

取消执行：
```php
$task = \Swlib\Archer::taskTimerAfter(1000, function() { echo 'aaa'; });
$task->clearTimer(); // 返回true为成功，若已执行则返回false

// 或这样：
$taskid = \Swlib\Archer::taskTimerAfter(1000, function() { echo 'aaa'; })->getId();
\Swlib\Archer::clearTimerTask($taskid); // 返回true为成功，若已执行则返回false
```
### 模式6：持续型Timer模式
该模式的Task不受[队列的配置](https://github.com/swlib/archer#%E9%85%8D%E7%BD%AE)影响  
（该模式与直接使用co::sleep()执行协程代码的区别在于：不直接阻塞当前协程；底层经过算法优化，会减少并行sleep()的协程数量，节约内存；可以在执行之前清除掉计时器；运行于不同的协程）
```php
\Swlib\Archer::taskTimerTick(int $tick_time_ms, callable $task_callback, ?array $params = null, ?int $first_time_after = null): \Swlib\Archer\Task\Timer\Tick;
```
- `$tick_time_ms` 执行间隔，单位为毫秒
- `$first_time_after` 初次执行计时器，单位为毫秒。若缺省则与`$tick_time_ms`相同
| 返回模式 | 协程说明 | 异常处理 |
| :-- | :-- | :-- |
| 返回 Task对象 | $task_callback与当前协程不是同一个 | Archer会捕获异常，并产生一个warnning |

取消执行：
```php
$task = \Swlib\Archer::taskTimerTick(1000, function() { echo 'tick'; });
$task->clearTimer(); // 返回true为成功，若已被清理则返回false

// 或这样：
$taskid = \Swlib\Archer::taskTimerTick(1000, function() { echo 'aaa'; })->getId();
\Swlib\Archer::clearTimerTask($taskid); // 返回true为成功，若已被清理则返回false
```


### psr风格
```php
$archer = \Swlib\Archer::psr();
$archer->setTaskCallback(
    function($par1, $par2) {
        // task to do
    }
)->setParams('foo', 'barr'); //支持任意多个参数

// 同 模式1：伪异步模式:
$archer->asyncExecute(
    function (int $task_id, $task_return_value, ?\Throwable $e) {
        // receive task result
    }
); 

// 同 模式2：协程同步返回模式（参数表示超时时间，单位为秒）
$archer->waitExecute(2.0); 

// 同 模式3：Defer模式
$task = $archer->deferExecute();
$task->recv(); 

// 同 模式4：Task集模式
$container = \Swlib\Archer::getMultiTask();
$archer->attachToMultiTask($container);
$archer->attachToMultiTask($container);
$container->waitForAll(2.0);

// 同 模式5：一次性Timer模式（参数表示计时时间，单位为毫秒）
$archer->afterTimeExecute(1000); 

// 同 模式6：持续型Timer模式（参数1表示执行间隔，参数2表示初次执行计时器，单位均为毫秒。参数2可缺省）
$archer->tickExecute(1000, 500); 
```
### 在Task内获取当前的Taskid
```php
\Swlib\Archer\Task::getCurrentTaskId(): ?int;
```
在Task内调用该方法可以获取当前的Taskid，在其他地方调用会返回false

### ~~注册一个全局回调函数~~
`swoole>=4.2.9`版本推荐在项目使用Context的时候通过[Coroutine::defer](https://wiki.swoole.com/wiki/page/1015.html)注册清理函数，无需在此注册
```php
\Swlib\Archer\Task::registerTaskFinishFunc(callable $func): void;
```
~~这里注册的回调函数会在每个Task结束时执行，不论Task是否抛出了异常，不论Task模式，格式如下：~~
```php
function (int $task_id, $task_return_value, ?\Throwable $e) {
    // $task_id 为\Swlib\Archer::task()或\Swlib\Archer\MultiTask->addTask() 返回的Task id。\Swlib\Archer::taskWait()由于无法获取Taskid，所以可以忽略该项。
    // $task_return_value 为Task闭包 $task_callback 的返回值，若没有返回值或抛出了异常，则该项为null
    // $e为Task闭包 $task_callback 中抛出的异常，正常情况下为null
}
```
~~不建议在该方法中执行会引起阻塞或协程切换的操作，因为会影响到Task运行结果的传递效率；也不要在该方法中抛出任何异常，会导致catch不到而使进程退出。~~  
~~该方法所处的协程与Task所处的协程为同一个，所以可以**利用该函数清理执行Task所留下的[Context](https://wiki.swoole.com/wiki/page/865.html)**。~~  
~~- Task为伪异步模式时，该方法会在 $finish_callback 之前执行~~
~~- Task为协程同步返回模式或集模式时，该方法会在返回或抛出异常给原协程之前调用。~~

## 配置
```php
\Swlib\Archer\Queue::setQueueSize(int $size): void;
\Swlib\Archer\Queue::setConcurrent(int $concurrent): void;
```
- 队列的size，默认为8192。当待执行的Task数量超过size时，再投递Task会导致协程切换，直到待执行的Task数量小于size后才可恢复
- 最大并发数concurrent，默认为2048，表示同时处于执行状态的Task的最大数量。
- 这两个方法，必须在第一次投递任何Task之前调用。建议在 `onWorkerStart` 中调用

## 异常
Archer会抛出以下几种异常：
- `Swlib\Archer\Exception\AddNewTaskFailException` 将task加入队列时发生错误，由 \Swoole\Coroutine\Channel->pop 报错引起，这往往是由内核错误导致的
- `Swlib\Archer\Exception\RuntimeException` Archer内部状态错误，通常由用户错误地调用了底层函数引起
- `Swlib\Archer\Exception\TaskTimeoutException` Task超时，因用户在某些地方设置了`timeout`，Task排队+执行时间超过了该时间引发的异常。用户应该在需要设置`timeout`的地方捕获这个异常以完成超时逻辑。注意Task执行时间超时不会引起Task中断或被移出队列。

## 例子
###### *假设所有场景均已处于协程环境之中；场景都是理想化，简易化的；除了例子中使用的闭包，Archer支持所有[callable类型](http://php.net/manual/zh/language.types.callable.php)
#### 场景：记录用户操作时间，但并不关心执行结果，也不想等待SQL执行完
```php
\Swlib\Archer::psr()
    ->setTaskCallback(function(int $user_id, int $timestamp): void {
        $swoole_mysql = new Swoole\Coroutine\MySQL();
        $swoole_mysql->connect([
            'host' => '127.0.0.1',
            'port' => 3306,
            'user' => 'user',
            'password' => 'pass',
            'database' => 'test',
        ]);
        $swoole_mysql->prepare('UPDATE `user` SET `optime`=? WHERE `id`=?')->execute([$timestamp, $user_id], 10);
    })
    ->setParams(1, time())
    ->asyncExecute();
```
#### 场景：执行某些协程Client（或由[Runtime::enableCoroutine()](https://wiki.swoole.com/wiki/page/965.html)变为协程的传统Client）时，未开启或无法开启[Defer特性](https://wiki.swoole.com/wiki/page/p-coroutine_multi_call.html)，但又想使用Defer功能。
```php
$task_redis = \Swlib\Archer::taskDefer(function() {
    $redis = new \Swoole\Coroutine\Redis();
    $redis->connect('127.0.0.1', 6379);
    return $redis->get('key');
});
$task_mysql = \Swlib\Archer::taskDefer(function() {
    $mysql = new \Swoole\Coroutine\MySQL();
    $mysql->connect([
        'host' => '127.0.0.1',
        'user' => 'user',
        'password' => 'pass',
        'database' => 'test',
    ]);
    return $mysql->query('select sleep(1)');
});
$task_http = \Swlib\Archer::taskDefer(function(string $url): string {
    $httpclient = new \Swoole\Coroutine\Http\Client('0.0.0.0', 9599);
    $httpclient->setHeaders(['Host' => "api.mp.qq.com"]);
    $httpclient->set(['timeout' => 1]);
    $httpclient->get('/');
    return $httpclient->body;
}, ['api.mp.qq.com']);
var_dump($task_redis->recv());
var_dump($task_mysql->recv());
var_dump($task_http->recv());
```
#### 场景：并发20条SQL并一起获取返回值
```php
$container = \Swlib\Archer::getMultiTask();
$task_callback = function(int $id): int {
    $mysql = new Swoole\Coroutine\MySQL();
    $mysql->connect([
        'host' => '127.0.0.1',
        'user' => 'user',
        'password' => 'pass',
        'database' => 'test',
    ]);
    $result = $mysql->query('SELECT COUNT(*) AS `c` FROM `order` WHERE `user`='.id);
    if (empty($result)) return 0;
    return current($result)['c'] ?? 0;
};
$map = [];
$map2 = [];
$results = [];
for ($id=1; $id<=20; ++$id) {// 我知道这个可以用 GROUP BY 一条SQL实现，这里只是举个例子
    $taskid = $container->addTask($task_callback, [$id]);
    $map[$taskid] = $id;
    $map2[$id] = $taskid;
}

foreach ($container->waitForAll(10) as $taskid=>$count)
    $results[$map[$taskid]] = $count;
    
for ($id=1; $id<=20; ++$id)
    if (array_key_exists($id, $results))
        echo "id:{$id} count:{$results[$id]}\n";
    else
        echo "id:{$id} error:". $container->getError($map2[$id])->getMessage() ."\n";
```
#### 场景：并发20条SQL，并将结果发给20个用户，每条运行完就立刻发送。
```php
$container = \Swlib\Archer::getMultiTask();
$archer = \Swlib\Archer::psr();
$archer->setTaskCallback(function(int $id): int {
    $mysql = new Swoole\Coroutine\MySQL();
    $mysql->connect([
        'host' => '127.0.0.1',
        'user' => 'user',
        'password' => 'pass',
        'database' => 'test',
    ]);
    $result = $mysql->query('SELECT COUNT(*) AS `c` FROM `order` WHERE `user`='.id);
    if (empty($result)) return 0;
    return current($result)['c'] ?? 0;
});
$map = [];
for ($id=1; $id<=20; ++$id) {
    $taskid = $archer->setParams($id)->attachToMultiTask($container);
    $map[$taskid] = $id;
}

foreach ($container->yieldEachOne(10) as $taskid=>$count) {
    $server->send($map[$taskid], $count); // 不要纠结为什么 fd 和 id 取值一样，这只是一个简化的场景例子，正式应用肯定更复杂
    unset($map[$taskid]);
}

foreach ($map as $taskid => $id)
    $server->send($id, 'Error: ' . $container->getError($taskid)->getMessage());

```

------

## 重中之重

**欢迎提交issue和PR.**

