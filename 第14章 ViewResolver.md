#14.2AbstractCachingViewResolver系列
------

AbstractCachingVrewResolver提供了统一的缓存功能，当视图解析过一次就被缓存起来，直到缓存被删除前视图的解析都会自动从缓存中获取它的直接继承类有一个：ResourceBundIeV1ewResoIver、XmIMewResoIver和UrIBased-MewResolvero第一个的用法在前面已经介绍过了，它是通过使用properties属性配置文件解析视图的；第二个跟第一个非常相似，只不过它使用了xml配置文件；第三个是所有直接将逻辑视图作为耐查找模板文件的MewResolver的基类，因为它设置了统一的查找模板的规则，所以它的子类只需要确定渲染方式也就是视图类型就可以了，它的每一个子类对应一种视图类型前两种解析器的实现原理非常简单，首先根据Locale将相应的配置文件初始化到BeanFactory,然后直接将逻辑视图作为beanName到cto里查找就可以了。它们两个的loadView的代码是一样的，如下：
```java

	@Override
	protected View loadView(String viewName, Locale locale) throws Exception {
		BeanFactory factory = initFactory(locale);
		try {
			return factory.getBean(viewName, View.class);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Allow for ViewResolver chaining...
			return null;
		}
	}
```
UriBasedViewResolver稍后再详细分析，先来看一下AbstractCachingViewResolver中解析视图的过程，代码如下：
```java
@Override
	public View resolveViewName(String viewName, Locale locale) throws Exception {
		if (!isCache()) {
			//如果不适用缓存，则创建视图
			return createView(viewName, locale);
		}
		else {
			//如果使用了缓存，需要检查是否已经存在缓存中，如果存在，则直接获取并返回
			Object cacheKey = getCacheKey(viewName, locale);
			View view = this.viewAccessCache.get(cacheKey);
			if (view == null) {
				synchronized (this.viewCreationCache) {
					view = this.viewCreationCache.get(cacheKey);
					if (view == null) {
						//如果不存在，则将当前视图放进缓存，并返回
						// Ask the subclass to create the View object.
						view = createView(viewName, locale);
						if (view == null && this.cacheUnresolved) {
							view = UNRESOLVED_VIEW;
						}
						if (view != null) {
							//如果缓存中存在，则直接返回
							this.viewAccessCache.put(cacheKey, view);
							this.viewCreationCache.put(cacheKey, view);
							if (logger.isTraceEnabled()) {
								logger.trace("Cached view [" + cacheKey + "]");
							}
						}
					}
				}
			}
			return (view != UNRESOLVED_VIEW ? view : null);
		}
	}
```
逻辑非常简单，首先判断是否开启了缓存功能，如果没开启则直接调用创建视图，否则检查是否已经存在缓存中，如果存在则直接获取并返回，否则使用createView创建一个，然后保存到缓存中并返回。createView内部直接调用了loadView方法，而loadView是一个模板方法，留给子类实际创建视图，这也是子类解析视图的入口方法。createVlew之所以调用了loadView而没有直接作为模板方法让子类使用是因为在loadView前可以统一做一些通用的解析，如果解析不到再交给执行，这点在UrlBasedViewResolver中有具体的体现。AbstractCachingViewResolver里有个cacheLimit参数需要说一下，它是用来设置最大缓存数的，当设置为0时不启用缓存，isCache就是判断它是否大于0，如果设置为一个大于0的数则它表示最多可以缓存视图的数量，如果往里面添加视图时超过了这个数那么最前面缓存的值将会删除。cacheLimit的默认值是1024，也就是最多可以缓存1024个视图

### 多知道一些
>LinkedHashMap中的自动删除功能

