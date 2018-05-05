# springboot集成websocket实时推送消息

> 最近的工作内容是把前段时间集成到app中的websocket替换成stomp，谁曾想，这个坑实在是太大了，差点没爬出来。在这里记录整个整合流程，顺便把websocket部分也整理记录在案，以备后面查阅。

#### 什么是websocket

简单来说，websocket是一个持久化的协议，可以实现请求的长连接，用来替换以前我们只能使用long pull才能实现的效果，而且它可以双向通信，通俗来讲就是服务端可以主动推送消息到客户端，当然客户端还是可以主动发送消息到服务端的:happy:。

WebSocket协议定义了两种消息类型,文本或字节,但是没定义它们的内容.它有意让客户端和服务端通过通用的子协议(例如,更高水平的协议)来定义消息语法.但是WebSocket协议的使用是可选的,客户端和服务端需要同意某些种类的协议来翻译这些消息 

这里不做太多的概念讲解，想要具体了解的可以参考[阮一峰老师的WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)、[知乎高票回答--WebSocket 是什么原理？为什么可以实现持久连接？](https://www.zhihu.com/question/20215561/answer/40316953)，知乎的高票回答讲解的还是比较生动形象的~~

#### 什么是sockjs

虽然主流浏览器已经很好的支持了websocket，但是某些厂商（是的，我就是在说IE）确实并没有支持，这时就需要sockjs了。

sockjs支持websocket、streaming、polling这三种方案，在系统不支持websocket协议的情况下，sockjs会选择另外的方案。所以我们可以在不用关注具体底层协议的情况下，使用统一api完成与服务端的通信。

#### stomp

STOMP即Simple (or Streaming) Text Orientated Messaging Protocol，简单(流)文本定向消息协议，它提供了一个可互操作的连接格式，允许STOMP客户端与任意STOMP消息代理（Broker）进行交互 。

客户端可以使用"SEND"或"SUBSCRIBE"命名来发送或订阅消息,需要一个目的地,用来描述这个消息是什么和谁来接收它.它会激活一个简单的发布订阅机制,可以用来通过代理向其他连接的客户端来发送消息或者向服务端发送消息来要求执行一些工作 

#### 前端示例

> 本文前端使用react技术栈，stomp和sockjs分别使用[@stomp/stompjs](https://www.npmjs.com/package/@stomp/stompjs)和[sockjs-client](https://www.npmjs.com/package/sockjs-client)

```js
var subscription = null;
var socket =  new SockJS(WS_URL)
var client = Stomp.over(socket);
client.connect({},function (iframe) {
    subscription = client.subscribe("/user/queue/monitor", function (message) {
        //do something
    };
    client.subscribe("/topic/subtest", function (message) {
        //do something
    });                                
}
subscription.unsubscribe();
```

#### 添加websocket支持

在pom.xml中添加如下依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-messaging</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-messaging</artifactId>
</dependency>
```

#### 启用消息代理

```java
/**
*AbstractSessionWebSocketMessageBrokerConfigurer实现了在handshake时获取httpsession，并且每次*websocket消息发生时也刷新了httpsession的时间。同时在websocket session中加入了SPRING.SESSION.ID字
*段。
**/
@Configuration
@EnableWebSocketMessageBroker // 开启使用STOMP协议来传输基于代理（message broker）的消息
public class WebsocketConfig extends AbstractSessionWebSocketMessageBrokerConfigurer<ExpiringSession> {
    
    	@Resource
	private ApplicationEventPublisher eventPublisher;
    
    @Bean
	public CustomWebSocktHandlerDecoratorFactory wsHandlerDecoratorFactory() {
		return new CustomWebSocktHandlerDecoratorFactory(eventPublisher);
	}
    
	@Override
	protected void configureStompEndpoints(StompEndpointRegistry stompEndpointRegistry) {
		stompEndpointRegistry.addEndpoint("/ws").
				addInterceptors(myHandshakeInterceptor()).//websocket拦截器
				setHandshakeHandler(myDefaultHandshakeHandler()).//websocket握手处理器
				setAllowedOrigins("*");//解决跨域问题
		stompEndpointRegistry.addEndpoint("/sockjs/ws").
				addInterceptors(myHandshakeInterceptor()).
				setHandshakeHandler(myDefaultHandshakeHandler()).
				setAllowedOrigins("*").
				withSockJS() //sockjs、stomp
				.setMessageCodec(new FastjsonSockJsMessageCodec())
		.setClientLibraryUrl("//cdn.jsdelivr.net/sockjs/1/sockjs.min.js");//解决前后端版本不一致的问题
	}
    
    	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
        //设置客户端接收消息地址的前缀（可不设置）
		registry.enableSimpleBroker("/topic","/queue");
        //设置客户端向服务器发送消息的地址前缀（可不设置）
		registry.setApplicationDestinationPrefixes("/ws");
	}
    
    @Override
	public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
		registration.setSendTimeLimit(5000);
        //添加WebSocket的装饰工厂，便于监听Session状态
		registration.addDecoratorFactory(wsHandlerDecoratorFactory());
	}
    
    	@Override
	public void configureClientInboundChannel(ChannelRegistration registration) {
		registration.taskExecutor()
			.corePoolSize(Runtime.getRuntime().availableProcessors()*4);
		registration.setInterceptors(new ChannelInterceptorAdapter() {
			@Override
			public Message<?> preSend(Message<?> message, MessageChannel channel) {
				System.out.println("recv:"+message);
				StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
				User user = ((WebSocketPrincipal)accessor.getSessionAttributes().get("user")).getUser();
				System.out.println(user.toString());
				return super.preSend(message, channel);
			}
		});
	}
    
    	@Override
	public void configureClientOutboundChannel(ChannelRegistration registration) {
		registration.taskExecutor().corePoolSize(Runtime.getRuntime().availableProcessors() *4);
		registration.setInterceptors(
				new ChannelInterceptorAdapter() {
					@Override
					public Message<?> preSend(Message<?> message, MessageChannel channel) {
						System.out.println("send:"+message);
						return super.preSend(message, channel);
					}
				}
		);
	}
    	/**
	 * WebSocket 握手拦截器
	 * 可做一些用户认证拦截处理
	 */
	private HandshakeInterceptor myHandshakeInterceptor(){
		return new HttpSessionHandshakeInterceptor() {
			/**
			 * websocket握手连接
			 * @return 返回是否同意握手
			 */
			@Override
			public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
				ServletServerHttpRequest req = (ServletServerHttpRequest) request;
				Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
				if(principal==null || "anonymousUser".equals(principal)){
					response.setStatusCode(HttpStatus.UNAUTHORIZED);
					logger.error("websocket权限拒绝");
					return false;
				}
				attributes.put("user",new WebSocketPrincipal((User) principal));
				return true;
			}

			@Override
			public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {

			}
		};
	}
    	//WebSocket 握手处理器
	private DefaultHandshakeHandler myDefaultHandshakeHandler(){
		return new DefaultHandshakeHandler(){
			@Override
			protected Principal determineUser(ServerHttpRequest request, WebSocketHandler wsHandler, Map<String, Object> attributes) {
				//设置认证通过的用户到当前会话中
				return (Principal)attributes.get("user");
			}
		};
	}
    
    	class WebSocketPrincipal implements Principal{

		private User user;

		public WebSocketPrincipal(User user) {
			this.user = user;
		}

		public WebSocketPrincipal() {
		}

		@Override
		public String getName() {
			return user.getUsername();
		}

		public User getUser() {
			return user;
		}

		public void setUser(User user) {
			this.user = user;
		}
	}
}
```

```java
public class CustomWebSocktHandlerDecoratorFactory implements WebSocketHandlerDecoratorFactory {

