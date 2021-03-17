# go-debug-delve
当你的应用只与kubernetes暴露的api一起工作时，你可以简单地在IDE上以调试模式启动你的应用并调试你的应用。
但是当你的应用需要连接到其他服务或组件时，这些服务或组件只能在Kubernetes集群内使用，那么这个方案将无法工作。
### repository
在本文中，我将使用【kubernetes-go-grpc repository】。你可以在这篇文章中找到更多的细节和部署说明如何开发Go gRPC微服务并在Kubernetes中部署。
由于本文专注于调试方案，所以我不打算描述关于项目的情况。

### Solution
在生成二进制Go时，编译器可以改变操作顺序，增加代码，删除代码或应用变换。所以，将go代码行映射到优化后的输出是比较困难的。
在调试过程中，我们需要在特定的go 代码行调试或停止执行。因此，我们需要生成没有优化的二进制。你可以通过使用`gcflags`向编译器传递参数来禁止优化。
执行下面的命令来生成不经过优化的二进制文件。
```
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -gcflags "all=-N -l" -o ./app
```
这里的"-N "将关闭优化，"-l "将关闭内联。要想远程调试，我们需要在机器上启动一个无头Delve服务器。
```
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./app
```
如果服务本身需要传递参数，使用如下方式
```
dlv --listen=:40000 --headless=true --api-version=2 --accept-multiclient exec ./app -- --logtostderr=true
```
### Dockerfile
```
FROM golang
RUN go get -u github.com/go-delve/delve/cmd/dlv
RUN mkdir app
WORKDIR /app
COPY app .
EXPOSE 40000
EXPOSE 8080
ENTRYPOINT ["/go/bin/dlv", "--listen=:2345", "--headless=true", "--api-version=2", "exec", "./app"]
```
将应用部署到k8s上，现在需要转发下2345 端口，就可以进行远程调试了
```
kubectl port-forward <YOUR_POD_NAME> 2345:2345
```
### Configure the IDE 
You need to create a Remote Debug and point it to localhost:2345 since you have activated the port forward.


#### 参考
[debugging-go-application-inside-kubernetes](https://hackernoon.com/debugging-go-application-inside-kubernetes-from-ide-h5683xeb)
<br>
[Debugging a go in k8s](https://itnext.io/debug-a-go-application-in-kubernetes-from-ide-c45ad26d8785)