LinkedHashMap中保存的值是右顺序的，不过除了这点还有一个功能，它可以自动删除最前面保存的值，这个很多人并不知道。LinkedHashMap中有一个removeEldestEntry方法，如果这个方法返回true，Map中最前面添加的内容将被删除，它是在添加属性的put或putAIl方法被调用后自动调用的。这个功能主要是用在缓存中，用来限定缓存的最大数量，以防止缓存无限地长。当新的值添加后，如果缓存达到了上限，最开头的值就会被删除，当然这需要设置，设置方法就是覆盖removeEldestEntry法，当这个方法返回true时就表示达到了上限，返回false就是没达到上限，而size()方法可以返回现在所保存对象的数量，一般用它和设置的值做比较就可以了。AbstractCachingViewResolver中的viewCreationCache就是使用的这种方式，代码如下：
```java
private final Map<Object, View> viewCreationCache =
	new LinkedHashMap<Object, View>(DEFAULT_CACHE_LIMIT, 0.75f, true) {
		@Override
		protected boolean removeEldestEntry(Map.Entry<Object, View> eldest) {
			if (size() > getCacheLimit()) {
				viewAccessCache.remove(eldest.getKey());
				return true;
			}
			else {
				return false;
			}
		}
	};
```
在`AbstractCachingViewResolver`中使用了两个`Map`做缓存，它们分别是`viewAccessCache`和`viewCreationCache`。前者是`ConcurrentHashMap`类型，它内部使用了细粒度的锁，支持并发访问，效率非常高，而后者主要提供了限制缓存最大数的功能，效率不如前者高。使用的最多的获取缓存是从前者获取的，而添加缓存会给两者同时添加，后者如果发现缓存数量已达到上限时会在删除白己最前面的缓存的同时也删除前者对应的缓存。这种将两种`Map`的优点结合起来的用法非常值得我们学习和借鉴。


##UrIBasedViewResolver
    UrIBasedViewResolver里面重写了父类的getCacheKey、createView和loadView三个方法。
    getCacheKey方法直接返回viewN ame，它用于父类AbstractCachingViewResolver中设置缓存的key，原来（AbstractCachingViewResolver中）使用的是viewName+“”+locale，也就是说UrlBasedViewResolver的缓存中key没有使用Locale只使用了viewN ame，从这里可以看出UrlBasedViewResolver不支持Locale。
    在createView中首先检查是否可以解析传人的逻辑视图，如果不可以则返回null让别的ViewResolver解析，接着分别检查是不是redirect视图成者forward视图，检查的方法是看是不是以“redirect：”或“forward：”开头，如果是则返回相应视图，如果都不是则交给父类的createView，父类中又调用了loadView，代码如下：
```java
//org. springframework.web. servlet .view.UrlBasedViewRe solver
protected View createView (String viewName,  Locale  locale)  throws  Exception  {
    ／／检查是否支持此逻辑视图，可以配置支持的模板
    if  (!canHandle (viewName,  locale))  {
    return null;
    )
    ／／检查是不是redirect视图
    if   (viewName. st artsWith (REDIRECT—URL—PREFIX))  {
    String redirectUrl=viewName.substring (REDIRECT URL  PREFIX.Length());
    RedirectView view=new RedirectView (redirectUrl,  isRedirectContextRelative(),
    isRedirectHttplOCompatible());
    return  applyLifecycleMethods (viewName,  view);
    )
    ，／检查是不是forward视图
    if   (viewName. startsWith( FORWARD_URL—PREFIX))  {
    String  forwardUrl=viewName.substring (FORWARD_URL  PREFIX.length (》 ;
    return  new  InternalResourceView ( forwardUrl) ;
    )
    ／／如果都不是则调用父类的createView，也就会调用loadView
    return  super. createView (viewName,   locale) ;
```
其实这里是为所有UrlBasedViewResolver子类解析器统一添加了检查是否支持传人的逻辑视图和传人的逻辑视图是不是redirect或者forward视图的功能。检查是否支持是调用的canHandle方法，它是通过可以配置的viewNames属性检查的，如果没有配置则可以解析所有逻辑视图，如果配置了则按配置的模式检查，配置的方法可以直接将所有可以解析的逻辑视图配置迸去，也可以配置逻辑视图需要满足的模板，如“*report”“goto*”“*from*”等，代码如下：
```java
	protected boolean canHandle(String viewName, Locale locale) {
		String[] viewNames = getViewNames();
		return (viewNames == null || PatternMatchUtils.simpleMatch(viewNames, viewName));
	}
```

