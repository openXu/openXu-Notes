
## 1. 什么是apt？它能帮我们做什么？有什么好处？

APT(Annotation Processing Tool)编译时注解处理器，是javac内置的一个用于编译时扫描和处理注解的工具。在**源代码编译阶段通过注解处理器获取到源文件内注解相关内容信息**，然后通过这些信息根据需求来自动的生成一些java源文件，通常注解处理器是用于**自动产生一些有规律性的重复代码，解决了手工编写重复代码的问题，大大提升编码效率**。APT被广泛应用在各种框架中，比如ButterKnife、EventBus3、Dagger2、ARouter、DataBinding等框架都运用到APT技术。这篇文章通过几个示例对apt相关知识进行一次实现和巩固。

虽然`ButterKnife`已经被google官方DataBinding替代，但是不得不承认它之前是一个非常被认可和被广泛使用的框架。`ButterKnife`框架就是使用了APT在编译时为每个使用了相关注解的Activity/Fragment生成一个`XxxActivity/Fragment_ViewBinding`类，这个类的构造方法中完成了对应findViewById()、getResources()、setOnClickListener()的操作。运行时调用`ButterKnife.bind(this)`则是拼装生成的类名称通过反射获取到构造方法，然后调用构造方法完成初始化方法的调用。

```java
public class MainActivity extends AppCompatActivity {
    //绑定控件 findViewById()
    @BindView(R.id.btn_butterknife)
    Button btn_butterknife;
    @BindView(R.id.btn_mybutterknife)
    Button btn_mybutterknife;
    @BindView(R.id.btn_arouter)
    Button btn_arouter;
    //绑定资源  getResource().getString()
    @BindString(R.string.app_name)
    String app_name;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        /**
         * ButterKnife.bind(this)执行过程：
         * 1. 根据target也就是MainActivity对象，拼装得到apt生成的MainActivity_ViewBinding类名，通过反射获取到它的构造方法
         * 2. 调用构造方法创建MainActivity_ViewBinding的对象
         * 3. 构造方法中完成了MainActivity中使用注解的成员变量初始化以及点击事件绑定
         *    //得到Activity的根View(是为了和Fragment的处理共用一套代码)
         *    View source = target.getWindow().getDecorView()
         *    //Activity中可以直接用target.findViewById()，但是为了和Fragment一致，所以上面获取其根窗口
         *    target.btn_butterknife = source.findViewById(R.id.btn_butterknife)
         *    target.app_name = target.getContext().getResources().getString(R.string.app_name);
         *    view.setOnClickListener(v->target.onClick(v));
         */
        ButterKnife.bind(this);
    }
    @OnClick({R.id.btn_butterknife, R.id.btn_mybutterknife, R.id.btn_arouter})
    public void onClick(View v){
        switch (v.getId()){
            case R.id.btn_butterknife:
                Toast.makeText(this, "我是正牌butterknife框架", Toast.LENGTH_LONG).show();
                break;
            case R.id.btn_mybutterknife:
                break;
            case R.id.btn_arouter:
                break;
        }
    }
}
```

`ButterKnife.bind(this)`的内部如果通过**运行时注解+反射**的方式也能实现其功能，但是运行时大量反射(每个注解至少需要一次反射)会**造成性能消耗**。而通过apt在编译时生成java文件，运行时每个类只需要一次反射(获取调用构造方法)就可以了，这样大大提高了程序性能。下面我们参考ButterKnife的功能自己实现一个ButterKnife。

## 2. 手撸ButterKnife

### 2.1 项目结构

```xml
AnnotationProcessing
	|
	|--annotation           //一个Java Library，用于定义注解
	|--annotation-compiler  //一个Java Library，用于编写运行时注解处理器，帮我们生成java文件
	|--annotation-runtime   //Android Library，也就是我们编写的框架库，项目模块中需要依赖它
	|--app                  //项目模块

```

build.gradle配置：

