#基本介绍
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/logo_butterknife.png" width="188" alt="ButterKnife Logo"/></div>

__ButterKnife__ 是一个注解框架，项目地址: [ButterKnife](http://jakewharton.github.io/butterknife/) 。 简单使用🌰如下:

```java
class ExampleActivity extends Activity {
	@Bind(R.id.title) TextView title;
	@Bind(R.id.subtitle) TextView subtitle;
	@Bind(R.id.footer) TextView footer;

	@Override public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.simple_activity);
		ButterKnife.bind(this);
		// TODO Use fields...
	}
	
	@OnClick(R.id.footer)
	public void submit() {
  		// TODO submit data to server...
	}
}
```
以上这段代码可以代替以下这段代码:

```java
public void bind(ExampleActivity activity) {
	activity.subtitle = (android.widget.TextView) activity.findViewById(2130968578);
	activity.footer = (android.widget.TextView) activity.findViewById(2130968579);
	activity.title = (android.widget.TextView) activity.findViewById(2130968577);
	
	activity.footer..setOnClickListener(new View.OnClickListener() {
		@Override
		public void onClick(View v) {
			submit();
		}
	}
}

```
我们看到，使用ButterKnife后可以不用再去写`findViewById`、`setOnClickListener`这样重复繁琐代码，通过`@Bind`、`@OnClick`注解就可以达到效果，代码简洁不少。

更多的例子和功能可以参见[官网示例](http://jakewharton.github.io/butterknife/)。

本文重点分析两个内容：

1. ButterKnife的注解定义方式；
2. ButterKnife是如何运用编译时注解来提高效率的（在Android上，反射的效率较低）； 

#知识储备
研究ButterKnife要具备一定的注解知识，CodeKK上有篇文章推荐阅读: [《公共技术点之 Java 注解 Annotation》](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E6%B3%A8%E8%A7%A3%20Annotation)。

关于运行时注解的开发，可以看这篇文章:[Java注解处理器](http://www.race604.com/annotation-processing/)。ButterKnife使用的是`com.google.auto.service:auto-service`包，通过其`AutoService`注解来实现编译时注解的开发。

另外也可以关注一下这个项目: [android-apt](https://bitbucket.org/hvisser/android-apt)。

#项目结构
如下是ButterKnife的项目结构:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/ButterKnife项目.png" width="250" alt="ButterKnife项目"/></div>

ButterKnife项目分为四个Module: Module butterknife-annotations中定义的是ButterKnife所支持的注解，butterknife-compiler中主要是编译时注解Processor。butterknife-sample则是一个使用例子。

下面我们就来逐步分析整个注解框架的实现。

#注解定义解析
ButterKnife支持的注解主要分为四类:

1. 资源绑定。通过Id引用Array、Bitmap、Bool、Color、Dimen、Drawable、Int、String等几类资源；
2. 事件绑定。包括onCheckedChanged、onClick、onItemClick、onItemLongClick等几种事件监听；
3. 视图绑定。通过id实例化xml中的View；
4. Mark注解。Unbinder和Optional；

##资源绑定注解定义
这类注解都以`BindXXX`的形式来命名，由于资源注解都只需要指定资源的id，因此定义的形式非常一致。下面以String资源为例，展示一下定义内容:

```java
/**
 * Bind a field to the specified string resource ID.
 * <pre><code>
 * {@literal @}BindString(R.string.username_error) String usernameErrorText;
 * </code></pre>
 */
@Retention(CLASS) @Target(FIELD)
public @interface BindString {
	/** String resource ID to which the field will be bound. */
	@StringRes int value();
}
```
这里还用到了Android提供的资源注解，以保证返回的Id符合特定的资源。注解是编译时注解(CLASS)，适用于属性(FIELD)。

##事件绑定注解
事件绑定注解比较复杂，主要是因为以下三个方面:

1. 为View添加监听的方法是不一样的，比如`setOnClickListener`、`addTextChangedListener`、`setOnFocusChangeListener`等，方法名字本身不一致，且无规律可言；
2. 这些方法只能绑定在特定的View上面；
3. 监听需要实现的方法有很大的差别，具体可以参考`OnLongClickListener`和`TextWatcher`需要实现的方法的区别；

因此在ButterKnife内部专门为此建立两个注解:`ListenerClass`和`ListenerMethod`（可以认为是ButterKnife支持事件绑定注解的元注解）。

###事件元注解
首先我们来看一下`ListenerMethod`:

```java
@Retention(RUNTIME) @Target(FIELD)
public @interface ListenerMethod {
	/** Name of the listener method for which this annotation applies. */
	String name();

	/** List of method parameters. If the type is not a primitive it must be fully-qualified. */
	String[] parameters() default { };

	/** Primitive or fully-qualified return type of the listener method. May also be {@code void}. */
	String returnType() default "void";

	/** If {@link #returnType()} is not {@code void} this value is returned when no binding exists. */
	String defaultReturn() default "null";
}
```
这个注解主要是定义一个方法的签名(方法名字、方法的参数、方法的返回类型)以及默认返回值。

再来看`ListenerClass`:

```java
@Retention(RUNTIME) @Target(ANNOTATION_TYPE)
public @interface ListenerClass {
	String targetType();

	/** Name of the setter method on the {@link #targetType() target type} for the listener. */
	String setter();

	/** Fully-qualified class name of the listener type. */
	String type();

	/** Enum which declares the listener callback methods. Mutually exclusive to {@link #method()}. */
	Class<? extends Enum<?>> callbacks() default NONE.class;

	/**
    * Method data for single-method listener callbacks. Mutually exclusive with {@link #callbacks()}
    * and an error to specify more than one value.
    */
	ListenerMethod[] method() default { };

	/** Default value for {@link #callbacks()}. */
	enum NONE { }
}
```
这个注解主要定义如下内容:

1. __setter():__ 监听的设置方法全称；
2. __type():__ 监听类的类名全称；
3. __method():__ 监听类中有哪些方法；
4. __callback():__ 当一个监听有多个回调时，指定当前方法在哪个回调中调用，后面有详细阐述；

注意，这两个注解都是用来修饰注解的：可以看到`@Target`都是`ANNOTATION_TYPE`类型的。

###事件注解
由于事件的复杂性，注解定义本身差距比较大，先看一个简单的:

```java
@Target(METHOD)
@Retention(CLASS)
@ListenerClass(
    targetType = "android.widget.CompoundButton",
    setter = "setOnCheckedChangeListener",
    type = "android.widget.CompoundButton.OnCheckedChangeListener",
    method = @ListenerMethod(
        name = "onCheckedChanged",
        parameters = {
            "android.widget.CompoundButton",
            "boolean"
        }
    )
)
public @interface OnCheckedChanged {
	/** View IDs to which the method will be bound. */
	@IdRes int[] value() default { View.NO_ID };
}
```
首先，该注解同样是编译时注解，并且是用于修饰一个方法的。
其次，使用“元注解”修饰该注解：该注解是用于为ComPoundButton通过`setOnCheckedChangeListener`添加`OnCheckedChangeListener`监听的，该监听有一个需要实现的方法`onCheckedChanged `，该方法传入一个CompoundButton对象和布尔值作为参数。

最后我们关注一下这个注解本身：它只需要返回设置一组id即可，默认返回的是`View.NO_ID`。

这样就完成了一个事件注解的定义。这样看上去很抽象，我们看一个官网对`OnClick`注解的使用例子就清楚了:

```java
@OnClick({ R.id.door1, R.id.door2, R.id.door3 })
public void pickDoor(DoorView door) {
	if (door.hasPrizeBehind()) {
		Toast.makeText(this, "You win!", LENGTH_SHORT).show();
	} else {
		Toast.makeText(this, "Try again", LENGTH_SHORT).show();
	}
}
```
如上，就是为三个door View添加`onClickListener`监听。

那么如果一个监听有多个方法需要回调，又应该怎么办呢？比如ViewPager的`OnPageChangeListener`:

```java
viewPager.setOnPageChangeListener(new ViewPager.OnPageChangeListener() {
	@Override
	public void onPageScrolled(int i, float v, int i1) {
	}
	
	@Override
	public void onPageSelected(int i) {
	}
	
	@Override
	public void onPageScrollStateChanged(int i) {
	}
});
```
如果像`@onClick`那样绑定，我怎么知道绑定的是哪个方法呢？也就是说，是在哪个回调里面执行这个方法呢？这个是通过`callback`来实现的，我们看一下`@OnPageChange`:

```java
@Target(METHOD)
@Retention(CLASS)
@ListenerClass(
    targetType = "android.support.v4.view.ViewPager",
    setter = "setOnPageChangeListener",
    type = "android.support.v4.view.ViewPager.OnPageChangeListener",
    callbacks = OnPageChange.Callback.class
)
public @interface OnPageChange {
	/** View IDs to which the method will be bound. */
	@IdRes int[] value() default { View.NO_ID };

	/** Listener callback to which the method will be bound. */
	Callback callback() default Callback.PAGE_SELECTED;

	/** {@code ViewPager.OnPageChangeListener} callback methods. */
	enum Callback {
		/** {@code onPageSelected(int)} */
		@ListenerMethod(
			name = "onPageSelected",
			parameters = "int"
		PAGE_SELECTED,

		/** {@code onPageScrolled(int, float, int)} */
		@ListenerMethod(
			name = "onPageScrolled",
			parameters = {
				"int",
				"float",
				"int"
			}
		)
		PAGE_SCROLLED,

		/** {@code onPageScrollStateChanged(int)} */
    	@ListenerMethod(
        	name = "onPageScrollStateChanged",
        	parameters = "int"
    	)
    	PAGE_SCROLL_STATE_CHANGED,
  	}
}
```
callback其实是个枚举值，是一组ListenerMethod，映射到监听类的若干个方法，`@OnPageChange`注解就定义了三组，callback默认是`Callback.PAGE_SELECTED`，这个值代表的方法是`onPageSelected`，即如果按照下面的用法:

```java
@OnPageChange(R.id.example_pager) void onPageSelected(int position) {
	Toast.makeText(this, "Selected " + position + "!", Toast.LENGTH_SHORT).show();
}
```
那么就是当`onPageSelected`回调时，该方法会被调用。如果我想绑定到另外一个回调接口里面去，应该怎么办呢？可以如下使用:

```java
@OnPageChange(value = R.id.example_pager, callback = PAGE_SCROLL_STATE_CHANGED)
void onPageStateChanged(int state) {
	Toast.makeText(this, "State changed: " + state + "!", Toast.LENGTH_SHORT).show();
}
```
这样就可以在`onPageScrollStateChanged`回调中调用这个方法了。

##视图绑定注解
和资源绑定注解很像，如下：

```java
/**
 * Bind a field to the view for the specified ID. The view will automatically be cast to the field
 * type.
 * <pre><code>
 * {@literal @}Bind(R.id.title) TextView title;
 * </code></pre>
 */
@Retention(CLASS) @Target(FIELD)
public @interface Bind {
	/** View ID to which the field will be bound. */
	@IdRes int[] value();
}
```
只不过这里返回的是一个int[]数组，因为`@Bind`实际支持将一组id绑定到一个数组或者List属性上。

##其余注解
1. __Unbinder__ 这个注解是为了给类生成一个Unbinder实例，这样可以将之前`bind`的变量全部解绑，后面有例子；
2. __Optional__ 可选项，有时候有些View、资源找不到，所以有些注入必须可选，否则就会Crash；

#注解解析大致过程
定义注解只是注解框架的一部分，代表着注解框架所支持的功能，解析注解是注解框架的另一个核心部分。下面我们来看看ButterKnife是如何实现编译时注解的。我们就从使用的例子上切入开始分析整个注解的运作过程。

##运行时解析
运行时解析源于ButterKnife的一行代码:

```java
ButterKnife.bind(this);
```
在任何需要使用ButterKnife的类中，这行代码都需要调用。我们看看这个方法做了什么:

```java
public static void bind(@NonNull Activity target) {
    bind(target, target, Finder.ACTIVITY);
}
  
static void bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder) {
    Class<?> targetClass = target.getClass();
    try {
        if (debug) Log.d(TAG, "Looking up view binder for " + targetClass.getName());
        ViewBinder<Object> viewBinder = findViewBinderForClass(targetClass);
        viewBinder.bind(finder, target, source);
    } catch (Exception e) {
        throw new RuntimeException("Unable to bind views for " + targetClass.getName(), e);
    }
}
```
`bind()`方法有很多重载方法，但是最终都会调用到方法`bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder)`。这个方法接受三个参数，我们关注一下第三个参数`Finder`，这个其实是一个枚举类，用于编译时生成的代码中，非常重要（后面就可以见到了），它主要的功能如下:
__为Activity、View、Dialog三种情景提供Context以及findViewById功能，并自带强制转换。__

这个方法会对targetClass进行解析，调用的方法是`findViewBinderForClass()`:

```java
static final Map<Class<?>, ViewBinder<Object>> BINDERS = new LinkedHashMap<>();
static final ViewBinder<Object> NOP_VIEW_BINDER = new ViewBinder<Object>() {

@NonNull
private static ViewBinder<Object> findViewBinderForClass(Class<?> cls) throws IllegalAccessException, InstantiationException {
    ViewBinder<Object> viewBinder = BINDERS.get(cls);
    if (viewBinder != null) {
        if (debug) Log.d(TAG, "HIT: Cached in view binder map.");
        return viewBinder;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
        if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
        return NOP_VIEW_BINDER;
    }
    try {
        Class<?> viewBindingClass = Class.forName(clsName + "$$ViewBinder");
        //noinspection unchecked
        viewBinder = (ViewBinder<Object>) viewBindingClass.newInstance();
        if (debug) Log.d(TAG, "HIT: Loaded view binder class.");
    } catch (ClassNotFoundException e) {
        if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
        viewBinder = findViewBinderForClass(cls.getSuperclass());
    }
    BINDERS.put(cls, viewBinder);
    return viewBinder;
}
```
这里其实是为一个类创建一个ViewBinder<Object>对象，并通过BINDERS做出缓存。这里注意一点: 当检测到时android或者java框架库中的类，则立即返回，否则就会默认去读取类中的`clsName + "$$ViewBinder"`类，并由这个类创建出ViewBinder<Object>对象（可以猜测出这个类就是ViewBinder<Object>类型的）。

可是我们在使用ButterKnife类的时候，并没有创建这个奇怪的类，那么这个类来自哪里呢？这个下一节再解释，我们继续往下分析，在`findViewBinderForClass`之后，调用的就是下面的方法:

```java
viewBinder.bind(finder, target, source);
```
ViewBinder只是一个接口，框架里面没有相关实现，根据前面的分析，具体的实现应该在`clsName + "$$ViewBinder"`类中。

所以，我们去找`clsName + "$$ViewBinder"`类吧。

##编译时注解——代码生成
研究编译时注解，我们需要找到Processor类，ButterKnife类的Processor类叫做`ButterKnifeProcessor`，它的`process`方法如下:

```java
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);

    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
        TypeElement typeElement = entry.getKey();
        BindingClass bindingClass = entry.getValue();

        try {
            bindingClass.brewJava().writeTo(filer);
        } catch (IOException e) {
            error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
            e.getMessage());
        }
    }
    
    return true;
}
```
第一个关键点是调用`findAndParseTargets`方法:

```java
private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
    Set<String> erasedTargetNames = new LinkedHashSet<>();

    // Process each @Bind element.
    for (Element element : env.getElementsAnnotatedWith(Bind.class)) {
        if (!SuperficialValidation.validateElement(element)) continue;
        try {
            parseBind(element, targetClassMap, erasedTargetNames);
        } catch (Exception e) {
            logParsingError(element, Bind.class, e);
        }
    }

    // Process each annotation that corresponds to a listener.
    for (Class<? extends Annotation> listener : LISTENERS) {
        findAndParseListener(env, listener, targetClassMap, erasedTargetNames);
    }

    // Process each @BindArray element.
    for (Element element : env.getElementsAnnotatedWith(BindArray.class)) {
        if (!SuperficialValidation.validateElement(element)) continue;
        try {
             parseResourceArray(element, targetClassMap, erasedTargetNames);
        } catch (Exception e) {
            logParsingError(element, BindArray.class, e);
        }
    }

    // Process each @BindBitmap element.
    // Process each @BindBool element.
    // Process each @BindColor element.
    // Process each @BindDimen element.
    // Process each @BindDrawable element.
    // Process each @BindInt element.
    // Process each @BindString element.    
    // Process each @Unbinder element.

    // Try to find a parent binder for each.
    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
        String parentClassFqcn = findParentFqcn(entry.getKey(), erasedTargetNames);
        if (parentClassFqcn != null) {
            entry.getValue().setParentViewBinder(parentClassFqcn + BINDING_CLASS_SUFFIX);
        }
     }
      
     return targetClassMap;
}
```
粗略一看，这个方法就是遍历并且整理所有使用ButterKnife注解的元素，详细暂时略去不表。

继续往下看：在遍历这些元素之后再遍历`targetClassMap`，然后调用`bindingClass.brewJava().writeTo(filer);`方法创建文件，这就算生成处理完了，那么到底生成了什么东西呢？

###编译时注解运行结果
这里偷个懒，也是为了更快捷的达到目标：因为有module butterknife-sample，所以ButterKnife可以直接当做应用运行。我们run一下，然后在该下面目录结构下就可以看到一些文件:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Butter-Knife生成类.png" height="250" alt="Butter-Knife生成类"/></div>

如图，在这里就可以看到运行时注解自动生成的类——类名是以`$$ViewBinder`结尾的，以`SimpleActivity$$ViewBinder`为例，它的内容如下:

```java
public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> {
    public SimpleActivity$$ViewBinder() {
    }

    public void bind(Finder finder, final T target, Object source) {
        SimpleActivity$$ViewBinder.Unbinder unbinder = new SimpleActivity$$ViewBinder.Unbinder(target);
        View view = (View)finder.findRequiredView(source, 2130968576, "field \'title\'");
        target.title = (TextView)finder.castView(view, 2130968576, "field \'title\'");
        view = (View)finder.findRequiredView(source, 2130968577, "field \'subtitle\'");
        target.subtitle = (TextView)finder.castView(view, 2130968577, "field \'subtitle\'");
        view = (View)finder.findRequiredView(source, 2130968578, "field \'hello\', method \'sayHello\', and method \'sayGetOffMe\'");
        target.hello = (Button)finder.castView(view, 2130968578, "field \'hello\'");
        unbinder.view2130968578 = view;
        view.setOnClickListener(new DebouncingOnClickListener() {
            public void doClick(View p0) {
                target.sayHello();
            }
        });
        view.setOnLongClickListener(new OnLongClickListener() {
            public boolean onLongClick(View p0) {
                return target.sayGetOffMe();
            }
        });
        view = (View)finder.findRequiredView(source, 2130968579, "field \'listOfThings\' and method \'onItemClick\'");
        target.listOfThings = (ListView)finder.castView(view, 2130968579, "field \'listOfThings\'");
        unbinder.view2130968579 = view;
        ((AdapterView)view).setOnItemClickListener(new OnItemClickListener() {
            public void onItemClick(AdapterView<?> p0, View p1, int p2, long p3) {
                target.onItemClick(p2);
            }
        });
        view = (View)finder.findRequiredView(source, 2130968580, "field \'footer\'");
        target.footer = (TextView)finder.castView(view, 2130968580, "field \'footer\'");
        target.headerViews = Utils.listOf(new View[]{(View)finder.findRequiredView(source, 2130968576, "field \'headerViews\'"), (View)finder.findRequiredView(source, 2130968577, "field \'headerViews\'"), (View)finder.findRequiredView(source, 2130968578, "field \'headerViews\'")});
        target.unbinder = unbinder;
    }

    private static final class Unbinder implements butterknife.ButterKnife.Unbinder {
        private SimpleActivity target;
        View view2130968578;
        View view2130968579;

        Unbinder(SimpleActivity target) {
            this.target = target;
        }

        public void unbind() {
            if(this.target == null) {
                throw new IllegalStateException("Bindings already cleared.");
            } else {
                this.target.title = null;
                this.target.subtitle = null;
                this.view2130968578.setOnClickListener((OnClickListener)null);
                this.view2130968578.setOnLongClickListener((OnLongClickListener)null);
                this.target.hello = null;
                ((AdapterView)this.view2130968579).setOnItemClickListener((OnItemClickListener)null);
                this.target.listOfThings = null;
                this.target.footer = null;
                this.target.headerViews = null;
                this.target.unbinder = null;
                this.target = null;
            }
        }
    }
}
```
这个类就是我们要找的类，也是 __注解解析__ 一节中调用`Binder.bind()`方法时用于绑定的类，注意这个类：它实现了ViewBinder接口。再看一下`bind`方法:

```java
static void bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder) {
    Class<?> targetClass = target.getClass();
    try {
        if (debug) Log.d(TAG, "Looking up view binder for " + targetClass.getName());
        ViewBinder<Object> viewBinder = findViewBinderForClass(targetClass);
        viewBinder.bind(finder, target, source);
    } catch (Exception e) {
        throw new RuntimeException("Unable to bind views for " + targetClass.getName(), e);
    }
}
```
`viewBinder.bind(finder, target, source);`调用的就是生成类中的`bind`方法。我们取方法中的一小段来看:

```java
View view = (View)finder.findRequiredView(source, 2130968576, "field \'title\'");
target.title = (TextView)finder.castView(view, 2130968576, "field \'title\'");
```
这里就通过finder的`findRequiredView`和`castView`两个方法来为target的title属性来赋值了。这里其实略有多余，不需要再添加`(TextView)`强转，看一下`castView`方法的实现就好了:

```java
public <T> T castView(View view, int id, String who) {
	try {
		return (T) view;
   } catch (ClassCastException e) {
   		if (who == null) {
        throw new AssertionError();
		}
      	String name = getResourceEntryName(view, id);
      	throw new IllegalStateException("View '"
          + name
          + "' with ID "
          + id
          + " for "
          + who
          + " was of the wrong type. See cause for more info.", e);
	}
}
```
这里已经使用泛型进行强转了。

到这里，我们大致可以明白ButterKnife的实现原理了:
__为注解类生成一个对应的ViewBinder类，自动生成`findViewById`等模板代码，在运行的时候，反射实例化ViewBinder类，调用它的`bind()`方法完成注解——这里只有在生成ViewBinder对象的时候使用了反射，其余代码均是正常的调用。__

从生成的代码里面还可以看到Unbinder这个内部类：生成它是因为SimpleActivity里面有这样的用法:

```java
@Unbinder ButterKnife.Unbinder unbinder;
```
在`unbinder()`方法里面，我们可以看到把注入的内容全部删除了，这也就是`@Unbinder`注解的使用方法。

好了，到了这里，我们大致能知道ButterKnife是怎么玩的了，但是具体如何生成类这一块还不是很清楚。__下一节我们来重点分析：ButterKnife是如何生成一个类的。__

#生成类
讲解这个需要我们基于前面的分析，在`findAndParseTargets`方法中，有很多的`parseXXX`方法调用。根据前面的分析，基本可以确定一个注解类XXX会对应生成`XXX$$ViewBinder`类，那么我们先确定：ButterKnife如何确定需要生成哪些类，属性又是如何规整到这些类里面去的？即：怎么知道生成一个类的所有信息。

##解析过程
我们来看一个`parseResourceInt `方法，这是处理所有的Int资源的类:

```java
private void parseResourceInt(Element element, Map<TypeElement, BindingClass> targetClassMap,
                                  Set<String> erasedTargetNames) {
	boolean hasError = false;
	// A
	TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

	// Verify that the target type is int.
	if (element.asType().getKind() != TypeKind.INT) {
		error(element, "@%s field type must be 'int'. (%s.%s)", BindInt.class.getSimpleName(),
		enclosingElement.getQualifiedName(), element.getSimpleName());
		hasError = true;
	}

	// Verify common generated code restrictions.
	hasError |= isInaccessibleViaGeneratedCode(BindInt.class, "fields", element);
	hasError |= isBindingInWrongPackage(BindInt.class, element);

	if (hasError) {
		return;
	}

	// Assemble information on the field.
	String name = element.getSimpleName().toString();
	int id = element.getAnnotation(BindInt.class).value();

	// B
	BindingClass bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
	FieldResourceBinding binding = new FieldResourceBinding(id, name, "getInteger", false);
	bindingClass.addResource(binding);

	erasedTargetNames.add(enclosingElement.toString());
}
```
看A处，这里调用了一个`getEnclosingElement()`方法，这个方法是干啥的呢？官网文档解释如下:
>返回此元素直接封装（非严格意义上）的元素。 类或接口被认为用于封装它直接声明的字段、方法、构造方法和成员类型。这包括所有（隐式）默认构造方法和枚举类型的隐式 values 和 valueOf 方法。 包封装位于其中的顶层类和接口，但不认为它封装了子包。 当前不认为其他种类的元素封装了任何元素；但是，随着此 API 或编程语言的发展，这种情况可能发生改变。

从ButterKnife的注解定义来看，注解使用在方法或者属性上面，那么通过这个方法就可以获取到封装这些元素的类。接下来我们看B处，先看一下`getOrCreateTargetClass`方法:

```java
private static final String BINDING_CLASS_SUFFIX = "$$ViewBinder";

private BindingClass getOrCreateTargetClass(Map<TypeElement, BindingClass> targetClassMap, TypeElement enclosingElement) {
	BindingClass bindingClass = targetClassMap.get(enclosingElement);
	if (bindingClass == null) {
		String targetType = enclosingElement.getQualifiedName().toString();
		String classPackage = getPackageName(enclosingElement);
		String className = getClassName(enclosingElement, classPackage) + BINDING_CLASS_SUFFIX;

		bindingClass = new BindingClass(classPackage, className, targetType);
		targetClassMap.put(enclosingElement, bindingClass);
	}
	return bindingClass;
}
```
代码很清楚：这里会往targetClassMap里面存储一个Entry，Key为enclosingElement（即外部类），Value为BindingClass。BindingClass初始化记录了三样东西: 原先的注解类类名、原先的注解类所在包名和要生成的注解类类名（`XXX$$ViewBinder`）。

我们继续看B处，在获取到这个BindingClass之后，就执行如下代码:

```java
FieldResourceBinding binding = new FieldResourceBinding(id, name, "getInteger", false);
bindingClass.addResource(binding);
```
这个代码生成一个FieldResourceBinding对象然后添加到BindingClass中。

由此我们可以看到整个解析过程: __遍历所有的注解元素，并通过`getEnclosingElement()`获取声明这些元素的类，所有需要创建的类信息都维护在`targetClassMap`Map数据结构中，Key为注解使用类，Value为BindingClass——后续将根据这个BindingClass生成代码。所有的注解元素都将经过解析存储到BindingClass中（类似FieldResourceBinding这样的属性），最终解析完成，所有需要生成的类都可以通过`targetClassMap`索引到__。

##写入过程
写入过程是这样的:

```java
for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
	TypeElement typeElement = entry.getKey();
	BindingClass bindingClass = entry.getValue();

	try {
		bindingClass.brewJava().writeTo(filer);
	} catch (IOException e) {
		error(typeElement, "Unable to write view binder for type %s: %s", typeElement, e.getMessage());
	}
}
```
这里就是遍历所有的BindingClass，调用`brewJava()`，这个方法涉及到的知识以及后面`writeTo()`方法均来自于[javapoet项目](https://github.com/square/javapoet)，这个项目可以很方便的根据一些信息生成一个Java类，此处不赘述。

#总结
ButterKnife不但使用方便，而且通过编译时注解，减少了反射的使用，提高了注解解析的效率，确实非常赞。

最后推荐一个Android Studio插件用于生成ButterKnife代码，地址如下: [android-butterknife-zelezny](https://github.com/avast/android-butterknife-zelezny)，可以用于快速生成ButterKnife注解代码。

鉴于某些项目由于方法数问题或者其余原因没有使用ButterKnife，程序员们被逼要写`findViewById`这样的代码，还得去XML里面翻找id，我基于以上插件，改写了另外一个插件，地址如下：[AndroidViewGenerator](https://github.com/BigFootprint/AndroidViewGenerator)，欢迎使用。