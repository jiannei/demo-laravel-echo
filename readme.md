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

![readme-2019-10-29-23-10-44](https://vuepress-pic-bed.oss-cn-beijing.aliyuncs.com/readme-2019-10-29-23-10-44)

* 启动 laravel-echo-server 服务

```
laravel-echo-server start
```

![readme-2019-10-29-23-12-52](https://vuepress-pic-bed.oss-cn-beijing.aliyuncs.com/readme-2019-10-29-23-12-52)

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

<p align="center">
<img src="https://i.loli.net/2019/10/30/QPJVEiAOtHFhGej.png"/>
</p>

<p align="center">
<img src="https://i.loli.net/2019/10/30/lm5BNwKeCvqSt7d.png"/>
</p>

* 新开窗口访问 `http://demo-laravel-echo.test/test-broadcast`

<p align="center">
<img src="https://i.loli.net/2019/10/30/Fc5dvV1MoYSn3fp.png"/>
</p>

<p align="center">
<img src="https://i.loli.net/2019/10/30/SQpuse5C7gDc4NI.png"/>
</p>

* 问题：可以看到后端事件准备被广播，但浏览器控制台并没有任何输出，仔细查看发现，laravel-echo-server 广播的实际 channel 名称为 `echo_database_test-event`，即`APP_NAME_database_channel_name`。
* 原因：`config/database.php` 中 redis 选项在 Laravel 6.x 版本多出了 prefix `'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),`
* 解决：
    * 可以暂时在 `.env` 中配置 `REDIS_PREFIX=null`来去掉前缀
    * 另一种方式是在前端收听 channel 的时候也加上前缀，本案例中，如`window.Echo.channel('echo_database_test-event')`
* 相关讨论：
    * [Why laravel broadcast channel has a prefix?](https://github.com/laravel/framework/issues/28210)
    * [Redis keyPrefix Problem](https://github.com/laravel/echo/issues/232)
    * [config/database.php redis prefix question](https://github.com/laravel/framework/issues/28701)

## 参考

* [使用 Laravel-echo-server 构建实时应用](https://learnku.com/laravel/t/13101/using-laravel-echo-server-to-build-real-time-applications)
