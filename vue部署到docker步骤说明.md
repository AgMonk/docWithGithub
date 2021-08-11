 配置文件的内容中我都补充了注释，注意查看以帮助理解

## 创建Dockerfile文件

在代码根目录下创建 Dockerfile 文件，无后缀名，其内容为

```
#使用nginx镜像为基础镜像
FROM nginx
#将当前目录中的dist文件夹（即vue打包完成的文件）内容复制到 镜像的 /usr/share/nginx/html/ 文件夹（即nginx的静态文件根目录）
COPY ./dist/ /usr/share/nginx/html/
#将nginx.conf 文件复制到  /etc/nginx/nginx.conf 这是nginx的运行配置文件
COPY nginx.conf /etc/nginx/nginx.conf
```

## 创建nginx运行配置文件

在代码根目录下创建 nginx.conf 文件，即上一步骤最后一行需要进行复制的文件，其内容如下

注意根据实际情况替换其中的 ${xxxxx} 

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    client_max_body_size   20m;
    server {
        listen       80;
        server_name  ${部署的服务器ip地址};

		# 配置静态文件根目录位置 注意这里需要与上一步骤中的路径一致
        location / {
	        # 根目录
            root   /usr/share/nginx/html;
            # 首页文件名
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }
        # 配置代理 应与vue的proxy（vue3）或proxyTable（vue2）的配置方案一致
        # 本例中是吧 /api 开头的地址 映射到了 ${api绝对路径}
        location ^~ /api/ {
            proxy_pass ${api绝对路径};
        }
      	# 静态资源目录 当需要通过静态方式访问资源（如下载文件、显示图片）时需要配置此项
      	# 本例中是吧 /attachment 开头的地址 映射到了 ${本容器中的文件夹路径}
      	# 注意：本容器中的文件在容器被重新创建之后就会丢失
      	# 所以后续我们需要把本容器的该路径映射到宿主机的相同路径下
        location ^~ /attachment/ {
            alias ${本容器中的文件夹路径};
        }
      
    }
}
```

## 创建部署脚本 build.sh

使用FinalShell 连接到目标服务器

进入 /home 文件夹

```
cd /home
```

新建一个文件夹，命名为你的项目名称，如：project

```
mkdir project
```

进入该文件夹

```
cd project
```

或后续再次部署时可以直接

```
cd /home/project
```

新建脚本文件 build.sh

```
touch build.sh
```

在FinalShell的文件列表中双击打开编辑，其内容为

注意根据实际情况替换前三行中的 ${xxxxx} 

```
name=${镜像和容器名称，一般为项目名称}
mapDir=${需要映射的文件夹路径}
port=${本容器对外开放的端口}

echo '停止容器运行';
docker container stop ${name}
echo '删除容器';
docker container rm ${name}
echo '删除镜像';
docker image rm ${name}:latest
echo '打包镜像';
docker build -t ${name} .
echo '使用镜像创建容器并运行';
docker run -p ${port}:80 -v ${mapDir}:${mapDir} -d --name ${name} ${name}:latest;
```

其中：本容器对外开放的端口，指部署完毕后，通过地址 http://服务器ip:端口/   可以访问到本前端，此处的端口，由运营管理员提供。

将 Dockerfile 、 nginx.conf 也上传到该目录

## 部署运行

进入项目目录

```
cd /home/project
```

将项目打包好的 dist 文件夹上传到该目录，然后执行之前创建好的脚本即可

```
sh build.sh
```

后续部署前需要先删除上次部署的 dist文件夹

```
rm -rf /home/project/dist
```

或者也可以把这句写在脚本的最后一句