	private final ApplicationEventPublisher eventPublisher;

	public CustomWebSocktHandlerDecoratorFactory(ApplicationEventPublisher eventPublisher) {
		this.eventPublisher = eventPublisher;
	}

	@Override
	public WebSocketHandler decorate(WebSocketHandler webSocketHandler) {
		return new CustomWebSocketHandler(webSocketHandler,eventPublisher);
	}
}
```

```java
/**
 * @author xueao
 * @create 2018-04-19 23:00
 * @desc 自定义websocket事件监听器
 **/
public class CustomWebSocketHandler extends WebSocketHandlerDecorator {

	private final ApplicationEventPublisher eventPublisher;
    
	private static final Logger logger = LoggerFactory.getLogger(CustomWebSocketHandler.class);

	public CustomWebSocketHandler(WebSocketHandler delegate,ApplicationEventPublisher eventPublisher) {
		super(delegate);
		this.eventPublisher=eventPublisher;
	}

	@Override
	public void afterConnectionEstablished(WebSocketSession session) throws Exception {

		System.out.println("WebSocket连接已建立，websocketId:"+session.getId()+"   user:"+session.getPrincipal().getName());
        //SocketSessionRegistry为用户登录后保存websocket session,可自行实现
		SocketSessionRegistry.registerSessionId(session.getPrincipal().getName(),session);

		super.afterConnectionEstablished(session);
	}

	@Override
	public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) throws Exception {

		System.out.println("WebSocket连接已关闭，websocketId:{}"+session.getId()+"   user:"+session.getPrincipal().getName());

		SocketSessionRegistry.unregisterSessionId(session.getPrincipal().getName(),session);

		super.afterConnectionClosed(session, closeStatus);
	}

	private void publishEvent(ApplicationEvent event) {
		try {
			eventPublisher.publishEvent(event);
		}
		catch (Throwable ex) {
			//忽略异常
			logger.error( "Failed to publisher event.", ex );
		}
	}
}
```

#### 推送消息

* @MessageMapping ：接收客户端消息 
* @SendTo：发送广播消息 
* @SendToUser：反馈客户端发送消息结果 
* SimpMessagingTemplate：消息模板发送广播消息和给指定客户端发消息 

```java
@Controller
public class WebsocketController {

    	@Autowired
	private SimpMessagingTemplate template;
    
    @MessageMapping("/addNotice")
	@SendTo("/topic/subtest")//全局订阅示例
	public WsMessage notice(String name,Principal fromUser){
        //WsMessage 自定义返回数据，可自行实现
		WsMessage msg = new WsMessage();
		msg.setFromName(fromUser.getName());
		msg.setContent("测试全局订阅消息推送");

		return msg;
	}
    
    @MessageMapping("/msg")
	//@SendToUser("/queue/msg/result")//点对点消息示例
	public WsMessage sendMsg(WsMessage msg,Principal fromUser){
		msg.setFromName(fromUser.getName());
		//向指定客户端发送消息
		//template.convertAndSendToUser(fromUser.getName(),"/queue/msg/new","test send to user");
		return msg;
	}

}
```





参考文章：

[Spring Boot 的 WebSocket 开发说明文档 ](https://blog.csdn.net/lnktoking/article/details/78678406)

[spring websocket项目实践 ](https://blog.csdn.net/Dwade_mia/article/details/79054590)

[spring websocket中 STOMP ](https://my.oschina.net/u/1590027/blog/879629)

