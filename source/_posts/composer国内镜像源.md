---
title: PHP Composer 国内全量镜像源整理
date: 2019-07-15 11:04:45
tags: php
urlname: composer-mirrors
---

近日，各大云服务厂商逐渐公布了自己的 PHP Composer 全量镜像，整理以下『全量镜像』供大家选择。

# PHP Composer 安装
```shell
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
mv composer.phar /usr/local/bin/composer
```

# [阿里云](https://developer.aliyun.com/composer)
自用稳定性很好，个人目前选择。
```shell
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```

# [腾讯云](https://mirrors.cloud.tencent.com/composer/)
```shell
composer config -g repos.packagist composer https://mirrors.cloud.tencent.com/composer/
```

# [phpcomposer](https://pkg.phpcomposer.com/)
国内很早开放的免费镜像服务，其他服务商跟进之前的首选。
```shell
composer config -g repo.packagist composer https://packagist.phpcomposer.com
```

# ~~[Laravel China 镜像](https://packagist.laravel-china.org)~~
完成历史使命，将于 2019-09-01 之后终止服务，感谢 Laravel-China 对开源事业的贡献！

# [华为云](https://mirrors.huaweicloud.com/)
```shell
composer config -g repo.packagist composer https://mirrors.huaweicloud.com/repository/php/
```

# [安畅网络](https://php.cnpkg.org/)
```shell
composer config -g repos.packagist composer https://php.cnpkg.org
```

# 关闭全局配置
```shell
composer config -g --unset repos.packagist
```