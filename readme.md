# Laravel-echo 使用示例项目

## 项目前准备

* [composer 镜像源](https://developer.aliyun.com/composer)修改 `composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/`
* yarn 镜像源修改 `yarn config set registry https://registry.npm.taobao.org`
* npm 镜像源修改 `npm config set registry=https://registry.npm.taobao.org`
* （可选）yarn 升级：`curl --compressed -o- -L https://yarnpkg.com/install.sh | bash`
* （可选）npm 升级：`sudo npm install -g npm`

## 示例过程

* 安装 (master 分支使用截止到目前的 laravel 最新版本 6.4.0)

```
composer create-project --prefer-dist laravel/laravel demo-laravel-echo
```

### laravel-echo-server 服务

* 安装

```
npm install -g laravel-echo-server
```

* 切换到 laravel 项目目录，初始化 Socket 服务：

```
cd demo-laravel-echo
laravel-echo-server init
```

<img src="https://i.loli.net/2019/10/30/MKFbCdqUra79PLB.png"/>

* 启动 laravel-echo-server 服务

```
laravel-echo-server start
```

<img src="https://i.loli.net/2019/10/30/YbtHGopfxC3iSVD.png"/>

### （后端）事件广播

* 启用 BroadcastServiceProvider 

```
// config/app.php
App\Providers\BroadcastServiceProvider::class,// 取消注释
```

* 创建事件 `php artisan make:event ExampleEvent`
* 广播事件 `class ExampleEvent implements ShouldBroadcast`
* 定义 chanel 
```
public function broadcastOn()
{
    return new Channel('test-event');
}
```

* 广播数据

```
public function broadcastWith()
{
    return [
        'data' => 'key'
    ];
}
```


* 测试事件广播

```
// routes/web.php
Route::get('test-broadcast', function(){
    broadcast(new \App\Events\ExampleEvent);
});
```

* 安装 redis 驱动

```
composer require predis/predis
```

* 配置启用 redis 队列

```
BROADCAST_DRIVER=redis
QUEUE_CONNECTION=redis
```

* 启动队列监听

```
php artisan queue:work
```

## （前端）收听广播

* 前端 vue 脚手架

```
composer require laravel/ui --dev
php artisan ui vue
```

* 安装前端依赖 `npm install --no-bin-links`
* 安装 Socket.io 客户端和 Laravel-Echo 包：`npm install --save socket.io-client laravel-echo`
* 引入 laravel-echo 

```
// resources/js/bootstrap.js
import Echo from 'laravel-echo'

window.io = require('socket.io-client');
window.Echo = new Echo({
    broadcaster: 'socket.io',
    host: window.location.hostname + ':6001'
});
```

* 使用 laravel-echo 收听 websocket 传来的广播消息，`test-event` 与后端的广播 channel 对应，`ExampleEvent` 与后端的广播事件对应

```
// resources/js/components/ExampleComponent.vue
window.Echo.channel('test-event')
    .listen('ExampleEvent', (e) => {
        console.log(e);
    });
```

* 编译 `npm run dev`

### 测试

* 预期效果：浏览器控制台输出 websocket 收听到的广播消息

* 访问 `http://demo-laravel-echo.test/`

<img src="https://i.loli.net/2019/10/30/QPJVEiAOtHFhGej.png"/>

<img src="https://i.loli.net/2019/10/30/lm5BNwKeCvqSt7d.png"/>

* 新开窗口访问 `http://demo-laravel-echo.test/test-broadcast`

<img src="https://i.loli.net/2019/10/30/Fc5dvV1MoYSn3fp.png"/>

<img src="https://i.loli.net/2019/10/30/SQpuse5C7gDc4NI.png"/>

* 问题一：可以看到后端事件准备被广播，但浏览器控制台并没有任何输出，仔细查看发现，laravel-echo-server 广播的实际 channel 名称为 `echo_database_test-event`，即`APP_NAME_database_channel_name`。
* 原因：`config/database.php` 中 redis 选项在 Laravel 6.x 版本多出了 prefix `'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),`
* 解决：
    * 可以暂时在 `.env` 中配置 `REDIS_PREFIX=null`来去掉前缀
    * 另一种方式是在前端收听 channel 的时候也加上前缀，本案例中，如`window.Echo.channel('echo_database_test-event')`
* 相关讨论：
    * [Why laravel broadcast channel has a prefix?](https://github.com/laravel/framework/issues/28210)
    * [Redis keyPrefix Problem](https://github.com/laravel/echo/issues/232)
    * [config/database.php redis prefix question](https://github.com/laravel/framework/issues/28701)
    
* 问题二：Please remove or rename the Redis facade alias in your "app" configuration file in order to avoid collision with the PHP Redis extension.
* 原因：laravel 6.x 版本使用phpredis 作为 redis 默认 driver，同样在`config/database.php` 中 redis 选项中可以看到 `'client' => env('REDIS_CLIENT', 'phpredis'),`
* 解决：
    * 在 `.env`文件中增加`REDIS_PREFIX=predis`来使用 predis 驱动
    * 安装 php 对应版本的的 phpredis 扩展：https://github.com/phpredis/phpredis/blob/develop/INSTALL.markdown （php 7.4 版本截止到 2019.10 并没有对应扩展），并在`config/app.php`中注释或删除` 'Redis' => Illuminate\Support\Facades\Redis::class,`
* 相关讨论：
    [Laravel 6 使用 Redis 注意事项](https://learnku.com/laravel/t/33538)

## 启动 laravel-echo-server 的正确姿势

* 移除原先全局安装的 laravel-echo-server:
    * `sudo npm uninstall -g laravel-echo-server`
    * 删除项目目录中的`laravel-echo-server.json`配置文件

* 进入到项目目录，将 laravel-echo-server 作为项目依赖进行安装

``
npm install laravel-echo-server --no-bin-links
``

* 创建 server.js，内容参考如下

```
require('dotenv').config();

const env = process.env;

require('laravel-echo-server').run({
    authHost: env.APP_URL,
    devMode: env.APP_DEBUG,
    database: "redis",
    databaseConfig: {
        redis: {
            host: env.REDIS_HOST,
            port: env.REDIS_PORT,
        }
    }
});
```

* 可以看到 server.js 依赖 dotenv，所以进行安装 `npm install dotenv --no-bin-links`
* 可以使用 `node server.js`启动 laravel-echo-server 服务。或者选择在 package.json 中增加一个启动 laravel-echo-server 的 script，使用`npm run start`启动服务。参考如下：

```
{
    "private": true,
    "scripts": {
        "start": "node server.js"
    },
    "dependencies": {
        "dotenv": "^8.2.0",
        "laravel-echo-server": "^1.5.9",
    }
}
```

### laravel-echo-server 两种启动方式对比

* 第一种方式需要全局安装 laravel-echo-server，并在项目目录使用 `laravel-echo-server init`初始化生成配置文件`laravel-echo-server.json`（有利于将 laravel-echo-server 独立部署成服务？）
* 第二种方式以项目依赖进行安装，然后与 laravel 后端共用 `.env`文件的配置

## 参考

* [使用 Laravel-echo-server 构建实时应用](https://learnku.com/laravel/t/13101/using-laravel-echo-server-to-build-real-time-applications)
* [Running Laravel Echo Server the right way](https://medium.com/@titasgailius/running-laravel-echo-server-the-right-way-32f52bb5b1c8)
