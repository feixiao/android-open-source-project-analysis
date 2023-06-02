# Android 窗口管理框架：Android 布局解析者 LayoutInflater

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

**文章目录**

- 一 获取 XmlResourceParser
- 二 解析 View 树
- 三 解析 View

> Instantiates a layout XML file into its corresponding {@link android.view.View}objects.

LayoutInflater 可以把 xml 布局文件里内容加载成一个 View，LayoutInflater 可以说是 Android 里的无名英雄，你经常用的到它，却体会不到它的好。因为隔壁的 iOS 兄弟是没有
这种东西的，他们只能用代码来写布局，需要应用跑起来才能看到效果。相比之下 Android 的开发者就幸福的多，但是大家有没有相关 xml 是如何转换成一个 View 的，今天我们就来分析
这个问题。

LayoutInflater 也是通过 Context 获取，它也是系统服务的一种，被注册在 ContextImpl 的 map 里，然后通过 LAYOUT_INFLATER_SERVICE 来获取。

```java
layoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

LayoutInflater 是一个抽象类，它的实现类是 PhoneLayoutInflater。LayoutInflater 会采用深度优先遍历自顶向下遍历 View 树，根据 View 的全路径名利用反射获取构造器
从而构建 View 的实例。整个逻辑还是很清晰的，我们来具体看一看。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/LayoutInflater_inflate_sequence.png"/>

我们先来看看总的调度方法 inflate()，这个也是我们最常用的

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)
```

这个方法有三个参数：

int resource：布局 ID，也就是要解析的 xml 布局文件，boolean attachToRoot 表示是否要添加到父布局 root 中去。这里面还有个关键的参数 root。它用来表示根布局，这个就很常见的，我们在用
这个方法的时候，有时候给 root 赋值了，有时候直接给了 null（给 null 的时候 IDE 会有警告提示），这个 root 到底有什么作用呢？🤔

它主要有两个方面的作用：

- 当 attachToRoot == true 且 root ！= null 时，新解析出来的 View 会被 add 到 root 中去，然后将 root 作为结果返回。
- 当 attachToRoot == false 且 root ！= null 时，新解析的 View 会直接作为结果返回，而且 root 会为新解析的 View 生成 LayoutParams 并设置到该 View 中去。
- 当 attachToRoot == false 且 root == null 时，新解析的 View 会直接作为结果返回。

注意第二条和第三条是由区别的，你可以去写个例子试一下，当 root 为 null 时，新解析出来的 View 没有 LayoutParams 参数，这时候你设置的 layout_width 和 layout_height 是不生效的。

说到这里，有人可能有疑问了，Activity 里的布局应该也是 LayoutInflater 加载的，我也没做什么处理，但是我设置的 layout_width 和 layout_heigh 参数都是可以生效的，这是为什么？🤔

> 这是因为 Activity 内部做了处理，我们知道 Activity 的 setContentView()方法，实际上调用的 PhoneWindow 的 setContentView()方法。它调用的时候将 Activity 的顶级 DecorView（FrameLayout）
> 作为 root 传了进去，mLayoutInflater.inflate(layoutResID, mContentParent)实际调用的是 inflate(resource, root, root != null)，所以在调用 Activity 的 setContentView()方法时
> 可以将解析出的 View 添加到顶级 DecorView 中，我们设置的 layout_width 和 layout_height 参数也可以生效。

具体代码如下：

```java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {

        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

了解了 inflate()方法各个参数的含义，我们正式来分析它的实现。

```java

public abstract class LayoutInflater {

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        //获取xml资源解析器XmlResourceParser
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);//解析View
        } finally {
            parser.close();
        }
    }
}
```

可以发现在该方法里，主要完成了两件事情：

1. 获取 xml 资源解析器 XmlResourceParser。
2. 解析 View

我们先来看看 XmlResourceParser 是如何获取的。

从上面的序列图可以看出，调用了 Resources 的 getLayout(resource)去获取对应的 XmlResourceParser。getLayout(resource)又去调用了 Resources 的 loadXmlResourceParser()
方法来完成 XmlResourceParser 的加载，如下所示：

```java
public class Resources {

     XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
             throws NotFoundException {
         final TypedValue value = obtainTempTypedValue();
         try {
             final ResourcesImpl impl = mResourcesImpl;
             //1. 获取xml布局资源，并保存在TypedValue中。
             impl.getValue(id, value, true);
             if (value.type == TypedValue.TYPE_STRING) {
                 //2. 加载对应的loadXmlResourceParser解析器。
                 return impl.loadXmlResourceParser(value.string.toString(), id,
                         value.assetCookie, type);
             }
             throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                     + " type #0x" + Integer.toHexString(value.type) + " is not valid");
         } finally {
             releaseTempTypedValue(value);
         }
     }
}
```

可以发现这个方法又被分成了两步：

1. 获取 xml 布局资源，并保存在 TypedValue 中。
2. 加载对应的 loadXmlResourceParser 解析器。

从上面的序列图可以看出，资源的获取涉及到 resources.arsc 的解析过程，这个我们已经在**Resources 的创建流程**简单聊过，这里就不再赘述。通过
getValue()方法获取到 xml 资源以后，就会调用 ResourcesImpl 的 loadXmlResourceParser()方法对该布局资源进行解析，以便得到一个 UI 布局视图。

我们来看看它的实现。

## 一 获取 XmlResourceParser

```java
public class ResourcesImpl {

    XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
               @NonNull String type)
               throws NotFoundException {
           if (id != 0) {
               try {
                   synchronized (mCachedXmlBlocks) {
                       //... 从缓存中查找xml资源

                       // Not in the cache, create a new block and put it at
                       // the next slot in the cache.
                       final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);
                       if (block != null) {
                           final int pos = (mLastCachedXmlBlockIndex + 1) % num;
                           mLastCachedXmlBlockIndex = pos;
                           final XmlBlock oldBlock = cachedXmlBlocks[pos];
                           if (oldBlock != null) {
                               oldBlock.close();
                           }
                           cachedXmlBlockCookies[pos] = assetCookie;
                           cachedXmlBlockFiles[pos] = file;
                           cachedXmlBlocks[pos] = block;
                           return block.newParser();
                       }
                   }
               } catch (Exception e) {
                   final NotFoundException rnf = new NotFoundException("File " + file
                           + " from xml type " + type + " resource ID #0x" + Integer.toHexString(id));
                   rnf.initCause(e);
                   throw rnf;
               }
           }

           throw new NotFoundException("File " + file + " from xml type " + type + " resource ID #0x"
                   + Integer.toHexString(id));
       }
}
```

我们先来看看这个方法的四个形参：

- String file：xml 文件的路径
- int id：xml 文件的资源 ID
- int assetCookie：xml 文件的资源缓存
- String type：资源类型

ResourcesImpl 会缓存最近解析的 4 个 xml 资源，如果不在缓存里则调用 AssetManger 的 openXmlBlockAsset()方法创建一个 XmlBlock。XmlBlock 是已编译的 xml 文件的一个包装类。

AssetManger 的 openXmlBlockAsset()方法的实现如下所示：

```java
public final class AssetManager implements AutoCloseable {
   /*package*/ final XmlBlock openXmlBlockAsset(int cookie, String fileName)
       throws IOException {
       synchronized (this) {
           //...
           long xmlBlock = openXmlAssetNative(cookie, fileName);
           if (xmlBlock != 0) {
               XmlBlock res = new XmlBlock(this, xmlBlock);
               incRefsLocked(res.hashCode());
               return res;
           }
       }
       //...
   }
}
```

可以看出该方法会调用 native 方法 openXmlAssetNative()去代开 fileName 指定的 xml 文件，成功打开该文件后，会得到 C++层的 ResXMLTree 对象的地址 xmlBlock，然后将 xmlBlock 封装进
XmlBlock 中返回给调用者。ResXMLTreed 对象会存放打开后 xml 资源的内容。

上述序列图里的 AssetManger.cpp 的方法的具体实现也就是一个打开资源文件的过程，资源文件一般存放在 APK 中，APK 是一个 zip 包，所以最终会调用 openAssetFromZipLocked()方法打开 xml 文件。

XmlBlock 封装完成后，会调用 XmlBlock 对象的 newParser()方法去构建一个 XmlResourceParser 对象，实现如下所示：

```java
final class XmlBlock {
    public XmlResourceParser newParser() {
        synchronized (this) {
            //mNative指向的是C++层的ResXMLTree对象的地址
            if (mNative != 0) {
                return new Parser(nativeCreateParseState(mNative), this);
            }
            return null;
        }
    }

