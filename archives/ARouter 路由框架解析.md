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
**基本用法**  

写一个 IInterceptor 的实现类注解 @Interceptor 并标明优先级，主要逻辑写在 process()方法中。
```java
@Interceptor(priority = 7)
public class Test1Interceptor implements IInterceptor {
    Context mContext;

    @Override
    public void process(final Postcard postcard, final InterceptorCallback callback) {
        if ("/test/activity4".equals(postcard.getPath())) {
            final AlertDialog.Builder ab = new AlertDialog.Builder(MainActivity.getThis());
            ab.setTitle("温馨提醒");
            ab.setMessage("想要跳转到Test4Activity么？(触发了\"/inter/test1\"拦截器，拦截了本次跳转)");
            ab.setNegativeButton("继续", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onContinue(postcard);
                }
            });
            ab.setNeutralButton("算了", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onInterrupt(null);
                }
            });
            ab.setPositiveButton("加点料", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    postcard.withString("extra", "我是在拦截器中附加的参数");
                    callback.onContinue(postcard);
                }
            });

            MainLooper.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    ab.create().show();
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }

    @Override
    public void init(Context context) {
        mContext = context;
        Log.e("testService", Test1Interceptor.class.getName() + " has init.");
    }
}
```
但是这样用有一个问题，就是判断哪些页面应该拦截哪些不拦截的任务落到了 interceptor 中，这就违背了高内聚（所有关于当前页面的配置写在当前页面中）的原则。所以某个 activity 如何标识出自己应不应该由某个拦截器拦截就成了一个问题。  
这就用到了 @Router 注解中的 extra 参数,extra 是一个 int 值。  
> 而为什么extras这个参数是int呢？其实是因为int本身在Java中是由4个字节实现的，每个字节是8位，所以一共是32个标志位，去除掉符号位还剩下31个，也就是说转化成为二进制之后，一个int中可以配置31个1或者0，而每一个0或者1都可以表示一项配置，这时候只需要从这31个位置中随便挑选出一个表示是否需要登录就可以了，只要将标志位置为1，就可以在刚才声明的拦截器中获取到这个标志位，通过位运算的方式判断目标页面是否需要登录





### 控制反转与服务
（TODO）
### 运行期动态修改路由
只要写一个类实现 PathReplaceService 接口就可以了。
```java
@Route(path = "/service/pathReplace")
public class PathReplaceServiceImpl implements PathReplaceService {
    @Override
    public String forString(String path) {
        if ("/test/activity2".equals(path)) {
            return "/test/activity1";
        } else {
            return path;
        }
    }

    @Override
    public Uri forUri(Uri uri) {
        return uri;
    }

    @Override
    public void init(Context context) {

    }
}
```
forUri()是从外部通过URI的形式跳转到页面的时候会使用到的一个方法，参数中的URI就是原始的URI，而 forString() 是正常跳转中用到的路由地址。  

这个功能的实现就是在 build 一个 Postcard 的时候查找有没有 PathReplaceService 的实现类，如果有就按照实现类中的逻辑改变 path 或 uri。
```java
protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return build(path, extractGroup(path));
        }
    }
```

### 降级
和动态改变路由类似，想要实现降级功能只要写一个 DegradeService 的实现类就可以了。  
```java
@Route(path = "/service/degrade")
public class DegradeServiceImpl implements DegradeService {

    @Override
    public void init(Context context) {

    }

    @Override
    public void onLost(Context context, Postcard postcard) {
        ARouter.getInstance().build("/test/webview")
                .withString("url", "https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.5qkuq2")
                .navigation();
    }
}
``` 
这里是统一跳转到 webview 页面打开某个错误提示页面。  
这个功能的实现是在 navigation 时捕获失败，判断如果没有单独降级的 callback 则采用全局降级的逻辑。  
```java
try {
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
                       if (null != callback) {
                callback.onLost(postcard);
            } else {    // No callback for this invoke, then we use the global degrade service.
                DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
                if (null != degradeService) {
                    degradeService.onLost(context, postcard);
                }
            }

            return null;
        }
```        

---

参考资料 

> [开源最佳实践：Android平台页面路由框架ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.5qkuq2)  开发者演讲及最佳实践