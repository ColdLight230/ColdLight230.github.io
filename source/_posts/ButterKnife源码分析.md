---
title: ButterKnife源码分析
date: 2017-9-19 21:57:47
tags: Android
categories: Android
---
ButterKnife 是 JakeWharton 大神出的一个编译时注解框架。在开发过程中，使用其注解可以极大的简化了我们代码。今天，我们就来研究一下大神的做法，并且了解编译时注解的原理。源码的版本是8.8.1 。

使用方法不做具体演示，[详情见这里](https://github.com/JakeWharton/butterknife)
这是使用时的具体代码：

<!--more-->

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.tv_title)
    TextView mTvTitle;
    @BindView(R.id.iv_logo)
    ImageView mIvLogo;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ButterKnife.bind(this);
    }

    @OnClick(R.id.iv_logo)
    public void onClick(View view) {
        Toast.makeText(MainActivity.this, "logo...", Toast.LENGTH_SHORT).show();
    }
}
```
编译一下，我们可以在 build -> generated -> source -> apt -> debug 里面看到自动生成了一个 MainActivity_ViewBinding.java 类。
```
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  private View view2131427423;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(final MainActivity target, View source) {
    this.target = target;

    View view;
    target.mTvTitle = Utils.findRequiredViewAsType(source, R.id.tv_title, "field 'mTvTitle'", TextView.class);
    view = Utils.findRequiredView(source, R.id.iv_logo, "field 'mIvLogo' and method 'onClick'");
    target.mIvLogo = Utils.castView(view, R.id.iv_logo, "field 'mIvLogo'", ImageView.class);
    view2131427423 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onClick(p0);
      }
    });
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.mTvTitle = null;
    target.mIvLogo = null;

    view2131427423.setOnClickListener(null);
    view2131427423 = null;
  }
}
```
可以看到，这里生成的 ViewBinding 类就是用来初始化对应的控件，以及给控件设置对应的事件。那么这个文件是怎么生成的了？我们来看看 `ButterKnife.bind()` 方法
`bind()` 方法有一系列的重载方法，最后都会调用到 `createBinding()` 方法
```
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
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
所以 `bind()` 最终的目的就是调用 MainActivity_ViewBinding 的构造方法，这样控件初始化就已经完成。
走到这里，似乎一切流程就已经明了。所以这里就存在着最关键的一步，MainActivity_ViewBinding.java 文件到底是怎么生成的。
首先，我们先来确定一个概念，编译时注解和运行时注解不同的地方就在于，编译时注解是不需要通过反射，而是通过apt(Annotation Processing Tool) 来处理 自己定义的 Annotation。
我们可以看到在 ButterKnife 源码中有依赖到两个 java library，butterknife-annotations 和 butterknife-compiler 。
其中 butterknife-annotations 是用来自定义 Annotation 。而 butterknife-compiler 则是对定义好的 annotations 进行对应处理。
首先 butterknife-compiler 是依赖了 `'com.google.auto.service:auto-service:1.0-rc3'` 和 `'com.squareup:javapoet:1.9.0'`
auto-service 的作用是在编译时，会去扫描到我们自定义的 Annotation 注解，每扫描到一个，就会在 `AbstractProcessor#process()` 方法中回调处理。
在 ButterKnife 中，ButterKnifeProcessor 是继承自 AbstractProcessor 的类。我们主要来看看下面两个方法：
```
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
`getSupportedAnnotationTypes()` 方法就是用来确定我们这个处理类需要处理的 Annotation 的种类。返回的就是我们在butterknife-annotations 里定义好的所有 Annotation。
```
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
`process()` 方法就是来具体处理每个 Annotation 相对应的逻辑。 首先 `findAndParseTargets()` 方法返回的是一个Map，以 `env.getEnclosingElement()` 返回值为key，也就是我们使用了 ButterKnife 注解的类的名称，值是一个 BindingSet 对象，用于生成这个类的 ViewBinding。
再接着就是遍历这个Map，将通过 JavaFile 生成的 XXX_ViewBinding 写成 .java文件。
我们再来看看 BindingSet 里到底是怎么处理的
```java
JavaFile brewJava(int sdk, boolean debuggable) {
    return JavaFile.builder(bindingClassName.packageName(), createType(sdk, debuggable))
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
  }
```
```java
private TypeSpec createType(int sdk, boolean debuggable) {
    TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
        .addModifiers(PUBLIC);
    if (isFinal) {
      result.addModifiers(FINAL);
    }

    if (parentBinding != null) {
      result.superclass(parentBinding.bindingClassName);
    } else {
      result.addSuperinterface(UNBINDER);
    }

    if (hasTargetField()) {
      result.addField(targetTypeName, "target", PRIVATE);
    }

    if (isView) {
      result.addMethod(createBindingConstructorForView());
    } else if (isActivity) {
      result.addMethod(createBindingConstructorForActivity());
    } else if (isDialog) {
      result.addMethod(createBindingConstructorForDialog());
    }
    if (!constructorNeedsView()) {
      // Add a delegating constructor with a target type + view signature for reflective use.
      result.addMethod(createBindingViewDelegateConstructor());
    }
    result.addMethod(createBindingConstructor(sdk, debuggable));

    if (hasViewBindings() || parentBinding == null) {
      result.addMethod(createBindingUnbindMethod(result));
    }

    return result.build();
  }
```
可以看到，在 `brewJava()` 方法中，就是根据我们的注解来构建出对应的 java 类的逻辑。这部分是 [JavaPoet](https://github.com/square/javapoet)  的用法，感兴趣可以去了解。 
于是，在我们每次编译完，就可以看到  XXX_ViewBinding 文件了，ButterKnife 源码分析到此结束。








