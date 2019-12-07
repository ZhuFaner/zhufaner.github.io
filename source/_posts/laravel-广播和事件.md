title: Laravel 事件和广播
author: 吉鹏
tags:
  - laravel
  - 事件
  - 广播
categories: []
date: 2019-12-06 17:59:00
---
# 事件

## 基础介绍

事件可以解耦系统程序代码，使系统模块分明。事件本质上是一对多的关系，即当某个事件发生，会触发一系列的系统更新操作。php 提供了 SplSubject, SplObserver, SplObjectStorage 标准库接口用来实现事件功能。  
laravel 的事件提供了一个简单的观察者实现，允许你在应用中订阅和监听各种发生的事件。在阅读之前，需要对 laravel 的服务容器和服务提供者有些微了解。

## 事件实现

### EventServiceProvider 服务提供者

在 laravel 系统中，EventServiceProvider 负责提供事件的实现与调度，作为 laravel 核心服务提供者，在系统初始化函数中就被注册，核心代码块为

```php
// vendor/laravel/framework/src/Illuminate/Events/EventServiceProvider.php
public function register()
{
    $this->app->singleton('events', function ($app) {
        return (new Dispatcher($app))->setQueueResolver(function () use ($app) {
            return $app->make(QueueFactoryContract::class);
        });
    });
}
```
*     核心是往 laravel 的服务容器中绑定事件接口和事件的实现类
*     Dispatcher 类是事件实现的核心类
*     QueueFactoryContract 是标注对应的队列实现类

为了简明逻辑，将核心放到事件实现上，可以忽略队列相关东西，可以将代码简化为

```php
public function register()
{
	$this->app->singleton('events', function ($app) {
		return (new Dispatcher($app));
	});
}
```

  

### Dispatcher 核心类
   Dispatcher 类是 laravel 提供事件服务的核心代码，事件本质上就两个核心函数
   1. listen 方法，负责绑定事件名称和事件监听器代码的对应关系，事件名称通过判断是否包含 " * " 分为明确事件名称和通配符事件名称
   2. dispatch 方法，负责调度监听器代码，完成系统事件更新
   
#### listen 方法解析

```php
public function listen($events, $listener)
{
	foreach ((array) $events as $event) {
		if (Str::contains($event, '*')) {
			$this->setupWildcardListen($event, $listener);
		} else {
			$this->listeners[$event][] = $this->makeListener($listener);
		}
	}
}
```
laravel 将事件映射分别存储到 wildcards 和 listeners 属性中
 
```php
public function makeListener($listener, $wildcard = false)
{
	if (is_string($listener)) {
		return $this->createClassListener($listener, $wildcard);
	}
	return function ($event, $payload) use ($listener, $wildcard) {
		if ($wildcard) {
			return $listener($event, $payload);
		}
		return $listener(...array_values($payload));
	};
}
```

在 makeListener 方法中，分依据  $listener 参数类型不同，分情况解析监听器代码
  1.  当 $listener 为字符串时，会通过 createClassListener 方法进一步解析处理
  2.  当 $listener 为闭包函数时，会进一步进行包装，将事件名和参数作为闭包函数的参数，在闭包函数内，依据 $wildcard 直接调用对应的监听器代码。这一步封装主要为了在调度时方便统一处理
 
```php
public function createClassListener($listener, $wildcard = false)
{
	return function ($event, $payload) use ($listener, $wildcard) {
		if ($wildcard) {
			return call_user_func($this->createClassCallable($listener), $event, $payload);
		}
		return call_user_func_array(
			$this->createClassCallable($listener), $payload
		);
	};
}

protected function createClassCallable($listener)
{
	[$class, $method] = $this->parseClassCallable($listener);
	***省了队列的一些处理****
	return [$this->container->make($class), $method];
}

protected function parseClassCallable($listener)
{
	return Str::parseCallback($listener, 'handle');
}
```
1.  先对字符进行处理，laravel 预期的字符串为 \mespace\XXXclass@method ，parseClassCallable 会用 @ 符号截取字符串，获得类名和方法名，方法名默认为 handle
2.  createClassCallable 方法会通过服务容器，解析出监听器类实例
3.  createClassListener 方法也会进行闭包封装，参数依然是事件名称和参数，这一点和上述对闭包的封装一样
 
 
#### dispatch 方法解析
当对应事件触发时，系统会通过 dispatch 方法进行调度，调用之前注册过的监听函数，完成事件更新任务。