```xml

# 1. annotation模块

apply plugin: 'java-library'
sourceCompatibility = "1.8"
targetCompatibility = "1.8"

# 2. annotation-compiler模块

apply plugin: 'java-library'
dependencies {
    //依赖注解模块
    implementation project(":annotation")
    //https://github.com/google/auto/tree/auto-service-1.0-rc6
    //依赖google的autoService，此处必须implementation && annotationProcessor，否则注解处理器不会执行
    implementation 'com.google.auto.service:auto-service:1.0-rc6'
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'

    //依赖javapoet，面向对象编程的方式通过建造者模式生成java文件
    //https://github.com/square/javapoet
    implementation 'com.squareup:javapoet:1.13.0'

}
sourceCompatibility = "1.8"
targetCompatibility = "1.8"

# 3. annotation-runtime模块

apply plugin: 'com.android.library'
android {
    ...
}
dependencies {
    //传递依赖注解库，这样项目模块中就只需要依赖本框架库即可
    api project(":annotation")
}

# 4. app模块

apply plugin: 'com.android.application'
apply plugin: 'com.jakewharton.butterknife'
android {
    ...
}

dependencies {
    implementation fileTree(dir: "libs", include: ["*.jar"])
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 

    //butterknife
    implementation 'com.jakewharton:butterknife:10.2.3'
    //If you are using Kotlin, replace annotationProcessor with kapt.
    annotationProcessor 'com.jakewharton:butterknife-compiler:10.2.3'

    /**自定义butterknife*/
    implementation project(":annotation-runtime")
    //Android Gradle插件2.2以上版本提供了annotationProcessor来代替apt，不需要在添加com.neenbedankt.android-apt插件
    annotationProcessor project(":annotation-compiler")
}
```

### 2.2 代码实现

**annotation**

这个模块下主要定义一些注解，这里实现了三个注解，如果需要其他功能，可以在这里继续定义：

```java
//表示该注解只保留在class文件阶段，项目运行是代码被加载后该注解就被去除了
@Retention(RetentionPolicy.CLASS)
//表示该注解标注在成员变量上
@Target({ElementType.FIELD})
public @interface BindView {
    int value();
}

@Retention(RetentionPolicy.CLASS)
@Target({ElementType.FIELD})
public @interface BindString {
    int value();
}

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface OnClick {
    int[] value() default {-1};
}
```

**annotation-compiler**

注解处理器模块中，定义一个类继承`AbstractProcessor`，实现其`process()`方法完成对源文件中注解的解析处理，生成相关java帮助文件，其实就是对`Element`的API操作，代码中注释已经很清楚了，这里就不赘述。

AbstractProcessor的实现类需要用`@AutoService(Processor.class)`注解标记，注解处理器才会被识别，并在构建时执行。

