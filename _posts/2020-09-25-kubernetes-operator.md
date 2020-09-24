---
layout: post
title: k8s apiserve/controller及operator工作流解析
tags: [kubernetes]
---

### k8s工作流简述
本文的apigroup用x.xx.x替换，你可以认为是deployment的group, 基于crd的controller分析， 自定义资源的名字为videoprocessor，可以任务是一个业务app。

架构图：k8s的架构，主要是master node节点结构， master上有apiserver/controller/scheduler等，node上有kubelet/kube-proxy等，容器网络一般是二层的flannel,或者三层的vxlan.
![architecture.png](http://www.mrzzjiy.cn/assets/architecture.png)

如下两张图，生动描述了k8s的一些工作流,扣的图，文章末尾有参考资料，大家也可以看下，写的很好。   

![scenes-from-kubernetes-page1.png](http://www.mrzzjiy.cn/assets/scenes-from-kubernetes-page1.png)
![scenes-from-kubernetes-page2.png](http://www.mrzzjiy.cn/assets/scenes-from-kubernetes-page2.png)


videoprocessors get流程  
    => GET https://172.20.26.162:6443/apis/algo.viper.sensetime.com/v1alpha2/namespaces/default/videoprocessors?limit=500；    
    => apiserver handler处理这个请求，从etcd获取配置信息；  
    => return json body  
    
videoprocessors put  post请求都是类似的过程，通过apiserver，验证schema没有问题，则存储到etcd中。  
apiserver如何知道videoprocessor的schema:   通过crd定义，apiserver会自动添加crd对应资源的rest apis接口。  


### etcd 
 
在分布式系统中，如何管理节点间的状态一直是一个难题，etcd像是专门为集群环境的服务发现和注册而设计，它提供了数据TTL失效、数据改变监视、多值、目录监听、分布式锁原子操作等功能，可以方便的跟踪并管理集群节点的状态。Etcd的特性如下：  

- 简单: curl可访问的用户的API（HTTP+JSON）
- 安全: 可选的SSL客户端证书认证 
- 快速: 单实例每秒1000次写操作
- 可靠: 使用Raft算法保证一致性

export ETCDCTL_API=3; etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem  --cert=/etc/ssl/etcd/ssl/admin-controller-162.pem --key=/etc/ssl/etcd/ssl/admin-controller-162-key.pem get /registry --prefix -w json
etcd存储的k8s key:value都进行base64 encode   

例如videoprocessors的存储的key如下：  
/registry/{group name}/{crd name}/{namespace}/{resource-name}  
/registry/x.xx.x/videoprocessors/default/engine-video-process-worker-face

value的话，就是我们kubectl get videoprocessors a-b-c-d -o json的json结果。

### apiserver
apiserver提供集群的restful api接口的统一入口，

有一个cluster-admin的clusterrole，我们可以基于该role，创建一个admin的用户，并拿到token，以便我们用curl进行api访问,apply如下的配置文件到default namespace下；

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-admin

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: myadmin
subjects:
  - kind: ServiceAccount
    name: my-admin
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io  
  
```

然后我们就可以通过curl来查询apiserver的接口了,既然已经拿到token了，那么我们在集群外也是可以访问的，比如通过postman
```
TOKEN=$(kubectl get secret $(kubectl get serviceaccount my-admin -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode )

curl -k https://172.20.25.160:6443/ --header "Authorization: Bearer $TOKEN"
```
以下为个人理解，有误请随时指出。  
我们可以看下apiserver的api组成结构， 由于历史原因是存在/api/v1这种组织形式的，这个的话可能就这样了吧，其实也就是一些核心的资源在这个rest下面，那新的组织方式是按照groupversion的方式来组织的，比如vps的资源apis/algo.viper.sensetime.com/v1alpha2/namespaces/default/videoprocessors，它的groupName是algo.viper.sensetime.com，版本是v1alpha2,后面的就是namespaces/{{namespace}}/{{资源名字}}, 需要指出的是namespaces是可选的，是可以不传的，默认的话就是default的namespace了。
```
#接口组织方式
/ 
  => /api
    => /v1
      => /configmap等
      
  => /apis
    => /algo.viper.sensetime.com/v1alpha1
      => /namespaces/default/videoprocessors
    => etc...  
    
#groupversion下提供了哪些能力，可以看到verbs里面能力，常用的get/delete/patch/update/watch等，这些接口个人理解都是包装的对etcd的操作，而且应该会有crd的controller来对这些资源做控制，自动注册路由，取消路由等这些操作。 至于具体的etcd操作，底层已经封装好了实现，接口调用就行了。
{
    "kind": "APIResourceList",
    "apiVersion": "v1",
    "groupVersion": "x.xx.x",
    "resources": [
        {
            "name": "videoprocessors",
            "singularName": "videoprocessor",
            "namespaced": true,
            "kind": "VideoProcessor",
            "verbs": [
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "create",
                "update",
                "watch"
            ],
            "storageVersionHash": "BJtAlAUT81o="
        },
        {
            "name": "videoprocessors/status",
            "singularName": "",
            "namespaced": true,
            "kind": "VideoProcessor",
            "verbs": [
                "get",
                "patch",
                "update"
            ]
        }
    ]
}


{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/x.xx.x",
    "/apis/x.xx.x/v1alpha1",
    "/apis/x.xx.x/v1alpha2",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/apps/v1beta1",
    "/apis/apps/v1beta2",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/configuration.konghq.com",
    "/apis/configuration.konghq.com/v1",
    "/apis/configuration.konghq.com/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/metrics.k8s.io",
    "/apis/metrics.k8s.io/v1beta1",
    "/apis/monitoring.coreos.com",
    "/apis/monitoring.coreos.com/v1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/ca-registration",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/version"
```
重点看下watch接口是干啥的，其实后面的client-go的lister informer机制最终还是用到了这个接口  
watch可以作为一个path param传入，也可以作为一个query传入如下url，实际上它是一个get请求;http keepalived长连接，Connection: Keep-Alive，http1.1开始好像默认都是开启的长连接；Transfer-Encoding：chunked， 传输的数据以块的方式进行，数据传送结束发送长度为0的chunked，apiserver具体能保持连接的时间好像是随机的，也可以通过选项来配置--min-request-timeout=1800；

curl -k --header "Authorization: Bearer $TOKEN" https://172.20.25.160:6443/apis/x.xx.x/v1alpha2/namespaces/default/videoprocessors?watch=true&continue=true&fieldSelector=metadata.name=xxxxx 

### client-go lister informer机制
前面有讲到apiserver的apigroup机制以及watch接口的使用，那现在讲的informer机制就依赖了apiserver的watch接口，直接以ips-operator的informer来看下具体是怎么实现的,借用下operator-sdk的图， 通过Reflector list Watch k8s资源，这里就是通过apiserver的watch接口实现，运行informer初始化会sync k8s的资源，接收watch事件并推送到deltaFIFO队列， 并通过indexer缓存起来，lister查询缓存；这样的机制不会给apiserver造成很大的压力。  
一个简单的controller示例
```
package main

import (
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	client, err := newK8sClient()
	if err != nil {
		panic(err)
	}

	stopChan := make(chan struct{})
	factory := informers.NewSharedInformerFactory(client, 10*time.Second)

	podInfomer := factory.Core().V1().Pods()
	podInfomer.Informer().AddEventHandler(eventHandler{})

	go podInfomer.Informer().Run(stopChan)

	if ok := cache.WaitForCacheSync(stopChan, podInfomer.Informer().HasSynced); !ok {
		panic("sync cache error, maybe apiserver have some problems")
	}

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, os.Interrupt, syscall.SIGTERM, syscall.SIGINT)

	sig := <-sigs
	println("controller stop , recevie sig ", sig)
}

func newK8sClient() (*kubernetes.Clientset, error) {
	config, err := clientcmd.BuildConfigFromFlags("", "/Users/zhouxiaoming/.kube/config")
	if err != nil {
		return nil, err
	}
	return kubernetes.NewForConfig(config)
}

type eventHandler struct{}

func (e eventHandler) OnAdd(obj interface{}) {
	println("this is on add event handler")
}

func (e eventHandler) OnUpdate(old, new interface{}) {
	println("this is on update event handler")
}

func (e eventHandler) OnDelete(obj interface{}) {
	println("this is on delete event handler")
}

```


![pic](http://www.mrzzjiy.cn/assets/operator-sdk.jpeg)

### apply一个videoprocssors资源的时候发生了啥？
通过下图，比较明了的表现了apply一个smoking的vps app的时候，k8s到底发生了啥，如果还需要深入了解的话，就得深入源码分析了。

![pic](http://www.mrzzjiy.cn/assets/k8s-crd-operator-flow.png)



参考资料:  
[etcd](https://www.tony-yin.site/2019/05/15/Etcd_Service_HA/ )  
[kubernetes api docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/)  
[apiserver list watch](http://dockone.io/article/1538)