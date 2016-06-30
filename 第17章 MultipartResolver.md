# 第17章 MultipartResolver
MultipartResolver用于处理上传请求，有两个实现类：StandardS ervletMultipartResolver和CommonsMultipartResolver。前者使用了Servlet3.0标准的上传方式，后者使用了Apache的commons-fileupload.

## 17.1StandardServletMultipartResolver
StandardS ervletMultipartResolver使用了Servlet3.0标准的上传方式，在Servlet3.0中上传文件非常简单，只需要调用request的getParts方法就可以获取所有上传的文件。如果想单独获取某个文件可以使用request．getPart( fileName)，获取到Part后直接调用它到write(saveFileName)方法就可以将文件保存为以saveFileName为文件名的文件，也可以调用getlnputStream获取InputStream，如果想要使用这种上传方式还需要在配置上传文件的Servlet时添加multipart-config属性，例如，我们使用的Spring MVC中所有的请求都在DispatcherServlet这个Servlet中，所以可以给它配置上multipart-config．如下所示：
```xml
<!--web.xml-->
<servlet>
	<servlet-name>let' sGo</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<load-on-startup>l</load-on-startup>
	<rnultipart-config>
		<location>/tmp</location>
		<max-file-size>-l</max-file-size>
		<max-request-size>-l</max-request-size>
		<file-size-threshold>0 </file-size-threshold>
	</multipart-config>
</servlet>
```
multipart-config有4个子属性可以配置，下面分别介绍：

>1. location：设置上传文件存放的根目录，也就是调用Part的write(saveFileName)方法保存文件的根目录。如果saveFileName带了绝对路径，将以saveFileName所带路径为准。
>2. max-file-size：设置单个上传文件的最大值，默认值为-1，表示无限制。
>3. max-request-size：设置一次上传的所有文件总和的最大值，默认值为-l，表示无限制。
>4. file-size-threshold：设置不写入硬盘的最大数据量，默认值为0，表示所有上传的文件都会作为一个临时文件写入硬盘。

下面看一下StandardServletMultipartResolver，它的代码非常简单。
```JAVA
public class StandardServletMultipartResolver implements MultipartResolver {

	private boolean resolveLazily = false;

	@Override
	public boolean isMultipart(HttpServletRequest request) {
		// Same check as in Commons FileUpload...
		if (!"post".equals(request.getMethod().toLowerCase())) {
			return false;
		}
		String contentType = request.getContentType();
		return (contentType != null && contentType.toLowerCase().startsWith("multipart/"));
	}

	@Override
	public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
		return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
	}

	@Override
	public void cleanupMultipart(MultipartHttpServletRequest request) {
		// To be on the safe side: explicitly delete the parts,
		// but only actual file parts (for Resin compatibility)
		try {
			for (Part part : request.getParts()) {
				if (request.getFile(part.getName()) != null) {
					part.delete();
				}
			}
		}
		catch (Exception ex) {
			LogFactory.getLog(getClass()).warn("Failed to perform cleanup of multipart items", ex);
		}
	}

}

```
如何判断是不是上传请求呢？在isMultipart方法中首先判断是不是post请求，如果是则再检查contentType是不是以"multipar"开头，如果也是则认为是上传请求。resolveMultipart方法直接将当前请求封装成StandardMultipartHttpServletRequest并返回。cleanupMultipart方法删除了缓存。
下面来看一下StandardMultipartHttpServletRequest。
```JAVA
private void parseRequest(HttpServletRequest request) {
	try {
		Collection<Part> parts = request.getParts();
		this.multipartParameterNames = new LinkedHashSet<String>(parts.size());
		MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<String, MultipartFile>(parts.size());
		for (Part part : parts) {
			String filename = extractFilename(part.getHeader(CONTENT_DISPOSITION));
			if (filename != null) {
				files.add(part.getName(), new StandardMultipartFile(part, filename));
			}
			else {
				this.multipartParameterNames.add(part.getName());
			}
		}
		setMultipartFiles(files);
	}
	catch (Exception ex) {
		throw new MultipartException("Could not parse multipart servlet request", ex);
	}
}
```
可以看到，它的大概思路就是通过request的getParts方法获取所有Part，然后使用它们创建出File并保存到对应的属性，以便在赴理器中可以直接调用。
## 17.2  CommonsMultipartResolver
CommonsMultipartResolver使用了commons-fileupload来完成具体的上传操作。
在CommonsMultipartResolver中，判断是不是上传请求的isMultipart，这将交给commons-fileupload的ServletFileUpload类完成，代码如下：
```JAVA
@Override
public boolean isMultipart(HttpServletRequest request) {
	return (request != null && ServletFileUpload.isMultipartContent(request));
}
```
CommonsMultipartResolver中实际处理request的方法是resolveMultipart，代码如下:
```JAVA
@Override
public MultipartHttpServletRequest resolveMultipart(final HttpServletRequest request) throws MultipartException {
	Assert.notNull(request, "Request must not be null");
	if (this.resolveLazily) {
		return new DefaultMultipartHttpServletRequest(request) {
			@Override
			protected void initializeMultipart() {
				MultipartParsingResult parsingResult = parseRequest(request);
				setMultipartFiles(parsingResult.getMultipartFiles());
				setMultipartParameters(parsingResult.getMultipartParameters());
				setMultipartParameterContentTypes(parsingResult.getMultipartParameterContentTypes());
			}
		};
	}
	else {
		MultipartParsingResult parsingResult = parseRequest(request);
		return new DefaultMultipartHttpServletRequest(request, parsingResult.getMultipartFiles(),
				parsingResult.getMultipartParameters(), parsingResult.getMultipartParameterContentTypes());
	}
}
```
它根据不同的resolveLazily配置使用了两种不同的方法，不过都是将Request转换为DefaultMultipartH ttp ServletRequest类型，而且都使用parseRequest方法进行处理。    