```java

/**
 * Author: openXu
 * Time: 2020/10/29 10:50
 * class: ViewBindProcessor
 * Description: APT的核心是AbstractProcessor类，自定义注解处理器需要继承它，帮我们在编译期生成java文件，
 *
 * AutoService是Google开发的，将在build/classes/java/main/META-INF/services/文件夹中生成javax.annotation.processing.Processor配置文件，
 * 该文件将包含被@AutoService标记了的AbstractProcessor实现类，当外部程序装配这个模块的时候，该配置文件找到具体的实现类名，并装载实例化，完成模块的注入。
 */
@AutoService(Processor.class)
public class ViewBindProcessor extends AbstractProcessor {
    //注解处理器上下文环境
    ProcessingEnvironment processingEnv;
    //文件相关的辅助类, 支持注释处理器创建新文件(java文件)
    public Filer filer; //文件相关的辅助类
    //元素相关的辅助类
    public Elements elements;
    //打印日志相关的辅助类
    public Messager messager;

    /**1. 初始化*/
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        this.processingEnv = processingEnv;
        messager = processingEnv.getMessager();
        elements = processingEnv.getElementUtils();
        filer = processingEnv.getFiler();
    }
    /**2. 指定当前正在使用的Java版本，processingEnv.getSourceVersion()返回我们脚本中配置的java版本*/
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return processingEnv.getSourceVersion();
    }
    /**★ 3. 该注解处理器支持处理（需要处理）的注解类型名称集合*/
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> set = new HashSet<>();
        set.add(BindView.class.getName());
        set.add(BindString.class.getName());
        set.add(OnClick.class.getName());
        return set;
    }
    /**
     * 处理注解
     * @param annotations
     * @param roundEnv 携带了项目中 所有的 标记了 这个注解处理器声明要处理的注解 的程序元素
     * @return 返回true表示本注解处理器处理了注解
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        messager.printMessage(Diagnostic.Kind.WARNING, "=====================开始处理注解======================");
        //★ 解析注解，并按照类封装到ElementForOneType的集合中
        Map<TypeElement, ElementForOneType> bindingMap = findAndParseTargets(roundEnv);
        //★ 为每个Activity/Fragment生成ViewBinding类java文件
        for (Map.Entry<TypeElement, ElementForOneType> entry : bindingMap.entrySet()) {
            createSource(entry);
        }
        return true;
    }

    /**
     * 根据注解元素信息为每个Activity/Fragment生成Xxxxx$ViewBinding类java文件
     * java文件将被输出到项目mudule下build/generated/ap_generated_sources文件夹下
     * @param entry
     */
    private void createSource(Map.Entry<TypeElement, ElementForOneType> entry){
        TypeElement typeElement = entry.getKey();
        ElementForOneType elementForOneType = entry.getValue();
        String activityName = typeElement.getSimpleName().toString();
        String packgeName = elements.getPackageOf(typeElement).getQualifiedName().toString();
        String className = activityName + "$ViewBinding";
        messager.printMessage(Diagnostic.Kind.WARNING, "-->生成："+packgeName + "." + className);
        //生成一个java文件
        try {
            JavaFileObject sourceFile = filer.createSourceFile(packgeName + "." + className);
            Writer writer = sourceFile.openWriter();
            StringBuffer stringBuffer = new StringBuffer();

            stringBuffer.append("package "+packgeName+";\n");
            stringBuffer.append("import "+packgeName+"."+activityName+";\n");
            stringBuffer.append("import android.view.View;\n");
            stringBuffer.append("public class "+className+"{\n");
            stringBuffer.append("public "+className+"(final "+activityName+" target, View source) {\n");
            //BindView -> target.tv_name = source.findViewById(R.id.tv_name);
            List<VariableElement> bindViewElements = elementForOneType.getBindViewElements();
            if(bindViewElements!=null && bindViewElements.size()>0){
                for(VariableElement element : bindViewElements){
                    //获取变量名字
                    String fieldName = element.getSimpleName().toString();
                    //获取注解中的id
                    int resId = element.getAnnotation(BindView.class).value();
                    stringBuffer.append("target."+fieldName+" = source.findViewById("+resId+");\n");
                }
            }
            //BindString -> target.app_name = source.getContext().getResources().getString(R.string.app_name);
            List<VariableElement> bindStringElements = elementForOneType.getBindStringElements();
            if(bindStringElements!=null && bindStringElements.size()>0){
                for(VariableElement element : bindStringElements){
                    String fieldName = element.getSimpleName().toString();
                    int resId = element.getAnnotation(BindString.class).value();
                    stringBuffer.append("target."+fieldName+" = source.getContext().getResources().getString("+resId+");\n");
                }
            }
            //OnClick -> source.findViewById(R.id.tv_name).setOnClickListener(v->{target.onClick(v)});
            List<ExecutableElement> clickElements = elementForOneType.getClickElements();
            if(clickElements!=null && clickElements.size()>0){
                for(ExecutableElement element : clickElements){
                    String methodName = element.getSimpleName().toString();
                    int[] resIds = element.getAnnotation(OnClick.class).value();
                    for(int resId : resIds){
                        stringBuffer.append("source.findViewById("+resId+").setOnClickListener(v->{target."+methodName+"(v);});\n");
                    }
                }
            }
            stringBuffer.append("}\n");
            stringBuffer.append("}\n");
            writer.write(stringBuffer.toString());
            writer.flush();
            writer.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 查找并解析所有的注解标记元素，以Activity为单位（key）组织起来，每个Activity对应一个ElementForOneType封装对象
     * ElementForOneType中保存了该Activity中 以注解类型划分 的 所有元素的集合，一种注解对应一个集合
     */
    private Map<TypeElement, ElementForOneType> findAndParseTargets(RoundEnvironment env) {
        //被标记的类，对应所有注解的分类集合
        Map<TypeElement, ElementForOneType> map = new LinkedHashMap<>();
        /**
         * 获取当前项目中所有被某个注解标记的程序元素，这些程序元素可能是以下几种类型，得看注解标注在什么上
         * TypeElement：类或接口程序元素
         * ExecutableElement：方法元素
         * VariableElement：成员变量元素
         *
         * 比如这里获取所有被BindView标记的元素，BindView标记在成员变量上，那么获取到的就是VariableElement集合
         */
        Set<? extends Element> bindViewElements = env.getElementsAnnotatedWith(BindView.class);
        messager.printMessage(Diagnostic.Kind.NOTE, "获取所有BindView注解元素："+bindViewElements.size());
        //获取所有标记了BindString的成员变量
        Set<? extends Element> bindStringElements = env.getElementsAnnotatedWith(BindString.class);
        messager.printMessage(Diagnostic.Kind.NOTE, "获取所有BindString注解元素："+bindStringElements.size());
        //获取所有标记了OnClick注解的方法元素
        Set<? extends Element> onClickElements = env.getElementsAnnotatedWith(OnClick.class);
        messager.printMessage(Diagnostic.Kind.NOTE, "获取所有OnClick注解元素："+onClickElements.size());
        //1. 处理BinderView标记的元素集合
        for (Element element : bindViewElements) {
            //转换成对应的子类，才能调用子类中的方法
            VariableElement variableElement = (VariableElement)element;
            //返回此变量的封闭元素，也就是成员变量所在的类节点
            TypeElement typeElement = (TypeElement)variableElement.getEnclosingElement();
            //获取map中指定类对应的元素封装
            ElementForOneType elementForOneType = map.get(typeElement);
            if(elementForOneType==null) {  //如果该类没有封装过就创建一个
                elementForOneType = new ElementForOneType();
                map.put(typeElement, elementForOneType);
            }
            //获取封装中所有bindView元素集合
            List<VariableElement> bindViewList = elementForOneType.getBindViewElements();
            if(bindViewList==null) {  //如果集合为空，则创建一个新的，并设置给封装类
                bindViewList = new ArrayList<>();
                elementForOneType.setBindViewElements(bindViewList);
            }
            //将节点元素放到封装类对应的集合中
            bindViewList.add(variableElement);
        }
        //2. 处理BindString标记的元素集合
        for (Element element : bindStringElements) {
            VariableElement variableElement = (VariableElement)element;
            TypeElement typeElement = (TypeElement)variableElement.getEnclosingElement();
            ElementForOneType elementForOneType = map.get(typeElement);
            if(elementForOneType==null) {
                elementForOneType = new ElementForOneType();
                map.put(typeElement, elementForOneType);
            }
            List<VariableElement> bindStringList = elementForOneType.getBindStringElements();
            if(bindStringList==null) {
                bindStringList = new ArrayList<>();
                elementForOneType.setBindStringElements(bindStringList);
            }
            bindStringList.add(variableElement);
        }
        //3. 处理OnClick标记的元素集合
        for (Element element : onClickElements) {
            ExecutableElement executableElement = (ExecutableElement)element;
            TypeElement typeElement = (TypeElement)executableElement.getEnclosingElement();
            ElementForOneType elementForOneType = map.get(typeElement);
            if(elementForOneType==null) {
                elementForOneType = new ElementForOneType();
                map.put(typeElement, elementForOneType);
            }
            List<ExecutableElement> clickList = elementForOneType.getClickElements();
            if(clickList==null) {
                clickList = new ArrayList<>();
                elementForOneType.setClickElements(clickList);
            }
            clickList.add(executableElement);
        }
        return map;
    }
}
```

