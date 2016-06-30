# 第16章HandlerExceptionResolver
HandlerExceptionResolver用于解析请求处理过程中所产生的异常，继承结构如图16-1所示。
![图16.1 继承结构][1]
其中HandlerExceptionResolverComposite作为容器使用，可以封装别的Resolver，前面已经多次介绍过，这里就不再叙述了。
HandlerExceptionResolver的主要实现都继承自抽象类AbstractHandlerExceptionResolver，它有五个子类，其中的AnnotatiionMethodHandlerExceptionResolver已经被弃用，剩下的还有四个：
>1. AbstractHandlerMethodExceptionResolver:和其子类ExceptionHandlerExceptionResolver一起完成使用@ExceptionHandler注释的方法进行异常解析的功能。
>2. De faultHandlerExceptionResolver：按不同类型分别对异常进行解析。
>3. ResponseStatusExceptionResolver：解析有@ResponseStatus注释类型的异常。
>4. SimpleMappingExceptionResolver：通过配置的异常类和view的对应关系来解析异常。

异常解析过程主要包含两部分内容：给ModeIAndView设置相应内容、设置response的相关属性。当然还可能有一些辅助功能，如记录日志等，在白定义的ExceptionHandler里还可以做更多的事情。
## 16.1  AbstractHandlerExceptionResolver
AbstractHandlerExceptionResolver是所有直接解析异常类的父类，里面定义了通用的解析流程，并使用了模板模式，子类只需要覆盖相应的方法即可，resolveException方法如下：
```JAVA
	public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
			Object handler, Exception ex) {

		if (shouldApplyTo(request, handler)) {
			// Log exception, both at debug log level and at warn level, if desired.
			if (logger.isDebugEnabled()) {
				logger.debug("Resolving exception from handler [" + handler + "]: " + ex);
			}
			logException(ex, request);
			prepareResponse(ex, response);
			return doResolveException(request, response, handler, ex);
		}
		else {
			return null;
		}
	}
```
首先使用shouldApplyTo方法判断当前ExceptionResolver是否可以解析所传入处理器所抛出的异常（可以指定只能处理指定的处理器抛出的异常），如果不可以则返回null，交给下一个ExceptionResolver解析，如果可以则调用logException方法打印日志，接着调用prepareResponse设置response，最后调用doResolveException实际解析异常，doResolveException是个模板方法，留给子类实现。
```JAVA
	protected boolean shouldApplyTo(HttpServletRequest request, Object handler) {
		if (handler != null) {
			if (this.mappedHandlers != null && this.mappedHandlers.contains(handler)) {
				return true;
			}
			if (this.mappedHandlerClasses != null) {
				for (Class<?> handlerClass : this.mappedHandlerClasses) {
					if (handlerClass.isInstance(handler)) {
						return true;
					}
				}
			}
		}
		// Else only apply if there are no explicit handler mappings.
		return (this.mappedHandlers == null && this.mappedHandlerClasses == null);
	}
```
这里使用了两个属性：mappedHandlers和mappedHandlerClasses，这两个属性可以在定义HandlerExceptionResolver的时候进行配置，用于指定可以解析处理器抛出的哪些异常，也就是如果设置了这两个值中的一个，那么这个ExceptionResolver就只能解析所设置的处理器抛出的异常。mappedHandlers用于配置处理器的集合，mappedHandlerClasses用于配置处理器类型的集合。检查方法非常简单，在此就不细说了，如果两个属性都没配置则将处理所有异常。
logException是默认记录日志的方法，代码如下：
```JAVA
	protected void logException(Exception ex, HttpServletRequest request) {
		if (this.warnLogger != null && this.warnLogger.isWarnEnabled()) {
			this.warnLogger.warn(buildLogMessage(ex, request), ex);
		}
	}
	
	protected String buildLogMessage(Exception ex, HttpServletRequest request) {
		return "Handler execution resulted in exception";
	}
```
logException方法首先调用buildLogMessage创建了日志消息，然后使用warnLogger将其记录下来。
prepareResponse方法根据preventResponseCaching杯示判断是否给response设置禁用缓存的属性，preventResponseCaching默认为false，代码如下：
```JAVA
protected void prepareResponse(Exception ex, HttpServletResponse response) {
	if (this.preventResponseCaching) {
		preventCaching(response);
	}
}

protected void preventCaching(HttpServletResponse response) {
	response.setHeader(HEADER_PRAGMA, "no-cache");
	response.setDateHeader(HEADER_EXPIRES, 1L);
	response.setHeader(HEADER_CACHE_CONTROL, "no-cache");
	response.addHeader(HEADER_CACHE_CONTROL, "no-store");
}

```
最后的doResolveException方法是模板方法，子类使用它具体完成异常的解析工作。
## 16.2 ExceptionHandlerExceptionResolver
ExceptionHandlerExceptionResolver继承自AbstractHandlerMethodExceptionResolver，后者继承自AbstractHandlerExceptionResolverc, AbstractHandlerMethodExceptionResolver重写了shouldApplyTo方法，并在处理请求的doResolveException方法中将实际处理请求的过程交给了模板方法doResolveHandlerMethodException。代码如下：
```JAVA
@Override
protected boolean shouldApplyTo(HttpServletRequest request, Object handler) {
	if (handler == null) {
		return super.shouldApplyTo(request, handler);
	}
	else if (handler instanceof HandlerMethod) {
		HandlerMethod handlerMethod = (HandlerMethod) handler;
		handler = handlerMethod.getBean();
		return super.shouldApplyTo(request, handler);
	}
	else {
		return false;
	}
}

@Override
protected final ModelAndView doResolveException(
		HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	return doResolveHandlerMethodException(request, response, (HandlerMethod) handler, ex);
}
protected abstract ModelAndView doResolveHandlerMethodException(
		HttpServletRequest request, HttpServletResponse response,
		HandlerMethod handlerMethod, Exception ex);

````
AbstractHandlerMethodException Resolver的作用其实相当于一个适配器。一般的处理器是类的形式，但HandlerMethod其实是将方法作为处理器耒使用的，所以需要进行适配。首先在shouldApplyTo中判断如果处理器是HandlerMethod类型则将处理器设置为其所在的类，然后冉交给父类判断，如果为空则直接交给父类判断，如果既不为空也不是HandlerMethod类型则返回false不处理。

doResolveException将处理传递给doResolveHandlerMethodException方法具体处理，这样做主要是为了层次更加合理，而且这样设计后如果有多个子类还可以在doResolveException中统一做一些事情。

下面来看ExceptionHandlerException Resolver，它其实就是一个简化版的RequestMappingHandlerAdapter，它的执行也是使用的ServletlnvocableHandlerMethod，首先根据handlerMethod和exception将其创建出来（大致过程是在处理器类里找出所有注释了@ExceptionHandler的方法，然后再根据其配置中的异常和需要解析的异常进行匹配），然后设置了argumentResolvers和returnValueHandlers．接着调用其invokeAndHandle方法执行处理，最后将处理结果封装成ModelAndView返回。如果RequestMappingHandlerAdapter理解了，再来看它就会觉得非常简单。代码如下：
```JAVA
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {

	ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
	if (exceptionHandlerMethod == null) {
		return null;
	}

	exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
	exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);

	ServletWebRequest webRequest = new ServletWebRequest(request, response);
	ModelAndViewContainer mavContainer = new ModelAndViewContainer();

	try {
		if (logger.isDebugEnabled()) {
			logger.debug("Invoking @ExceptionHandler method: " + exceptionHandlerMethod);
		}
		exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception);
	}
	catch (Exception invocationEx) {
		if (logger.isErrorEnabled()) {
			logger.error("Failed to invoke @ExceptionHandler method: " + exceptionHandlerMethod, invocationEx);
		}
		return null;
	}

	if (mavContainer.isRequestHandled()) {
		return new ModelAndView();
	}
	else {
		ModelAndView mav = new ModelAndView().addAllObjects(mavContainer.getModel());
		mav.setViewName(mavContainer.getViewName());
		if (!mavContainer.isViewReference()) {
			mav.setView((View) mavContainer.getView());
		}
		return mav;
	}
}

