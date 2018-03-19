---
layout:     post
title:      "ButterKnife源码解析"
date:       2017-05-25 13:23:43
author:     "afayp"
catalog:    true
tags:
    - Android
    - 源码解析
---

# ButterKnife

ButterKnife的核心就是利用APT在编译期间动态生成一些辅助代码，然后利用这些辅助代码完成view、listener等的绑定。

看下官方的一个demo：

```java
public class SimpleActivity extends Activity {
    private static final ButterKnife.Action<View> ALPHA_FADE = new ButterKnife.Action<View>() {
        @Override
        public void apply(@NonNull View view, int index) {
            AlphaAnimation alphaAnimation = new AlphaAnimation(0, 1);
            alphaAnimation.setFillBefore(true);
            alphaAnimation.setDuration(500);
            alphaAnimation.setStartOffset(index * 100);
            view.startAnimation(alphaAnimation);
        }
    };

    @BindView(R2.id.title)
    TextView title;
    @BindView(R2.id.subtitle)
    TextView subtitle;
    @BindView(R2.id.hello)
    Button hello;
    @BindView(R2.id.list_of_things)
    ListView listOfThings;
    @BindView(R2.id.footer)
    TextView footer;

    @BindViews({R2.id.title, R2.id.subtitle, R2.id.hello})
    List<View> headerViews;

    private SimpleAdapter adapter;

    @OnClick(R2.id.hello)
    void sayHello() {
        Toast.makeText(this, "Hello, views!", LENGTH_SHORT).show();
        ButterKnife.apply(headerViews, ALPHA_FADE);
    }

    @OnLongClick(R2.id.hello)
    boolean sayGetOffMe() {
        Toast.makeText(this, "Let go of me!", LENGTH_SHORT).show();
        return true;
    }

    @OnItemClick(R2.id.list_of_things)
    void onItemClick(int position) {
        Toast.makeText(this, "You clicked: " + adapter.getItem(position), LENGTH_SHORT).show();
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.simple_activity);
        ButterKnife.bind(this);

        // Contrived code to use the bound fields.
        title.setText("Butter Knife");
        subtitle.setText("Field and method binding for Android views.");
        footer.setText("by Jake Wharton");
        hello.setText("Say Hello");

        adapter = new SimpleAdapter(this);
        listOfThings.setAdapter(adapter);
    }
}
```

对应生成的`SimpleActivity_ViewBinding`类：

```java
public class SimpleActivity_ViewBinding<T extends SimpleActivity> implements Unbinder {
  protected T target;

  private View view2130968578;

  private View view2130968579;

  @UiThread
  public SimpleActivity_ViewBinding(final T target, View source) {
    this.target = target;

    View view;
    target.title = Utils.findRequiredViewAsType(source, R.id.title, "field 'title'", TextView.class);
    target.subtitle = Utils.findRequiredViewAsType(source, R.id.subtitle, "field 'subtitle'", TextView.class);
    view = Utils.findRequiredView(source, R.id.hello, "field 'hello', method 'sayHello', and method 'sayGetOffMe'");
    target.hello = Utils.castView(view, R.id.hello, "field 'hello'", Button.class);
    view2130968578 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.sayHello();
      }
    });
    view.setOnLongClickListener(new View.OnLongClickListener() {
      @Override
      public boolean onLongClick(View p0) {
        return target.sayGetOffMe();
      }
    });
    view = Utils.findRequiredView(source, R.id.list_of_things, "field 'listOfThings' and method 'onItemClick'");
    target.listOfThings = Utils.castView(view, R.id.list_of_things, "field 'listOfThings'", ListView.class);
    view2130968579 = view;
    ((AdapterView<?>) view).setOnItemClickListener(new AdapterView.OnItemClickListener() {
      @Override
      public void onItemClick(AdapterView<?> p0, View p1, int p2, long p3) {
        target.onItemClick(p2);
      }
    });
    target.footer = Utils.findRequiredViewAsType(source, R.id.footer, "field 'footer'", TextView.class);
    target.headerViews = Utils.listOf(
        Utils.findRequiredView(source, R.id.title, "field 'headerViews'"), 
        Utils.findRequiredView(source, R.id.subtitle, "field 'headerViews'"), 
        Utils.findRequiredView(source, R.id.hello, "field 'headerViews'"));
  }

  @Override
  @CallSuper
  public void unbind() {
    T target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");

    target.title = null;
    target.subtitle = null;
    target.hello = null;
    target.listOfThings = null;
    target.footer = null;
    target.headerViews = null;

    view2130968578.setOnClickListener(null);
    view2130968578.setOnLongClickListener(null);
    view2130968578 = null;
    ((AdapterView<?>) view2130968579).setOnItemClickListener(null);
    view2130968579 = null;

    this.target = null;
  }
}
```



