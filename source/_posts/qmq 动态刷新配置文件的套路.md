
---
title: qmq 动态刷新配置文件的套路
date: 2018-12-11 19:19:00
author: pleuvoir
img: /images/code.jpg
tags:
  - 开源
  - 套路
categories:
  - 技术
---

本文是对 qmq 中动态加载配置类的简要分析以及学习。该功能实现了 `properties` 文件向通用接口的转换，可以方便的获取属性文件中对应的值，并且实现了实时刷新的功能。

## 设计概览

### 接口及现有的实现

`DynamicConfig` 接口是作者对 `properties` 文件对象转换的抽象，提供了方便的方法让我们来获取文件中定义的值。

```
public interface DynamicConfig {
	
    void addListener(Listener listener);

    String getString(String name);

    String getString(String name, String defaultValue);

    int getInt(String name);

    boolean exist(String name);

    Map<String, String> asMap();

	// 省略其他...
}
```

`DynamicConfigFactory` 是一个动态配置创建工厂方法接口，目前该项目中只实现了本地文件的创建，后期可能会加入远程文件的读取创建实现。

```java
public interface DynamicConfigFactory {
	DynamicConfig create(String name, boolean failOnNotExist);
}
```

`DynamicConfigLoader` 动态配置加载器，该类即是程序的入口。通过调用 `load` 方法，即可获得 `DynamicConfig`

```java
public final class DynamicConfigLoader {

    private static final DynamicConfigFactory FACTORY;

    static {
        ServiceLoader<DynamicConfigFactory> factories = ServiceLoader.load(DynamicConfigFactory.class);
        DynamicConfigFactory instance = null;
        for (DynamicConfigFactory factory : factories) {
            instance = factory;
            break;
        }

        FACTORY = instance;
    }

    private DynamicConfigLoader() {
    }

    public static DynamicConfig load(final String name) {
        return load(name, true);
    }

    public static DynamicConfig load(final String name, final boolean failOnNotExist) {
        return FACTORY.create(name, failOnNotExist);
    }
}
```

通过工厂方法创建一个动态配置。目前只有本地文件读取创建，这个类由于是名义上的单例，所以成员变量必须考虑为线程安全的：

```java
public class LocalDynamicConfigFactory implements DynamicConfigFactory {
	private final ConfigWatcher watcher = new ConfigWatcher();	// 配置文件观察者，当文件发生变动时会重新加载文件，并通知监听器
	private final ConcurrentMap<String, LocalDynamicConfig> configs = new ConcurrentHashMap<>();	// 每一个文件都对应一个 LocalDynamicConfig 类

	@Override
	public DynamicConfig create(final String name, final boolean failOnNotExist) {
		if (configs.containsKey(name)) {
			return configs.get(name);
		}

		return doCreate(name, failOnNotExist);
	}

	private LocalDynamicConfig doCreate(final String name, final boolean failOnNotExist) {
		// putIfAbsent 如果不存在就 put 并且不返回值，否则就 get，正常情况第一次放返回为 null
		final LocalDynamicConfig prev = configs.putIfAbsent(name, new LocalDynamicConfig(name, failOnNotExist));
		// 不管之前有没有，反正现在一定是可以拿到的
		final LocalDynamicConfig config = configs.get(name);
		// 如果之前的为空，则代表是并发第一次操作，加入监听，触发事件
		if (prev == null) {
			watcher.addWatch(config);
			config.onConfigModified();
		}
		return config;
	}
}
```

这里为什么非要 `putIfAbsent` 原因就在于它类似于 `redis` 的 `setNX`，可以用来判断是否占位成功。如果为空则是我们预期的，可以通知监听器（如果有的话），开启监听线程。否则会出现，并发时加入多次的问题。


`ConfigWatcher` 配置文件观察者，使用单线程池轮询所有文件 `lastModified` 变化，当侦听到变化时会更新对象中值，用于下次做比较。同时调用 `LocalDynamicConfig` 的 `onConfigModified` 方法，该方法会执行实际的文件加载，以及通知所有的自定义的监听器。同样该类也是理论单例，所以设计时也是线程安全的。