```
这里只是返回了ModelAndView，并没有对response进行设置，如果需要可以自己在异常处理器中设置。
## 16.3 DefaultHandlerExceptionResolver
DefaultHandlerExceptionResolver昀解析过程是根据异常类型的不同，使用不同的方法进行处理，doResolveException代码如下：
```JAVA
@Override
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	try {
		if (ex instanceof NoSuchRequestHandlingMethodException) {
			return handleNoSuchRequestHandlingMethod((NoSuchRequestHandlingMethodException) ex, request, response,
					handler);
		}
		else if (ex instanceof HttpRequestMethodNotSupportedException) {
			return handleHttpRequestMethodNotSupported((HttpRequestMethodNotSupportedException) ex, request,
					response, handler);
		}
		else if (ex instanceof HttpMediaTypeNotSupportedException) {
			return handleHttpMediaTypeNotSupported((HttpMediaTypeNotSupportedException) ex, request, response,
					handler);
		}
		else if (ex instanceof HttpMediaTypeNotAcceptableException) {
			return handleHttpMediaTypeNotAcceptable((HttpMediaTypeNotAcceptableException) ex, request, response,
					handler);
		}
		else if (ex instanceof MissingServletRequestParameterException) {
			return handleMissingServletRequestParameter((MissingServletRequestParameterException) ex, request,
					response, handler);
		}
		else if (ex instanceof ServletRequestBindingException) {
			return handleServletRequestBindingException((ServletRequestBindingException) ex, request, response,
					handler);
		}
		else if (ex instanceof ConversionNotSupportedException) {
			return handleConversionNotSupported((ConversionNotSupportedException) ex, request, response, handler);
		}
		else if (ex instanceof TypeMismatchException) {
			return handleTypeMismatch((TypeMismatchException) ex, request, response, handler);
		}
		else if (ex instanceof HttpMessageNotReadableException) {
			return handleHttpMessageNotReadable((HttpMessageNotReadableException) ex, request, response, handler);
		}
		else if (ex instanceof HttpMessageNotWritableException) {
			return handleHttpMessageNotWritable((HttpMessageNotWritableException) ex, request, response, handler);
		}
		else if (ex instanceof MethodArgumentNotValidException) {
			return handleMethodArgumentNotValidException((MethodArgumentNotValidException) ex, request, response, handler);
		}
		else if (ex instanceof MissingServletRequestPartException) {
			return handleMissingServletRequestPartException((MissingServletRequestPartException) ex, request, response, handler);
		}
		else if (ex instanceof BindException) {
			return handleBindException((BindException) ex, request, response, handler);
		}
		else if (ex instanceof NoHandlerFoundException) {
			return handleNoHandlerFoundException((NoHandlerFoundException) ex, request, response, handler);
		}
	}
	catch (Exception handlerException) {
		logger.warn("Handling of [" + ex.getClass().getName() + "] resulted in Exception", handlerException);
	}
	return null;
}
```
其体的解析方法也非常简单，主要是设置response的相关属性，下面介绍前两个异常的处理方法，也就是没找到处理器执行方法和request的Method类型不支持的异常处理，代码如下：
```JAVA
protected ModelAndView handleNoSuchRequestHandlingMethod(NoSuchRequestHandlingMethodException ex,
		HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {

	pageNotFoundLogger.warn(ex.getMessage());
	response.sendError(HttpServletResponse.SC_NOT_FOUND);
	return new ModelAndView();
}

protected ModelAndView handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex,
		HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {

	pageNotFoundLogger.warn(ex.getMessage());
	String[] supportedMethods = ex.getSupportedMethods();
	if (supportedMethods != null) {
		response.setHeader("Allow", StringUtils.arrayToDelimitedString(supportedMethods, ", "));
	}
	response.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED, ex.getMessage());
	return new ModelAndView();
}
```
可以看到其处理方法就是给response设置了相应属性，然后返回一个空的ModelAndView。其中response的sendError方法用于设置错误类型，它有两个重载方法sendError(int)和sendError(int，String)，int参数用于设置404）500等错误类型，String类型的参数用于设置附加的错误信息，可以在页面中获取到。sendError和setStatus方法的区别是前者会返回web.xml中定义的相应错误页面，后者只是设置了status而不会返回相应错误页面。其他处理方法也都大同小异，就不解释了。

## 16.4  ResponseStatusExceptionResolver
ResponseStatusExceptionResolver用来解析注释了@ResponseStatus的异常（如自定义的注释了@ResponseStatus的异常），代码如下：
```JAVA
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	ResponseStatus responseStatus = AnnotationUtils.findAnnotation(ex.getClass(), ResponseStatus.class);
	if (responseStatus != null) {
		try {
			return resolveResponseStatus(responseStatus, request, response, handler, ex);
		}
		catch (Exception resolveEx) {
			logger.warn("Handling of @ResponseStatus resulted in Exception", resolveEx);
		}
	}
	return null;
}

