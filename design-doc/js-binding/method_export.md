#Lynx使用Java拓展JS

Lynx 通过前端的方式开发native应用，其中基础的组件都是经过包装后暴露给前端的。一个完整的JS拓展流程可以参考[这里](https://hxxft.github.io/lynx-book/design-doc/js-binding/js-binding-part1.html)。同时Lynx在JavavScript的基础上也提供了Java层的JS拓展。下面就介绍一下Lynx如何将native JAVA方法暴露给JS的。

目前支持的方法暴露有两种场景，一种是JSComponent子类的方法的暴露；另一种则是LynxUI子类方法的暴露。两者的区别在于JSComponent的调用是在JS线程的，而LynxUI的方法则是运行在UI线程。JS线程调用的JAVA方法能够支持有参数返回的类型。UI线程的拓展方法只能是无参数返回的类型。由JS方法调用的JAVA方法入参，目前仅支持int，long，boolean，float，double，string，LynxArray，LynxMap类型。

##1. JSComponent子类的拓展
以PageNavigator为例子，只需要给PageNavigator类加上@JSObject，并将需要暴露的方法加上@JSMethod注解即可。

```
@JSObject
public class PageNavigator extends JSComponent {
    private static final String DEFAULT_SCHEME = "lynx://";

    private Context mContext;

    public PageNavigator(LynxRuntime runtime) {
        super(runtime);
        mContext = runtime.getContext();
    }

    @JSMethod
    public void startNewPage(String page) {
        Bundle bundle = new Bundle();
        bundle.putString("page", page);
        Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(DEFAULT_SCHEME + page));
        intent.putExtras(bundle);
        mContext.startActivity(intent);
    }
}
```

这样在JS端只需要通过PageNavigator.startNewPage("test")，就能调用到Java端的方法了。其中@JSObject支持@JSOBject(name="xxxx")的形式，xxx为JS端的别名，默认情况下为Java的类名。设置了别名的情况下就需要使用xxx别名的方式来调用相应的方法。


##2. LynxUI子类的拓展
与JSComponent类似，LynxUI的子类使用的是@UIComponent和@JSMethod注解项结合的方式。

```
@UIComponent(type = 2)
public class LynxUILabel extends LynxUI<AndroidLabel> {

    ....
    
    @JSMethod
    public void test(){
        Log.e("lynx","test");
    }
}
```
其中UIComponent的type为相应标签的数值，已有的标签可以参考**LynxUIFactory**中的type。LynxUI
的拓展方法是通过LynxUI#setData调用到暴露的方法的。所以暴露的方法需要一个方法id，可以通过@jSMethod(id=666)的方式来设置，默认情况用下的id为方法名的hashcode。另外LynxUI的子类可以通过添加UIComponent注解来新增一个前端标签。如给对应的LynxUI的子类增加@UIComponent(type = 19,tag="viewpage")注解，前端则可以使用新的<viewpage></viewpage>标签。


##3. sdk外module拓展
之前介绍都是在sdk内的拓展，sdk外的拓展方式与上述的介绍基本类似，都是通过apt生成代码来完成拓展的。只需要加额外上一些额外配置即可。


在对应的app/build.gradle里加上

```
android {
  defaultConfig {
    javaCompileOptions {
      annotationProcessorOptions {
        arguments = ['package': 'xxx','':'moduleName' : 'XXXX']
      }
    }
  }
}

dependencies {
    ...
    annotationProcessor project(":apt")
    ...

}

```
根据需要给相应的类加上注解，然后rebuild project，就会根据配置生成对应的xxx.XXXX类。最后在生成LynxApplicationDelegate实例时，将xxx.XXXX和其他的拓展模块通过LynxApplicationDelegate#setExtModules方法统一注入。