如果resolveLazily为true，则会将parsingResult方法放在DefaultMultipartHttpServletRequest类重写的initializeMultipart方法中，initializeMultipart方法只有在调用相应的get方法(getMultipartFiles、getMultipartParameters或getMultipartParameterContentTypes)时才会被调用。

如果resolveLazily为false，则将会先调用parseRequest万法来处理request，然后将处理的结果传人De faultM ultipartH ttpServletRequest。

parseRequest方法是使用commons-fileupload中的FileUpload组件解析出fileltems，然后再调用parseFileltems方法将fileltems分为参数和文件两类，并设置到三个Map中，三个Map分别用于保存参数、参数的ContentType和上传的文件，代码如下：
```JAVA
protected MultipartParsingResult parseRequest(HttpServletRequest request) throws MultipartException {
	String encoding = determineEncoding(request);
	FileUpload fileUpload = prepareFileUpload(encoding);
	try {
		List<FileItem> fileItems = ((ServletFileUpload) fileUpload).parseRequest(request);
		return parseFileItems(fileItems, encoding);
	}
	catch (FileUploadBase.SizeLimitExceededException ex) {
		throw new MaxUploadSizeExceededException(fileUpload.getSizeMax(), ex);
	}
	catch (FileUploadException ex) {
		throw new MultipartException("Could not parse multipart servlet request", ex);
	}
}

protected MultipartParsingResult parseFileItems(List<FileItem> fileItems, String encoding) {
	MultiValueMap<String, MultipartFile> multipartFiles = new LinkedMultiValueMap<String, MultipartFile>();
	Map<String, String[]> multipartParameters = new HashMap<String, String[]>();
	Map<String, String> multipartParameterContentTypes = new HashMap<String, String>();

	//将fileItems分为文件和参数两类 ,并设置到对应的Map
	for (FileItem fileItem : fileItems) {
		//如果是参数类型
		if (fileItem.isFormField()) {
			String value;
			String partEncoding = determineEncoding(fileItem.getContentType(), encoding);
			if (partEncoding != null) {
				try {
					value = fileItem.getString(partEncoding);
				}
				catch (UnsupportedEncodingException ex) {
					if (logger.isWarnEnabled()) {
						logger.warn("Could not decode multipart item '" + fileItem.getFieldName() +
								"' with encoding '" + partEncoding + "': using platform default");
					}
					value = fileItem.getString();
				}
			}
			else {
				value = fileItem.getString();
			}
			String[] curParam = multipartParameters.get(fileItem.getFieldName());
			if (curParam == null) {
				// 单个参数
				multipartParameters.put(fileItem.getFieldName(), new String[] {value});
			}
			else {
				// 数组参数
				String[] newParam = StringUtils.addStringToArray(curParam, value);
				multipartParameters.put(fileItem.getFieldName(), newParam);
			}
			multipartParameterContentTypes.put(fileItem.getFieldName(), fileItem.getContentType());
		}
		else {
			// 如果是文件类型
			CommonsMultipartFile file = new CommonsMultipartFile(fileItem);
			multipartFiles.add(file.getName(), file);
			if (logger.isDebugEnabled()) {
				logger.debug("Found multipart file [" + file.getName() + "] of size " + file.getSize() +
						" bytes with original filename [" + file.getOriginalFilename() + "], stored " +
						file.getStorageDescription());
			}
		}
	}
	return new MultipartParsingResult(multipartFiles, multipartParameters, multipartParameterContentTypes);
}
```
通过上面这些步骤就将request转换为DefaultMultipartHttpServletRequest类型，并将解析出的三类参数设置到对应的Map。最后来看一下清理上传缓存的方法。代码如下：
```JAVA
public void cleanupMultipart(MultipartHttpServletRequest request) {
	if (request != null) {
		try {
			cleanupFileItems(request.getMultiFileMap());
		}
		catch (Throwable ex) {
			logger.warn("Failed to perform multipart cleanup for servlet request", ex);
		}
	}
}

protected void cleanupFileItems(MultiValueMap<String, MultipartFile> multipartFiles) {
	for (List<MultipartFile> files : multipartFiles.values()) {
		for (MultipartFile file : files) {
			if (file instanceof CommonsMultipartFile) {
				CommonsMultipartFile cmf = (CommonsMultipartFile) file;
				cmf.getFileItem().delete();
				if (logger.isDebugEnabled()) {
					logger.debug("Cleaning up multipart file [" + cmf.getName() + "] with original filename [" +
							cmf.getOriginalFilename() + "], stored " + cmf.getStorageDescription());
				}
			}
		}
	}
}
```
清理缓存的cleanupMultipart方法获取到request里的multiFileMap（保存着上传文件）后将具体清理—亡作交给了cleanupFileltems方法，后者遍历传人的文件并将它们删除，因为删除的是MultNalueMap类型，所以使用了两层循环来操作。

## 17.3小结
本章分析了MultipartResolver的两个实现类，MultipartResolver的作用就是将上传请求包装成可以直接获取File的Request，从而方便操作。所以MultipartResolver的重点是从Request中解析出上传的文件并设置到相应上传类型的Request中。具体解析上传文件的过程使用Servlet的标准上传和Apache的commons-fileupload两种方式来完成，它们所对应的Request分别为StandardMultipartHttpServletRequest和De faultMultipartHttp ServletRequest类型。