loadView方法代码如下：
```java
	@Override
	protected View loadView(String viewName, Locale locale) throws Exception {
		AbstractUrlBasedView view = buildView(viewName);
		View result = applyLifecycleMethods(viewName, view);
		return (view.checkResource(locale) ? result : null);
	}
```
loadView -共执行了三句代码：（使用buildView方法创建View；（别吏用applyLifecycleMethods方法对创建的View初始化；③检查vlew对应的模板是否存在，如果存在则将初始化的视图返回，否则返回null交给下一个ViewResolver处理。    
applyLifecycleMethods方法是通过容器获取到Factory然后实现的。
 ```JAVA
//org.springframework.web. servlet.view.UrlBasedViewResolver
	private View applyLifecycleMethods(String viewName, AbstractView view) {
		return (View) getApplicationContext().getAutowireCapableBeanFactory().initializeBean(view, viewName);
	}
```
下面来看一下buildView方法，它用于具体创建View，理解了这个方法就知道AbstractUrIBasedView系列中View是怎么创建的了，它的子类只是在这里创建出来的视图的基础上设置了一些属性。所以这是AbstractUrIBasedView申最重要的方法，代码如下：
```JAVA
protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(getViewClass());
		view.setUrl(getPrefix() + viewName + getSuffix());
		//如果exposePathVariables不为null，将其值设置给view，它用于标示是否让view
		//使用PathVariables，可以在ViewResolver中配置。PathVariables就是处理器中
		//@PathVariables注释的参数
		String contentType = getContentType();
		if (contentType != null) {
			view.setContentType(contentType);
		}

		view.setRequestContextAttribute(getRequestContextAttribute());
		view.setAttributesMap(getAttributesMap());

		Boolean exposePathVariables = getExposePathVariables();
		if (exposePathVariables != null) {
			view.setExposePathVariables(exposePathVariables);
		}
		Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
		if (exposeContextBeansAsAttributes != null) {
			view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
		}
		String[] exposedContextBeanNames = getExposedContextBeanNames();
		if (exposedContextBeanNames != null) {
			view.setExposedContextBeanNames(exposedContextBeanNames);
		}

		return view;
	}
```
View的创建过程也非常简单，首先根据使用BeanUtils根据geMewClass有法的返回值创建出view，然后将viewName加上前缀、后缀设置为url，前缀和后缀可以在配置ViewResolver时进行设置，这样Mew就创建完了，接下来根据配置给View设置一些参数，具体内容已经注释到代码上了。这里的getViewClass返回其中的viewClass属性，代表View的视图类型，可以在子类通过setViewClass方法进行设置。另外还有一个requiredViewClass方法，它用于在设置视图时判断所设置的类型是否支持，在UrIBasedMewResolver中默认返回AbstractUrIBasedView类型，requiredViewClass使用在设置视图的setViewClass方法中，代码如下：
```JAVA
	public void setViewClass(Class viewClass) {
		if (viewClass == null || !requiredViewClass().isAssignableFrom(viewClass)) {
			throw new IllegalArgumentException(
					"Given view class [" + (viewClass != null ? viewClass.getName() : null) +
					"] is not of type [" + requiredViewClass().getName() + "]");
		}
		this.viewClass = viewClass;
	}

```
 UriBasedViewResolver的代码就分析完了，通过前面的分析可知，只需要给它设置AbstractUrIBasedView类型的viewClass就可以直接使用了，我们可以直接注册配置了viewClass的UrIBasedViewResolver来使用，不过最好还是使用相应的子类。UrIBasedViewResolver的子类主要做i件事：①通过重写requiredViewClass方法修改了必须符合的视图类型的值；②使用setViewClass方法设置了所用的视图类型；③给创建出来的视图设置一些属性。下面来看一下使用得非常多的InternalResourceViewResolver和FreeMarkerViewResolver，前者用于解析jsp视图后者用于解析FreeMarker视图，其他实现类也都差不多。InternaIResourceViewResolver直接继承自UrIBasedViewResolver，它在构造方法中设置了viewClass．在buildView中对父类创建的View设置了一些属性，requiredViewClass方法返回InternalResourceView类型，代码如下：
