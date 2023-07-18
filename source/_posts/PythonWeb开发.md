---
title: Python web
categories: [python]
---

## Python web
## WSGI 概念

​	django 只是一个解析框架，他不能直接处理 HTTP 请求和 HTTP 响应，要想它能够直接部署在生产环境中，你必须得安装符合 WSGI:Python Web Server Gateway Interface v1.0 规范的 web 服务器。
​	WSGI 它是 PEP3333中定义的（PEP3333的目标建立一个简单的普遍适用的服务器与Web框架之间的接口）WSGI是Python应用程序或框架和Web服务器之间的一种接口 WSGI 被广泛接受, 已基本达成它了可移植性方面的目标，它其实就类似于 Java 中的"servlet" API 。
​	uwsgi 实现了 WSGI， uwsgi 的配置文件在的 Django 的同名包目录下，uwsgi 才是一个真正的WEB服务器，它就类似与 Java 中的 tomcat（但是他不仅仅是 web 服务器，它还支持很多其他的协议 。Django 只能接收到他翻译好的结果 ，才能够解析具体的请求和响应。
​	为什么 Java 和 python 要这样设计呢？我认为是增加了中间层，将复杂的通讯io处理封装成单独的一层，如，servlet API 和 WSGI ，把业务逻辑和分离出去，降低了复杂性。业务逻辑中的框架都满足 servlet API 或 WSGI。类似与网络分层，这些只要满足了这些标准，我们就可以随意切换应用框架，以满足自己的需求。

## 轻量 HTTP 服务

python 标准库里面的工具类自带一个轻量的 HTTP 服务

```python
from wsgiref.simple_server import make_server


def hello_world_app(environ, start_response):
    status = "200 OK"  # HTTP Status
    headers = [("Content-type", "text/plain; charset=utf-8")]  # HTTP Headers
    start_response(status, headers)

    # The returned object is going to be printed
    return [b"Hello World"]

with make_server("", 8000, hello_world_app) as httpd:
    print("Serving on port 8000...")

    # Serve until process is killed
    httpd.serve_forever()
```



## 分发部署

### 环境导出

#### 一、如果使用的 pip
```python
python3 -m pip freeze > requirements.txt
```
#### 二、如果使用的是 anaconda
迁移到其他 anaconda 中有三种方式：
##### 第一种

是使用 requirements.txt。先使用如下命令生成 anaconda 的 requirement.txt

```python
conda list --export > requirements.txt
```
再在另外一个 anaconda 中导入
```python
conda create --name <env> --file requirements.txt
```
##### 第二种

是使用 yaml 的方式,先使用如下命令导出 yaml

```python
conda env export > environment.yml
```
再在其他 anaconda 中使用如下命令导入
```python
onda env create -f environment.yml
```
##### 第三种

若是想生成 pip 那种格式的 requirement.txt 可以加参数，来格式化一下 

```python
pip list --format=freeze > requirements.txt
```


### 容器分发

编写 dokcer file

```dockerfile
FROM python:3.9

RUN mkdir /app
WORKDIR /app
RUN pip install --upgrade pip
COPY requirements.txt /app/

RUN pip install -r requirements.txt
COPY . /app/

EXPOSE 8080

ENTRYPOINT ["python", "main.py"]
```

生成镜像

1.通过 Dockerfile 生成

```python
docker build --tag python-docker .
```
2.使用 export 和 import 命令

- 这两个命令是通过容器来导入和导出镜像的。首先，需要使用`docker export`命令根据容器ID将镜像导出成一个文件，例如

  ```shell
  docker export f299f501774c > some_server.tar
  ```

- 然后，可以使用`docker import`命令将这个文件导入成一个新的镜像，例如

- ```shell
  `docker import - new_hangger_server < some_server.tar`
  ```

  这种方案的优点是导出的文件体积较小，而且可以为导入的镜像指定新名称。但是缺点是会丢失镜像的历史记录和元数据信息，只能保存容器当时的快照状态。

3.使用save和load命令

- 这两个命令是通过镜像来保存和加载镜像文件的。首先，需要使用`docker save`命令根据镜像名称或ID将镜像保存成一个文件，例如

  ```shell
  docker save -o ./ubuntu18.tar ubuntu:18.04
  ```

- 然后，可以使用`docker load`命令将这个文件加载成一个镜像，例如

  ```shell
  docker load < ubuntu18.tar
  ```

- 这种方案的优点是不会丢失镜像的历史记录和元数据信息，而且可以同时将多个镜像打包成一个文件。但是缺点是保存的文件体积较大，而且不能对加载的镜像重命名。

打上标签

```shell
docker tag python-docker:latest
```
测试部署
```shell
docker run python-docker
```
### 常见问题：
#### 1.Django Invalid HTTP_HOST header

解决方法：
修改 settings.py 中 ALLOWED_HOSTS

```python
ALLOWED_HOSTS = ['192.168.2.157','127.0.0.1']
```
值为'*'，可以使所有的网址都能访问Django项目了，失去了保护的作用，可以用于测试
```python
ALLOWED_HOSTS = ['*']
```


## 参考链接

1、stackoverflow requirements export : [https://stackoverflow.com/questions/50777849/from-conda-create-requirements-txt-for-pip3](https://stackoverflow.com/questions/50777849/from-conda-create-requirements-txt-for-pip3)

2、Python和Java服务器通信实现的理解和比较 ：https://birdben.github.io/2017/03/05/Java/Python%E5%92%8CJava%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%80%9A%E4%BF%A1%E5%AE%9E%E7%8E%B0%E7%9A%84%E7%90%86%E8%A7%A3%E5%92%8C%E6%AF%94%E8%BE%83/

3、django-python :  https://docs.docker.com/language/python/run-containers

4、Django Invalid HTTP_HOST header : https://www.cnblogs.com/baby123/p/12122703.html

5、python 标准库 API： https://docs.python.org/zh-cn/3/library/wsgiref.html
