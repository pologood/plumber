#plumber

**plumber** 是一款实现页面 pagelet 并发执行的 java web 框架，支持以 [**BigPipe**](http://www.cubrid.org/blog/dev-platform/faster-web-page-loading-with-facebook-bigpipe/) 的方式返回你的 pagelet 执行结果，以一切可能并发的方式提高你的页面响应速度。



##Core Features

* 支持将页面划分成多个可以并发执行的 pagelet , 每个 pagelet 有着自身独立的 mvc , 同时不同页面也可以复用这些 pagelet 。

* pagelet 提供同步 ( **barrier** ) 和异步 ( **pipe** ) 两种返回方式， **barrier** 方式实现的所有 pagelet 将作为页面的首次内容返回， **pipe** 方式实现的 pagelet 将在自身执行完成后以 [**BigPipe**](http://www.cubrid.org/blog/dev-platform/faster-web-page-loading-with-facebook-bigpipe/) 的方式继续返回给客户端。




##WorkFlow
![image](http://img.hb.aicdn.com/43b62f21e78717f8fb35bf4c47dfbb91a998661ad188-A01agD_fw658)

为设置、解析和控制 **barrier** **pipe** 等不同类型 pagelet， 保证它们的执行和返回顺序， **plumber** 有着自己的 **controller** ，在与 struts/spring mvc 等 web 框架一同运用的时候，strtus 等 MVC 框架的C将只承担请求参数转发的作用，真正的 control 逻辑需要用户在 **plumber** 的 **controller** 层来完成。

当然用户并不需要关心 pagelet 具体的执行和返回顺序，这些都将由 **plumber** 来控制和保证，用户只需要在 **controller** 层通过框架提供的 **pb-barrier** 和 **pb-pipe** 这两个页面标签，来设置哪些 pagelet 为 **barrier** 类型，它们作为第一次 response 返回，哪些 pagelet 为 **pipe** 类型以 BigPipe 的方式后续继续返回。

一个 request 经由 struts/spring mvc 等 web 框架转发进入 **plumber** 的运行环境后，执行代码即为需要用户来实现的一个 **controller** 加多个用户自主划分的 **barrier** 或 **pipe** 。




##Get Started

###1) 添加 pom 依赖

	<dependency>
	  <groupId>com.dianping</groupId>
	  <artifactId>plumber</artifactId>
	  <version>{latest-version}</version>
	</dependency>
	
###2) 添加 plumber.yaml 配置
在 **resources** 下添加 plumber.yaml 配置文件 ：

	response:
    	contentType: text/html;charset=UTF-8
    	timeout: 3000
	view:
	    encoding: UTF-8
	    viewSourceLoaderFactory: com.dianping.plumber.view.support.loader.ViewSourceUnderClassPathLoaderFactory
	    viewRendererFactory: com.dianping.plumber.view.support.renderer.freemarker.FreemarkerRendererFactory
	concurrent:
	    timeout: 1000
	    threadPool:
	        corePoolSize: 50
	        maximumPoolSize: 50
	        keepAliveTime: 0
	        blockingQueueCapacity: 1000
	        


###3）添加 spring 配置

	<bean id="plumber" class="com.dianping.plumber.core.Plumber">
    </bean>

###4）让我们来完成一个demo
demo 页面基于 struts+spring+plumber 来开发，模板引擎使用 **plumber** 默认提供的 freemarker，模板资源放在 resources 下。

demo 页面包含 headBarrier rightBarrier 和 mainPipe 3个 pagelet, 其中 headBarrier 和 rightBarrier 将作为首次内容输出，mainPipe 以 BigPipe 的方式后续输出。

![image](http://img.hb.aicdn.com/edd20546044223879ad06a1fb18a017aa1d9266a16f17-lFxzti_fw658)

mainPipe 类似于微博中 feed 这样比较耗时的模块，为了缓解用户等待页面加载的焦虑感，我们可以先返回一个执行起来不太耗时的框子页面（包含页头和右边栏），让用户看到部分输出，而不是对着一个空白页，且返回该框子页面后，客户端浏览器已经可以开始加载相关静态资源了。mainPipe 在处理完成后以 BigPipe 的方式作为第二次 response 返回。



#####struts
	public class StrutsDemo extends ActionSupport {

    	private Plumber plumber;

        @Override
        public String execute() throws Exception {


            String plumberControllerName = "demoController";

            Map<String, Object> paramsForController = new HashMap<String, Object>();
            paramsForController.put("demoDesc", "StrutsDemo");

            HttpServletResponse response = ServletActionContext.getResponse();
            HttpServletRequest request = ServletActionContext.getRequest();
            
            ResultType resultType = plumber.execute(plumberControllerName, paramsForController, request, response);
            
        	if ( resultType==ResultType.SUCCESS ) {
            	return null;
        	} else {
            	return "error";
        	}
        }

        public void setPlumber(Plumber plumber) {
            this.plumber = plumber;
        }
    }
    

    <package name="struts" namespace="/struts" extends="struts-default">

        <action name="demo" class="com.dianping.struts.StrutsDemo">
        	<result name="error">/error.jsp</result>
        </action>

    </package>
    
    
    
    

#####PlumberController

**业务逻辑 DemoController** :

	public class DemoController extends PlumberController {

	    @Override
	    public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsForPagelets, Map<String, Object> modelForView) {

	        paramsForPagelets.put("param", "test paramFromController");

	        modelForView.put("title", "plumber-tutorial of " + paramsFromRequest.get("demoDesc"));

	        return ResultType.SUCCESS;
	    }

	}

 使用 **plumber** 时，struts 只是作为用户请求的转发器，controller 都是由实现了 PlumberController 的实现类真正接管。
 
 *paramsFromRequest* 来自于用户请求的参数，由struts、spring mvc 等 web 框架转发
 
 *paramsForPagelets* 提供给子级 pagelet 共用的参数
 
 *modelForView* 填充 demo.ftl 的 model
 
 
 
**controller Spring 配置** :
 
 	<bean id="demoController" class="com.dianping.plumber.DemoController">
        <property name="viewPath" value="/view/demo.ftl" />
    </bean>
    
无论是 **controller** 还是 **barrier** 或 **pipe** , 在 **plumber** 中都需要被配置成一个 spring 的 bean ，因为 **plumber** 是直接基于 spring 来做对象管理的，并且 **plumber** 会自行将所有的 **controller** **barrier** 以及 **pipe** 的 scope 设置成 prototype。


 
**容器页面 demo.ftl** :

	<html>
        <head>
            <title>${title}</title>
            <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
            <link rel="stylesheet" type="text/css" href="/static/css/demo.css">
        </head>

    <body>

        <div id="head" pb-barrier="headBarrier">
            ${headBarrier}
        </div>

        <div id="content">
            <div id="main" pb-pipe="mainPipe">
                main
            </div>

            <div id="right" pb-barrier="rightBarrier">
                ${rightBarrier}
            </div>

            <div class="clear-both"></div>
        </div>

        <script src="/static/js/lib/jquery-min.js"></script>
        <script src="/static/js/demo.js"></script>

    <#--</body>-->    
    <#--</html>-->
    
demo.ftl 将作为所有 **barrier** 类型 pagelet 的容器页面，待所有的 **barrier** 执行完成并将其渲染结果填充到该页面后，作为第一次 response 返回。
   
**plumber** 提供了 **pb-barrier** 和 **pb-pipe** 两种页面属性，分别对应 barrier 和 pipe 两种返回方式， **plumber** 会自动解析该模板页面，识别出其中这两种类型的 pagelet 。

例如这里的 pb-barrier="headBarrier" ，**plumber** 将识别出 demo.ftl 中有一个 **barrier** 类型的 pagelet，然后从 **spring** 容器中需找一个 id 为 headBarrier 的 pagelet ，它的执行渲染结果将以一个名为 headBarrier 的变量填充到 demo.ftl 中。

headBarrier 和 rightBarrier 将以并发的方式得到执行，待他们都执行完成，渲染结果填充到 demo.ftl 中后，demo.ftl 将作为第一次 response 发送到客户端。

而名为 mainPipe 这个 **pipe** 类型的 pagelet 将继续执行，待它执行完成后，将以 [**chunked**](http://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81) 的方式继续往客户端发送。如果你有多个 **pipe** 类型的 pagelet 的话，他们将无干扰的在各自执行完成后自行返回给客户端。

也许你已经注意到在 demo.ftl 中，我们未曾将 < body > 和 < html > 标签闭合，这是理所当然的， demo.ftl 只是第一次返回给客户端的内容，此时页面 dom 并未完整， **pipe** 的执行结果作为页面的其余部分将继续以 **chunked** 的方式往客户端发送。如果将两个标签闭合后，浏览器将不再接受服务端的 response ，那样即使我们的 **pipe** 类型的 response 发送到客户端，浏览器也不会接受并加以解析。

所以如果你的页面中用到了 **pipe** 类型的 pagelet ，请不要为你的容器页面添加上述闭合标签， **plumber** 会在该页面的所有 **pipe** 类型的 pagelet 成功返回后自行补上该闭合标签。

而如果你的页面只包含 **barrier** 类型的 pagelet ，那还是在你的容器页面中，直接将标签闭合吧。




#####Barrier

	public class HeadBarrier extends PlumberPagelet {

        private Logger logger = Logger.getLogger(RightBarrier.class);

        @Override
        public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsFromController, Map<String, Object> modelForView) {

            String demoDesc = (String) paramsFromRequest.get("demoDesc");

            String param = (String) paramsFromController.get("param");

            modelForView.put("msg", "Get HeadBarrier Content! "+ param + " " + demoDesc);

            try {
                Thread.sleep(40);
            } catch (InterruptedException e) {
                logger.error(e);
            }

            return ResultType.SUCCESS;
        }

    }
    

    <bean id="headBarrier" class="com.dianping.plumber.HeadBarrier">
        <property name="viewPath" value="/view/headBarrier.ftl" />
    </bean>
    

    headBarrier.ftl:
    ${msg}
    
    

#####Pipe
	public class MainPipe extends PlumberPagelet {

        private Logger logger = Logger.getLogger(MainPipe.class);

        @Override
        public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsFromController, Map<String, Object> modelForView) {

            try {
                Thread.sleep(4000);
                SimpleDateFormat time=new SimpleDateFormat("HH:mm:ss");
                modelForView.put("msg", "Get MainPipe Content! " + time.format(new Date()));
            } catch (InterruptedException e) {
                logger.error(e);
            }

            return ResultType.SUCCESS;
        }
    }


    <bean id="mainPipe" class="com.dianping.plumber.MainPipe">
        <property name="viewPath" value="/view/mainPipe.ftl" />
    </bean>
    

    mainPipe.ftl:
    <script>
        $("#main").html("${msg}");
    </script>

当页面存在多个 **pipe** 类型的 pagelet 时，因我们无法保证它们的先后执行顺序，所以一般的处理方式是，在容器页面中设置一个 placeholder 的 dom 节点，如 demo.ftl 中的 ：

	<div id="main" pb-pipe="mainPipe">
        main
    </div>
而将 **pipe** 的执行结果包在一段 javascript 中， 通过 javascript append 到 placeholder 的 dom 节点中，从而保证 **pipe** 类型 pagelet 的执行结果能放置到你预期的页面位置，当然，这可能对你的页面 SEO 有一定的影响。


如果你跟着我们一步一步做，至此你应该已经可以将这个 demo 部署到任意 web 容器，run 起来看一下执行结果了。

同时，你也可以打开浏览器控制台，看一下这个 request 请求，是不是和我一样，response header 中包含 Transfer-Encoding:chunked。 事实上 **plumber** 中 BigPipe 的实现正是基于 http [**chunked**](http://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81) 来达成的。

上述 demo 的所有代码，你均可以在 [plumber-tutorial](https://github.com/zhongliangjun1/plumber-tutorial) 中找到。



##Advanced

###1) plumber.yaml 配置详解

	configOverriderFactory: com.dianping.plumber.DemoConfigOverriderFactory
	env: dev
	response:
    	contentType: text/html;charset=UTF-8
    	timeout: 3000
	view:
	    encoding: UTF-8
	    viewSourceLoaderFactory: com.dianping.plumber.view.support.loader.ViewSourceUnderWebContextLoaderFactory
	    viewRendererFactory: com.dianping.plumber.view.support.renderer.freemarker.FreemarkerRendererFactory
	concurrent:
	    timeout: 1000
	    threadPool:
	        corePoolSize: 50
	        maximumPoolSize: 50
	        keepAliveTime: 0
	        blockingQueueCapacity: 1000
	        
配置介绍：

* **configOverriderFactory** 用于实现配置外部化。你可以提供一个 PlumberConfigOverriderFactory 的实现，在该 factory 生产的 PlumberConfigOverrider 中覆盖 plumber.yaml 中的配置。

* **env** 配置为 dev 环境时（ 默认为 product ），**plumber** 会直接将抛出异常的 pagelet 的错误堆栈信息作为模块内容输出到页面上，便于查看出错信息，配置为 product 时，**plumber** 将丢弃该 pagelet ，仅输出页面其余 pagelet 内容。你可以通过 **configOverriderFactory** 确保生产环境对该配置的覆盖，避免 dev/product 频繁切换造成疏漏。

* **response.contentType** 设置 response 的 Content-Type。

* **response.timeout** 设置整个页面的响应超时时间，超过这个时间的 **pipe** 类型的 pagelet 将被丢弃。

* **view.encoding** 页面模板文件的编码方式。

* **view.viewSourceLoaderFactory** 页面模板文件 loader 的工厂类。 **plumber** 提供了从 classpath 和 WEB-INF 下加载页面模板文件的两种默认实现： ViewSourceUnderClassPathLoaderFactory 和 ViewSourceUnderWebContextLoaderFactory。需要注意的是，使用 ViewSourceUnderWebContextLoaderFactory 时需要在 web.xml 中将 ViewSourceUnderWebContextLoader 添加为 listener ，且须在 spring 的 ContextLoaderListener 之前。

* **view.viewRendererFactory** 页面模板引擎的工厂类。 **plumber** 默认提供了对 freemarker 引擎的支持：FreemarkerRendererFactory ，当然你也可以使用任意其他模板引擎，提供一个 ViewRendererFactory 的实现即可。

* **concurrent** plumber 线程池的相关配置，你可以根据自己的业务特性做相应调整。


###2) 通过注解进行参数注入： @ParamFromRequest @ParamFromController

在 **plumber** 的 **controller** 中你可以获取从 struts/spring mvc 等 web 框架转发来的请求参数( **Map<String, Object> paramsFromRequest** )，同时也可以在 **Map<String, Object> paramsForPagelets** 中为下一级的 **barrier** 或 **pipe** prepare 一些共用参数：

	public abstract class PlumberController {

        ...

        /**
         * entrance of request. Controller should prepare common params for its pagelets ,
         * then execute its business logic to fill the modelForView.
         * View of this Controller will be the first time response send to client,
         * you can also set pagelets to be barrier ,then it will be sent with this view.
         * @param paramsFromRequest
         * @param paramsForPagelets
         * @param modelForView
         * @return
         */
        public abstract ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsForPagelets, Map<String, Object> modelForView);

        ...
    }
而在 **plumber** 的 **barrier** 或 **pipe** 中，你则既可以从 **Map<String, Object> paramsFromRequest** 中获取转发来的请求参数，又可以从 **Map<String, Object> paramsFromController** 中获取由 **controller** prepare 的共用参数：

	public interface PlumberPagelet {

        public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsFromController, Map<String, Object> modelForView);

    }

但是反复从 Map 中获取参数，并且做类型强转是一件相对麻烦的事情，**plumber** 提供了 **@ParamFromRequest** 和 **@ParamFromController** 这样两个注解，为你解决这个问题。

例如 PlumberBarrier 原来是这样写：

	public class HeadBarrier extends PlumberPagelet {

        private Logger logger = Logger.getLogger(RightBarrier.class);

        @Override
        public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsFromController, Map<String, Object> modelForView) {

            String demoDesc = (String) paramsFromRequest.get("demoDesc");

            String param = (String) paramsFromController.get("param");

            modelForView.put("msg", "Get HeadBarrier Content! "+ param + " " + demoDesc);

            try {
                Thread.sleep(40);
            } catch (InterruptedException e) {
                logger.error(e);
            }

            return ResultType.SUCCESS;
        }

    }

现在可以这样写：

	public class HeadBarrier extends PlumberPagelet {

        private Logger logger = Logger.getLogger(RightBarrier.class);

        @ParamFromController
        private String param;

        @ParamFromRequest
        private String demoDesc;

        @Override
        public ResultType execute(Map<String, Object> paramsFromRequest, Map<String, Object> paramsFromController, Map<String, Object> modelForView) {

            modelForView.put("msg", "Get HeadBarrier Content! "+ param + " " + demoDesc);
            try {
                Thread.sleep(40);
            } catch (InterruptedException e) {
                logger.error(e);
            }

            return ResultType.SUCCESS;
        }

    }
    
打过注解的成员变量，**plumber** 将以你的参数名为 key , 自动到相应的 Map 中去获取对应的 value ，帮你做类型转换，然后注入进来。 

**@ParamFromRequest** 可以在 **controller** **barrier** 和 **pipe** 中使用， 而 **@ParamFromController** 则只可以在 **barrier** 和 **pipe** 中使用。

###3) 移动 H5 页面 SEO 优化

在前面我们已经指出 **pipe** 类型的 pagelet 作为后续输出返回到 client 时， 我们需要把它插入到页面我们想要的位置，所以会将 **pipe** 的 dom 包在一段 javascript 中，通过 javascript append 到预期的位置，当然这种方式或多或少会对我们的页面 seo 产生一些影响。

不过对于移动 H5 页面，它的 dom 结构天然的具有“顺序性”，而且尤其适合于 **big-pipe**, 因为我们确实可以优先只显示首屏的内容，首屏往下的其余部分，我们都可以通过 **big-pipe** 后续返回。

而如果对后续返回的这些 **pagelet**，我们可以施加一定的顺序控制，如下面的 mobileFirstPipe mobileSecondPipe mobileThirdPipe 和 mobileFourthPipe （ mobileSecondPipe 会等 mobileFirstPipe 返回后再返回，而不会等待 mobileThirdPipe 或 mobileFourthPipe；同样 mobileThirdPipe 会等 mobileFirstPipe 和 mobileSecondPipe 返回后再返回，而不会等 mobileFourthPipe ），这样我们便不再需要通过 javascript 来控制这些 **pagelet** 的显示位置了，他们会按照你设定的优先级顺序，依次拼接在首次返回的 dom 后面，最终构成完整的 dom tree。

	<html>

    ...

    <div class="header" pb-barrier="mobileHeadBarrier">
        ${mobileHeadBarrier}
    </div>

    <div class="bs-callout bs-callout-orange item" pb-barrier="mobileMainBarrier">
        ${mobileMainBarrier}
    </div>

    <script src="/static/common/js/jquery.min.js"></script>
    <script src="/static/common/js/bootstrap.min.js"></script>

    <div class="hide" pb-pipe="mobileFirstPipe"></div>
    <div class="hide" pb-pipe="mobileSecondPipe"></div>
    <div class="hide" pb-pipe="mobileThirdPipe"></div>
    <div class="hide" pb-pipe="mobileFourthPipe"></div>

    <#--</body>-->
	<#--</html>-->

现在在 **plumber** 中要实现这样的效果非常简单，你只需要在指定 **pb-pipe** name 的同时，像下面这样指定它的优先级即可：

	<div class="hide" pb-pipe="mobileFirstPipe@4000"></div>
    <div class="hide" pb-pipe="mobileSecondPipe@3000"></div>
    <div class="hide" pb-pipe="mobileThirdPipe@2000"></div>
    <div class="hide" pb-pipe="mobileFourthPipe@1000"></div>
    
**@** 后面指定的优先级就跟 css 中的 z-index 类似，权值越大，优先级越高。指定 **pagelet** 的优先级之后，**plumber** 便会按照你指定的顺序来返回这些 **pagelet** 。

需要注意的是，采取指定优先级的方式后，你需要为该页面所有的 **pipe** 类型的 **pagelet** 均指定优先级，不可以有些指定有些不指定。

###4）通过 spring xml 配置 pagelet 信息

除了可以在容器页面中通过 **pb-barrier** 和 **pb-pipe** 这两个页面属性来配置 pagelet 信息外，你也可以在 controller 的 spring xml 中进行配置。如下：

	<bean id="springConfigController" class="com.dianping.plumber.mobile.MobileController">
    	<property name="viewPath" value="/WEB-INF/view/test/demo.ftl" />
    	<property name="barrierNames">
        	<list value-type="java.lang.String">
            	<value>mobileHeadBarrier</value>
            	<value>mobileMainBarrier</value>
        	</list>
    	</property>
    	<property name="pipeNames">
        	<list value-type="java.lang.String">
            	<value>mobileFirstPipe@4000</value>
            	<value>mobileSecondPipe@3000</value>
            	<value>mobileThirdPipe@2000</value>
            	<value>mobileFourthPipe@1000</value>
        	</list>
    	</property>
	</bean>
	
如果你在容器页面和 spring xml 中都进行了 pagelet 的相关配置, **plumber** 会合并这两边的配置信息，它们是不冲突的。

###5）pipe 类型的 pagelet 中不可修改 response header

因为服务端内容在第一次 flush 到客户端后，resposne header 便不可再被修改（客户端已经读取 header 内容了），所以在 **pipe** 类型的 pagelet 中不可再对 response header 进行相关修改操作，如添加、修改 cookie 等。

###6）plumber.js 让你的 js 执行不用再久久等待 dom ready

将你的静态资源引用作为第一次 **chunked** 输出，可以让你的静态资源提前得到加载，但如果你的 js 需要等待 dom ready 事件，那么你将面临你的 js 在所有 **pipe** 类型的 pagelet 抵达前均无法得到执行。如此所有你提前发送到客户端的 pagelet 将都陷入能看，却没有绑定 js 事件的境地。

事实上属于每个模块的 js 并不一定需要 dom ready 时才可以执行，现在 [**plumber.js**](https://github.com/zhongliangjun1/plumber.js) 可以让你的 js 只要等待所需操作的 dom 节点抵达客户端并被创建并入 dom tree 即可。

	<html>

    <head>
        <title>${title}</title>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <link rel="stylesheet" href="/static/common/css/bootstrap.min.css">
        <link rel="stylesheet" href="/static/mobile/css/demo.css">
        <script type="text/javascript">
            ${plumberJS}
        </script>
    </head>


    <body>

    <div class="bs-callout bs-callout-orange item" pb-barrier="jsFirstPagelet">
        ${jsFirstPagelet}
    </div>

    <div class="bs-callout bs-callout-grey item" pb-barrier="jsSecondPagelet">
        ${jsSecondPagelet}
    </div>

    <script src="/static/common/js/jquery.min.js"></script>
    <script src="/static/common/js/bootstrap.min.js"></script>
    <script src="/static/js/demo.js"></script>

    <div class="hide" pb-pipe="jsThirdPagelet@1"></div>
    <div class="hide" pb-pipe="jsFourthPagelet@2"></div>

    <#--</body>-->
	<#--</html>-->
	
如上，现在你只需在页面的初始，运行 **plumber.js** (框架会自动为你的 **controller** 的 **modelForView** 填充一个名为 **plumberJS** 的变量，它的值即为 **plumber.js** 的具体实现代码)，这个超级轻量的 js lib 为你提供了一个仅仅包含 **ready:function(name)** 和 **execute:function(name,fn)** 这样两个方法的全局 **plumber** 对象。

你的 js 代码现在只需要这样写:

	plumber.execute("jsThirdPagelet" , function(){
    	console.log($("#jsThirdPagelet").text());
	});
	
在接收到 **jsThirdPagelet** 这个事件（即 **jsThirdPagelet** 这个模块的 dom 抵达客户端并被创建并入 dom tree）之后，callback 函数便会得到执行，它再也不用等待整个页面的 dom ready 事件了。

而在每个页面模块的后面，你只需要加入类似如下这段 js :

	<div class="bs-callout bs-callout-orange item">
    	<h4 id="jsThirdPagelet" class="bs-callout-title-orange">${title}</h4>
    	<p>This is the thirdPagelet.</p>
	</div>

	<script type="text/javascript">
    	plumber.ready("jsThirdPagelet");
	</script>

当页面执行到这段 js 的时候，你所需操作的 dom 节点也已经被创建了，**jsThirdPagelet** 事件被触发，你的 callback 函数可以安全地得到执行。

事实上你应该尽可能地让你的 js 不要等到 dom ready 之后才会得以执行，这会严重拖慢你的页面速度：[Don't Let Document Ready Slow You Down](https://alexsexton.com/blog/2010/01/dont-let-document-ready-slow-you-down/)



##Change Log

####1.3.1

* [Bugfix] 解决了原来在 **PlumberController** 中注入的 **paramsForPagelets** 及 **modelForView** 为 ConcurrentHashMap 类型， 当用户向其 put 的 value 值为 **null** 时会报出 NPE 的问题( [Why does ConcurrentHashMap prevent null keys and values?](http://stackoverflow.com/questions/698638/why-does-concurrenthashmap-prevent-null-keys-and-values) ) 。

* [Change] 当页面包含 **pipe** 类型的 pagelet 时，默认关闭 nginx 的 **proxy_buffering** 功能。在使用 nginx 做反向代理时，nginx 默认开启了 **proxy_buffering**，它会将从服务器端接收到的返回数据都丢进 **buffer** 里，最后一起返回，这样服务端的 **pipe** 输出便等同于被 nginx 截流并蓄积起来了，**big-pipe** 的效果也便无法达到了。通过将 **X-Accel-Buffering** 这个 response header 设置为 **no** ,可以关闭 nginx 对该response 的 buffer 操作（ [ngx_http_proxy_module#proxy_buffering](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering) ）。

####1.4.0

* [Feature] 现在你也可以在 controller 的 spring xml 中配置页面 pagelet 的相关信息，并且它们与在容器页面中通过 **pb-barrier** 和 **pb-pipe** 的配置方式是兼容的。

* [Feature] 基类 **PlumberController** 中新增了 **barrierNames** 和 **pipeNames** 这两个成员变量，运行时框架会帮你自动注入，现在可以在 controller 的实例中获取 pagelet 的相关信息了。需要注意的是，它们是上述两种配置方式合并后的结果。

* [Change] 重构 **plumber** 内部异常体系，以 **PlumberInitializeFailureException** 和 **PlumberRuntimeException** 为基础建立异常树。

* [Change] 重构代码，重新组织了 core package 内部结构，划分出 **definitions** 和 **workers** 两个 package ，分别对应于 pagelet 的相关定义信息和并发任务 worker 。

####1.5.0

* [Feature] 加入 [**plumber.js**](https://github.com/zhongliangjun1/plumber.js)，你先期抵达客户端的页面模块再也不用等待 dom ready 了，具体见 *Advanced(5)*。

####1.6.0

* [Feature] 现在在 **dev** 环境下，**FreemarkerRenderer** 会在页面模板文件动态刷新后始终 render 最新的模板。

####1.7.0
* [Change] 重构了门面类 **plumber.java**，代码更精简了。
* [Feature] 原来指定 **pipe** 返回顺序的方式类似于指定数组下标，这种方式的问题在于改动一处下标后，后续的下标都需要更改，太麻烦。现在改成类似于 css 中 **z-index** 的方式，**@** 后的数值越大，优先级越高，返回越早。
* [Feature] 原来控制 **pipe** 返回顺序的方式是在 **worker** 线程中通过 **lock** 和 **condition** 来实现，这种实现方式存在 **worker** 线程利用率不高和过度锁竞争的问题。在 1.7.0 中引入了 **monitor** 的概念，**pipe** 类型的 task **worker** 做完之后，会直接 commit 给 **monitor**，由 **monitor** 去接管控制返回顺序的问题，**worker** 解放出来继续去做其他的事情。在 **monitor** 中实现了一个 **event loop** 来控制保证 **pipe** 的返回顺序。