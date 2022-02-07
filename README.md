# 0.1测试版本：Kubernetes集群部署

##若出现部分组件无法使用的情况，可替换成自己的组件，

该项目不涉及不商业使用，仅供技术交流，本人在学习搭建SSM博客使用的是该项目的前端：https://github.com/saysky/ForestBlog 在此感谢作者。

## 1、准备工作

首先是准备Kubernetes所需的集群，这里以一主两仆架构，网络采取桥接模式，具体如下：

| 节点   | IP地址         | 操作系统   | 配置            |
| ------ | -------------- | ---------- | --------------- |
| Master | 192.168.31.112 | CentOS 7.9 | 2核CPU，2GB内存 |
| Node1  | 192.168.31.192 | CentOS 7.9 | 2核CPU，2GB内存 |
| Node2  | 192.168.31.172 | CentOS 7.9 | 2核CPU，2GB内存 |

### （1）安装版本

这里选用Docker（18.06.3），kubeadm（1.17.4）、kubelet（1.17.4）、kubectl（1.17.4）

### （2）剩余安装步骤省略

安装步骤可参考官方文档：https://kubernetes.io/zh/docs/tasks/tools/

## 2、搭建mysql

### （1）首先搭建RC

```yaml
apiVersion: v1
kind: ReplicationController										#副本控制器RC
metadata:
  name: mysql																	#RC的名称，全局唯一
spec:
  replicas: 1																	#Pod副本的期待数量
  selector:
    app: mysql																#符合目标的Pod拥有此标签
  template:																		#根据此模版创建Pod的副本
    metadata:
      labels:
        app: mysql														#Pod副本拥有的标签，对应RC的Selector
    spec:
      containers:															#Pod内容器的定义部分
      - name: mysql														#容器的名称
        image: mysql													#容器对应的Docker Image
        ports:
        - containerPort: 3306									#容器应用监听的端口号
        env:																	#注入容器内的环境变量
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```

创建成功后界面如下：

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118201822047.png)

### （2）然后创建Kubernetes Service，选择nodePort用以外部连接

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: NodePort
  ports:
    - port: 3306
      nodePort: 30060
  selector:
    app: mysql
```

创建成功后界面如下：

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203453890.png)

### （3）写好服务后使用Navicat连接，连接成功后将服务所需的数据库导入

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203244826.png)

### （4）连接成功后进入idea，将项目的jdbc地址改为Host地址进行测试

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203259814.png)

出现如下页面则成功：

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203312650.png)

## 3、打包并将SSM项目上传至DockerHub

### （1）准备工作

首先利用idea的maven插件将项目打成war包，更名成ROOT.war。然后将war文件和其他文件放在同一文件夹（这部分文件主要是webapps包里的）

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118200114711.png)

### （2）编写Dockfile，并将Dockerfile也放入文件夹中

```dockerfile
FROM tomcat:9.0.46
COPY ROOT.war /usr/local/tomcat/webapps
COPY docs /usr/local/tomcat/webapps
COPY examples /usr/local/tomcat/webapps
COPY host-manager /usr/local/tomcat/webapps
EXPOSE 8080
```

### （3）执行命令，创建镜像

```sh
docker build -t testtomcatcz1 .
```

然后创建并测试容器，出现2（3）的效果则为创建成功

### （4）登陆docker账户并上传镜像

```powershell
# 1 登陆docker账户
docker login

# 2 更改镜像名称，镜像名称格式为：“dockerID/镜像名”，否则会提示上传失败
docker tag testtomcatcz1 mark2first/testtomcatcz:v1

# 3 上传镜像
docker pull mark2first/testtomcatcz:v1
```

## 4、部署集群

### （1）配置deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testforward
spec:
  replicas: 3										# 这里用三个副本
  selector:
    matchLabels:
      run: testforward
  template:
    metadata:
      labels:
        run: testforward
    spec:
      containers:
      - image: mark2first/testtomcatcz:v1					# 这里以之前构建的镜像为例
        name: testforward
        ports:
        - containerPort: 8080
          protocol: TCP
```

DashBoard出现如下界面则创建成功：

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203408797.png)

### （2）创建Service，步骤同mysql创建Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-forward2
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: testforward
  type: NodePort
```

创建成功后界面如下：

![avatar](http://mark2first.top:9000/?explorer/share/fileOut&shareID=7zdB_p2w&path=%7BshareItemLink%3A7zdB_p2w%7D%2Fimage-20220118203339519.png)

### （3）访问http://192.168.31.112:32376，等待片刻出现2（3）的页面则整个集群创建成功

