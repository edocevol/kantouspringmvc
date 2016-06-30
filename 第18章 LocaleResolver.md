# 第18章 LocaleResolver
LocaleResolver的作用是使用request解析出Locale，它的继承结构如图18-1所示。
![图18.1 Locale的继承结构图][1]

虽然LocaleResolver的实现类结构看起来比较复杂，但是实现却非常简单。在LocaleResolver的实现类中，AcceptHeaderLocaleResolver直接使用了Header里的"acceptlanguage"值，不可以在程序中修改；FixedLocaleResolver用于解析出同定的Locale，也就是说在创建时就设置好确定的Locale，玄后无法修改；SessionLocaleResolver用于将Locale保存到Session中，可以修改；CookieLocaleResolver用于将Locale保存到Cookie中，可以修改。

另外，从Spring MVC4.0开始，LocaleResolver添加了一个子接口LocaleContextResolver，其中增加了获取和设置LocaleContext的能力，并添加了抽象类AbstractLocaleContextResolver，抽象类添加了对TimeZone也就是时区的支持。LocaleContextResolver接口定义如下：
```java
public interface LocaleContextResolver extends LocaleResolver {

	LocaleContext resolveLocaleContext(HttpServletRequest request);
	
	void setLocaleContext(HttpServletRequest request, HttpServletResponse response, LocaleContext localeContext);

}
```

下面分别看一下这些实现类，先来看AcceptHeaderLocaleResolver，这个类直接实现的LocaleResolver接口，代码非常简单，如下所示：
```java
public class AcceptHeaderLocaleResolver implements LocaleResolver {

	@Override
	public Locale resolveLocale(HttpServletRequest request) {
		return request.getLocale();
	}

	@Override
	public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
		throw new UnsupportedOperationException(
				"Cannot change HTTP accept header - use a different locale resolution strategy");
	}

}
```

再来看LocaleResolver的抽象实现类AbstractLocaleResolver。这个类也很简单，只是添加默认Locale的defaultLocale属性，并添加getter/setter方法，并未实现接口方法，代码如下：

```java
public abstract class AbstractLocaleResolver implements LocaleResolver {

	private Locale defaultLocale;


	public void setDefaultLocale(Locale defaultLocale) {
		this.defaultLocale = defaultLocale;
	}

	protected Locale getDefaultLocale() {
		return this.defaultLocale;
	}

}

```

下面来看Spring MVC新增加的AbstractLocaleContextResolver类，它里面做了两件事：①添加默认时区的属性defaultTimeZone．以及其getter/setter方法；②提供LocaleResolver接口的默认实现，实现方法是使用LocaleContext。代码如下：
```java
public abstract class AbstractLocaleContextResolver extends AbstractLocaleResolver implements LocaleContextResolver {

	private TimeZone defaultTimeZone;


	/**
	 * Set a default TimeZone that this resolver will return if no other time zone found.
	 */
	public void setDefaultTimeZone(TimeZone defaultTimeZone) {
		this.defaultTimeZone = defaultTimeZone;
	}

	/**
	 * Return the default TimeZone that this resolver is supposed to fall back to, if any.
	 */
	public TimeZone getDefaultTimeZone() {
		return this.defaultTimeZone;
	}


	@Override
	public Locale resolveLocale(HttpServletRequest request) {
		return resolveLocaleContext(request).getLocale();
	}

	@Override
	public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
		setLocaleContext(request, response, (locale != null ? new SimpleLocaleContext(locale) : null));
	}

}

```
剩下的三个类孰是实际解析Locale的类了，下面分别看一下。先来看使用固定Locale的FixedLocaleResolver。

```java
public class FixedLocaleResolver extends AbstractLocaleContextResolver {

	/**
	 * Create a default FixedLocaleResolver, exposing a configured default
	 * locale (or the JVM's default locale as fallback).
	 * @see #setDefaultLocale
	 * @see #setDefaultTimeZone
	 */
	public FixedLocaleResolver() {
		setDefaultLocale(Locale.getDefault());
	}

	/**
	 * Create a FixedLocaleResolver that exposes the given locale.
	 * @param locale the locale to expose
	 */
	public FixedLocaleResolver(Locale locale) {
		setDefaultLocale(locale);
	}

	/**
	 * Create a FixedLocaleResolver that exposes the given locale and time zone.
	 * @param locale the locale to expose
	 * @param timeZone the time zone to expose
	 */
	public FixedLocaleResolver(Locale locale, TimeZone timeZone) {
		setDefaultLocale(locale);
		setDefaultTimeZone(timeZone);
	}


	@Override
	public Locale resolveLocale(HttpServletRequest request) {
		Locale locale = getDefaultLocale();
		if (locale == null) {
			locale = Locale.getDefault();
		}
		return locale;
	}

	@Override
	public LocaleContext resolveLocaleContext(HttpServletRequest request) {
		return new TimeZoneAwareLocaleContext() {
			@Override
			public Locale getLocale() {
				return getDefaultLocale();
			}
			@Override
			public TimeZone getTimeZone() {
				return getDefaultTimeZone();
			}
		};
	}

	@Override
	public void setLocaleContext(HttpServletRequest request, HttpServletResponse response, LocaleContext localeContext) {
		throw new UnsupportedOperationException("Cannot change fixed locale - use a different locale resolution strategy");
	}

}

```
FixedLocaleResolver继承自AbstractLocaleContextResolver，也就具有了defaultLocale和defaultTimeZone属性。FixedLocaleResolver在构造方法里对这两个属性进行了设置，它一共有三个构造方法，其中，如果有Locale、TimeZone参数则将其设置为默认值，无参数的构造方法会使用Locale,getDefault()作为默认Locale，这时一般为java虚拟机所在环境的Locale，也可以人为修改。