### ButterKnifeProcessor

首先看下`ButterKnifeProcessor#getSupportedAnnotationTypes`，该方法的返回值表示此注解处理器要处理的注解集合。

```java
  @Override public Set<String> getSupportedAnnotationTypes() {
    Set<String> types = new LinkedHashSet<>();
    for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
      types.add(annotation.getCanonicalName());
    }
    return types;
  }

  private Set<Class<? extends Annotation>> getSupportedAnnotations() {
    Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();

    annotations.add(BindAnim.class);
    annotations.add(BindArray.class);
    annotations.add(BindBitmap.class);
    annotations.add(BindBool.class);
    annotations.add(BindColor.class);
    annotations.add(BindDimen.class);
    annotations.add(BindDrawable.class);
    annotations.add(BindFloat.class);
    annotations.add(BindFont.class);
    annotations.add(BindInt.class);
    annotations.add(BindString.class);
    annotations.add(BindView.class);
    annotations.add(BindViews.class);
    annotations.addAll(LISTENERS);

    return annotations;
  }
```

接着`ButterKnifeProcessor#process`方法：

```java
@Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

  for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
    TypeElement typeElement = entry.getKey();
    BindingSet binding = entry.getValue();

    JavaFile javaFile = binding.brewJava(sdk, debuggable);
    try {
      javaFile.writeTo(filer);
    } catch (IOException e) {
      error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
    }
  }

  return false;
}
```

在 `process` 中主要做了两件事，第一是`findAndParseTargets`找出被注解的元素，封装成 `Map<TypeElement, BindingSet>` ，第一个参数可以想成类，第二个参数可以想成所有有关信息的封装；第二就是遍历这个 map ，然后针对每个类生成对应的 ViewBinder 类。

`ButterKnifeProcessor#findAndParseTargets`中会找出被A注解的所有元素，接着调用`parseA`方法对这些元素进行解析。

```Java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
  Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
  Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
  // .......

  // Process each @BindView element.
  for (Element element : env.getElementsAnnotatedWith(BindView.class)) {//拿到被BindView注解的元素
    // we don't SuperficialValidation.validateElement(element)
    // so that an unresolved View type can be generated by later processing rounds
    try {
      parseBindView(element, builderMap, erasedTargetNames);//解析
    } catch (Exception e) {
      logParsingError(element, BindView.class, e);
    }
  }
  // .......

  return bindingMap;
}
```

这里我们挑一个bindView注解解析过程看看：

