---
layout: post
title:  "EventSource实现服务器推送"
date:   2018-01-18
author: Dickie Yang
tags: 
    - 推送 
    - es
---

### 利用Eventsource对象实现服务器推送
  - **处理过程**  
	客户端建立EventSource对象，对服务器通过http协议不断进行请求。服务器对客户端的响应数据格式有四部分构成，event，data，id，空格行。客户端接收到服务器端的响应数据之后，根据event事件值，找到EventSource对象对应的事件监听器。
	- 例如，event值为load，那么客户端收到响应数据之后，解析到event值为load。客户端为EventSource对象添加该事件的监听器，`EventSource.onLoad = function(){ //处理服务器端的响应数据 }或者EventSource.addEventListener("load",function(){ //处理服务器端的响应数据 })`。
	- EventSource有三个默认的监听器，分别监听open，message，error事件。客户端和服务器端进行连接时，将会触发open事件，执行EventSource.onOpen = function(){}或者 EventSource.addEventListener("open",function(){ })。对于message事件，当服务器端响应的数据没有指定事件类型时，将会默认触发客户端的message事件。
	- 服务器端响应的报文数据中，id 表示事件event的id，用户可以自定义。并且响应的类型为 text/event-stream 类型。

	EventSource推送的前端代码实例：

		<script type="text/javascript">

    	if(window.EventSource){

        var eventSource = new EventSource("http://localhost:9090/sse");

        //只要和服务器连接，就会触发open事件
        eventSource.addEventListener("open",function(){
           console.log("和服务器建立连接");
        });

        //处理服务器响应报文中的load事件
        eventSource.addEventListener("load",function(e){
            console.log("服务器发送给客户端的数据为:" + e.data);
        });

        //如果服务器响应报文中没有指明事件，默认触发message事件
        eventSource.addEventListener("message",function(e){
            console.log("服务器发送给客户端的数据为:" + e.data);
        });

        //发生错误，则会触发error事件
        eventSource.addEventListener("error",function(e){
            console.log("服务器发送给客户端的数据为:" + e.data);
        });

    	}
    	else{
        	console.log("服务器不支持EvenSource对象");
    	}
		</script>

	EventSource推送的后端实现代码：

		protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        	//媒体类型为 text/event-stream
        	response.setContentType("text/event-stream");
        	response.setCharacterEncoding("utf-8");
        	PrintWriter out = response.getWriter();

        	//响应报文格式为:
        	//data:Hello World
        	//event:load
        	//id:140312
			//retry:3 重试时间(秒)
        	//换行符(/r/n)

        	out.println("data:Hello World");
        	out.println("event:load");
        	out.println("id:140312");
        	out.println();
        	out.flush();
        	out.close();
		}


	>服务器端的响应需要包含 Access-Control-Allow-Origin 头，用来声明允许从哪些域访问该 URL。“*”表示允许来自任何域的访问，不推荐使用该值。一般使用与当前应用相同的域，限制只允许来自当前域的访问。

	- **小结**  
	如果需要从服务器端推送数据给浏览器，可以使用的基于 HTML 5 规范标准的技术包括 WebSocket 和服务器推送事件。开发人员可以根据应用的具体需求来选择合适的技术。如果只是需要从服务器端推送数据，服务器推送事件的规范更加简单，实现起来更容易。


