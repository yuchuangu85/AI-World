<h1 align="center">Docker n8n ollama本地模型</h1>



## 安装

官方安装命令，要先安装docker客户端。

```
docker volume create n8n_data
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

安装到第二步时报错：

```
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
Unable to find image 'docker.n8n.io/n8nio/n8n:latest' locally
```

需要翻墙

或者

根据：[docker.n8n.io/n8nio/n8n:latest · Issue #3081 · kubesre/docker-registry-mirrors](https://github.com/kubesre/docker-registry-mirrors/issues/3081)提示使用：

```
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n:latest
```

或者使用镜像的：

```
# Docker 拉取命令
docker pull swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest
docker tag swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest docker.n8n.io/n8nio/n8n:latest
# 然后启动
# 或者直接执行如下命令
docker run -d --name n8n -p 5678:5678 swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest

# Containerd 拉取命令
ctr images pull swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest
ctr images tag swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest docker.n8n.io/n8nio/n8n:latest

# Shell 快速替换命令
sed -i s#docker.n8n.io/n8nio/n8n:latest#swr.cn-east-3.myhuaweicloud.com/kubesre/docker.n8n.io/n8nio/n8n:latest#g 你的文件名
```

可以拉取成功

然后启动，

![](./media/docker-n8n-01.png)

开启后自动跳转到如下页面，点击下面链接，

![](./media/docker-n8n-02.png)



## 使用n8n创建工作流

点击打开网页

![](./media/docker-n8n-03.png)

注册进入，点击左上角，创建工作流

![](./media/docker-n8n-04.png)

创建完成如下图：

![](./media/docker-n8n-05.png)

点击中间加号，创建节点，

![](./media/docker-n8n-06.png)

选择On chat message,

![](./media/docker-n8n-07.png)

点击Back to canvas

![](./media/docker-n8n-08.png)

继续点击后面的加号，然后点击Advanced AI,

<img src="./media/docker-n8n-09.png" style="zoom:50%;" />

选择`Basic LLM Chain`

![](./media/docker-n8n-10.png)

点击加号，然后选择 `Ollama Chat Model`

![](./media/docker-n8n-11.png)

点击`Select Credential` -> `Create new credential`

![](./media/docker-n8n-12.png)

docker中启动的修改Base URL(http://localhost:11434)为：`http://host.docker.internal:11434`，点击保存

![](./media/docker-n8n-13.png)

链接成功，返回选择模型：

<img src="./media/docker-n8n-14.png" style="zoom:50%;" />

选择完成后：

![](./media/docker-n8n-15.png)

就有了选好的模型节点，在输入框输入信息，

![](./media/docker-n8n-16.png)

等待加载信息

![](./media/docker-n8n-17.png)

返回了信息，这个就是本地模型Gemma返回的。

后面会介绍一下n8n + MCP的创建，还有一些实战工作流的创建。



## 参考

* [利用本地DeepSeek R1和n8n框架，创建AI智能体_n8n deepseek-CSDN博客](https://blog.csdn.net/Code1994/article/details/145653684)
* [AI工作流+MCP：零代码打造最强AI Agent，一键接入海量工具 | N8N+MCP实战教程！](https://www.youtube.com/watch?v=c2Ecz0tI7IU)
* 