    private static final native long nativeCreateParseState(long obj);
}
```

mNative 指向的是 C++层的 ResXMLTree 对象的地址，native 方法 nativeCreateParseState()根据这个地址找到 ResXMLTree 对象，利用 ResXMLTree 对象对象构建一个 ResXMLParser 对象，并将 ResXMLParser 对象
的地址封装进 Java 层的 Parser 对象中，构建一个 Parser 对象。所以他们的对应关系如下所示：

- XmlBlock <--> ResXMLTree
- Parser <--> ResXMLParser

就是建立了 Java 层与 C++层的对应关系，实际的实现还是由 C++层完成。

等获取了 XmlResourceParser 对象以后就可以调用 inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) 方法进行 View 的解析了，在解析 View 时
，会先去调用 rInflate()方法解析 View 树，然后再调用 createViewFromTag()方法创建具体的 View，我们来详细的看一看。

## 二 解析 View 树

1. 解析 merge 标签，rInflate()方法会将 merge 下面的所有子 View 直接添加到根容器中，这里我们也理解了为什么 merge 标签可以达到简化布局的效果。
2. 不是 merge 标签那么直接调用 createViewFromTag()方法解析成布局中的视图，这里的参数 name 就是要解析视图的类型，例如：ImageView。
3. 调用 generateLayoutParams()f 方法生成布局参数，如果 attachToRoot 为 false，即不添加到根容器里，为 View 设置布局参数。
4. 调用 rInflateChildren()方法解析当前 View 下面的所有子 View。
5. 如果根容器不为空，且 attachToRoot 为 true，则将解析出来的 View 添加到根容器中，如果根布局为空或者 attachToRoot 为 false，那么解析出来的额 View 就是返回结果。返回解析出来的结果。

接下来，我们分别看下 View 树解析以及 View 的解析。

```java
public abstract class LayoutInflater {

    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
           synchronized (mConstructorArgs) {
               Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

               final Context inflaterContext = mContext;
               final AttributeSet attrs = Xml.asAttributeSet(parser);

               //Context对象
               Context lastContext = (Context) mConstructorArgs[0];
               mConstructorArgs[0] = inflaterContext;

               //存储根视图
               View result = root;

               try {
                   // 获取根元素
                   int type;
                   while ((type = parser.next()) != XmlPullParser.START_TAG &&
                           type != XmlPullParser.END_DOCUMENT) {
                       // Empty
                   }

                   if (type != XmlPullParser.START_TAG) {
                       throw new InflateException(parser.getPositionDescription()
                               + ": No start tag found!");
                   }

                   final String name = parser.getName();

                   if (DEBUG) {
                       System.out.println("**************************");
                       System.out.println("Creating root view: "
                               + name);
                       System.out.println("**************************");
                   }

                   //1. 解析merge标签，rInflate()方法会将merge下面的所有子View直接添加到根容器中，这里
                   //我们也理解了为什么merge标签可以达到简化布局的效果。
                   if (TAG_MERGE.equals(name)) {
                       if (root == null || !attachToRoot) {
                           throw new InflateException("<merge /> can be used only with a valid "
                                   + "ViewGroup root and attachToRoot=true");
                       }

                       rInflate(parser, root, inflaterContext, attrs, false);
                   } else {
                       //2. 不是merge标签那么直接调用createViewFromTag()方法解析成布局中的视图，这里的参数name就是要解析视图的类型，例如：ImageView
                       final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                       ViewGroup.LayoutParams params = null;

                       if (root != null) {
                           if (DEBUG) {
                               System.out.println("Creating params from root: " +
                                       root);
                           }
                           //3. 调用generateLayoutParams()f方法生成布局参数，如果attachToRoot为false，即不添加到根容器里，为View设置布局参数
                           params = root.generateLayoutParams(attrs);
                           if (!attachToRoot) {
                               // Set the layout params for temp if we are not
                               // attaching. (If we are, we use addView, below)
                               temp.setLayoutParams(params);
                           }
                       }

                       if (DEBUG) {
                           System.out.println("-----> start inflating children");
                       }

                       //4. 调用rInflateChildren()方法解析当前View下面的所有子View
                       rInflateChildren(parser, temp, attrs, true);

                       if (DEBUG) {
                           System.out.println("-----> done inflating children");
                       }

                       //如果根容器不为空，且attachToRoot为true，则将解析出来的View添加到根容器中
                       if (root != null && attachToRoot) {
                           root.addView(temp, params);
                       }

                       //如果根布局为空或者attachToRoot为false，那么解析出来的额View就是返回结果
                       if (root == null || !attachToRoot) {
                           result = temp;
                       }
                   }

               } catch (XmlPullParserException e) {
                   final InflateException ie = new InflateException(e.getMessage(), e);
                   ie.setStackTrace(EMPTY_STACK_TRACE);
                   throw ie;
               } catch (Exception e) {
                   final InflateException ie = new InflateException(parser.getPositionDescription()
                           + ": " + e.getMessage(), e);
                   ie.setStackTrace(EMPTY_STACK_TRACE);
                   throw ie;
               } finally {
                   // Don't retain static reference on context.
                   mConstructorArgs[0] = lastContext;
                   mConstructorArgs[1] = null;

                   Trace.traceEnd(Trace.TRACE_TAG_VIEW);
               }

               return result;
           }
     }
}
```

上面我们已经提到 View 树的解析是有 rInflate()方法来完成的，我们接着来看看 View 树是如何被解析的。

```java
public abstract class LayoutInflater {