protected ModelAndView resolveResponseStatus(ResponseStatus responseStatus, HttpServletRequest request,
		HttpServletResponse response, Object handler, Exception ex) throws Exception {

	int statusCode = responseStatus.value().value();
	String reason = responseStatus.reason();
	if (this.messageSource != null) {
		reason = this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale());
	}
	if (!StringUtils.hasLength(reason)) {
		response.sendError(statusCode);
	}
	else {
		response.sendError(statusCode, reason);
	}
	return new ModelAndView();
}
```
doResolveException万法中首先使用AnnotationUtils找到ResponseStatus注释，然后调用resolveResponseStatus方法进行解析，后者使用注释里的value和reason作为参数调用了Response的sendError方法。
## 16.5 SimpleMappingExceptionResolver
SimpleMappingExceptionResolver需要提前配置异常类和view的对应关系然后才能使用，doResolveException代码如下：
```JAVA
@Override
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) {

	// Expose ModelAndView for chosen error view.
	String viewName = determineViewName(ex, request);
	if (viewName != null) {
		// Apply HTTP status code for error views, if specified.
		// Only apply it if we're processing a top-level request.
		Integer statusCode = determineStatusCode(request, viewName);
		if (statusCode != null) {
			applyStatusCodeIfPossible(request, response, statusCode);
		}
		return getModelAndView(viewName, ex, request);
	}
	else {
		return null;
	}
}
```
这里首先调用determineViewName方法根据异常找到显示异常的逻辑视图，然后调用determineStatusCode方法判断逻辑视图是否有对应的statusCode，如果有则调用applyStatusCodelfPossible方法设置到response，最后调用getModelAndView将异常和解析出的viewName封装成ModeIAndView并返回。
查找视图的determineViewName方法如下:
```JAVA
protected String determineViewName(Exception ex, HttpServletRequest request) {
	String viewName = null;
	//如果异常在设置的excludedExceptions中所包含则返回null
	if (this.excludedExceptions != null) {
		for (Class<?> excludedEx : this.excludedExceptions) {
			if (excludedEx.equals(ex.getClass())) {
				return null;
			}
		}
	}
	// 调用findMatchingViewName方法实际查
	if (this.exceptionMappings != null) {
		viewName = findMatchingViewName(this.exceptionMappings, ex);
	}
	// 如果没找到viewName并且配置了defaultErrorView，则使用defaultErrorView
	if (viewName == null && this.defaultErrorView != null) {
		if (logger.isDebugEnabled()) {
			logger.debug("Resolving to default view '" + this.defaultErrorView + "' for exception of type [" +
					ex.getClass().getName() + "]");
		}
		viewName = this.defaultErrorView;
	}
	return viewName;
}
```
这里首先检查异常是不是配置在excludedExceptions中（excludedExceptions用于配置不处理的异常），如果是则返回null，否则调用findMatchingViewName实际查找viewName，如果没找到而且配置了defaultErrorView，则使用defaultErrorView。findMatchingViewName从传人的参数就可以肴出来它是根据配置的exceptionMappings参数匹配当前异常的，不过并不是直接完全匹配的，而是只要配置异常的字符在当前处理的异常或其父类中存在就可以了，如配置“BindingException”可以匹配“xxx.UserBindingException”“xxx.DeptBindingException”等，而“java.lang.Exception”可以匹配所有它的子类，即所有“CheckedExceptions”，其代码如下：
```JAVA
protected String findMatchingViewName(Properties exceptionMappings, Exception ex) {
	String viewName = null;
	String dominantMapping = null;
	int deepest = Integer.MAX_VALUE;
	for (Enumeration<?> names = exceptionMappings.propertyNames(); names.hasMoreElements();) {
		String exceptionMapping = (String) names.nextElement();
		int depth = getDepth(exceptionMapping, ex);
		if (depth >= 0 && (depth < deepest || (depth == deepest &&
				dominantMapping != null && exceptionMapping.length() > dominantMapping.length()))) {
			deepest = depth;
			dominantMapping = exceptionMapping;
			viewName = exceptionMappings.getProperty(exceptionMapping);
		}
	}
	if (viewName != null && logger.isDebugEnabled()) {
		logger.debug("Resolving to view '" + viewName + "' for exception of type [" + ex.getClass().getName() +
				"], based on exception mapping [" + dominantMapping + "]");
	}
	return viewName;
}
````
大致过程就是遍历配置文件，然后调用getDepth查找，如果返回值大于等于0则说明可以匹配，而且如果有多个匹配项则选择最优的，选择方法是判断两项内容：①匹配的深度；②匹配的配置项文本的长度。深度越浅越好，配置的文本越长越好。深度是指如果匹配的是异常的父类而不是异常本身，那么深度就是异常本身到被匹配的父类之间的继承层数。getDepth方法的代码如下：
```JAVA
protected int getDepth(String exceptionMapping, Exception ex) {
	return getDepth(exceptionMapping, ex.getClass(), 0);
}

private int getDepth(String exceptionMapping, Class<?> exceptionClass, int depth) {
	//如果异常的类名里包含查找的配置文本则匹配成功
	if (exceptionClass.getName().contains(exceptionMapping)) {
		// Found it!
		return depth;
	}
	// 查找到异常的根类Throwable还没匹配，则说明不匹配，返回-1
	if (exceptionClass.equals(Throwable.class)) {
		return -1;
	}
	return getDepth(exceptionMapping, exceptionClass.getSuperclass(), depth + 1);
}
```
getDepth申调用了同名的带depth参数的递归方法getDepth，并给depth传人了0。后面的getDepth方法判断传人的类名中是否包含匹配的字符串，如果找到则返回相应的depth，如果没有则对depth加1后递归检查异常的父类，直到检查的类变成Throwable还不匹配则返回-l。
分析完determineViewName，再回过头去分析determineStatusCode，代码如下：
```JAVA
protected Integer determineStatusCode(HttpServletRequest request, String viewName) {
	if (this.statusCodes.containsKey(viewName)) {
		return this.statusCodes.get(viewName);
	}
	return this.defaultStatusCode;
}
```
determineStatusCode方法非常简单，就是直接从配置的statusCodes中用viewName获取，如果获取不到则返回默认值defaultStatusCode．statusCodes和defaultStatusCode都可以在定义SimpleMappingExceptionResolver的时候进行配置。找到statusCode后会调用applyStatus-
CodelfPossible方法将其设置到response上，代码如下：
```java
protected void applyStatusCodeIfPossible(HttpServletRequest request, HttpServletResponse response, int statusCode) {
	if (!WebUtils.isIncludeRequest(request)) {
		if (logger.isDebugEnabled()) {
			logger.debug("Applying HTTP status code " + statusCode);
		}
		response.setStatus(statusCode);
		request.setAttribute(WebUtils.ERROR_STATUS_CODE_ATTRIBUTE, statusCode);
	}
}
```
最后调用getModeIAndView生成ModelAndView并返回，生成过程是将解析出的viewName没置为View，如果exceptionAttribute不为空则将异常添加到Model，代码如下：
```java
protected ModelAndView getModelAndView(String viewName, Exception ex, HttpServletRequest request) {
		return getModelAndView(viewName, ex);
	}
	
protected ModelAndView getModelAndView(String viewName, Exception ex) {
	ModelAndView mv = new ModelAndView(viewName);
	if (this.exceptionAttribute != null) {
		if (logger.isDebugEnabled()) {
			logger.debug("Exposing Exception as model attribute '" + this.exceptionAttribute + "'");
		}
		mv.addObject(this.exceptionAttribute, ex);
	}
	return mv;
}
```
SimpleMappingExceptionResolver就分析完了，这里面有不少可配置的选项，总结如下：
>1. exceptionMappings：用于配置异常类（字符串类型）和viewName的对应关系，异常类可以是异常（包含包名的完整名）的一部分，还可以是异常父类的一部分。
>2. excludedExceptions：用于配置不处理的异常。
>3. defaultErrorView：用于配置当无法从exceptionMappings中解析出视图时使用的默认视图．
>4. statusCodes：用于配置解析出的viewName和statusCode对应关系。
>5. defaultStatusCode：用于配置statusCodes中没有配置相应的viewName时使用的默认statusCode。
>6. exceptionAttribute：用于配置异常在Model中保存的参数名，默认为”exception”，如果为null，异常将不保存到Model中


