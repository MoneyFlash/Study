## 引言
工作中已经有多次需要用到WebSocket，于是乎总结一下子使用过程以及方法

*参考链接：*
   > [阮一峰 WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)  
   
   > [静默虚空 WebSocket 详解教程](https://www.cnblogs.com/jingmoxukong/p/7755643.html)  
   
   > [菜鸟课程 HTML5 WebSocket](http://www.runoob.com/html/html5-websocket.html)  

~~我知道你们都不会去看链接的~~  
下面进入我们的代码环节

***
华丽的分割线-----
***
* 初始化
``` 
// 初始化WebSocket
initWebSocket() { 
  // websock地址
  let websockUrl = '你的websock的链接地址';
  // 新建websock链接
  this.websock = new WebSocket(websockUrl);
  // 接受websock数据
  this.websock.onmessage = this.websocketOnMessage;
  // 关闭websock链接
  this.websock.onclose = this.websocketClose;
  // 处理websock报错
  this.websock.onerror = this.websocketError;
}
```

* 数据接收
``` 
// 数据接收
websocketOnMessage(e) { 
  // 接收的数据需要JSON转译
  const res = JSON.parse(e.data);
  console.log(res);
}
```

* 关闭链接
``` 
// 关闭链接
websocketClose(e) { 
  console.log("connection closed (" + e.code + ")");
}
```

* 处理错误
``` 
// 处理错误
websocketError(e) { 
  
}
```
* 实际调用方法
``` 
// 实际调用方法
threadPoxi(data) { 
  //参数
  let params = {
    
  }
  // 转换JSON
  let agentData = JSON.stringify(params);
  // 根据websock当前状态进行不同操作
  // CONNECTING：值为0，表示正在连接
  // OPEN：值为1，表示连接成功，可以通信了
  // CLOSING：值为2，表示连接正在关闭
  // CLOSED：值为3，表示连接已经关闭，或者打开连接失败
  if(this.websock.readyState === this.websock.OPEN) { // 连接成功
    this.websocketSend(agentData);
  }
  else if(this.websock.readyState === this.websock.CONNECTING) { // 正在连接
    let that = this; //保存当前对象this
    setTimeout(function() {
      that.websocketSend(agentData);
    }, 300);
  }
  else { // 其他状态
    this.initWebSocket();
    setTimeout(() => {
      this.websocketSend(agentData);
    }, 500);
  }
}
```

* 发送数据
``` 
// 发送数据
websocketSend(data) { 
  // 可能需要转译才能传给服务端，否则服务端无法接收数据
  let agentData = encodeURIComponent(agentData);
  this.websock.send(agentData);
}
```