```java
private void parseBindView(Element element, Map builderMap,
      Set erasedTargetNames) {
    // 得到注解 @BindView 元素所在的类元素
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    // Start by verifying common generated code restrictions.
    // ---------- 类型校验逻辑 start ---------------
    // 判断是否被注解在属性上，如果该属性是被 private 或者 static 修饰的，则出错
    // 判断是否被注解在错误的包中，若包名以“android”或者“java”开头，则出错
    boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
        || isBindingInWrongPackage(BindView.class, element);

    // Verify that the target type extends from View.
    TypeMirror elementType = element.asType();
    if (elementType.getKind() == TypeKind.TYPEVAR) {
      TypeVariable typeVariable = (TypeVariable) elementType;
      elementType = typeVariable.getUpperBound();
    }
    // 判断元素是不是View及其子类或者Interface
    if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
      if (elementType.getKind() == TypeKind.ERROR) {
        note(element, "@%s field with unresolved type (%s) "
                + "must elsewhere be generated as a View or interface. (%s.%s)",
            BindView.class.getSimpleName(), elementType, enclosingElement.getQualifiedName(),
            element.getSimpleName());
      } else {
        error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
            BindView.class.getSimpleName(), enclosingElement.getQualifiedName(),
            element.getSimpleName());
        hasError = true;
      }
    }
    // 如果有错误 不执行下面代码
    if (hasError) {
      return;
    }
    //---------------- 类型校验逻辑 end -----------------

    // Assemble information on the field.  //得到被注解的注解值，即 R.id.xxx
    int id = element.getAnnotation(BindView.class).value();
    // 根据所在的类元素去查找 builder
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    // 如果相应的 builder 已经存在
    if (builder != null) {
      // 得到相对应的 View 绑定的属性名
      String existingBindingName = builder.findExistingBindingName(getId(id));
      // 若该属性名已经存在，则说明之前已经绑定过，会报错
      if (existingBindingName != null) {
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      // 如果没有对应的 builder ，就通过 getOrCreateBindingBuilder 方法生成，并且放入 builderMap 中
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }
    // 得到注解名
    String name = element.getSimpleName().toString();
    // 得到注解元素的类型
    TypeName type = TypeName.get(elementType);
    boolean required = isFieldRequired(element);
    // 根据 id ，添加相对应的 Field 的绑定信息
    builder.addField(getId(id), new FieldViewBinding(name, type, required));

    // Add the type-erased version to the valid binding targets set.
    // 添加到待 unbind 的序列中
    erasedTargetNames.add(enclosingElement);
}
```



方法分为前后两部分，首先对该 `element` 去一些校验，包括不能被private、static修饰，包名校验等。

若校验通过之后，生成该 `element` 所在的类元素对应的 builder ，builder 中添加相应的 Field 绑定信息，最后添加到待 unbind 的序列中去。

得到包含注解信息的 `bindingMap` 后，就会利用利用 `binding` 中的信息生成相应的 Java 源码。

```
JavaFile javaFile = binding.brewJava(sdk, debuggable);

JavaFile brewJava(int sdk, boolean debuggable) {
  return JavaFile.builder(bindingClassName.packageName(), createType(sdk, debuggable))
      .addFileComment("Generated code from Butter Knife. Do not modify!")
      .build();
}
```

类的具体构建动作在createType中完成：

```java
private TypeSpec createType(int sdk) {
    // 生成类名为 bindingClassName 的公共类，比如 MainActivity_ViewBinding
    TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
            .addModifiers(PUBLIC);
    // 是否修饰为 final ，默认是 false
    if (isFinal) {
        result.addModifiers(FINAL);
    }

    if (parentBinding != null) {
        // 如果有父类的话，那么要继承父类
        result.superclass(parentBinding.bindingClassName);
    } else {
        // 如果没有父类，那么实现 Unbinder 接口
        result.addSuperinterface(UNBINDER);
    }

    // 增加一个变量名为target，类型为targetTypeName的成员变量
    if (hasTargetField()) {
        result.addField(targetTypeName, "target", PRIVATE);
    }
    // 如果没有 View 绑定
    if (!constructorNeedsView()) {
        // Add a delegating constructor with a target type + view signature for reflective use.
        // 该生成的构造方法被 @deprecated ，一般作为反射使用
        result.addMethod(createBindingViewDelegateConstructor(targetTypeName));
    }
    // 生成构造方法，另外 findViewById 类似的代码都在这里生成
    // Xxxx_ViewBinding 一般都是执行这个方法生成构造器
    result.addMethod(createBindingConstructor(targetTypeName, sdk));

    if (hasViewBindings() || parentBinding == null) {
        //生成unBind方法
        result.addMethod(createBindingUnbindMethod(result, targetTypeName));
    }

    return result.build();
}
```

