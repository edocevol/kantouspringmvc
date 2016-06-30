# 第20章 FlashMapManager

FlashMapManager用来管理FlashMap，FlashMap用于在redirect时传递参数，前面已经介绍过。FlashMapManager的继承结构如图20-1所示。

![enter description here][1]

在Spring MVC中FlashMapManager的实现结构非常简单，一个抽象类和一个具体实现类，抽象类采用模板模式定义了整体流程，具体实现类SessionFlashMapManager通过模板方法提供了具体操作FlashMap的功能。
    
在具体分析代码前先说明两点：第一，实际在Session中保存的FlashMap是List<FlashMap>类型，也就是说FlashMap保存着一套Redriect转发所传递的参数；第二，FlashMap继承自HashMap，它除了具有HashM印的功能和设置有效期，还可以保存Redirect后的目标路径和通过url传递的参数，这两项内容主要用来从Session保存的多个FlashMap中查找当前请求的FlashMap。

明白上面两点AbsrtactFlashMapManager后就容易分析了，下面先来看保存FlashMap的saveoutputFlashMap方法。

```java
@Override
public final void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response) {
	if (CollectionUtils.isEmpty(flashMap)) {
		return;
	}

	String path = decodeAndNormalizePath(flashMap.getTargetRequestPath(), request);
	flashMap.setTargetRequestPath(path);
	decodeParameters(flashMap.getTargetRequestParams(), request);

	if (logger.isDebugEnabled()) {
		logger.debug("Saving FlashMap=" + flashMap);
	}
	//设置有效期
	flashMap.startExpirationPeriod(getFlashMapTimeout());
	//用于获取互斥变量，是模板方法，如果子类返回值不为null则同步执行，否则不需要同步
	Object mutex = getFlashMapsMutex(request);
	if (mutex != null) {
		synchronized (mutex) {
			//取回保存的List<FlashMap>，如果没获取到则新建一个，然后添加现有的flas hMap
			List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
			allFlashMaps = (allFlashMaps != null ? allFlashMaps : new CopyOnWriteArrayList<FlashMap>());
			allFlashMaps.add(flashMap);
			//将添加完的List<FlashMap>更新到存储介质，是模板方法，子类实现
			updateFlashMaps(allFlashMaps, request, response);
		}
	}
	else {
		List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
		allFlashMaps = (allFlashMaps != null ? allFlashMaps : new LinkedList<FlashMap>());
		allFlashMaps.add(flashMap);
		updateFlashMaps(allFlashMaps, request, response);
	}
}
```
整个过程是这样的，首先对fiashMap中的目标地址和url参数进行编码，编码格式使用当前request获取；其次设置有效期，有效期可以通过flashMapTimeout参数配置，默认值是180秒；然后将fiashMap添加到整体的List中并更新。

最后一步添加到List中并更新的过程首先通过模板方法getFlashMapsMutex获取互斥变量，如果可以获取到则使用同步方式更新，如果获取不到则不使用同步。更新过程是先将原来保存的List获取到，如果原来没有则新建一个，然后将现在的fiashMap添加进去冉调用updateFlashMaps模板方法进行更新。retrieveFlashMaps、getFlashMapsMutex和updateFlashMaps方法都在子类SessionFlashMapManager中实现，代码如下：
```java
protected List<FlashMap> retrieveFlashMaps(HttpServletRequest request) {
	HttpSession session = request.getSession(false);
	return (session != null ? (List<FlashMap>) session.getAttribute(FLASH_MAPS_SESSION_ATTRIBUTE) : null);
}

protected void updateFlashMaps(List<FlashMap> flashMaps, HttpServletRequest request, HttpServletResponse response) {
	WebUtils.setSessionAttribute(request, FLASH_MAPS_SESSION_ATTRIBUTE, (!flashMaps.isEmpty() ? flashMaps : null));
}

protected Object getFlashMapsMutex(HttpServletRequest request) {
	return WebUtils.getSessionMutex(request.getSession());
}
```

