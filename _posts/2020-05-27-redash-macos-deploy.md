---
layout: post
title: "[BIG DATA] Redash 9.0 on macOS 安装部署"
tags: 
  - Big Data
  - Redash
  - Dashboard
---

## Redash简介

现在市面上有三大开源数据可视化工具, 分别是 Metebase, Superset 和 Redash. Redash相对于前两种虽然功能性和可视化图表相对少了一些, 但是也因为操作简单, 用户体验更好, 而且代码的可读性更强, 更易于二次开发, 功能定制.

源码地址: https://github.com/getredash/redash.git


Redash架构图:

<center>
    <img style="border-radius: 0.3125em;box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" src="https://pic1.zhimg.com/v2-50c3f04b9772f0f8f6d3c740a3794b56_r.jpg">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;display: inline-block;color: #999;padding: 2px;">图片来自网络, 侵删</div>
</center>


## 安装部署(macOS)

### Docker一键安装

下载源码后, 切换到项目路径, 分别执行一下命令:
```shell
# 拉取所有依赖镜像并初始化数据库
docker-compose run –rm server create_db
# 创建容器并运行, -d为后台执行
docker-compose up -d
```
然后访问 http://127.0.0.1:5000 即可进入Redash

如果出现界面样式失效, 可能是因为 node 容器没有创建导致前端资源没有编译, 可以在执行以下命令手动编译:
```shell
# 下载所有依赖
npm install
# 编译运行
npm run build
```

如果需要自定义配置, 可以修改docker-compose.yml中的制定配置项

其他命令
```shell
# 停止容器
docker-compose stop
# 停止并删除相关容器
docker-compose down
```

### 自助安装

安装以下主要组件依赖, macOS 下可用 homebrew 一键安装

- python3
- redis
- nodejs
- freetds
- postgresql

postgresql 的初始化的主要命令:
```shell
# 初始化
initdb /usr/local/var/postgres
# 根据当前用户创建一个db
created
# 进入postgres cli
psql
# 创建一个用户
CREATE USER postgres WITH PASSWORD '123456';
# 删除自动生成'postgres'db
DROP DATABASE postgres;
# 创建一个属于新创建用户的db
CREATE DATABASE postgres OWNER postgres;
# 赋予权限
GRANT ALL PRIVILEGES ON DATABASE postgres to postgres;
# 角色
ALTER ROLE postgres CREATEDB;
```

**修改配置** 

所有配置项基本就在此文件下: *redash/settings/\_\_init\_\_.py* 

如修改数据库配置 **REDASH_DATABASE_URL** 改为如下内容:
```python
SQLALCHEMY_DATABASE_URI = os.environ.get(
    "REDASH_DATABASE_URL", os.environ.get("DATABASE_URL", "postgresql://postgres:123456@127.0.0.1:5432/postgres")
)
```

**python 相关**

推荐使用 python 环境运行, 主要命令:
```shell
# 进入到项目目录下
# 下载虚拟环境模块
pip3 install virtualenv
# 创建虚拟环境
virtualenv -p python3 venv
# 运行虚拟环境
source venv/bin/activate
```

所有 python 依赖包拉取:
```shell
# 安装所有基础依赖和数据库连接依赖
pip install -r requirements.txt -r requirements_all_ds.txt
```

可能出现的问题:
安装 psycopg2 可能会出现: *Error: pg_config executable not found.*

解决: 把 postgresql 的bin目录加入环境变量, 或者不安装 psycopg2 直接安装 psycopy2-binary(requirements.txt文件中注释掉该行)

安装 mysqlclient 可能会出现: *Error: mysql_config not found.*

解决: 安装 mysql-connector-c 并加入环境变量, 如果已安装过mysql相关模块仍报此问题, 可能就是环境变量问题.

安装完成后执行: 
```shell
python manage.py check_settings
```
如果没问题的话就会打印出当前所有设置项内容

**前端编译**

*(注意 nodejs 版本不能太低)*
```shell
npm install
npm run build
```

**初始化元数据库**

```shell
python manage.py database create_tables
```

**运行**

需要运行三个模块: **Web Server** 和 **RQ worker** & **scheduler**

所以可以开三个终端窗口分别运行:
```shell
pyhton manage.py runserver –debugger –reload -h 0.0.0.0 -p 5000
python manage.py rq worker
python manage.py rq scheduler  ##(此调度程序非必须运行)
```

**Done**

访问 http://127.0.0.1:5000 注册一个账户开启 Redash 吧

