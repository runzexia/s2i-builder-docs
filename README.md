# 创建 S2I 的构建器镜像与构建器模版

S2I使用源代码和构建器镜像生成新的Docker镜像，在我们的项目当中提供了部分常用的构建器镜像，例如[Python](https://github.com/kubesphere/s2i-python-container/)、[Java](https://github.com/kubesphere/s2i-java-container/)，您也可以定义自己的构建器镜像扩展S2i。

在详细介绍构建器影响之前，我们会先解释一下S2I的工作原理以及构建器镜像的作用。


1. 下载S2I构建器镜像（镜像提供构建脚本）。
2. 下载源代码。
3. 将源代码传入构建器镜像当中。
4. 运行构建器镜像所提供的`assemble`脚本进行应用构建。
5. 保存制品镜像。

构建器镜像需要有一些必要的内容才能完成所有工作。  

首先由于构建器镜像负责构建应用程序，因此它必须包含构建和运行应用程序所有需要的库和工具。例如Java构建器镜像将安装JDK、Maven等，而Python构建器镜像则可能需要pip等工具。  

其次构建器镜像需要提供脚本来执行构建和运行操作。这部分脚本在S2I当中为：

* assemble - 负责构建应用程序
* run - 负责运行应用程序

在以下的步骤中，我们将向您展示如何创建一个[nginx](https://www.nginx.com/)服务的构建器镜像。

## 第一步 使用S2I命令行工具引导构建器镜像所需目录

[S2I命令行工具](https://github.com/openshift/source-to-image/releases)带有一个方便的命令，可以引导构建器所需的目录结构，我们将使用以下命令生成我们的目录结构：

```bash
$ s2i create nginx-centos7 s2i-builder-docs
```
我使用`nginx-centos7`作为构建器镜像的名字创建了初始目录，目录结构如下所示

`s2i-builder-docs/`
  * `Dockerfile` - 一个标准的Dockerfile，定义了构建器镜像。
  * `Makefile` - 用于测试和构建构建器镜像的帮助脚本。
  * `test/`
    * run - 测试脚本，测试构建器镜像是否正常工作。
    * test-app/ - 用于测试的应用程序的目录
  * `s2i/bin`
    * assemble - 负责构建应用程序的脚本
    * run - 负责运行应用程序的脚本
    * usage - 负责打印构建器镜像用法的脚本

## 第二步修改 `Dockerfile` 定义构建器镜像

```Dockerfile
# nginx-centos7
FROM kubespheredev/s2i-base-centos7:1


LABEL maintainer="Runze Xia <runzexia@yunify.com>"

# 声明当前应用的版本
ENV NGINX_VERSION=1.6.3


LABEL io.k8s.description="Nginx Webserver" \
      io.k8s.display-name="Nginx 1.6.3" \
      io.kubesphere.expose-services="8080:http" \
      io.kubesphere.tags="builder,nginx,html"

# 安装nginx并且清理yum cache
RUN yum install -y epel-release && \
    yum install -y --setopt=tsflags=nodocs nginx && \
    yum clean all

# 修改nginx的默认开放端口
RUN sed -i 's/80/8080/' /etc/nginx/nginx.conf
RUN sed -i 's/user nginx;//' /etc/nginx/nginx.conf

# 将s2i的脚本复制到构建器镜像当中
COPY ./s2i/bin/ /usr/libexec/s2i

RUN chown -R 1001:1001 /usr/share/nginx
RUN chown -R 1001:1001 /var/log/nginx
RUN chown -R 1001:1001 /var/lib/nginx
RUN touch /run/nginx.pid
RUN chown -R 1001:1001 /run/nginx.pid
RUN chown -R 1001:1001 /etc/nginx

USER 1001

# 声明默认使用的端口
EXPOSE 8080

# 修改构建器的默认启动命令，以展示构建器镜像的用法
CMD ["/usr/libexec/s2i/usage"]


```

## 第三步 处理s2i构建器脚本

当我们完成了`Dockerfile`的定义，我们现在可以完成构建器镜像的其他部分。我们现在添加S2I脚本，我们将从`assemble`（负责构建应用程序）开始。在我们的例子中，它只是把`nginx`的配置文件以及静态内容复制到目标容器中：

```bash
#!/bin/bash -e

if [[ "$1" == "-h" ]]; then
	exec /usr/libexec/s2i/usage
fi

echo "---> Building and installing application from source..."
if [ -f /tmp/src/nginx.conf ]; then
  mv /tmp/src/nginx.conf /etc/nginx/nginx.conf
fi

if [ "$(ls -A /tmp/src)" ]; then
  mv /tmp/src/* /usr/share/nginx/html/
fi
```

默认情况下，`s2i build`将应用程序源代码放在`/tmp/src`目录中，在上面的命令当中，我们将应用源代码复制到了了`kubespheredev/s2i-base-centos7:1`定义的工作目录`/opt/app-root/src`当中。  

现在我们可以来处理第二个脚本`run`(用于启动应用程序)，在我们的例子当中，它只是启动`nginx`服务器：

```bash
#!/bin/bash -e

exec /usr/sbin/nginx -g "daemon off;"
```

我们使用`exec`命令将执行`run`脚本替换为执行 `nginx` 服务器的主进程。我们这样做是为了让所有`docker`发出的信号都可以被`nginx`收到，并且可以让`nginx`使用容器的标准输入和标准输出流。

我们在例子当中被没有实现增量构建，因此我们可以直接删除`save-artifacts`脚本。

最后我们在`usage`脚本当中添加一些使用信息：
```bash
#!/bin/bash -e
cat <<EOF
This is the nginx-centos7 S2I image:
To use it, install S2I: https://github.com/kubesphere/s2i-operator
Sample invocation:
s2i build test/test-app kubespheredev/nginx-centos7 nginx-centos7-app
You can then run the resulting image via:
docker run -d -p 8080:8080 nginx-centos7-app
and see the test via http://localhost:8080
EOF
```

## 第四步 构建与运行

当我们完成了`Dockerfile`和S2I的脚本，我们现在修改一下`Makefile当中的镜像名称`:
```makefile
IMAGE_NAME = kubespheredev/nginx-centos7-s2ibuilder-sample

.PHONY: build
build:
	docker build -t $(IMAGE_NAME) .

.PHONY: test
test:
	docker build -t $(IMAGE_NAME)-candidate .
	IMAGE_NAME=$(IMAGE_NAME)-candidate test/run

```
可以执行`make build`命令来构建我们的构建器镜像了。

```bash
$ make build
docker build -t kubespheredev/nginx-centos7-s2ibuilder-sample .
Sending build context to Docker daemon  164.9kB
Step 1/17 : FROM kubespheredev/s2i-base-centos7:1
 ---> 48f8574c05df
Step 2/17 : LABEL maintainer="Runze Xia <runzexia@yunify.com>"
 ---> Using cache
 ---> d60ebf231518
Step 3/17 : ENV NGINX_VERSION=1.6.3
 ---> Using cache
 ---> 5bd34674d1eb
Step 4/17 : LABEL io.k8s.description="Nginx Webserver"       io.k8s.display-name="Nginx 1.6.3"       io.kubesphere.expose-services="8080:http"       io.kubesphere.tags="builder,nginx,html"
 ---> Using cache
 ---> c837ad649086
Step 5/17 : RUN yum install -y epel-release &&     yum install -y --setopt=tsflags=nodocs nginx &&     yum clean all
 ---> Running in d2c8fe644415

…………
…………
…………

Step 17/17 : CMD ["/usr/libexec/s2i/usage"]
 ---> Running in c24819f6be27
Removing intermediate container c24819f6be27
 ---> c147c86f2cb8
Successfully built c147c86f2cb8
Successfully tagged kubespheredev/nginx-centos7-s2ibuilder-sample:latest
```

可以看到我们的镜像已经构建成功了，现在我们执行 `s2i build ./test/test-app kubespheredev/nginx-centos7-s2ibuilder-sample:latest sample app` 命令来构建我们的应用镜像。

```bash
$ s2i build ./test/test-app kubespheredev/nginx-centos7-s2ibuilder-sample:latest sample-app
---> Building and installing application from source...
Build completed successfully
```

当我们完成了应用镜像的构建，我们可以在本地运行这个应用镜像看构建出的应用是否符合我们的要求：

```bash
$ docker run -p 8080:8080  sample-app
```

在浏览器中访问可以看到我们的网页已经可以正常访问了:
![Hello-World](images/hello-world.jpg)


## 第五步 推送镜像并添加 KubeSphere 构建器模版

当我们在本地完成S2I构建器镜像的测试之后，就可以推送镜像到镜像仓库当中，并创建构建器模版`yaml`文件：

```yaml
apiVersion: devops.kubesphere.io/v1alpha1
kind: S2iBuilderTemplate
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: nginx-demo
spec:
  baseImages: # 构建器镜像名称，同一代码框架的多个不同版本。
    - kubespheredev/nginx-centos7-s2ibuilder-sample
  codeFramework: nginx # 代码框架类型
  defaultBaseImage: kubespheredev/nginx-centos7-s2ibuilder-sample # 默认使用的构建器镜像
  version: 0.0.1 # 构建器模版的版本
  description: "This is a S2I builder template for Nginx builds whose result can be run directly without any further application server.." # 构建器模版的描述信息

```

在创建好构建器模版后我们可以使用 `kubectl` 将构建器模版提交到KubeSphere环境当中：
```bash
$ kubectl apply -f s2ibuildertemplate.yaml
s2ibuildertemplate.devops.kubesphere.io/nginx created
```

现在我们来到KubeSphere的控制台界面，我们已经可以选择添加的构建器模版了：

![Hello-World](images/s2i-in-dashborad.jpg)

至此我们就完成了S2I构建器镜像与构建器模版的创建。