```java
/**
 * Author: openXu
 * Time: 2020/10/29 14:29
 * class: ElementForOneType
 * Description: 一个类上所有标记了注解的各类元素集合的封装，每一个集合表示一种注解标记的素有元素
 */
public class ElementForOneType {
    //所有标记BindView注解的成员变量元素集合
    List<VariableElement> bindViewElements;
    //所有标记BindString注解的成员变量元素集合
    List<VariableElement> bindStringElements;
    //所有标记OnClick注解的方法元素集合
    List<ExecutableElement> clickElements;
    public List<VariableElement> getBindViewElements() {
        return bindViewElements;
    }
    public void setBindViewElements(List<VariableElement> bindViewElements) {
        this.bindViewElements = bindViewElements;
    }
    public List<VariableElement> getBindStringElements() {
        return bindStringElements;
    }
    public void setBindStringElements(List<VariableElement> bindStringElements) {
        this.bindStringElements = bindStringElements;
    }
    public List<ExecutableElement> getClickElements() {
        return clickElements;
    }
    public void setClickElements(List<ExecutableElement> clickElements) {
        this.clickElements = clickElements;
    }
}
```

**annotation-runtime**

这是自定义的一个框架库，主要包含一个`BindLife`类，相当于`ButterKnife`类，用于在运行时完成绑定初始化操作：