```java
	private final CopyOnWriteArrayList<Watch> watches; // 线程安全的 list，并且适合多读少些，显然这里读操作多，基本上 addWatcher 只会在启动时调用
	private final ScheduledExecutorService watcherExecutor;

	ConfigWatcher() {
		this.watches = new CopyOnWriteArrayList<>();
		this.watcherExecutor = Executors.newSingleThreadScheduledExecutor();	// 同时只有一个线程执行，可以保证线程执行的顺序和投入线程池的顺序一致（此处并未用到此特性）
		start();
	}

	private void start() {
		watcherExecutor.scheduleWithFixedDelay(new Runnable() {
			@Override
			public void run() {
				checkAllWatches();
			}
		}, 10, 10, TimeUnit.SECONDS); 	// 10 秒执行一次文件检查
	}

	// 如果有多个线程在检测文件变化，那么同一个对象的修改时间一定是要及时刷新的
	private void checkWatch(final Watch watch) {
		final LocalDynamicConfig config = watch.getConfig();
		final long lastModified = config.getLastModified();
		if (lastModified == watch.getLastModified()) {
			return;
		}
		
		watch.setLastModified(lastModified);
		config.onConfigModified();
	}

	private static final class Watch {
		private final LocalDynamicConfig config;
		private volatile long lastModified;		// 注意这里是 volatile

		private Watch(final LocalDynamicConfig config) {
			this.config = config;
		}
		// 省略 ..
	}
```


`LocalDynamicConfig` 是动态接口的实现，主要功能是完成文件的加载，以及

```java

	// 这些都是 volatile 原因是会共享
    private volatile File file;
    private volatile boolean loaded = false;
    private volatile Map<String, String> config;

	// 获取文件，目的是为了获取文件的上次修改时间
   private File getFileByName(final String name) {
        try {
            final URL res = this.getClass().getClassLoader().getResource(name);
            if (res == null) {
                return null;
            }
            return Paths.get(res.toURI()).toFile();
        } catch (URISyntaxException e) {
            throw new RuntimeException("load config file failed", e);
        }
    }

  // 文件转化为 Properties
  private void loadConfig() {
        try {
            final Properties p = new Properties();
            try (Reader reader = new BufferedReader(new FileReader(file))) {
                p.load(reader);
            }
			// 转为 map，每次重新载入时都会 new 新的 map
            final Map<String, String> map = new LinkedHashMap<>(p.size());
            for (String key : p.stringPropertyNames()) {
                map.put(key, tryTrim(p.getProperty(key)));
            }
            config = Collections.unmodifiableMap(map);	// 再走一步，转为不可修改的，防止小白修改了对象属性
        } catch (IOException e) {
        	e.printStackTrace();
        }
    }

  synchronized void onConfigModified() {
        if (file == null) {
            return;
        }

        loadConfig();
        executeListeners(); // 其实就是循环调用 listener.onLoad(this); 监听器载入本类，嗯很合理。这个接口需要我们自己实现。
        loaded = true;
    }
```

### 如何使用

```java
// DynamicConfig 是一个通用的接口，可以方便的获取值
DynamicConfig config = DynamicConfigLoader.load("config/appconfig.properties");
config.addListener(new Listener() {
	@Override
	public void onLoad(DynamicConfig config) {
		System.out.println("当文件被修改，被检测到后会执行我被重新载入了。。");
	}
});

// 可以修改文件试试，会发现动态刷新了
while(true) {
	TimeUnit.SECONDS.sleep(1);
	System.out.println(config.getString("app.name")); 
}
```

## 相关技术

### SPI

```java
private static final DynamicConfigFactory FACTORY;

static {
    ServiceLoader<DynamicConfigFactory> factories = ServiceLoader.load(DynamicConfigFactory.class);
    DynamicConfigFactory instance = null;
    for (DynamicConfigFactory factory : factories) {
        instance = factory;		//第一个即为实现，忽略后面的
        break;
    }

    FACTORY = instance;
}
```

此项技术使用了 java 的 spi，约定如下，在 `resource` 目录下新建 `META-INF\services` 目录，新建文件名称为接口的权限定类名（注意此文件是无格式的），在此例中为 `io.github.pleuvoir.qmq.DynamicConfigFactory`，文件内容为该接口的实现类，这里是 `io.github.pleuvoir.qmq.local.LocalDynamicConfigFactory`。这样当如上的代码加载时，`DynamicConfigFactory` 即指向了文件中配置的实现，同时实现类会被初始化。

### j.u.c

- `CopyOnWriteArrayList`
- `Executors.newSingleThreadScheduledExecutor`
- `scheduleWithFixedDelay`
- `putIfAbsent`
- `volatile`