createBindingConstructor：

```java
private MethodSpec createBindingConstructor(TypeName targetType, int sdk) {
    // 创建构造方法，方法修饰符为 public ，并且添加注解为UiThread
    MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
            .addAnnotation(UI_THREAD)
            .addModifiers(PUBLIC);
    // 如果有方法绑定，比如 @OnClick
    if (hasMethodBindings()) {
        // 如果有，那么添加 targetType 类型，final 修饰，参数名为 target 的构造方法参数
        constructor.addParameter(targetType, "target", FINAL);
    } else {
        // 如果没有，和上面比起来就少了一个 final 修饰符
        constructor.addParameter(targetType, "target");
    }
    // 如果有注解的 View
    if (constructorNeedsView()) {
        // 那么添加 View source 参数
        constructor.addParameter(VIEW, "source");
    } else {
        // 否则添加 Context context 参数
        constructor.addParameter(CONTEXT, "context");
    }

    if (hasUnqualifiedResourceBindings()) {
        // Aapt can change IDs out from underneath us, just suppress since all will work at runtime.
        constructor.addAnnotation(AnnotationSpec.builder(SuppressWarnings.class)
                .addMember("value", "$S", "ResourceType")
                .build());
    }

    // 如果有父类，那么会根据不同情况调用不同的 super 语句
    if (parentBinding != null) {
        if (parentBinding.constructorNeedsView()) {
            constructor.addStatement("super(target, source)");
        } else if (constructorNeedsView()) {
            constructor.addStatement("super(target, source.getContext())");
        } else {
            constructor.addStatement("super(target, context)");
        }
        constructor.addCode("\n");
    }
    // 如果有绑定 Field 或者方法，那么添加 this.target = target 语句
    if (hasTargetField()) {
        constructor.addStatement("this.target = target");
        constructor.addCode("\n");
    }
    // 如果有 View 绑定
    if (hasViewBindings()) {
        if (hasViewLocal()) {
            // Local variable in which all views will be temporarily stored.
            constructor.addStatement("$T view", VIEW);
        }
        for (ViewBinding binding : viewBindings) {
            // 为 View 绑定生成类似于 findViewById 之类的代码
            addViewBinding(constructor, binding);
        }
        // 为 View 的集合或者数组绑定
        for (FieldCollectionViewBinding binding : collectionBindings) {
            constructor.addStatement("$L", binding.render());
        }

        if (!resourceBindings.isEmpty()) {
            constructor.addCode("\n");
        }
    }
    // 绑定 resource 资源的代码
    if (!resourceBindings.isEmpty()) {
        if (constructorNeedsView()) {
            constructor.addStatement("$T context = source.getContext()", CONTEXT);
        }
        if (hasResourceBindingsNeedingResource(sdk)) {
            constructor.addStatement("$T res = context.getResources()", RESOURCES);
        }
        for (ResourceBinding binding : resourceBindings) {
            constructor.addStatement("$L", binding.render(sdk));
        }
    }

    return constructor.build();
}
```

addViewBinding：

