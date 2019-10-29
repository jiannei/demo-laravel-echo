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
composer create-project --prefer-dist laravel/laravel demo-echo
```

### laravel-echo-server

* 安装

```
npm install -g laravel-echo-server
```

* 切换到 laravel 项目目录，初始化 Socket 服务：

```
cd demo-echo
laravel-echo-server init
```

![readme-2019-10-29-23-10-44](https://vuepress-pic-bed.oss-cn-beijing.aliyuncs.com/readme-2019-10-29-23-10-44)


* 启动 laravel-echo-server 服务

```
laravel-echo-server start
```

![readme-2019-10-29-23-12-52](https://vuepress-pic-bed.oss-cn-beijing.aliyuncs.com/readme-2019-10-29-23-12-52)


## 参考

* [使用 Laravel-echo-server 构建实时应用](https://learnku.com/laravel/t/13101/using-laravel-echo-server-to-build-real-time-applications)
