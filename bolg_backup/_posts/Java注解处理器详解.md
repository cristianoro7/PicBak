---
layout: post
title: Java注解处理器实战
data: 2017/11/30
subtitle: ""
description: ""
tags:
  - Java#基础
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fm0c4an974j30m80b4qhy.jpg)

> ### 注解处理器

注解强大的地方在于: 我们可以在运行时或者编译时处理注解. 在编译时或者运行时处理注解的机制都可以称为一个注解处理器.

> ### 注解处理器的类型

注解处理器的类型分为两大类:

* 运行时处理器: 这种机制是在程序运行时利用``反射``机制去处理注解.

* 编译时处理器: 这种机制是在程序编译时利用``javac``提供的一个``apt``工具来处理注解.

运行时处理器的特点: 该机制基于反射机制, 因此灵活性很大, 但是却相对耗性能. 编译时处理器的特点:这种方法是在编译时处理注解, 因此不存在性能的问题, 但是灵活性也相对比较低.

在已有的开源库实现中, 普遍是结合``编译时处理器``和``反射机制``这两种方法. 举个例子: ``ButterKnife``. 它是一个``View``注入的库. 内部实现的原理: 编译时利用``apt``生成``findViewById``等一些列模板代码, 然后在运行时利用反射去实例生成的模板代码类, 这样完成了一次注入. 从这里可以看出, ``ButterKnife``并没有完全基于编译时注解处理器. 而是加了反射.

这样做的好处: 开发者不需要手动实例``apt``生成的类. 换句话说: 开发者不需要了解生成类的命令规则. 也许你可能会觉得利用反射会耗性能. 但是仔细想想, 如果没有利用反射的话, 开发者都需要手动编译, 然后再实例化生成的类. 这个过程也是挺繁琐的. 而且, 库内部是有做相应的缓存的, 所以耗的性能还是相对比较低的.


对于反射的利用, 应该适当使用, 而不是避而不用或者滥用.

接下来, 讲讲``APT``的一些知识, 最后再仿写一个简单的``ButterKnife``作为实战.

> ### AbstractProcessor

开发注解处理器的第一个步骤就是继承``AbstractProcessor``. 然后重写其四个方法:

* ``public synchronized void init(ProcessingEnvironment processingEnvironment)``: 这个方法一般是做一些初始化的工作.

* ``public SourceVersion getSupportedSourceVersion()``: 该处理器所支持的``JDK``版本, 一般是支持到最新: ``SourceVersion.latestSupported()``.

* ``public Set<String> getSupportedAnnotationTypes()``: 你所要处理的注解的类型.

* ``public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment)``: 处理注解的方法. 这个是我们主要实现的方法. 返回值表示当前这个注解类型是否需要被随后的注解处理器处理. ``true``表示不需要, ``false``表示后面的注解处理器可能会对本次处理的注解.

认识这4个方法后, 我们还需要熟悉一下编写处理器涉及到的一些概念和``API``.

#### ProcessingEnvironment

``ProcessingEnvironment``是在``init``方法传入的. 它表示``APT``框架的一个处理时上下文. 这个类主要是提供一些工具类. 详细看下面代码注释:

```java
public interface ProcessingEnvironment {

    //获取外部配置的参数
    Map<String,String> getOptions();

    //获取Messager对象, 用于打印一些日志
    Messager getMessager();

    //获取Filer对象, 该对象用于创建文件
    Filer getFiler();

    //获取Elements, 它包含处理Element的一些工具方法
    Elements getElementUtils();

    //获取Types, 它包含处理TypeMirror的一些工具方法
    Types getTypeUtils();

    SourceVersion getSourceVersion();

    Locale getLocale();
}

```

#### RoundEnvironment