```JAVA
public class InternalResourceViewResolver extends UrlBasedViewResolver {

	private static final boolean jstlPresent = ClassUtils.isPresent(
			"javax.servlet.jsp.jstl.core.Config", InternalResourceViewResolver.class.getClassLoader());

	private Boolean alwaysInclude;

	public InternalResourceViewResolver() {
		Class viewClass = requiredViewClass();
		if (viewClass.equals(InternalResourceView.class) && jstlPresent) {
			viewClass = JstlView.class;
		}
		setViewClass(viewClass);
	}

	@Override
	protected Class requiredViewClass() {
		return InternalResourceView.class;
	}

	public void setAlwaysInclude(boolean alwaysInclude) {
		this.alwaysInclude = alwaysInclude;
	}

	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		InternalResourceView view = (InternalResourceView) super.buildView(viewName);
		if (this.alwaysInclude != null) {
			view.setAlwaysInclude(this.alwaysInclude);
		}
		view.setPreventDispatchLoop(true);
		return view;
	}

}
```
 buildMew方法中给创建出来的View设置的alwayslnclude用于标示是否在可以使用forward的情况下也强制使用include，默认为false，可以在注册解析器时配置。setPreventDispatchLoop(true)用于阻止循环调用，也就是请求处理完成后又转发回了原来使用的处理器的情况。
    FreeMarkerViewResolver继承自UrIBasedViewResolver的子类AbstractTemplateViewResolver，AbstractTemplateViewResolver是所用模板类型ViewResolver的父类，它里面主要对创建的View设置了一些属性，并将requiredViewClass的返回值设置为AbstractTemplateView类型。代码如下：
```JAVA
public class AbstractTemplateViewResolver extends UrlBasedViewResolver {

	private boolean exposeRequestAttributes = false;

	private boolean allowRequestOverride = false;

	private boolean exposeSessionAttributes = false;

	private boolean allowSessionOverride = false;

	private boolean exposeSpringMacroHelpers = true;


	@Override
	protected Class requiredViewClass() {
		return AbstractTemplateView.class;
	}

	/**
	 * Set whether all request attributes should be added to the
	 * model prior to merging with the template. Default is "false".
	 * @see AbstractTemplateView#setExposeRequestAttributes
	 */
	public void setExposeRequestAttributes(boolean exposeRequestAttributes) {
		this.exposeRequestAttributes = exposeRequestAttributes;
	}

	/**
	 * Set whether HttpServletRequest attributes are allowed to override (hide)
	 * controller generated model attributes of the same name. Default is "false",
	 * which causes an exception to be thrown if request attributes of the same
	 * name as model attributes are found.
	 * @see AbstractTemplateView#setAllowRequestOverride
	 */
	public void setAllowRequestOverride(boolean allowRequestOverride) {
		this.allowRequestOverride = allowRequestOverride;
	}

	/**
	 * Set whether all HttpSession attributes should be added to the
	 * model prior to merging with the template. Default is "false".
	 * @see AbstractTemplateView#setExposeSessionAttributes
	 */
	public void setExposeSessionAttributes(boolean exposeSessionAttributes) {
		this.exposeSessionAttributes = exposeSessionAttributes;
	}

	/**
	 * Set whether HttpSession attributes are allowed to override (hide)
	 * controller generated model attributes of the same name. Default is "false",
	 * which causes an exception to be thrown if session attributes of the same
	 * name as model attributes are found.
	 * @see AbstractTemplateView#setAllowSessionOverride
	 */
	public void setAllowSessionOverride(boolean allowSessionOverride) {
		this.allowSessionOverride = allowSessionOverride;
	}

	/**
	 * Set whether to expose a RequestContext for use by Spring's macro library,
	 * under the name "springMacroRequestContext". Default is "true".
	 * @see AbstractTemplateView#setExposeSpringMacroHelpers
	 */
	public void setExposeSpringMacroHelpers(boolean exposeSpringMacroHelpers) {
		this.exposeSpringMacroHelpers = exposeSpringMacroHelpers;
	}

	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		AbstractTemplateView view = (AbstractTemplateView) super.buildView(viewName);
		view.setExposeRequestAttributes(this.exposeRequestAttributes);
		view.setAllowRequestOverride(this.allowRequestOverride);
		view.setExposeSessionAttributes(this.exposeSessionAttributes);
		view.setAllowSessionOverride(this.allowSessionOverride);
		view.setExposeSpringMacroHelpers(this.exposeSpringMacroHelpers);
		return view;
	}

}
```