它们都是使用WebUtils对Session进行操作的。
下面介绍获取flashMap的retrieveAndUpdate方法，代码如下:
```java
@Override
public final FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response) {
	//从存储介质中获取List<FlashMap>，是模板方法，子类实现
	List allFlashMaps = retrieveFlashMaps(request);
	if (CollectionUtils.isEmpty(allFlashMaps)) {
		return null;
	}

	if (logger.isDebugEnabled()) {
		logger.debug("Retrieved FlashMap(s): " + allFlashMaps);
	}
	//检查过期的fla shMap，并将它们设置到mapsToRemove变量
	List mapsToRemove = getExpiredFlashMaps(allFlashMaps);
	//获取与当前request匹配的fla shMap，并设置到match变量
	FlashMap match = getMatchingFlashMap(allFlashMaps, request);
	//如果有匹配的则将其添加到mapsToRemove．待下面删除
	if (match != null) {
		mapsToRemove.add(match);
	}
	//删除mapsToRemove中保存的变量
	if (!mapsToRemove.isEmpty()) {
		if (logger.isDebugEnabled()) {
			logger.debug("Removing FlashMap(s): " + mapsToRemove);
		}
		Object mutex = getFlashMapsMutex(request);
		if (mutex != null) {
			synchronized (mutex) {
				allFlashMaps = retrieveFlashMaps(request);
				if (allFlashMaps != null) {
					allFlashMaps.removeAll(mapsToRemove);
					updateFlashMaps(allFlashMaps, request, response);
				}
			}
		}
		else {
			allFlashMaps.removeAll(mapsToRemove);
			updateFlashMaps(allFlashMaps, request, response);
		}
	}

	return match;
}
```

整个过程是这样的：首先使用retrieveFlashMaps模板方法获取List；然后检查其中已经过期的FlashMap并保存，检查方法通过保存时设置的过期时间进行判断；接着调用getMatchingFlashMap方法从获取的List中找出和当前request相匹配的FlashMap最后将过期的和与当前请求相匹配的FlashMap从List中删除并更新到Session中，将与当前request匹配的返回。    

查找与当前Request相匹配的FlashMap的getMatchingFlashMap方法，代码如下：
```java
/**
 * Return a FlashMap contained in the given list that matches the request.
 * @return a matching FlashMap or {@code null}
 */
private FlashMap getMatchingFlashMap(List<FlashMap> allMaps, HttpServletRequest request) {
	List<FlashMap> result = new LinkedList<FlashMap>();
	for (FlashMap flashMap : allMaps) {
		if (isFlashMapForRequest(flashMap, request)) {
			result.add(flashMap);
		}
	}
	if (!result.isEmpty()) {
		Collections.sort(result);
		if (logger.isDebugEnabled()) {
			logger.debug("Found matching FlashMap(s): " + result);
		}
		return result.get(0);
	}
	return null;
}
```

这里遍历了所有的FlashMap然后调用isFlashMapForRequest方法实际检查是否匹配，如果匹配则保存到临时变量result，遍历完后可能会有多个匹配的结果，最后将它们排序并返回第一个，isFlashMapForRequest方法代码如下：
```java
/**
 * Whether the given FlashMap matches the current request.
 * Uses the expected request path and query parameters saved in the FlashMap.
 */
protected boolean isFlashMapForRequest(FlashMap flashMap, HttpServletRequest request) {
	String expectedPath = flashMap.getTargetRequestPath();
	if (expectedPath != null) {
		String requestUri = getUrlPathHelper().getOriginatingRequestUri(request);
		if (!requestUri.equals(expectedPath) && !requestUri.equals(expectedPath + "/")) {
			return false;
		}
	}
	MultiValueMap<String, String> targetParams = flashMap.getTargetRequestParams();
	for (String expectedName : targetParams.keySet()) {
		for (String expectedValue : targetParams.get(expectedName)) {
			if (!ObjectUtils.containsElement(request.getParameterValues(expectedName), expectedValue)) {
				return false;
			}
		}
	}
	return true;
}
```

这里的检查方法就是通过FlashMap中保存的目标地址和url参数与Request进行比较的，如果保存的目标地址和Request的url以及url+“／”都不一样则返回false，如果FlashMap中保存的url参数在Request中没有也返回false，如果这两项都合格则返回true。

整个取回FlashMap的过程分析完毕，从这里可以看}}{两件事情：①每次取回FlashMap时都会对所有保存的参数检查是否过期，而且只有在取回的时候才会检查，也就是说虽然设置的过期时间是180秒，但是如果在更长的时间内没有过取回FlashMap的操作，那么即使过期也不会被删除；②使用request匹配FlashMap是在所有的FlashMap中进行的，也就是说其结果可能包含已经过期但还没有被删除的。不过这并不会造成什么影响，因为这里的FlashMap正常情况下只有一个，参数使用完后会自动删除，所以全部遍历也不会有什么问题，另外FlashMap主要用在redirect中，这个过程是不需要人1二参与的，时间一般会很短，而且即使过期了也不会有什么影响。
    
对FlashMapManager酌分析就讲完了。




















  [1]: ./images/1467209826743.jpg "1467209826743.jpg"