当``APT``启动时, 它会将扫描源文件, 然后进入``process``方法处理, 如果这个过程生成了新的文件的话, 新文件会被``APT``再次作为输入, 接着``process``会被``apt``再次调用, 如此循环下去, 直到没有新的文件产生. ``RoundEnvironment``为处理轮次的上下文环境. 下面是``RoundEnvironment``的方法简单介绍:

```java
public interface RoundEnvironment {

    //用于判断是否处理完了, 只有当没有新文件生成时, 证明已经处理完了, 才会返回true
    boolean processingOver();

    //上一次处理是否存在错误
    boolean errorRaised();

    //获取前一次处理的根元素集合
    Set<? extends Element> getRootElements();

    //获取被指定的类型元素标注的元素集
    Set<? extends Element> getElementsAnnotatedWith(TypeElement a);

    //获取被指定的注解标注的元素集
    Set<? extends Element> getElementsAnnotatedWith(Class<? extends Annotation> a);
}
```

#### Element

``Element``代表``Java``静态语言结构的元素. 这样听起来可能有点难以理解, 不过看下面的例子就会很好理解了:

```java
package com.desperado.processors; //PackageElement

public class Test { //TypeElement: 代表类或接口元素

    private int xixi; //VariableElement:

    public Test() { //ExecutableElement

    }

    public void haha(String arg) { //haha方法为TExecutableElement
        int v; //VariableElement
    }
}
```

* ``TypeElement``: 代表类或接口元素

* ``VariableElement``: 代表字段, 枚举常量, 方法或构造函数的参数, 局部变量, 资源变量和异常参数.

* ``PackageElement``: 代表包元素

* ``ExecutableElement``: 代表的方法, 构造方法,和接口或者类的初始化块.

#### DeclaredType, TypeElement和TypeMirror

``TypeElement``表示类或接口元素, 可以从中获取类名, 但是不能获得类本身的信息, 比如父类.

``TypeMirror``: 表示``Java``语言中的类型, 其中包括: 基本类型, 声明类型(类类型和接口类型), 数组类型, 类型变量和空类型. 也代表通配类型参数，可执行文件的签名和返回类型.

``DeclaredType``: 代表声明的类型, 类类型还是接口类型，当然也包括参数化类型，比如Set<String>，也包括原始类型.

> ### 开发一个注解处理器

掌握了上面的概念后, 我们开始来开发一个仿``ButterKnife``的注解处理器.

#### 步骤

* 继承``AbstractProcessor``并重写其中的四个方法.

* 注册注解处理器.


##### 继承``AbstractProcessor``

```java
@AutoService(Processor.class)
public class ViewBindingProcessor extends AbstractProcessor {

    private Types types;

    private Filer filer;

    private Messager messager;

    private Elements elements;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        types = processingEnvironment.getTypeUtils();
        filer = processingEnvironment.getFiler();
        messager = processingEnvironment.getMessager();
        elements = processingEnvironment.getElementUtils();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        //process
        return true;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(BindView.class.getCanonicalName());
        return annotations;
    }
}
```

在``getSupportedAnnotationTypes()``中, 我们直接返回了需要处理的注解的权限定名称集合.

##### 注册处理器

写好的处理器还需要处理告诉``Javac``, 让它编译时运行我们的处理器. 注册的步骤为: 在项目新建一个``resources``目录, 然后在``resources``内新建这样的目录结构: ``/META-INF/services/``. 最后在``services``中新建文件: `` javax.annotation.processing.Processor``. 并且在文件上填写你的处理器的全路径名称. 例如: ``com.desperado.processors.ViewBindingProcessor``.

其实还有一种更为简单的方法. 添加``com.google.auto.service:auto-service:1.0-rc2``这个库(Google开源的). 然后在你的处理器上添加注解: ``@AutoService(Processor.class)
``. 这样, 编译时就会自动帮我们生成注册文件.

> #### 如何组织处理器结构

为了不将注解处理器的代码打包进``APK``. 我们需要将注解和注解处理器分开. 具体的组织如下:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fm07qpsw5qj30cw047aa1.jpg)

