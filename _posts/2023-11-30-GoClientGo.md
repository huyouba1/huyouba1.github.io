---
layout: post
title: Client-go 介绍
date: 2023-08-28
tags: Go
---


## GVR/GVK
### 简介
在围绕Kubernetes的功能开发中，经常会调用到Kubernetes提供的API，本章就来介绍一下Kubernetes的API接口规则

Kubernetes是通过HTTP协议咦RESTFUL的形式提供的，同时支持JSON和Protobuf的数据格式，Protobuf是方便集群内部调用而支持的，我们自己平时调用Kubernetes接口，一般都是使用JSON数据格式的。

在Kubernetes API中，我们一般使用GVR或GVK来区分特定的资源，即根据不同的分组、版本以及资源，进行URL定义，有了分组和多版本的支持，即便是后续的版本中，需要去掉资源对象的某些字段或者重构API资源，也可以保证版本之间的兼容性。

同时，因为历史原因，Kubernetes的API分组是分`吴无组名资源组`和`有组名资源组`的，`无组名资源组`也被称之为核心资源组，Core Group。

**有组名资源组**
![](/images/posts/media/clientgo1.png)

**无组名资源组**
![](/images/posts/media/clientgo2.png)


### GVR/GVK 含义介绍
- G(Group)： 资源组，包含一组资源操作的集合
- V(Version)：资源版本，用于区分不同API的稳定程度及兼容性
- R(Resource): 资源信息，用于区分不同的资源API
- K(Kind)：资源对象的类型，每个资源类型都需要Kind来区分他自身代表的资源类型。

一般在接口调用的时候，我们只需要知道GVR即可。通过GVR操作对应的资源对象。

通过GVR组成RESTFUL API请求路径。例如针对apps/v1下面Deployment的RESTFUL API请求路径如下所示:

```
GET /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

GVK，相反，通过GVK信息则可以获取要读取得资源对象的GVR，进而构建RESTFUL API请求获取对应的资源。这种GVK与GVR的映射叫做RESTMapper。

RESTMapper其主要作用是在ListerWatcher时，根据schema定义的类型GVK解析出GVR，向APIserver发起HTTP请求获取资源，然后watch





## Client-Go 简介
Client-go 是负责与Kubernetes Apiserver服务进行交互的客户端库，利用Client-Go与Kubernetes进行的交互访问，以此来对Kubernetes中的各类资源对象进行管理操作，包括内置的资源对象以及CRD。

Client-go不仅被Kubernetes项目本身使用，其他围绕Kubernetes的生态，也被大量的使用，例如：kubectl，ETCD-operator等

### Client-go客户端对象
Client-go提供了4种与KubernetesApiserver交互的客户端对象，分表是RESTClient、DiscoveryClient、ClientSet、DynamicClient。

- RESTClient：最基础的客户端，主要是对HTTP请求进行封装，支持Json和Protobuf格式的数据。
- DiscoveryClient：发现客户端，负责发现APIServer支持的资源组、资源版本和资源信息
- ClientSet：负责操作Kubernetes内置的资源对象，例如Pod、Service等
- DynamicClient：动态客户端，可以对任意的Kubernetes资源对象进行通用操作，包括CRD


![](/images/posts/media/clientgo3.png)


### RESTClient
RESTClient 是所有客户端的父类，是最基础的客户端。

它提供了RESTful对应的方法封装，如：Get()、Put()、Post()、Delete()等。通过这些封装的方法与Kubernetes Apiserver进行交互。

因为他是所有客户端的父类，所以他可以操作Kubernetes内置的所有资源对象以及CRD。

**演示**

> 运行以下代码，将得到default下的pod部分相关资源信息

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 1. 加载配置文件，生成config对象
	config, err := clientcmd.BuildConfigFromFlags("", "config")
	if err != nil {
		panic(err)
	}

	// 2. 配置api路径
	config.APIPath = "api" // pods, /api/v1/pods
	//config.APIPath = "apis" // deployments, /apis/apps/v1/namespaces/{namespace}/deployments/{deployment}

	// 3. 配置分组版本
	config.GroupVersion = &corev1.SchemeGroupVersion // 无名资源组 group: "", version: "v1"

	// 4.配置数据的编解码工具
	config.NegotiatedSerializer = scheme.Codecs

	// 5. 实例化RESTFUL对象
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
	// 6.定义接收返回值的变量
	result := &corev1.PodList{}

	// 7. 跟APIserver交互
	err = restClient.Get().
		Namespace("default").
		Resource("pods").
		VersionedParams(&metav1.ListOptions{}, scheme.ParameterCodec). // 参数及参数的序列化工具
		Do(context.TODO()).                                            // 触发请求
		Into(result)                                                   // 写入返回结果

	if err != nil {
		panic(err.Error())
	}

	for _, item := range result.Items {
		fmt.Println(item.Namespace, "=====", item.Name)
	}
}
```

