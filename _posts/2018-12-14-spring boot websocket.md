---
layout: post
title:  "spring boot websocket"
date:   2018-12-14
author: Dickie Yang
categories: Java
tags: springboot websocket
---


## 原生方式实现
### 后台服务代码
> WebSocketConfig.java  

```
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
	@Autowired
	private SocketHandler socketHandler;

    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
		registry.addHandler(socketHandler, "/ώsocket")
		.setAllowedOrigins("*")
		;
	}
}
```
> SocketHandler.java  

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Slf4j
@Component
public class SocketHandler extends TextWebSocketHandler {

    public static final ConcurrentMap<String,WebSocketSession> loginMap = new ConcurrentHashMap<>();

    public void sendToUser(String uid,String message){
		WebSocketSession session = loginMap.get(uid);
		try {
			session.sendMessage(new TextMessage(message));
		} catch (IOException e) {
			log.warn("WebSocket send message fail! from {},content: {},ex:{}",uid,message,e.getMessage());
		}
	}

	public void topic(String message){
    	loginMap.forEach((k,v) -> sendToUser(k,message));
	}

	@Override
	public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
		loginMap.forEach((k,v) -> {
            if (session.equals(v)) {
                loginMap.remove(k, v);
				log.info("{}'s webSocket connection had leave! code {}",session.getUri().getQuery(),status.getCode());
            }
        });
	}

	@Override
	public void afterConnectionEstablished(WebSocketSession session) throws Exception {
		String uid = session.getUri().getQuery();
		loginMap.put(uid,session);
		log.info("Received a new webSocket connection from {}",uid);
		session.sendMessage(new TextMessage("success login!"));
	}
}
```

### 简单测试js
```
<!DOCTYPE HTML>
<html>
<head>
    <title>My WebSocket</title>
</head>
<body>
Welcome<br/>
<input id="text" type="text" />
	<button onclick="send()">Send</button>  
  <button onclick="closeWebSocket()">Close</button>
<div id="message">
</div>
</body>

<script type="text/javascript">
    var websocket = new WebSocket("wss://xxx.com/api_test/ώsocket?usernaem");
    //连接发生错误的回调方法
    websocket.onerror = function(){
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    websocket.onopen = function(event){
        setMessageInnerHTML("open");
    }

    //接收到消息的回调方法
    websocket.onmessage = function(event){
        setMessageInnerHTML(event.data);
    }

    //连接关闭的回调方法
    websocket.onclose = function(){
        setMessageInnerHTML("close");
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭websocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        websocket.close();
    }

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML){
        document.getElementById('message').innerHTML += innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket(){
        websocket.close();
    }

    //发送消息
    function send(){
        var message = document.getElementById('text').value;
        websocket.send(message);
    }
</script>
</html>
```
## STOMP实现
### 服务端代码
> WebSocketConfig.java

```
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.scheduling.concurrent.DefaultManagedTaskScheduler;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;


@Configuration
@EnableWebSocketMessageBroker
public class StompWebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        //表示客户端订阅地址的前缀信息，也就是客户端接收服务端消息的地址的前缀信息
        registry.enableSimpleBroker("/topic")
                .setTaskScheduler(new DefaultManagedTaskScheduler())
                .setHeartbeatValue(new long[]{5000,5000});
//        registry.enableStompBrokerRelay().
//        //指服务端接收地址的前缀，意思就是说客户端给服务端发消息的地址的前缀
//        registry.setUserDestinationPrefix("/user");
//        registry.setApplicationDestinationPrefixes("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry stompEndpointRegistry) {
        //这个方法的作用是添加一个服务端点，来接收客户端的连接。
        stompEndpointRegistry.addEndpoint("/socket")
                .setAllowedOrigins("*");
//                .withSockJS();
    }
}
```
> SocketEventListener.java

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Component
@Slf4j
public class WebSocketEventListener {

    private final String HEADER_LOGIN = "login";

    public static final ConcurrentMap<String,String> loginMap = new ConcurrentHashMap<>();

    @EventListener
    public void handleWebSocketConnectListener(SessionConnectedEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String login = getHeader(accessor, HEADER_LOGIN);
        loginMap.put(login,accessor.getSessionId());
        log.info("Received a new webSocket connection from {}",login);
    }

    @EventListener
    public void handleWebSocketDisconnectListener(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        loginMap.forEach((k,v) -> {
            if (accessor.getSessionId().equals(v)) {
                loginMap.remove(k, v);
                log.info("A webSocket connection had leave! {}",k);
            }
        });
    }

    private String getHeader(StompHeaderAccessor accessor,String headName){
        GenericMessage simpConnectMessage = (GenericMessage) accessor.getMessageHeaders().get("simpConnectMessage");
        Map<String,List<String>> headers = (Map<String,List<String>>) simpConnectMessage.getHeaders().get("nativeHeaders");
        return headers.get(headName).get(0);
    }
}
```
### 前端测试代码
```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8" />
    <title>聊天页面</title>
    <script src="https://cdn.bootcss.com/sockjs-client/1.3.0/sockjs.min.js"></script>
    <script src="https://cdn.bootcss.com/stomp.js/2.3.3/stomp.js"></script>
    <script src="https://cdn.bootcss.com/jquery/3.3.1/jquery.min.js"></script>
</head>
<body>
<p>
    聊天室
</p>

<form id="JanetForm">
    <textarea rows="4" cols="60" name="text"></textarea>
    <input type="submit"/>
	<button id="stop">Stop</button>
</form>
<div id="output"></div>

<script th:inline="javascript">
    $('#JanetForm').submit(function (e) {
        e.preventDefault();
        var text = $('#JanetForm').find('textarea[name="text"]').val();
        sendSpittle(text);
    });
//    连接endpoint为"/socket"的节点
    var sock = new WebSocket("ws://192.168.31.219:82/socket");
	
    var stomp = Stomp.over(sock);
//    连接WebSocket服务端
    stomp.connect('userid','passCode',function (frame) {
		console.log(frame);
        stomp.subscribe("/topic/chat/userid",handleNotification);
		stomp.subscribe("/topic/looking",handleNotification)
    });
    function handleNotification(message) {
        $('#output').append("<b>收到了:" + message.body + "</b><br/>")
    }
    function sendSpittle(text) {
//        表示向后端路径/chat发送消息请求，这个是在控制器中@MessageMapping中定义的。
        stomp.send("/chat",{},text);
    }
    $('#stop').click(function () {
        {sock.close()}
    });
</script>

</body>
</html>
```