* ``annotations``: ``annotations``是一个``java lib``. 是我们自定义的注解

* ``processors``: ``processors``也是一个``java lib``. 是我们的注解处理器

* ``viewbinding``: 为提供为``app``的api.

> #### annotations

首先我们在``annotation``模块定义一个注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface BindView {
    int value() default -1;
}
```

> #### processors

在讲处理器模块前, 我们先看看期望生成的代码模板:

```java
public class MainActivity$ViewBinding {

  public MainActivity$ViewBinding(MainActivity activity, View view) {
    if (activity == null) {
      return;
    }
    activity.mTvTextView = view.findViewById(2131165249);
  }
}
```

生成的代码的类名为: XXXX$ViewBinding

然后在构造函数中对传入的``activity``中的``View``进行赋值.

##### 定义处理规则.

* ``BindView``只能标注字段.

* ``BindView``不能标注接口中的字段, 只能标注类中的字段.

* ``BindView``不能标注抽象类中的字段.

* ``BindView``不能标注被``private``, ``static``或者``final``修饰的字段

* ``BindView``标注字段所属的类必须是``Activity``的子类.

* ``BindView``标注的字段必须是``View``的子类


定义好这些规则后, 我们就可以来看处理器的代码了.

##### 处理过程

```java
@AutoService(Processor.class)
public class ViewBindingProcessor extends AbstractProcessor {

    private Types types;

    private Filer filer;

    private Messager messager;

    private Elements elements;

    private Map<String, ViewClass> map = new HashMap<>();

    private static final String TYPE_ACTIVITY = "android.app.Activity";

    private static final String TYPE_VIEW = "android.view.View";

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        types = processingEnvironment.getTypeUtils();
        filer = processingEnvironment.getFiler();
        messager = processingEnvironment.getMessager();
        elements = processingEnvironment.getElementUtils();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        map.clear(); //该方法会被调用多次, 所以每次进入时, 首先清空之前的脏数据
        logNote("start process");
        for (Element e : roundEnvironment.getElementsAnnotatedWith(BindView.class)) {
            if (!isValid(e)) { //首先判断被标注的元素是否符合我们上面定的规则
                return true; //出现错误, 停止编译
            }
            logNote("start parse annotations");
            performParseAnnotations(e); //开始解析注解
        }
        logNote("start generate code");
        generateCode(); //生成代码
        return true;
    }

    private boolean isValid(Element element) {
        if (!(element instanceof VariableElement)) {
            logError("BindView只能标注字段");
            return false;
        }

        VariableElement variableElement = (VariableElement) element;
        TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();

        if (typeElement.getKind() != ElementKind.CLASS) {
            logError("只能标注类中的字段");
            return false;
        }
        if (typeElement.getModifiers().contains(Modifier.ABSTRACT)) {
            logError("不能标注抽象类中的字段");
            return false;
        }

        for (Modifier modifier : element.getModifiers()) {
            if (modifier == Modifier.PRIVATE || modifier == Modifier.STATIC ||
                    modifier == Modifier.FINAL) {
                logError("BindView不能标注被static," +
                        "private或者final的字段");
                return false;
            }
        }

        if (!isSubtype(typeElement.asType(), TYPE_ACTIVITY)) { //判断被标注的字段的类是不是Activity的子类
            logError(typeElement.getSimpleName() + "必须是 Activity的子类");
            return false;
        }
        if (!isSubtype(variableElement.asType(), TYPE_VIEW)) { //判断被标注的字段是不是View的子类
            logError("BindView只能标注View的子类");
            return false;
        }
        return true;
    }

    private boolean isSubtype(TypeMirror tm, String type) {
        boolean isSubType = false;
        while (tm != null) { //循环获取父类信息
            if (type.equals(tm.toString())) { //通过全路径是否相等
                isSubType = true;
                break;
            }
            TypeElement superTypeElem = (TypeElement) types.asElement(tm);
            if (superTypeElem != null) {
                tm = superTypeElem.getSuperclass();
            } else { //如果为空, 说明没了父类, 所以直接退出
                break;
            }
        }
        return isSubType;
    }

    private void performParseAnnotations(Element element) {

        VariableElement variableElement = (VariableElement) element;
        TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();

        String className = typeElement.getSimpleName().toString();

        ViewClass viewClass = map.get(className);
        ViewField field = new ViewField(variableElement);

        if (viewClass == null) {
            viewClass = new ViewClass(elements.getPackageOf(variableElement).getQualifiedName().toString(),
                    className, typeElement);
            map.put(className, viewClass);
        }
        viewClass.addField(field);
    }

    private void generateCode() {
        for (ViewClass vc : map.values()) {
            try {
                vc.generateCode().writeTo(filer);
            } catch (IOException e) {
                e.printStackTrace();
                logError("error in parse");
            }
        }
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(BindView.class.getCanonicalName());
        return annotations;
    }


    private void logError(String msg) {
        log(Diagnostic.Kind.ERROR, msg);
    }

    private void logNote(String msg) {
        log(Diagnostic.Kind.NOTE, msg);
    }

    private void log(Diagnostic.Kind kind, String msg) {
        messager.printMessage(kind, msg);
    }
}
```

为了使代码结构更为清晰, 这里将注解标注的字段的信息抽象为``ViewField``类, 字段所属的类的信息抽象为``ViewClass``.


```java
public class ViewField {