    void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        //1. 获取树的深度，执行深度优先遍历
        final int depth = parser.getDepth();
        int type;

        //2. 逐个进行元素解析
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();

            if (TAG_REQUEST_FOCUS.equals(name)) {
                //3. 解析添加ad:focusable="true"的元素，并获取View焦点。
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                //4. 解析View的tag。
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                //5. 解析include标签，注意include标签不能作为根元素。
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                //merge标签必须为根元素
                throw new InflateException("<merge /> must be the root element");
            } else {
                //6. 根据元素名进行解析，生成View。
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                //7. 递归调用解析该View里的所有子View，也是深度优先遍历，rInflateChildren内部调用的也是rInflate()方
                //法，只是传入了新的parent View
                rInflateChildren(parser, view, attrs, true);
                //8. 将解析出来的View添加到它的父View中。
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            //9. 回调根容器的onFinishInflate()方法，这个方法我们应该很熟悉。
            parent.onFinishInflate();
        }
    }

    //rInflateChildren内部调用的也是rInflate()方法，只是传入了新的parent View
    final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }

}
```

上述方法描述了整个 View 树的解析流程，我们来概括一下：

1. 获取树的深度，执行深度优先遍历.
2. 逐个进行元素解析。
3. 解析添加 ad:focusable="true"的元素，并获取 View 焦点。
4. 解析 View 的 tag。
5. 解析 include 标签，注意 include 标签不能作为根元素，而 merge 必须作为根元素。
6. 根据元素名进行解析，生成 View。
7. 递归调用解析该 View 里的所有子 View，也是深度优先遍历，rInflateChildren 内部调用的也是 rInflate()方法，只是传入了新的 parent View。
8. 将解析出来的 View 添加到它的父 View 中。
9. 回调根容器的 onFinishInflate()方法，这个方法我们应该很熟悉。

你可以看到，负责解析单个 View 的正是 createViewFromTag()方法，我们再来分析下这个方法。

## 三 解析 View

```java
public abstract class LayoutInflater {