```java
public class BindLife {
    //绑定Activity ->  BindLife.bind(this);
    public static void bind(Activity target) {
        View sourceView = target.getWindow().getDecorView();
        bind(target, sourceView);
    }
    //绑定Fragment -> BindLife.bind(this, view);
    public static void bind(Object target, View rootView){
        String bindingClassName = target.getClass().getName()+"$ViewBinding";
        Log.d("BindLife", "找到"+target+"对应的Binding类："+bindingClassName);
        try {
            //得到apt生成的绑定类
            Class clazz = target.getClass().getClassLoader().loadClass(bindingClassName);
            //反射获取构造方法
            Constructor constructor = clazz.getConstructor(target.getClass(), View.class);
            //调用构造方法完成初始化操作
            constructor.newInstance(target, rootView);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**使用**

在项目Activity/Fragment中使用自定义注解后，Build/Make Project，会在`module\build\generated\ap_generated_sources\debug\out\...`文件夹下生成对应的java文件`MyBindActivity$ViewBinding.java`和`MyBindFragment$ViewBinding.java`：

```java
package com.openxu.apt.mybtlf;
import android.view.View;
public final class MyBindActivity$ViewBinding {
  public MyBindActivity$ViewBinding(MyBindActivity target, View source) {
    target.btn_butterknife = source.findViewById(2131165266);
    target.btn_mybutterknife = source.findViewById(2131165267);
    target.btn_arouter = source.findViewById(2131165265);
    target.app_name = source.getContext().getResources().getString(2131492891);
    source.findViewById(2131165266).setOnClickListener(v->{target.onClick(v);});
    source.findViewById(2131165267).setOnClickListener(v->{target.onClick(v);});
    source.findViewById(2131165265).setOnClickListener(v->{target.onClick(v);});
  }
}
```

运行时通过`BindLife.bind(this)`完成绑定初始化。

```java
//注意，导入的是我们自定义的注解
import com.openxu.annotation.BindView;
import com.openxu.annotation.BindString;
import com.openxu.annotation.OnClick;
import com.openxu.apt.R;
import com.openxu.runtime.BindLife;
public class MyBindActivity extends AppCompatActivity {
    //绑定控件 findViewById()
    @BindView(R.id.btn_butterknife)
    Button btn_butterknife;
    @BindView(R.id.btn_mybutterknife)
    Button btn_mybutterknife;
    @BindView(R.id.btn_arouter)
    Button btn_arouter;
    //绑定资源  getResource().getString()
    @BindString(R.string.app_name)
    String app_name;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        setTitle("手撸butterknife");
        BindLife.bind(this);
    }
    @OnClick({R.id.btn_butterknife, R.id.btn_mybutterknife, R.id.btn_arouter})
    public void onClick(View v){
    }
}