    private String fieldName;

    private int id;

    public ViewField(VariableElement variableElement) {
        id = variableElement.getAnnotation(BindView.class).value();
        fieldName = variableElement.getSimpleName().toString();
    }

    public String getFieldName() {
        return fieldName;
    }

    public int getId() {
        return id;
    }
}

public class ViewClass {

    private String packName;

    private String className;

    private List<ViewField> fields = new ArrayList<>();

    private TypeElement typeElement;

    public ViewClass(String packName, String className, TypeElement typeElement) {
        this.packName = packName;
        this.className = className;
        this.typeElement = typeElement;
    }

    public void addField(ViewField viewField) {
        fields.add(viewField);
    }

    public JavaFile generateCode() { //为了方便, 这里生成代码用了JavaPoet库来生成代码
        MethodSpec.Builder con = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(TypeName.get(typeElement.asType()), "activity")
                .addParameter(ClassName.get("android.view", "View"), "view")
                .beginControlFlow("if (activity == null)")
                .addStatement("return")
                .endControlFlow();

        for (ViewField f : fields) {
            con.addStatement("activity.$N = view.findViewById($L)", f.getFieldName(), f.getId());
        }

        FieldSpec.Builder fid = FieldSpec.builder(TypeName.get(typeElement.asType()), "activity");
        TypeSpec typeSpec = TypeSpec.classBuilder(className + "$ViewBinding")
                .addModifiers(Modifier.PUBLIC)
                .addField(fid.build())
                .addMethod(con.build())
                .build();
        return JavaFile.builder(packName, typeSpec).build();
    }
}
```

处理过程其实并不复杂, 只是需要我们熟悉``API``而已, 因此, 接下来总结一下获取一些常用信息的方法:

##### 常用的api

```java
public class ViewBindingProcessor extends AbstractProcessor {

    private Types types;

    private Filer filer;

    private Messager messager;

    private Elements elements;

    private static final String TYPE_ACTIVITY = "android.app.Activity";