```Java
private void addViewBinding(MethodSpec.Builder result, ViewBinding binding) {
    if (binding.isSingleFieldBinding()) {
        // Optimize the common case where there's a single binding directly to a field.
        FieldViewBinding fieldBinding = binding.getFieldBinding();
        // 注意这里直接使用了 target. 的形式，所以属性肯定是不能 private 的
        CodeBlock.Builder builder = CodeBlock.builder()
                .add("target.$L = ", fieldBinding.getName());
        // 下面都是 View 绑定的代码
        boolean requiresCast = requiresCast(fieldBinding.getType());
        if (!requiresCast && !fieldBinding.isRequired()) {
            builder.add("source.findViewById($L)", binding.getId().code);
        } else {
            builder.add("$T.find", UTILS);
            builder.add(fieldBinding.isRequired() ? "RequiredView" : "OptionalView");
            if (requiresCast) {
                builder.add("AsType");
            }
            builder.add("(source, $L", binding.getId().code);
            if (fieldBinding.isRequired() || requiresCast) {
                builder.add(", $S", asHumanDescription(singletonList(fieldBinding)));
            }
            if (requiresCast) {
                builder.add(", $T.class", fieldBinding.getRawType());
            }
            builder.add(")");
        }
        result.addStatement("$L", builder.build());
        return;
    }

    List requiredBindings = binding.getRequiredBindings();
    if (requiredBindings.isEmpty()) {
        result.addStatement("view = source.findViewById($L)", binding.getId().code);
    } else if (!binding.isBoundToRoot()) {
        result.addStatement("view = $T.findRequiredView(source, $L, $S)", UTILS,
                binding.getId().code, asHumanDescription(requiredBindings));
    }

    addFieldBinding(result, binding);
    // OnClick 等监听事件绑定
    addMethodBindings(result, binding);
}
```

至此我们就得到了前面贴的`SimpleActivity_ViewBinding`类了。之后我们只需在运行时调用这个`SimpleActivity_ViewBinding`的构造方法就可以完成依赖注入了。



### ButterKnife.bind()

运行时我们通过ButterKnife.bind()方法来完成绑定。

```java
@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
  View sourceView = target.getWindow().getDecorView();
  return createBinding(target, sourceView);
}

@NonNull @UiThread
public static Unbinder bind(@NonNull View target) {
  return createBinding(target, target);
}
```

接着看createBinding:

```java
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    // 得到 target 的类名，比如 MainActivity 
    Class targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    // 找到 target 对应的构造器
    Constructor constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      // 创建对应的对象
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
}
```

通过`findBindingConstructorForClass`方法找到`targetActivity`对应的`targetActivity_ViewBinding`类。

```java
@VisibleForTesting
static final Map, Constructor> BINDINGS = new LinkedHashMap<>();

@Nullable @CheckResult @UiThread
private static Constructor findBindingConstructorForClass(Class cls) {
    // 对构造器的查找进行了缓存，可以直接从 Map 中获取
    Constructor bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      // 得到对应的 class 对象，比如 MainActivity_ViewBinding
      Class bindingClass = Class.forName(clsName + "_ViewBinding");
      //noinspection unchecked
      // 得到对应的构造器
      bindingCtor = (Constructor) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    // 进行缓存
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
}
```

可以看到，使用了 `Class.forName()` 反射来查找 `Class`，然后拿到其构造器，这里也做了缓存机制。

最后创建出`targetActivity_ViewBinding`对象，在其构造函数中完成依赖注入。



### 总结

ButterKnife是一个依赖注入框架，其利用APT技术在编译期间动态生成一些辅助类来帮助我们完成findViewById、setOnClickListener等操作。

ButterKnife实现的注解处理器是`ButterKnifeProcessor`,这个处理器会扫描代码中的`BindView`、`OnClick`

之类的注解，利用`JavaPoet`和`Filer`来生成对应的`XXX_ViewBinding`类。在这个类的构造器中，利用`findViewById`、`setOnClickListener`完成依赖绑定。

在运行时我们会调用`ButterKnife#bind`方法，这个方法内部会利用`Class.forName()`反射调用前面说的`XXX_ViewBinding`类的构造函数，从而实现依赖绑定。



### 参考

[ButterKnife源码简析](http://yydcdut.com/2017/04/19/butterknife-analyse/)

[ButterKnife 源码分析](http://yuqirong.me/2016/12/18/ButterKnife%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

























