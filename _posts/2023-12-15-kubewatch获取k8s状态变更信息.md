---
layout: post

title: kubewatch 获取k8s状态变更信息

categories: [云]




---



### kubewatch 获取k8s状态变更信息


kubewatch 是一个 Kubernetes 观察程序，用于将通知发布到可用的协作中心/通知通道。在 k8s 集群中运行它，您将通过 webhook 获得事件通知。

我也是在使用robusta的时候发现了它，他在Robusta-Forwarder中，kubewatch推送K8s的状态变更信息到Robusta-Forwarder中来触发triggers

github源码地址：[GitHub - robusta-dev/kubewatch: Watch k8s events and trigger Handlers](https://github.com/robusta-dev/kubewatch)

kubewatch安装，步骤如下:

1. 克隆代码 git clone https://github.com/bitnami-labs/kubewatch.git

2. 代码编译 go build 生成kubewatch可执行文件

   ![image-20231208154624872](https://yimacro.github.io/pics/post/image-20231208154624872.png)

3. 验证是否正常执行 

   ![image-20231208154807601](https://yimacro.github.io/pics/post/image-20231208154807601.png)

4. 执行如下命令启动，这里推送到webhook地址

   ```
   ./kubewatch config add webhook -u http://127.0.0.1:8080/webhook
     251  ./kubewatch resource add --po --svc
     252  ./kubewatch 
   ```

5. 在home目录下生成配置文件，也可以直接修改配置文件

   ![image-20231208155102859](https://yimacro.github.io/pics/post/image-20231208155102859.png)

6. 启动完成，编写处理程序验证（这里配置后pod状态有变更会发送信息到webhook，这时可以根据发送信息定位到代码）

   ![image-20231208155249503](https://yimacro.github.io/pics/post/image-20231208155249503.png)

demo编写与运行（其中WebhookMessage、EventMeta struct来之kubewatch的源码）:

  ```
  package main
  
  import (
  	"encoding/json"
  	"fmt"
  	"io/ioutil"
  	"net/http"
  	"time"
  )
  
  // WebhookMessage for messages
  type WebhookMessage struct {
  	EventMeta EventMeta `json:"eventmeta"`
  	Text      string    `json:"text"`
  	Time      time.Time `json:"time"`
  }
  
  // EventMeta containes the meta data about the event occurred
  type EventMeta struct {
  	Kind      string `json:"kind"`
  	Name      string `json:"name"`
  	Namespace string `json:"namespace"`
  	Reason    string `json:"reason"`
  }
  
  func handler(w http.ResponseWriter, r *http.Request) {
  	// 读取请求主体
  	body, err := ioutil.ReadAll(r.Body)
  	if err != nil {
  		http.Error(w, "Error reading request body", http.StatusInternalServerError)
  		return
  	}
  
  	// 解码JSON数据到WebhookMessage结构体
  	var webhookMessage WebhookMessage
  	err = json.Unmarshal(body, &webhookMessage)
  	if err != nil {
  		http.Error(w, "Error decoding JSON", http.StatusBadRequest)
  		return
  	}
  
  	// 处理接收到的数据
  	fmt.Printf("Received Webhook Message: %+v\n", webhookMessage)
  
  	// 发送响应
  	w.WriteHeader(http.StatusOK)
  	w.Write([]byte("Success"))
  }
  
  func main() {
  	// 设置路由
  	http.HandleFunc("/webhook", handler)
  
  	// 启动HTTP服务器
  	port := 8080
  	fmt.Printf("Server listening on :%d\n", port)
  	err := http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
  	if err != nil {
  		fmt.Printf("Error starting server: %s\n", err)
  	}
  }
  
  ```

  1. 上传到服务器，执行代码

     ![image-20231208162723151](https://yimacro.github.io/pics/post/image-20231208162723151.png)

  2. 触发pod变更，验证是否能收到消息

     ```
     kubectl apply -f https://gist.githubusercontent.com/robusta-lab/283609047306dc1f05cf59806ade30b6/raw
     ```

  3. 执行指令后，验证能收到信息

     kubewatch能成功发送信息

     ![image-20231208162942773](https://yimacro.github.io/pics/post/image-20231208162942773.png)

     demo能正常接收打印信息

     ![image-20231208162843525](https://yimacro.github.io/pics/post/image-20231208162843525.png)

  ps: 

  1. demo执行时遇到奇怪的问题，demo程序收不到不打印任何信息，直到http://127.0.0.1:8080把webhook重新修改地址才接收到打印，后面再改回去http://127.0.0.1:8080/webhook又能正常返回了。

 from: *[Yimacro](https://yimacro.github.io/)关注：[公众号](https://mp.weixin.qq.com/s?__biz=Mzg4Njc0NTY0OQ==&mid=2247483746&idx=1&sn=ee6d72ec1250512f7d38d5dff456eb80&chksm=cf95be3cf8e2372ad9914c0e2dcaf2040139d410e6f6bfea6b1bb457a1b3d53d4494aa7ad096#rd)*