输出
```
default ===== mysql-cluster-0
default ===== mysql-cluster-1
default ===== mysql-cluster-2
default ===== nacos-k8s-0
default ===== nacos-k8s-1
default ===== nacos-k8s-2
default ===== nfs-client-provisioner-cd4dfd779-vhbj6
default ===== tomcat-deploy-8458c7c775-7nf5p
default ===== tomcat-deploy-8458c7c775-b8lhw
```

### ClientSet
前面介绍了RESTClient，他虽然可以操作Kubernetes的所有资源对象，但是使用起来确实比较复杂，需要配置的参数过于繁琐，因此，为了更优雅更方便的与Kubernetes Apiserver进行交互，则需要进一步的封装。

ClientSet是基于RESTClient的封装，同时ClientSet是使用预生成的API对象与APIserver进行交互的，这样做更方便我们进行二次开发。

ClientSet是一组资源对象客户端的集合，例如负责操作Pods、Services等资源的CoreV1Client，负责操作Deployments、DaemonSets等资源的AppsV1Client等。通过这些资源对象客户端提供的操作方法，即可对Kubernetes内置的资源对象进行Create、Update、Get、List、Delete等操作

**演示**
> 运行以下代码，将得到default下的pod部分相关资源信息

```go
package main

import (
	"context"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}

	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	pods, err := clientSet.
		CoreV1(). // 返回CoreV1client实例
		Pods("default").
		List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}

	for _, item := range pods.Items {
		fmt.Println(item.Namespace, "==", item.Name)
	}

}

```

输出
```
default == mysql-cluster-0
default == mysql-cluster-1
default == mysql-cluster-2
default == nacos-k8s-0
default == nacos-k8s-1
default == nacos-k8s-2
default == nfs-client-provisioner-cd4dfd779-vhbj6
default == tomcat-deploy-8458c7c775-7nf5p
default == tomcat-deploy-8458c7c775-b8lhw
```


### DynamicClient
DynamicClient是一种动态客户端，通过动态指定资源组、资源版本和资源等信息，来操作任意的Kubernetes资源对象的一种客户端。即不仅仅是操作Kubernetes内置的资源对象，还可以操作CRD。这也是与ClientSet最明显的一个区别。

使用ClientSet的时候，程序会将所用的版本与类型紧密耦合。而DynamicClient使用嵌套的map[string]interface{}结构存储Kubernetes Apiserver的返回值，使用反射机制，在运行的时候，进行数据绑定，这种方式更加灵活，但是缺无法获取强数据类型的检查和验证。

此外，还需要了解另外两个重要的知识点，Object.runtime接口和Unstructured结构体。

- Object.runtime: Kubernetes中的所有资源对象，都实现了这个接口，其中包含DeepCopyObject和GetObjectKind的方法，分别用于对象深拷贝和获取对象的具体资源类型。
- Unstructured: 包含map[string]interface{}类型字段，在处理无法预知结构的数据时，将数据值存入interface{}中，待运行时利用反射判断。该结构体提供了大量的工具方法，便于处理非结构化的数据。