下面解释这5个属性的含义：
>1. exposeRequestAttributes：是否将所有RequestAttributes暴露给vlew使用，默认为false。
>2. allowRequestOverride：当RequestAttributes中存在Model中同名的参数时，是否允许
>3. 使用RequestAttributes中的值将Model中的值覆盖，默认为false。
>4. exposeSessionAttributes：是否将所有SessionAttributes暴露给view使用，默认为false。
>5. allowSessionOverride：当SessionAttributes中存在Model中同名的参数时，是否允许使用RequestAttributes中的值将Model中的值覆盖，默认为false。
>6. exposeSpringMacroHelpers：是否将RequestContext暴露给view为spring的宏使用，默认为true。

FreeMarkerViewResolver的代码就非常简单了，只是覆盖requiredViewClass方法返回FreeMarkerView类型，并在构造方法中调用setViewClass方法设置了viewClass。
```JAVA
public class FreeMarkerViewResolver extends AbstractTemplateViewResolver {

	public FreeMarkerViewResolver() {
		setViewClass(requiredViewClass());
	}

	/**
	 * Requires {@link FreeMarkerView}.
	 */
	@Override
	protected Class requiredViewClass() {
		return FreeMarkerView.class;
	}

}
```
#14.3 小结
UrIBasedViewResolver就分析完了，它实际完成了根据子类提供的viewClass类型创建视图的功能，它的子类只需要提供viewClass就可以了，有的子类会在创建完的视图上设置一些属性。
   
本章详细分析了Spring MVC中ViewResolver的各种实现方式。大部分实现类都继承自AbstractCachingViewResolver，它提供了对解析结果进行缓存的统一解决方案，它的子类中ResourceBundleViewResolver和XmlViewResolver分别通过properties邗xml配置文件进行解析，UrlBasedViewResolver将viewName添加前后缀后用作url，它的子类只需要提供视图类型就可以了。
    除了AbstractCaclungViewResolver外，还有i个类：BeanNameViewResolver、Content-NegotiatingViewResolver和ViewResolverComposite。第一个通过在spring容器里使用viewName查找bean来作为View；第二个是使用内部封装的ViewResolver解析后再根据MediaType或后缀找出最优的视图；第三个是直接遍历内部封装的ViewResolver进行解析。这三个类中第一个的视图是在spring容器中，后两个是通过别的ViewResolver进行解析的，都不需要直接创建View，所以也不需要缓存。
    解析视图的核心T作就是查找模板文件和视图类型，而查找的主要参数只有viewName -个（Locale只能起辅助作用）。这就产生了三种解析思路：①使用viewName查找模板文件；②使用viewName查找视图类型；③使用viewName同时查找模板文件和视图类型。Spring MVC对这三种思路都提供了实现的方式，第一种思路对应的是UrlBasedViewResolver系列；第二种思路对应的是BeanNameViewResolver；第三种思路对应的是ResourceBundleViewResolver和XmlViewResolver。理解了这层意思就能明白SpringMVC中ViewResolver的结构为什么这么安排了，如果ResourceBundleViewResolver和XmlViewResolver再提取一个父类出来就更清晰了。另外需要注意的是，并不是每个请求都需要使用ViewResolver．只有当处理器执行后vlew是String类型时才需要用它来解析。