```php
public function dispatch($event, $payload = [], $halt = false)
{
	[$event, $payload] = $this->parseEventAndPayload(
		$event, $payload
	);

	****  省略队列处理代码  ****
	$responses = [];
	foreach ($this->getListeners($event) as $listener) {
		$response = $listener($event, $payload);
		if ($halt && ! is_null($response)) {
			return $response;
		}
		if ($response === false) {
			break;
		}
		$responses[] = $response;
	}
	return $halt ? null : $responses;
}
```
这是事件调度的核心代码

```php
protected function parseEventAndPayload($event, $payload)
{
	if (is_object($event)) {
		[$payload, $event] = [[$event], get_class($event)];
	}
	return [$event, Arr::wrap($payload)];
}
```
该方法主要是为了解析一下事件名和参数，$event 解析为字符串，$payload 解析为数组。当参数 $event 为对象时，  laravel 会解析出类名作为事件名称，并且将类实例作为  $payload 数组参数返回

```php
public function getListeners($eventName)
{
	$listeners = $this->listeners[$eventName] ?? [];
	$listeners = array_merge(
		$listeners,
		$this->wildcardsCache[$eventName] ?? $this->getWildcardListeners($eventName)
	);
	return class_exists($eventName, false)
				? $this->addInterfaceListeners($eventName, $listeners)
				: $listeners;
}

protected function getWildcardListeners($eventName)
{
	$wildcards = [];
	foreach ($this->wildcards as $key => $listeners) {
		if (Str::is($key, $eventName)) {
			$wildcards = array_merge($wildcards, $listeners);
		}
	}
	return $this->wildcardsCache[$eventName] = $wildcards;
}

protected function addInterfaceListeners($eventName, array $listeners = [])
{
	foreach (class_implements($eventName) as $interface) {
		if (isset($this->listeners[$interface])) {
			foreach ($this->listeners[$interface] as $names) {
				$listeners = array_merge($listeners, (array) $names);
			}
		}
	}
	return $listeners;
}
```
1.  getWildcardListeners 方法会解析 wildcards 中的监听事件，这一部分主要是带通配符的事件名称，并且解析完后会进行内存缓存
2.  addInterfaceListeners 方法会向上发散，会找到事件类所有实现的接口类，并且进一步解析 listeners 中是否有对应接口类的监听器函数，借此实现了类似 JavaScript 中的事件冒泡原理
3.  获取到事件的所有监听器函数之后，会按照顺序依次调用，由参数 halt 或者 监听器函数返回值( false ) 来决定是否停止继续执行剩余监听器代码，所以，在绑定事件监听器时，绑定的顺序也是很重要的

##### Dispatcher 的其余代码
打开 Dispatcher 实现的接口类，发现还有一些其他方法，例如：push、flush、forget、hasListeners 等等，这些都是一些辅助的方法函数，都是对 listen / dispatch 的调用，或者是对 listeners / wildcards / wildcardsCache 的处理


### laravel 对队列的使用

#### 事件注册机制
  通过上述分析，事件注册本质上就是调用 listener 函数，进行事件名和事件处理函数的关系绑定。
```php
$event  = $app->make("events");

// 绑定事件名称 和 类字符串
$event->listen('order',App\Listeners\OrderListenerOne::class);
$event->listen(App\Events\OrderEvent::class,App\Listeners\OrderListenerOne::class);

// 绑定事件名称 和 闭包函数 
$event->listen('order',function( $a , $b ){
  echo "<hr>";
  echo $a, "<br>";
  echo $b, "<br>";
  echo "<hr>";
});

```

#### 事件触发调度
```php

// 字符串名称触发
$event->dispatch("order",[1,11,22]);

// 类事件触发
$one = new App\Events\OrderEvent(1);
$event->dispatch($one,[1,11,22]); // 如果是类的话，后边参数会被类覆盖，
```


