---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 访问ApiServer
Apiserver 做为 Kubernetes master 的核心组件之一起着非常重要的作用，因此如何访问 apiserver 就变得非常的重要，我们分为2部分来介绍一下如何访问 apiserver。

## ① 容器外
容器外指的是 `我们并不是在 Pod 内容器访问 apiserver`，比如我们在集群的节点上通过console访问，或者在其它集群通过HTTP协议进行访问的方式。容器外访问 apiserver 主要有以下几种方式，这些使用方式主要用在运维排查场景。

1. [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)  
   ```
   kubectl 是最常使用的方式，我们日常的运维开发过程中使用 kubectl 命令都是直接访问 apiserver，从而间接控制 etcd 的状态。
   kubectl 默认情况下会读取 ~/.kube/config 做为 kubeconfig 文件内容，我们也可以使用 --kubeconfig 选项来指定kubeconfig文件路径。  
   访问 apiserver 需要进行权限控制，因此我们需要有相关的访问凭证，而这些访问凭证会存储在 kubeconfig 文件内。
   ```
    
2. kubectl proxy  
   ```
   kubectl proxy 实际上是启动一个代理的 HTTP 服务，使得我们可以通过 HTTP 协议进行访问 apiserver，具体的步骤如下  
   1). 启动proxy: kubectl proxy --port=8080
   2). 访问api: curl http://localhost:8080/api/v1/namespaces/kube-system/services/hostnames 
   
   api url path跟具体的 Kubernetes API 对象有关，具体可以看 API 对象的 metadata.selfLink
   ```
   
3. 直接访问  
   ```
   APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
   SECRET_NAME=$(kubectl get secrets | grep ^default | cut -f1 -d ' ')
   TOKEN=$(kubectl describe secret $SECRET_NAME | grep -E '^token' | cut -f2 -d':' | tr -d " ")

   curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
   ```

## ② 容器内
容器内大部分场景指的是我们在代码层面访问 apiserver，例如我们需要实现一个 Kubernetes watcher 组件用于监听 Pod 事件，那这个时候我们就需要访问 apiserver 获取 Pod 的事件。由于 Kubernetes 是 Golang 开发的，建议业务代码使用 Golang 开发，那么就可以直接 Kubernetes 社区提供的 [client-go](https://github.com/kubernetes/client-go)。初次之外，还有 [python client](https://github.com/kubernetes-client/python) 可以使用。

最常见的代码如下

```
package main

import (
	"fmt"
	"time"

	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func main() {
	// creates the in-cluster config
	config, err := rest.InClusterConfig()
	if err != nil {
		panic(err.Error())
	}
    
	// creates the clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
    
	for {
		// get pods in all the namespaces by omitting namespace
		// Or specify namespace to get pods in particular namespace
		pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
		if err != nil {
			panic(err.Error())
		}
		fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
      
        // TODO @cgl
        
		time.Sleep(10 * time.Second)
	}
}
```


# 二. 访问业务Server
