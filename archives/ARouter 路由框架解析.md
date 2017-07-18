---

[TOC]

---

## 为什么要用路由框架？

 - 开发与协作  
 - 组件化  
 - Native&H5  

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%904.png)

---
 
## ARouter 介绍
ARouter是阿里巴巴开源的Android平台中对页面、服务提供路由功能的中间件，提倡的是简单且够用。

### ARouter 的7个优势

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%901.png)

### Arouter 怎么用
略

---

## ARouter 的技术方案

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%902.png)

整个代码结构分为3个 module  

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%903.png)


  - annotation 定义了一些注解，通过注解处理器自动生成路由表等信息
  - compiler 是注解处理器
    - Router Processor 用于处理路由信息的注解，生成路由表的信息。  
    - Interceptor Processor 用于处理拦截器相关的信息。  
    - Autowire Processor 用于处理跳转目标对象中自动绑定的变量，当URL携带参数跳转时候，可以直接对变量传值。  
  - api
    - Launcher 是对外暴露的接口。 
    - Service 是对外跳转过程中，系统定义的一些Service能力，如拦截，降级，替换跳转路径等。  
    - Ware House主要存储了ARouter在运行期间加载的一些配置文件以及映射关系。
    - Logistics Center 主要加载路由表，组装参数。
    - 还有一个图上没有画出的重要的类 Postcard，是承载路由跳转信息的容器。
  
  
### 页面注册：注解&注解处理器 

关于 [注解处理器](https://github.com/mmga/AnnotationProcessorDemo)  

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%905.png)

通过注解处理器生成的类是这样的： 

Root  

```java
public class ARouter$$Root$$app implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("service", ARouter$$Group$$service.class);
    routes.put("test", ARouter$$Group$$test.class);
  }
}

```

Group test  
```java
public class ARouter$$Group$$test implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/test/activity1", RouteMeta.build(RouteType.ACTIVITY, Test1Activity.class, "/test/activity1", "test", new java.util.HashMap<String, Integer>(){{put("pac", 9); put("obj", 10); put("name", 8); put("boy", 0); put("age", 3); put("url", 8); }}, -1, -2147483648));
    atlas.put("/test/activity2", RouteMeta.build(RouteType.ACTIVITY, Test2Activity.class, "/test/activity2", "test", new java.util.HashMap<String, Integer>(){{put("key1", 8); }}, -1, -2147483648));
    atlas.put("/test/activity3", RouteMeta.build(RouteType.ACTIVITY, Test3Activity.class, "/test/activity3", "test", new java.util.HashMap<String, Integer>(){{put("name", 8); put("boy", 0); put("age", 3); }}, -1, -2147483648));
    atlas.put("/test/activity4", RouteMeta.build(RouteType.ACTIVITY, Test4Activity.class, "/test/activity4", "test", null, -1, -2147483648));
    atlas.put("/test/fragment", RouteMeta.build(RouteType.FRAGMENT, BlankFragment.class, "/test/fragment", "test", null, -1, -2147483648));
    atlas.put("/test/webview", RouteMeta.build(RouteType.ACTIVITY, TestWebview.class, "/test/webview", "test", null, -1, -2147483648));
  }
}
```

### 加载：分组管理，按需加载

ARouter 初始化时调用 ARouter.init(), 实际是调用了 LogisticsCenter 的 init()。

```java
// These class was generate by arouter-compiler.
            List<String> classFileNames = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);

            //
            for (String className : classFileNames) {
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // This one of root elements, load root.
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // Load interceptorMeta
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // Load providerIndex
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }
```

先通过包名去扫描注解编译器生成的映射文件，拿到一个类名对应的集合，然后通过反射的方式，实例化接口，通过loadinfo方法，将映射文件中，定义的路由关系信息，载入到warehouse中数据结构中，完成了在内存中缓存一份的操作。  
为什么只加载了 root 的信息而不加载 group 呢？这就是所说的分组管理，按需加载。按照一定的业务规则或者命名规范把一部分页面聚合成一个分组，每个分组其实就相当于路径中的第一段。当某一个分组下的某一个页面第一次被访问的时候，整个分组的全部页面都会被加载进去

![](http://osx5yzuma.bkt.clouddn.com/image/ARouter%E8%A7%A3%E6%9E%906.png)

下面这段是开始跳转时的逻辑 (去掉了 log 和 try-catch)
```java
RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            
                // Load route and cache it into memory, then delete from metas.
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                completion(postcard);   // Reload
            }
        } else {
        //...
        //把数据装载到 postcard 中
        }
        
```     

### 跳转流程
以最简单的跳转为例    
```java
ARouter.getInstance()
        .build("/test/activity2")
        .navigation();
```
 build时创建了一个 Postcard 对象   
 
```java 
 public final class Postcard extends RouteMeta {
    // Base
    private Uri uri;
    private Object tag;             // A tag prepare for some thing wrong.
    private Bundle mBundle;         // Data to transform
    private int flags = -1;         // Flags of route
    private int timeout = 300;      // Navigation timeout, TimeUnit.Second
    private IProvider provider;     // It will be set value, if this postcard was provider.
    private boolean greenChannel;
    private SerializationService serializationService;
    // Animation
    private Bundle optionsCompat;    // The transition animation of activity
    private int enterAnim;
    private int exitAnim;
    //...
    }
```

他继承自 RouteMeta  
```java
public class RouteMeta {
    private RouteType type;         // Type of route
    private Element rawType;        // Raw type of route
    private Class<?> destination;   // Destination
    private String path;            // Path of route
    private String group;           // Group of route
    private int priority = -1;      // The smaller the number, the higher the priority
    private int extra;              // Extra data
    private Map<String, Integer> paramsType;  // Param type
    
    //...
```

在 navigation() 时通过 LogisticsCenter 的 completion 方法填充跳转的参数  

```java
postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();

                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }

                    // Save params name which need autoinject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }

                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }
```

最终在 _ARouter 的 _navigation() 方法中使用原生的跳转流程进行跳转 
```java
case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Navigation in main looper.
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        if (requestCode > 0) {  // Need start for result
                            ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
                        } else {
                            ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
                        }

                        if ((0 != postcard.getEnterAnim() || 0 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
                            ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
                        }

                        if (null != callback) { // Navigation over.
                            callback.onArrival(postcard);
                        }
                    }
                });

                break;
```

### 拦截器
（未完待续）
### 控制反转与服务
（未完待续）
### 运行期动态修改路由
（未完待续）
### 降级
（未完待续）

---

参考资料 

> [开源最佳实践：Android平台页面路由框架ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.5qkuq2)  开发者演讲及最佳实践