public class MyBindFragment extends Fragment {
    @BindView(R.id.btn_butterknife)
    EditText btn_butterknife;
    @BindString(R.string.app_name)
    String app_name;
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.activity_main, container, false);
        BindLife.bind(this, view);
        return view;
    }
    @OnClick({R.id.btn_butterknife})
    public void onClick(View v){
    }
}
```

### JavaPoet

上面生成代码是通过`StringBuffer`拼装java代码字符串实现的，这种方式实现起来很不雅观，我们可以借助[JavaPoet](https://github.com/square/javapoet)通过面向对象的编程方式生成java文件。JavaPoet是square推出的开源java代码生成框架，简化了Java代码生成的开发难度，通过建造者模式，使调用更加人性化，可读性提升。JavaPoet中，大部分数据类型使用了APT中通用的类型，结合APT自动化产生代码非常方便快速。下面我们重写`ViewBindProcessor`类的`createSource()`方法，使用JavaPoet生成java文件(JavaPoet的使用请参考[JavaPoet文档](https://github.com/square/javapoet)，非常详细)：

```xml
implementation 'com.squareup:javapoet:1.13.0'
```

```java
private void createSourceByJavaPoet(Map.Entry<TypeElement, ElementForOneType> entry){
    TypeElement typeElement = entry.getKey();
    ElementForOneType elementForOneType = entry.getValue();
    String activityName = typeElement.getSimpleName().toString();
    String packgeName = elements.getPackageOf(typeElement).getQualifiedName().toString();
    String className = activityName + "$ViewBinding";
    messager.printMessage(Diagnostic.Kind.WARNING, "-->生成："+packgeName + "." + className);
    //生成一个java文件
    try {
        //创建构造方法Buider对象
        MethodSpec.Builder constractBuider = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                //添加参数，ClassName.bestGuess()方法用于获取一个类，会自动完成import导包
                .addParameter(ClassName.bestGuess(packgeName+"."+activityName), "target")
                .addParameter(ClassName.bestGuess("android.view.View"), "source");
        //BindView -> target.tv_name = source.findViewById(R.id.tv_name);
        List<VariableElement> bindViewElements = elementForOneType.getBindViewElements();
        if(bindViewElements!=null && bindViewElements.size()>0){
            for(VariableElement element : bindViewElements){
                /**
                 * addStatement添加方法代码字符串不需要分号和换行符，字符串代码可使用类似String.format()的语法格式
                 * $L: 字面量值占位
                 * $S: 字符串占位，会自动插入""
                 * $T: 类占位，可以自动导包
                 * $N: 获取对象的名称引用
                 */
                constractBuider.addStatement("target.$L = source.findViewById($L)",
                        element.getSimpleName().toString(),
                        element.getAnnotation(BindView.class).value());
            }
        }
        //BindString -> target.app_name = source.getContext().getResources().getString(R.string.app_name);
        List<VariableElement> bindStringElements = elementForOneType.getBindStringElements();
        if(bindStringElements!=null && bindStringElements.size()>0){
            for(VariableElement element : bindStringElements){
                constractBuider.addStatement("target.$L = source.getContext().getResources().getString($L)",
                        element.getSimpleName().toString(),
                        element.getAnnotation(BindString.class).value());
            }
        }
        //OnClick -> source.findViewById(R.id.tv_name).setOnClickListener(v->{target.onClick(v)});
        List<ExecutableElement> clickElements = elementForOneType.getClickElements();
        if(clickElements!=null && clickElements.size()>0){
            for(ExecutableElement element : clickElements){
                int[] resIds = element.getAnnotation(OnClick.class).value();
                for(int resId : resIds){
                    constractBuider.addStatement("source.findViewById($L).setOnClickListener(v->{target.$N(v);})",
                            resId, element.getSimpleName().toString());
                }
            }
        }
        //创建一个public final class Xxx$ViewBinding的java类对象
        TypeSpec bindType = TypeSpec.classBuilder(className)
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                .addMethod(constractBuider.build())   //添加上面的构造方法
                .build();
        //创建一个java文件
        JavaFile javaFile = JavaFile.builder(packgeName, bindType)
                .build();
        javaFile.writeTo(filer);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 手撸Arouter

[ARouter](https://github.com/alibaba/ARouter/blob/master/README_CN.md)

在组件化架构中，一个app被分割成很多模块，对代码进行高度的解耦使模块分离，能极大地提高开发效率，方便后期维护工作。但是组件化框架的页面跳转就遇到了问题。

```java
//显式跳转
startActivity(new Intent(getContext(), MediaMainActivity.class));
//反射显式跳转
startActivity(new Intent(getContext(), Class.forName("com.xxx.xxx.MediaMainActivity")));
//隐式跳转
Intent intent = new Intent();
intent.setAction("com.xxx.xxx.action");
startActivity(intent);
```

两个module之间怎样相互启动对方的Activity？要清楚，这两个module之间是不可能相互依赖的(只能单向依赖)，通常情况下业务模块之间甚至连单向依赖都没有，这样的话A模块中不能拿到B模块XxxActivity.class，所以显式跳转是无法实现的。当然，我们可以通过反射获取到Activity的class进行跳转，但是每次跳转通过反射在使用上就不方便，可能存在类全名写错的问题，而且反射可能对性能造成影响。我们还可以使用隐式跳转，但是维护清单文件中的action又是一个麻烦的工作。

阿里的路由框架ARouter帮我们实现了模块之间的页面路由，我们看一下它的实现原理。

### Arouter原理

#### 编译阶段

Arouter通过apt在编译阶段处理被`@Route(path = /groupName/xxx)`标记的类，按照groupName进行分组，放入一个`groupMap<groupName, path>`中，然后为每个group生成一个`ARouter$$Group$$groupName`的java文件，每个module模块生成一个`ARouter$$Root$$moduleName`。一个模块允许有多个分组group，他们都需要在root中注册，运行时首先通过路由的groupName找到对应的`ARouter$$Group$$groupName`，然后根据path找到对应的`RouteMeta`对象。

```java
public class ARouter$$Group$$module_common implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/module_common/splash", RouteMeta.build(RouteType.ACTIVITY, SplashActivity.class, "/module_common/splash", "module_common", null, -1, -2147483648));
    atlas.put("/module_common/systemSet", RouteMeta.build(RouteType.FRAGMENT, SystemSettingFragment.class, "/module_common/systemset", "module_common", null, -1, -2147483648));
    ...
  }
}

public class ARouter$$Root$$module_common implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("module_common", ARouter$$Group$$module_common.class);
  }
}
```

#### 运行时初始化

程序启动时调用`ARouter.init(mApplication)`初始化框架，首先会查找所有`com.alibaba.android.arouter.routes`包下的类，然后遍历这些类根据类名分别进行处理，比如处理上面的`ARouter$$Root$$module_common`时将模块中的分组数据存储到一个`Map<String, Class<? extends IRouteGroup>> groupsIndex`容器中。init方法中并没有加载`ARouter$$Group$$Xxx`，只有在分组中的某一个路径第一次被访问的时候，该分组才会被初始化。

Arouter的设计理念：分组管理，按需加载。一个应用可能有上百个页面，用户打开应用时并不会将所有页面都打开一遍，一次性将所有页面数据都加载会影响性能占用内存。

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
	...
    try {
        if (registerByPlugin) {
        } else {
            Set<String> routerMap;
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                //遍历包名"com.alibaba.android.arouter.routes"下的所有class存入routerMap中（即把所有编译期自动生成的class读取出来）
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                }
                PackageUtils.updateVersion(context);
            } else {
            	//从sp缓存中获取
                routerMap = new HashSet<>(context.getSharedPreferences(...));
            }
            ...
            //遍历类，根据类名前缀分别进行处理
            for (String className : routerMap) {
            	//读取com.alibaba.android.arouter.routes.ARouter$$Root $$ xxx，将数据保存到 Map<String, Class<? extends IRouteGroup>> groupsIndex中
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                }
                ... 
            }
        }

       ...
    } catch (Exception e) {
    }
}
```

#### 跳转

```java
ARouter.getInstance().build("/module_common/systemSet")
	//传递基本数据类型
    .withLong("key1", 666L)
    .withString("key3", "888")
    //将Object转换成json字符串，传递接受后再解析成对象
    .withObject("key4", new Test("Jack", "Rose")) 
    //传递Bundle
    .withBundle("Bundle",new Bundle())
    //传递Parcelable
    .withParcelable("User",new User())
    .navigation();