resolveLocaleContext万法返回新建的TimeZoneAwareLocaleContext匿名类，其中getLocale和getTimeZone使用了defaultLocale和defaultTimeZone。

setLocaleContext方法不支持使用，相应的setLocale方法也不支持使用，因为它在父类中调用了setLocaleContext。也就是说，这里的Locale和TimeZone都是不可以修改的，最初设置的什么就是什么，设置之后就不可以修改。设置Locale和TimeZone的方法是在配置FixedLocaleResolver时设置的，可以通过设置构造方法的参数来设置，也可以直接设置defaultLocale和defaultTimeZone属性。

SessionLocaleResolver和FixedLocaleResolver的实现差不多，只不过把从默认值获取变成从Session中获取，不过如果获取不到还会使用默认值。另外SessionLocaleResolver添加了没置（也就是修改）LocaleContext的支持。解析Locale的resolveLocale方法代码如下：
```java
@Override
public LocaleContext resolveLocaleContext(final HttpServletRequest request) {
	return new TimeZoneAwareLocaleContext() {
		@Override
		public Locale getLocale() {
			Locale locale = (Locale) WebUtils.getSessionAttribute(request, LOCALE_SESSION_ATTRIBUTE_NAME);
			if (locale == null) {
				locale = determineDefaultLocale(request);
			}
			return locale;
		}
		@Override
		public TimeZone getTimeZone() {
			TimeZone timeZone = (TimeZone) WebUtils.getSessionAttribute(request, TIME_ZONE_SESSION_ATTRIBUTE_NAME);
			if (timeZone == null) {
				timeZone = determineDefaultTimeZone(request);
			}
			return timeZone;
		}
	};
}
```
这里首先从Session中获取，如果获取不到会调用determineDefaultLocale万法获取默认值，determineDefaultLocale方法会先获取defaultLocale，如果获取不到会调用Request的getLocale方法获取Request头的Locale.
    
解析LocaleContext的方法除resolveLocale方法外，还可以使用resolveLocaleContext方法，它和前者的解析方式一样，只是在前者的基础上增加了对TimeZone解析的内容，代码如下:
```java
@Override
public LocaleContext resolveLocaleContext(final HttpServletRequest request) {
	return new TimeZoneAwareLocaleContext() {
		@Override
		public Locale getLocale() {
			Locale locale = (Locale) WebUtils.getSessionAttribute(request, LOCALE_SESSION_ATTRIBUTE_NAME);
			if (locale == null) {
				locale = determineDefaultLocale(request);
			}
			return locale;
		}
		@Override
		public TimeZone getTimeZone() {
			TimeZone timeZone = (TimeZone) WebUtils.getSessionAttribute(request, TIME_ZONE_SESSION_ATTRIBUTE_NAME);
			if (timeZone == null) {
				timeZone = determineDefaultTimeZone(request);
			}
			return timeZone;
		}
	};
}

protected TimeZone determineDefaultTimeZone(HttpServletRequest request) {
	return getDefaultTimeZone();
}
```

其中解析TimeZone的过程和解析Locale的过程一样，先从Session中获取，如果获取不到则使用默认值。

下面看一下设置LocaleContext的setLocaleContext方法，代码如下：
```java
@Override
public void setLocaleContext(HttpServletRequest request, HttpServletResponse response, LocaleContext localeContext) {
	Locale locale = null;
	TimeZone timeZone = null;
	if (localeContext != null) {
		locale = localeContext.getLocale();
		if (localeContext instanceof TimeZoneAwareLocaleContext) {
			timeZone = ((TimeZoneAwareLocaleContext) localeContext).getTimeZone();
		}
	}
	WebUtils.setSessionAttribute(request, LOCALE_SESSION_ATTRIBUTE_NAME, locale);
	WebUtils.setSessionAttribute(request, TIME_ZONE_SESSION_ATTRIBUTE_NAME, timeZone);
}
```

这里先从LocaleContext中获取Locale和TimeZone，然后设置到Session中。

SessionLocaleResolver就分析完毕。CookieLocaleResolver和SessionLocaleResolver的处理方式非常相似，只是将Session变成Cookie来保存属性。另外由于CookieLocaleResolver为了处理Cookie方便而继承了CookieGenerator，所以它就不能继承AbstractLocaleContextResolver．不过它仍然提供保存默认Locale和TimeZone的属性defaultLocale和defaultTimeZone，这只是自己定义的，具体代码就不叙述了.












  [1]: ./images/1467208129756.jpg "1467208129756.jpg"