## 16.6 小结

本章详细分析了Spring MVC中异常解析组件HandlerExceptionResolver的所有实现类。实现类中除了HandlerExceptionResolverComposite，每个类处理一种异常处理方式，它们具体做的工作主要包括将异常相关信息设置到Model中和给response设置相应属性两个方面。

mvc:annotation-driven会自动将ExceptionHandlerExceptionResolver、DefaultHandlerExceptionResolver和ResponseStatusExceptionResolver配置到Spring MVC中，SimpleMappingExceptionResolver如果想使用需要自己配置，其实SimpleMappingExceptionResolver的使用还是很方便的，只需要将异常类型和错误页面的对应关系设置进去就可以了，而且还可以通过设置父类将某种类型的所有异常对应到指定页面。另外ExceptionHandlerExceptionResolver不仅可似使用处理器类中注释的@ExceptionHandler方法处理异常，还可以使用@ControllerAdvice注释的类里有@ExceptionHandler注释的全局异常处理方法。

一般刚开始接触编程的人都会对异常感到害怕，也讨厌异常。不过等学会调试并随着编程经验逐渐增多以后慢慢会觉得异常还是挺有用，是排查问题中非常有用的武器。随着自己慢慢成长，直到有一天自己也开始定义并使用异常的时候，才会发现异常原来是那么有用、那么方便！

HandlerExceptionResolver是SpringMVC提供的非常方便的通用异常处理工具，不过需要注意的是，它只能处理请求处理过程中抛出的异常，异常处理本身所抛出的异常和视图解析过程中抛出的异常它是不能处理的。

  [1]: ./images/1467204589895.jpg "1467204589895.jpg"