        View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
                boolean ignoreThemeAttr) {

            //1. 解析view标签。注意是小写view，这个不太常用，下面会说。
            if (name.equals("view")) {
                name = attrs.getAttributeValue(null, "class");
            }

            //2. 如果标签与主题相关，则需要将context与themeResId包裹成ContextThemeWrapper。
            if (!ignoreThemeAttr) {
                final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
                final int themeResId = ta.getResourceId(0, 0);
                if (themeResId != 0) {
                    context = new ContextThemeWrapper(context, themeResId);
                }
                ta.recycle();
            }

            //3. BlinkLayout是一种会闪烁的布局，被包裹的内容会一直闪烁，像QQ消息那样。
            if (name.equals(TAG_1995)) {
                // Let's party like it's 1995!
                return new BlinkLayout(context, attrs);
            }

            try {
                View view;

                //4. 用户可以设置LayoutInflater的Factory来进行View的解析，但是默认情况下
                //这些Factory都是为空的。
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

                //5. 默认情况下没有Factory，而是通过onCreateView()方法对内置View进行解析，createView()
                //方法进行自定义View的解析。
                if (view == null) {
                    final Object lastContext = mConstructorArgs[0];
                    mConstructorArgs[0] = context;
                    try {
                        //这里有个小技巧，因为我们在使用自定义View的时候是需要在xml指定全路径的，例如：
                        //com.guoxiaoxing.CustomView，那么这里就有个.了，可以利用这一点判定是内置View
                        //还是自定义View，Google的工程师很机智的。😎
                        if (-1 == name.indexOf('.')) {
                            //内置View解析
                            view = onCreateView(parent, name, attrs);
                        } else {
                            //自定义View解析
                            view = createView(name, null, attrs);
                        }
                    } finally {
                        mConstructorArgs[0] = lastContext;
                    }
                }

                return view;
            } catch (InflateException e) {
                throw e;

            } catch (ClassNotFoundException e) {
                final InflateException ie = new InflateException(attrs.getPositionDescription()
                        + ": Error inflating class " + name, e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;

            } catch (Exception e) {
                final InflateException ie = new InflateException(attrs.getPositionDescription()
                        + ": Error inflating class " + name, e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            }
        }
}
```

单个 View 的解析流程也很简单，我们来梳理一下：

1. 解析 View 标签。
2. 如果标签与主题相关，则需要将 context 与 themeResId 包裹成 ContextThemeWrapper。
3. BlinkLayout 是一种会闪烁的布局，被包裹的内容会一直闪烁，像 QQ 消息那样。
4. 用户可以设置 LayoutInflater 的 Factory 来进行 View 的解析，但是默认情况下这些 Factory 都是为空的。
5. 默认情况下没有 Factory，而是通过 onCreateView()方法对内置 View 进行解析，createView()方法进行自定义 View 的解析。这里有个小技巧，因为
   我们在使用自定义 View 的时候是需要在 xml 指定全路径的，例如：com.guoxiaoxing.CustomView，那么这里就有个.了，可以利用这一点判定是内置 View
   还是自定义 View，Google 的工程师很机智的。😎

**关于 view 标签**

```xml
<view
    class="RelativeLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

在使用时，相当于所有控件标签的父类一样，可以设置 class 属性，这个属性会决定 view 这个节点会是什么控件。

**关于 BlinkLayout**

这个也是个冷门的控件，本质上是一个 FrameLayout，被它包裹的控件会像电脑版的 QQ 小企鹅那样一直闪烁。

```xml
<blink
    android:layout_width="wrap_content"
    android:layout_height="wrap_content">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="这个控件会一直闪烁"/>

</blink>
```

**关于 onCreateView()与 createView()**

这两个方法在本质上都是一样的，只是 onCreateView()会给内置的 View 前面加一个前缀，例如：android.widget，方便开发者在写内置 View 的时候，不用谢全路径名。
前面我们也提到了 LayoutInflater 是一个抽象类，我们实际使用的 PhoneLayoutInflater，这个类的实现很简单，它重写了 LayoutInflater 的 onCreatView()方法，该
方法就是做了一个给内置 View 加前缀的事情。

```java
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };

    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {

        //循环遍历三种前缀，尝试创建View
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }
        return super.onCreateView(name, attrs);
    }
    public LayoutInflater cloneInContext(Context newContext) {
        return new PhoneLayoutInflater(this, newContext);
    }
}
```

这样一来，真正的 View 构建还是在 createView()方法里完成的，createView()主要根据完整的类的路径名利用反射机制构建 View 对象，我们具体来
看看 createView()方法的实现。

```java
public abstract class LayoutInflater {

    public final View createView(String name, String prefix, AttributeSet attrs)
                throws ClassNotFoundException, InflateException {

            //1. 从缓存中读取构造函数。
            Constructor<? extends View> constructor = sConstructorMap.get(name);
            if (constructor != null && !verifyClassLoader(constructor)) {
                constructor = null;
                sConstructorMap.remove(name);
            }
            Class<? extends View> clazz = null;

            try {
                Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

                if (constructor == null) {
                    // Class not found in the cache, see if it's real, and try to add it

                    //2. 没有在缓存中查找到构造函数，则构造完整的路径名，并加装该类。
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);

                    if (mFilter != null && clazz != null) {
                        boolean allowed = mFilter.onLoadClass(clazz);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    }
                    //3. 从Class对象中获取构造函数，并在sConstructorMap做下缓存，方便下次使用。
                    constructor = clazz.getConstructor(mConstructorSignature);
                    constructor.setAccessible(true);
                    sConstructorMap.put(name, constructor);
                } else {
                    //4. 如果sConstructorMap中有当前View构造函数的缓存，则直接使用。
                    if (mFilter != null) {
                        // Have we seen this name before?
                        Boolean allowedState = mFilterMap.get(name);
                        if (allowedState == null) {
                            // New class -- remember whether it is allowed
                            clazz = mContext.getClassLoader().loadClass(
                                    prefix != null ? (prefix + name) : name).asSubclass(View.class);

                            boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                            mFilterMap.put(name, allowed);
                            if (!allowed) {
                                failNotAllowed(name, prefix, attrs);
                            }
                        } else if (allowedState.equals(Boolean.FALSE)) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    }
                }

                Object[] args = mConstructorArgs;
                args[1] = attrs;

                //5. 利用构造函数，构建View对象。
                final View view = constructor.newInstance(args);
                if (view instanceof ViewStub) {
                    // Use the same context when inflating ViewStub later.
                    final ViewStub viewStub = (ViewStub) view;
                    viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
                }
                return view;

            } catch (NoSuchMethodException e) {
                final InflateException ie = new InflateException(attrs.getPositionDescription()
                        + ": Error inflating class " + (prefix != null ? (prefix + name) : name), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;

            } catch (ClassCastException e) {
                // If loaded class is not a View subclass
                final InflateException ie = new InflateException(attrs.getPositionDescription()
                        + ": Class is not a View " + (prefix != null ? (prefix + name) : name), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (ClassNotFoundException e) {
                // If loadClass fails, we should propagate the exception.
                throw e;
            } catch (Exception e) {
                final InflateException ie = new InflateException(
                        attrs.getPositionDescription() + ": Error inflating class "
                                + (clazz == null ? "<unknown>" : clazz.getName()), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }
}
```

好了，到这篇文章为止，我们对整个 Android 显示框架的原理分析就算是告一段落了，在这些文章里我们侧重的是 Client 端的分析，WindowManagerService、SurfaceFlinger 这些 Server 端的
并没有过多的涉及，因为对大部分开发者而言，扎实的掌握 Client 端的原理就足够了。等到你完全掌握了 Client 端的原理或者是需要进行 Android Framework 层的开发，可以进一步去深入 Server
端的原理。

关于 Android 显示框架主要包括五篇文章：

- [01Android 显示框架：Android 显示框架概述](./doc/Android系统应用框架篇/Android显示框架/01Android显示框架：Android显示框架概述.md)
- [02Android 显示框架：Android 应用视图的载体 View](./doc/Android系统应用框架篇/Android显示框架/02Android显示框架：Android应用视图载体View.md)
- [03Android 显示框架：Android 应用视图的管理者 Window](./doc/Android系统应用框架篇/Android显示框架/03Android显示框架：Android应用视图管理者Window.md)
- [04Android 显示框架：Android 应用窗口管理者 WindowManager](./doc/Android系统应用框架篇/Android显示框架/04Android显示框架：Android应用窗口管理者WindowManager.md)
- [05Android 显示框架：Android 布局解析者 LayoutInflater](./doc/Android系统应用框架篇/Android显示框架/05Android显示框架：Android布局解析者LayoutInflater.md)

后续我们会接着进行

- Android 组件框架
- Android 动画框架
- Android 通信框架
- Android 多媒体框架

等 Android 子系统的分析，后续的内容可以关注[Android open source project analysis](https://github.com/guoxiaoxing/android-open-source-project-analysis)项目。