**演示**
> 运行以下代码，将得到default下的pod部分相关资源信息

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}

	dynamicCli, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	gvr := schema.GroupVersionResource{
		Group:    "",
		Version:  "v1",
		Resource: "pods",
	}

	unStructData, err := dynamicCli.Resource(gvr).Namespace("default").List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}

	// 将unStructData转换为结构化数据
	podList := &corev1.PodList{}

	err = runtime.DefaultUnstructuredConverter.FromUnstructured(unStructData.UnstructuredContent(), podList)

	for _, item := range podList.Items {
		fmt.Println(item.Namespace, "==", item.Name)
	}
}
```
输出
```
default == mysql-cluster-0
default == mysql-cluster-1
default == mysql-cluster-2
default == nacos-k8s-0
default == nacos-k8s-1
default == nacos-k8s-2
default == nfs-client-provisioner-cd4dfd779-vhbj6
default == tomcat-deploy-8458c7c775-7nf5p
default == tomcat-deploy-8458c7c775-b8lhw
```


### DiscoveryClient

前面介绍的3种客户端对象，都是针对与资源对象管理的。而DiscoveryClient则是针对于GVR的。用于查看当前Kubernetes集群支持那些资源组、资源版本、资源信息。


> 运行以下代码，将得到Kubernetes集群的资源列表。


```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}

	disCli, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}

	_, apiResources, err := disCli.ServerGroupsAndResources()
	if err != nil {
		panic(err)
	}

	for _, list := range apiResources {
		gv, _ := schema.ParseGroupVersion(list.GroupVersion)

		for _, resource := range list.APIResources {
			fmt.Println(resource.Name, gv.Group, gv.Version)
		}
	}
}