```

`ARouter.getInstance().build("/module_common/systemSet")`会根据传入的path通过字符串分割解析到group，然后通过`new Postcard(path, group)`构造了一个Postcard对象。

接下来调用`Postcard.navigation()->_ARouter.navigation()`方法进行跳转。首先调用了`LogisticsCenter.completion(postcard)`，该方法从一个map中获取path对应的RouteMeta对象，如果没有获取到则去**加载对应group**的`ARouter$$Group$$Xxx`类，通过反射构造一个对象后调用loadInto()将该group下所有的RouteMeta缓存起来，获取到RouteMeta对象后，将其携带的信息填充给跳卡Postcard对象。最后跳转是调用了`_navigation()`方法，在里面根据跳转类型分别处理。

```java
//com.alibaba.android.arouter.launcher._ARouter

protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    ...
    //根据postcard中的path找到
    LogisticsCenter.completion(postcard);
    ...
    //不是绿色通道，需要拦截
    if (!postcard.isGreenChannel()) {   
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {
            @Override
            public void onContinue(Postcard postcard) {
            	//继续跳转
                _navigation(context, postcard, requestCode, callback);
            }
            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    callback.onInterrupt(postcard);
                }
            }
        });
    } else {
    	//直接跳转
        return _navigation(context, postcard, requestCode, callback);
    }
    return null;
}

private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    final Context currentContext = null == context ? mContext : context;
    //判断类型
    switch (postcard.getType()) {
        case ACTIVITY:   //activity跳转
            //postcard.getDestination()就是需要跳转的Activity.class
            final Intent intent = new Intent(currentContext, postcard.getDestination());
            intent.putExtras(postcard.getExtras());
            ...
            // Navigation in main looper.
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                	//在主线程中完成跳转
                    startActivity(requestCode, currentContext, intent, postcard, callback);
                }
            });
            break;
        case PROVIDER:
            return postcard.getProvider();
        case BOARDCAST:
        case CONTENT_PROVIDER:
        case FRAGMENT:
            Class fragmentMeta = postcard.getDestination();
            try {
                Object instance = fragmentMeta.getConstructor().newInstance();
                if (instance instanceof Fragment) {
                    ((Fragment) instance).setArguments(postcard.getExtras());
                } else if (instance instanceof android.support.v4.app.Fragment) {
                    ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                }
                return instance;
            } catch (Exception ex) {
                ...
            }
        case METHOD:
        case SERVICE:
        default:
            return null;
    }

    return null;
}
```

### 手动实现

Arouter








