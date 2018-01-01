### 框架应用  

- servlet  

生命周期:加载->实例化->服务->销毁  
servlet(非线程安全)是单例多线程的.主要方法有init(),service(),destroy();  
init():仅会执行一次的方法,在服务器装载某个servlet时,用来初始化servlet的各部分数据;  
service():接收响应请求,一般不会直接使用该方法,多数是通过继承HttpServlet后重写doGet()与doPost()方法(或者其他HTTP方法),重写service()方法最终也是方法给各种HTTP方法,这点查阅HttpServlet类会更清楚;    
destroy():也是仅执行一次的方法,在servlet生命结束时调用,负责释放相关资源.  

备注:servlet是SpringMVC的本质,由一个dispatch servlet来统一分发请求(内部有个映射器),个人初学servlet时觉得原生也挺不错呀,后来意识到当需要编写的
接口非常多时,我们就需要为每个接口重新建一个类,这就导致类规模剧增,复杂度也上升了,而使用框架就很好的解决了这点,扩展性也提高了.实质上,在进行微信开发的时候(当时还未接触框架),就想到且建立一个统一的负责转发请求的总servlet,这种思想
很SpringMVC的思想差不多.

- 转发(forward)与重定向(redirect)  

forward:request.getRequestDispatcher("xxx").forward(request,response);这是一种服务端内部的行为,因此仅触发一次请求与响应,客户端的URL地址栏不会产生变化;

redirect:response.sendRedirect(“xxx”);通知客户端发出新请求去访问响应的页面,因此这是触发二次请求,URL地址栏会相应变化.  

SpringMVC转发与重定向的方法参考Spring实战.  

- bean的加载过程
- Spring IOC & AOP
- 注入方式
- 事务管理 声明式事务 事务传播
- SpringMVC原理
- SpringMVC常用注解
- 转发与重定向
- ORM框架
- 缓存
- 批量提交
- SpringBoot
- RESTFul