### laravel 事件说明
1. laravel 事件处理，每个地方都在依据是否带有通配符进行分情况处理，带通配符的话，会将事件名作为参数传递
2. 如果触发事件的是类实例，laravel 会解析出类名作为事件名称
3. laravel 的事件也有向上冒泡功能  


     
# 广播

## laravel 广播介绍
   laravel 的广播模块，本质是利用 websocket 建立客户端和服务端的双向链接，通过各种频道进行通信，完成信息的更新显示
	
## 开工前准备
* laravel 6.0 版本
* laravel 队列，本例使用 redis 作为队列驱动
* laravel/ui 提供的前端脚手架
* laravel 事件


## 配置
* redis 相关配置
* 开启广播服务提供者
    在 config/app.php 中取消 App\Providers\BroadcastServiceProvider::class,  前边的注释
* 设置队列、广播为 redis 驱动
   在 .env 中，设置  
	BROADCAST_DRIVER=redis  
	QUEUE_CONNECTION=redis

    
## 开始

### 创建广播事件
1.  使用 php artisan make:event PushMsgEvent，创建事件类 
2.  编写事件类

```php
class PushMsgEvent implements ShouldBroadcast
{  
	use Dispatchable, InteractsWithSockets, SerializesModels;
	public $msg;

	/**
	 *  定义 msg 变量，保存弹幕消息
	 */
	public function __construct( $msg )
	{
		// 简单的消息列表
		$this->msg = $msg;
	}

	/**
	 *  弹幕所有人都可以收到，所以返回公共频道就 OK 的
	 */
	public function broadcastOn()
	{
		return new Channel('push');
	}

	/**
	 *  重命名一下广播名称，一般默认为类名
	 */
	public function broadcastAs()
	{
	 return 'push.msg';
	}
}
```

3. 编写路由及控制器，用于展示弹幕和发送弹幕

```php
class BroadcastingController extends Controller
{
    /**
	 * 展示弹幕显示页面
	 */
	public function index()
	{
		return view("broadcasting.index");
	}


	/** 
	 * 接受前端弹幕请求，并触发广播事件
	 */
	public function put( Request $request )
	{
		broadcast(  new PushMsgEvent( $request->msg )  );
    }
}
```

### 安装各种需要的 node 包

 1. laravel-echo-server 包，与 laravel 兼容的 Socket.IO 服务器
   * npm install -g laravel-echo-server  全局安装
   * laravel-echo-server init  初始化配置信息，在根目录下生成 laravel-echo-server.json 文件
   * laravel-echo-server start | stop ， Socket.IO  的开启或关闭，这个需要使用守护进程在服务器上运行
 
 2. socket.io  客户端
    * npm install  --save laravel-echo 
	* npm install  --save socket.io-client
     

### 前端页面编写
    
1. 修改 webpack.min.js ，生成单独使用的 broadcasting.js 文件
 
  ```javascript
       mix.js('resources/js/app.js', 'public/js')
             .js('resources/js/vue-component.js', 'public/js')
             .js('resources/js/broadcasting.js', 'public/js')  /// 新增
             .sass('resources/sass/app.scss', 'public/css');
  ```

2. 编写 resources/js/broadcasting.js

  ```javascript  
  // 引入该引入的包
  import Echo from 'laravel-echo';
  window.io = require('socket.io-client');
  window.Echo = new Echo({
    broadcaster: 'socket.io',
    host: window.location.hostname + ':6001'
  });
  window.Echo.channel('push')
      .listen('.push.msg', (e) => {
          alert( e.msg );  // 这里只是弹出弹幕消息，滚动的效果就不做了
      });
  ```
3. 运行 npm run dev 生成 broadcasting.js 文件
4. 编写 broadcasting.index blade 文件

  ```html
  <div class="my-container container">
        <div class="col-xs-12" style="min-height: 600px;">
        </div>
  </div>
  <script src="{{ asset('js/broadcasting.js') }}" defer></script>
  ```

# 开始验收效果
 1. 开启 Socket.IO 服务端进程，laravel-echo-server start 
 2. 开始队列处理进程 php artisan queue:work --timeout=30
 3. 利用 php 提供的内部服务器 php artisan serve
 4. 在输入弹幕消息之后，弹幕展示页面便会将消息弹出