---
title: LayoutInflater源码分析
date: 2017-10-10 22:41:43
tags: Android
categories: Android
---
在 Android 开发中，LayoutInflater 经常被使用，它的作用就是将所需要的 xml 布局填充到我们的 View 对象中。
首先，我们看一下它的用法，获取其实例的方法有三种：
```java
 // 1. 在 Activity 中获取 LayoutInflater：
  LayoutInflater layoutInflater = getLayoutInflater();
  // 2. 通过 LayoutInflater 的静态方法获取
  LayoutInflater layoutInflater = LayoutInflater.from(context);
  // 3 通过 Context 中的 getSystemService 方法获取
  LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```
<!--more-->
通过查看源码，我们可以发现此三种获取实例的方式最后都是通过第三种方式获取到 LayoutInflater 的实例的。我们来看看 ContextImpl 类中的 getSystemService 方法：
```java
@Override
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```
我们再看看 SystemServiceRegistry 类中的代码：
```java
    static {
        //...省略部分代码

        registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});

      //...省略部分代码
    }

    /**
    * Gets a system service from a given context.
    */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```
关于这部分内容，SystemServiceRegistry 在什么时候初始化，在什么时候注册这些服务，不是本文的主题，这里不做赘述。最关键的是，我们看到最终我们拿到的是 PhoneLayoutInflater 对象。我们先将这个对象放在这里，看看获取到的 LayoutInflater 接下来干的活。
我们在使用的时候都是调用了 inflater 方法：
```java
View view = inflater.inflate(layoutId, parent, attachToParent)
```
在 LayoutInflater 中，重载了几个 inflater 方法，最后都是调用了下面这个重载方法：
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            // Context 对象
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            // 存储父视图
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                final String name = parser.getName();
                // 解析 merge 标签
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    // 这里就是通过 xml 的 tag 来解析 layout 根视图
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        // Create layout params that match root, if supplied   生成布局参数
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // 如果  attachToRoot 为 false  给 temp 设置布局参数
                            temp.setLayoutParams(params);
                        }
                    }

                    // Inflate all children under temp against its context. 解析 temp 下的所有子 View
                    rInflateChildren(parser, temp, attrs, true);

                    // 如果 root 不为空，且 attachToRoot 为 true，将 temp 添加到 父视图
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // 如果 root 为空或者 attachToRoot 为 false，返回的就是 temp
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } 
            //省略 catch finaly 代码

            return result;
        }
    }
```
这里的流程很清晰，首先会去解析 xml 中的根标签，如果是 merge，就会调用 rInflater 方法，将 merge 下的所有子 View 直接添加到根标签中；如果是普通元素，就会调用 `createViewFromTag` 方法来解析，再接着 调用 rInflater 对 temp 下的子 View 解析，并添加到 temp 下。最后就将解析到的根视图返回。我们关键来看看 `createViewFromTag` 方法：
```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }
        // ...代码省略
        try {
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        // 内置 View 控件的解析
                        view = onCreateView(parent, name, attrs);
                    } else {
                        // 自定义控件的解析
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } 
        // ... 省略 catch 代码
    }

```
这里，重点代码就是判断 View 是否为自带的控件，如果是自带的控件，就会将 name 拼全，这里就是 PhoneLayoutInflater 这个实现类起作用的时候了。
在 PhoneLayoutInflater 重写的 onCreateView 方法中，会将 "android.widget."，"android.webkit."， "android.app." 这个几个前缀拼在 View 的 name 前面，然后调用 `createView` 方法来创建一个 View ：
```java
  public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        // 从缓存中获取构造函数
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

            if (constructor == null) {
                // 如果 prefix 不为空，构造完整的 View 路径，并且加载该类
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                // ...代码省略
                // 获取构造函数
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);   
                // 将构造函数添加到缓存中
                sConstructorMap.put(name, constructor);
            } else {
               // ...代码省略
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;
            // 通过构造函数实例化View
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;

        } 
        // 省略catch finaly 代码
    }
```
这里主要功能就是通过 ClassLoader 加载这个特定的View的类，然后通过这个类的构造函数来实例化这个 View。这样子，我们单个 View 就被解析出来了，由于我们的 xml 是一个视图树，在 LayoutInflater 中，我们通过 `rInflate` 递归调用，来解析整个视图树。在将解析到的子 View 添加到它的 parent 中。

总结：
        LayoutInflater 加载布局文件就是通过 Android 提供的 pull 解析方式来一层一层解析布局文件，然后将最后生成的视图返回。由于解析也是需要一定时间，况且一个布局文件层级过多的话，解析的时间就会增加，这样就会导致页面卡顿。所以在实际开发中，我们应该尽量将布局文件少些层级，来保证 App 操作的流畅度。