```
输出结果
```
bindings  v1
componentstatuses  v1
configmaps  v1
endpoints  v1
events  v1
limitranges  v1
namespaces  v1
namespaces/finalize  v1
namespaces/status  v1
nodes  v1
nodes/proxy  v1
nodes/status  v1
persistentvolumeclaims  v1
persistentvolumeclaims/status  v1
persistentvolumes  v1
persistentvolumes/status  v1
pods  v1
pods/attach  v1
pods/binding  v1
pods/eviction  v1
pods/exec  v1
pods/log  v1
pods/portforward  v1
pods/proxy  v1
pods/status  v1
podtemplates  v1
replicationcontrollers  v1
replicationcontrollers/scale  v1
replicationcontrollers/status  v1
resourcequotas  v1
resourcequotas/status  v1
secrets  v1
serviceaccounts  v1
serviceaccounts/token  v1
services  v1
services/proxy  v1
services/status  v1
apiservices apiregistration.k8s.io v1
apiservices/status apiregistration.k8s.io v1
apiservices apiregistration.k8s.io v1beta1
apiservices/status apiregistration.k8s.io v1beta1
controllerrevisions apps v1
daemonsets apps v1
daemonsets/status apps v1
deployments apps v1
deployments/scale apps v1
deployments/status apps v1
replicasets apps v1
replicasets/scale apps v1
replicasets/status apps v1
statefulsets apps v1
statefulsets/scale apps v1
statefulsets/status apps v1
events events.k8s.io v1
events events.k8s.io v1beta1
tokenreviews authentication.k8s.io v1
tokenreviews authentication.k8s.io v1beta1
localsubjectaccessreviews authorization.k8s.io v1
selfsubjectaccessreviews authorization.k8s.io v1
selfsubjectrulesreviews authorization.k8s.io v1
subjectaccessreviews authorization.k8s.io v1
localsubjectaccessreviews authorization.k8s.io v1beta1
selfsubjectaccessreviews authorization.k8s.io v1beta1
selfsubjectrulesreviews authorization.k8s.io v1beta1
subjectaccessreviews authorization.k8s.io v1beta1
horizontalpodautoscalers autoscaling v1
horizontalpodautoscalers/status autoscaling v1
horizontalpodautoscalers autoscaling v2beta1
horizontalpodautoscalers/status autoscaling v2beta1
horizontalpodautoscalers autoscaling v2beta2
horizontalpodautoscalers/status autoscaling v2beta2
jobs batch v1
jobs/status batch v1
cronjobs batch v1beta1
cronjobs/status batch v1beta1
certificatesigningrequests certificates.k8s.io v1
certificatesigningrequests/approval certificates.k8s.io v1
certificatesigningrequests/status certificates.k8s.io v1
certificatesigningrequests certificates.k8s.io v1beta1
certificatesigningrequests/approval certificates.k8s.io v1beta1
certificatesigningrequests/status certificates.k8s.io v1beta1
ingressclasses networking.k8s.io v1
ingresses networking.k8s.io v1
ingresses/status networking.k8s.io v1
networkpolicies networking.k8s.io v1
ingressclasses networking.k8s.io v1beta1
ingresses networking.k8s.io v1beta1
ingresses/status networking.k8s.io v1beta1
ingresses extensions v1beta1
ingresses/status extensions v1beta1
poddisruptionbudgets policy v1beta1
poddisruptionbudgets/status policy v1beta1
podsecuritypolicies policy v1beta1
clusterrolebindings rbac.authorization.k8s.io v1
clusterroles rbac.authorization.k8s.io v1
rolebindings rbac.authorization.k8s.io v1
roles rbac.authorization.k8s.io v1
clusterrolebindings rbac.authorization.k8s.io v1beta1
clusterroles rbac.authorization.k8s.io v1beta1
rolebindings rbac.authorization.k8s.io v1beta1
roles rbac.authorization.k8s.io v1beta1
csidrivers storage.k8s.io v1
csinodes storage.k8s.io v1
storageclasses storage.k8s.io v1
volumeattachments storage.k8s.io v1
volumeattachments/status storage.k8s.io v1
csidrivers storage.k8s.io v1beta1
csinodes storage.k8s.io v1beta1
storageclasses storage.k8s.io v1beta1
volumeattachments storage.k8s.io v1beta1
mutatingwebhookconfigurations admissionregistration.k8s.io v1
validatingwebhookconfigurations admissionregistration.k8s.io v1
mutatingwebhookconfigurations admissionregistration.k8s.io v1beta1
validatingwebhookconfigurations admissionregistration.k8s.io v1beta1
customresourcedefinitions apiextensions.k8s.io v1
customresourcedefinitions/status apiextensions.k8s.io v1
customresourcedefinitions apiextensions.k8s.io v1beta1
customresourcedefinitions/status apiextensions.k8s.io v1beta1
priorityclasses scheduling.k8s.io v1
priorityclasses scheduling.k8s.io v1beta1
leases coordination.k8s.io v1
leases coordination.k8s.io v1beta1
runtimeclasses node.k8s.io v1
runtimeclasses node.k8s.io v1beta1
endpointslices discovery.k8s.io v1beta1
flowschemas flowcontrol.apiserver.k8s.io v1beta1
flowschemas/status flowcontrol.apiserver.k8s.io v1beta1
prioritylevelconfigurations flowcontrol.apiserver.k8s.io v1beta1
prioritylevelconfigurations/status flowcontrol.apiserver.k8s.io v1beta1
apps catalog.cattle.io v1
apps/status catalog.cattle.io v1
operations catalog.cattle.io v1
operations/status catalog.cattle.io v1
clusterrepos catalog.cattle.io v1
clusterrepos/status catalog.cattle.io v1
ipamconfigs crd.projectcalico.org v1
ipamhandles crd.projectcalico.org v1
ippools crd.projectcalico.org v1
bgppeers crd.projectcalico.org v1
felixconfigurations crd.projectcalico.org v1
globalnetworksets crd.projectcalico.org v1
ipamblocks crd.projectcalico.org v1
networkpolicies crd.projectcalico.org v1
blockaffinities crd.projectcalico.org v1
clusterinformations crd.projectcalico.org v1
globalnetworkpolicies crd.projectcalico.org v1
hostendpoints crd.projectcalico.org v1
bgpconfigurations crd.projectcalico.org v1
networksets crd.projectcalico.org v1
kubecontrollersconfigurations crd.projectcalico.org v1
prometheusrules monitoring.coreos.com v1
probes monitoring.coreos.com v1
alertmanagers monitoring.coreos.com v1
servicemonitors monitoring.coreos.com v1
prometheuses monitoring.coreos.com v1
podmonitors monitoring.coreos.com v1
thanosrulers monitoring.coreos.com v1
alertmanagerconfigs monitoring.coreos.com v1alpha1
features management.cattle.io v3
clusters management.cattle.io v3
settings management.cattle.io v3
preferences management.cattle.io v3
```