    private static final String TYPE_VIEW = "android.view.View";

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        types = processingEnvironment.getTypeUtils();
        filer = processingEnvironment.getFiler();
        messager = processingEnvironment.getMessager();
        elements = processingEnvironment.getElementUtils();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        for (Element e : roundEnvironment.getElementsAnnotatedWith(BindView.class)) {
            String packName = elements.getPackageOf(e).getQualifiedName().toString(); //获取包名

            //2: 获取类名.
            //如果标注的是类的话
            TypeElement t = (TypeElement)e;
            String className = t.getSimpleName().toString();

            //如果被标注的是字段或者是方法
            TypeElement t = (TypeElement)e.getEnclosingElement();
            String className = t.getSimpleName().toString();

            //3. 获取方法名字, 或者字段名字
            String name = e.getSimpleName().toString();

            //4: 获取标注的注解
            BindView annotation = e.getAnnotation(BindView.class);
            int id = annotation.value();

            //5: 判断被标注的元素类型, 以字段类型为例子
            ElementKind kind = e.getKind();
            if (kind == ElementKind.FIELD) {

            }

            //6: 获取元素的修饰符. 我们以被修饰的字段为例子.
            for (Modifier modifier : e.getModifiers()) {
                if (modifier == Modifier.PRIVATE || modifier == Modifier.STATIC ||
                modifier == Modifier.FINAL) {
                  logError("BindView不能标注被static," +
                    "private或者final的字段");
                }
            }

            //7: 判读被标注的元素的是不是某个类的子类, 以View类为例子, 判读被修饰的字段是否是View的子类
            TypeMirror tm = e.asType();
            boolean isSubType = false;
            while (tm != null) {
                if ("android.view.View".equals(tm.toString())) {
                    isSubType = true;
                    break;
                }
                TypeElement t = (TypeElement)types.asElement(tm);
                if (t != null) {
                    tm = t.getSuperclass(); //获取父类信息
                } else {
                    //如果为空, 说明没了父类, 所以直接退出
                    break;
                }
            }

        }

        return true;
    }
}
```

> #### viewbinding

模板代码已经生成了, 但是我们还需要实例化这些类. 这个工作交由``viewbinding``. 它提供一些绑定的``API``给``app``. 这样我们就不需要手动调用了.

实现的思路: 运行时利用反射去实例化对应的模板类. 为了降低反射的消耗, 内部会做响应的缓存操作.

```java
public class ViewBinding {

    private static String LATTER_NAME = "$ViewBinding";

    private static Map<Class<?>, Constructor<?>> CACHE = new HashMap<>();

    public static <T extends Activity> void bind(T target) {
        View sourceView = target.getWindow().getDecorView();
        bind(target, sourceView);
    }

     public static <T> void bind(T target, View sourceView) {
        Class<? extends Activity> cl = target.getClass();
        try {
            Class<?> bindClass = findBindClass(cl);
            Constructor<?> constructor = findBindConstructor(bindClass, cl);
            constructor.newInstance(target, sourceView);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }

    private static Class<?> findBindClass(Class<?> targetClass) throws ClassNotFoundException {
        return targetClass.getClassLoader().loadClass(targetClass.getName() + LATTER_NAME);
    }

    private static Constructor<?> findBindConstructor(Class<?> bindClass, Class<? extends Activity> targetClass) throws NoSuchMethodException {
        Constructor<?> constructor = CACHE.get(bindClass);
        if (constructor == null) {
            constructor = bindClass.getConstructor(targetClass, View.class);
            CACHE.put(bindClass, constructor);
        }
        return constructor;
    }
}
```

> #### 使用

注解处理器现在算是开发好了. 最后的步骤就是在``app``中添加依赖, 然后实现绑定.

在``app``的``build.gradle``添加下面的依赖:

```java
implementation project(':viewbinding')
implementation project(':annotations')
annotationProcessor project(':processors')
```

然后在``MainActivity``中实现注入:

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.main_tv_hello)
    TextView mTvTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ViewBinding.bind(this);
        mTvTextView.setText("xuixi");
    }
}
```

完...

> ### 参考资料

http://jinchim.com/2017/08/23/JBind/

http://blog.csdn.net/dd864140130/article/details/53875814
