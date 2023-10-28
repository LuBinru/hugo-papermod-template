---
author: "BeiYu"
title: "MyBatisPlus分页不生效后的探索"
date: "2023-07-28"
description: "研究一下META-INF/spring.factories"
tags: ["Spring"]
ShowToc: false
math: true
ShowBreadCrumbs: false
---

# MyBatisPlus分页不生效后的探索

今天我在使用MyBatisPlus的IPage接口进行分页查询时，遇到了一个问题：分页并没有生效，结果直接返回了整个数据集。我查阅了官方文档，但没有找到我这几行代码的问题所在。于是我开始搜索关于"MyBatisPlus分页不生效"的原因和解决办法，但大多数结果都在讨论"未配置分页拦截器"这个问题。

经过测试，我发现分页拦截器根本没有启动。

在浏览了几页的搜索结果后，我没有找到解决方案。

实际上，问题出在META-INF目录的位置上。META-INF目录需要放置在resources根目录下，否则无法被正确扫描到。META-INF没有被扫描，在这之前没碰到过的一个小问题。

META-INF目录用于存放元数据信息，例如包和扩展的配置数据。

与分页拦截器是否启动直接相关的是spring.factories文件。

spring.factories的作用是自动加载Spring扩展的实现类。它需要以properties格式（键值对）进行配置。这个机制提供了一种将外部包中的类注入到Spring容器中的方式。换句话说，如果Spring要加载根目录以外的Bean，例如Maven依赖中的Bean，就需要使用spring.factories。

这是一种解耦的扩展机制。

基本原理是在spring-core包中的SpringFactoriesLoader类中，它实现了对META-INF/spring.factories文件的检索，并根据配置信息实例化这些功能类，最后将它们注册到Spring容器中。

相关代码如下：

```java
public final class SpringFactoriesLoader {
    //文件路径
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    ...
    /**
	* 根据给定的类型返回一个包含这些实现类对象的列表
	*/
	public static <T> List<T> loadFactories(Class<T> factoryType, @Nullable ClassLoader classLoader) {
        Assert.notNull(factoryType, "'factoryType' must not be null");
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }

        List<String> factoryImplementationNames = loadFactoryNames(factoryType, classLoaderToUse);
        if (logger.isTraceEnabled()) {
            logger.trace("Loaded [" + factoryType.getName() + "] names: " + factoryImplementationNames);
        }

        List<T> result = new ArrayList(factoryImplementationNames.size());
        Iterator var5 = factoryImplementationNames.iterator();

        while(var5.hasNext()) {
            String factoryImplementationName = (String)var5.next();
            result.add(instantiateFactory(factoryImplementationName, factoryType, classLoaderToUse));
        }

        AnnotationAwareOrderComparator.sort(result);
        return result;
    }
    /**
	* 根据给定的类型加载类路径的全限定名
	*/
    public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }

        String factoryTypeName = factoryType.getName();
        return (List)loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
    }
    /**
	* 根据类加载器，加载Factory实现类，返回一个包含这些实现类的Map对象
	*/
	private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = (Map)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            Map<String, List<String>> result = new HashMap();

            try {
                Enumeration<URL> urls = classLoader.getResources("META-INF/spring.factories");

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Map.Entry<?, ?> entry = (Map.Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        String[] var10 = factoryImplementationNames;
                        int var11 = factoryImplementationNames.length;

                        for(int var12 = 0; var12 < var11; ++var12) {
                            String factoryImplementationName = var10[var12];
                            ((List)result.computeIfAbsent(factoryTypeName, (key) -> {
                                return new ArrayList();
                            })).add(factoryImplementationName.trim());
                        }
                    }
                }

                result.replaceAll((factoryType, implementations) -> {
                    return (List)implementations.stream().distinct().collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
                });
                cache.put(classLoader, result);
                return result;
            } catch (IOException var14) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var14);
            }
        }
    }
```