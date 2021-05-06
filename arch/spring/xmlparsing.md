# Spring's XML parsing machinism

## 机制链 切入点 MEAT-INF/spring.handlers 、 META-INF/spring.schemas
xml 文件解释两大关键点:schema（节点合法性校验）、handler（节点处理动作）
* schema: 声明namespace以及其下的element
* handler: 声明该namespace下的各个element的处理过程

## spring项目全局搜索spring.handlers

可以找到类DefaultNamespaceHandlerResolver的定义

```java
public class DefaultNamespaceHandlerResolver implements NamespaceHandlerResolver {
	/**
	 * The location to look for the mapping files. Can be present in multiple JAR files.
	 */
	public static final String DEFAULT_HANDLER_MAPPINGS_LOCATION = "META-INF/spring.handlers"; // xml namespace 和 handler的 mapping 的默认配置位置
  
  private final String handlerMappingsLocation;// xml namespace 和 handler的 mapping 的配置位置
  
  public DefaultNamespaceHandlerResolver(ClassLoader classLoader) {
		this(classLoader, DEFAULT_HANDLER_MAPPINGS_LOCATION);
	}
  
  public DefaultNamespaceHandlerResolver(ClassLoader classLoader, String handlerMappingsLocation) {
		Assert.notNull(handlerMappingsLocation, "Handler mappings location must not be null");
		this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
		this.handlerMappingsLocation = handlerMappingsLocation;
	}
  
  // 针对xml namespace "namespaceUri" 查找这个namespace的handler
  public NamespaceHandler resolve(String namespaceUri) {
		Map<String, Object> handlerMappings = getHandlerMappings(); // 这里子过程代处理
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			
      /* ... classloader load class for className ... */
		}
	}
  
  private Map<String, Object> getHandlerMappings() {
		if (this.handlerMappings == null) {
			synchronized (this) {
				if (this.handlerMappings == null) {
					try {
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader); // 从this.handlerMappingsLocation 查找配置
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings); // 配置构造为 map
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return this.handlerMappings;
	}

}
```

## 一个简单的handler实现如下(NamespaceHandlerSupport是NamespaceHandler的一个虚实现类)：
```java
public class MyNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		MyBeanDefinitionParser myBeanDefinitionParser = new MyBeanDefinitionParser();
    
    /** 
     * 将该namespace 下的 "service" element 的handler注册为myBeanDefinitionParser, 
     * 这样xml解释到此namespace的"service"元素就会回调对应的BeanDefinitionParser方法
     */
		registerBeanDefinitionParser("service", myBeanDefinitionParser); 
	}

}
```

## BeanDefinitionParser
实际spring 会调用 BeanDefinitionParser 的 parse(Element element, ParserContext parserContext) 方法。
我们实现 BeanDefinitionParser 时可以继承虚实现类 AbstractBeanDefinitionParser, 实现其 parseInternal(Element element, ParserContext parserContext) 方法

```java
public class MyBeanDefinitionParser extends AbstractBeanDefinitionParser {

	@Override
	protected AbstractBeanDefinition parseInternal(Element rootElmt, ParserContext pc) {
		/**
		 * init all props
		 */
		PropertiesUtils.getPropsConfig(rootElmt);

		// Parse service element, obtain "name" attribute
		String serviceName = rootElmt.getAttribute("name");

		/* ... construct BeanDefinition ...*/
    
    return abstractBeanDefinition;
	}
```

## 上面返回的 AbstractBeanDefinition 最终会被注册到 ParserContext 里，实例化为具体的 bean 并注入 spring IOC, 至此，整个xml解释的机制链已经走完
