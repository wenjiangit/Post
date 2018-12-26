# RecyclerView中Item不能居中显示

**问题**

使用``View.inflate()``加载``RecyclerView``的``item``布局时,导致``item``内容无法居中显示.

```java
   @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View inflate = View.inflate(parent.getContext(), R.layout.rv_item_dance_indicator, null);
        return new ViewHolder(inflate);
    }
```

当时脑子抽风了,想偷个懒,用这种方式加载item布局,结果发现item竟然不能居中显示,一直以为是item布局的问题,这里把布局文件也贴一下.

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:paddingEnd="12dp"
    android:paddingStart="12dp"
    >

    <ImageView
        android:layout_gravity="center_vertical"
        android:id="@+id/iv_indicator"
        android:src="@drawable/indicator_selector"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        tools:ignore="ContentDescription" />

</FrameLayout>
```

各种尝试还是找不到原因,真无语了,怎么就居中不了呢?

最后还是百度大法,搜索关键字**recyclerview item不能居中显示**,出来一大堆,说什么的都有,比如更改一下根布局,改成线性布局啊,一看有不靠谱,最后还是找到一点有用的信息,[原文地址](https://blog.csdn.net/qq_24229973/article/details/52980743)

**解决方案**

动手一试,改成如下代码,果然成了,可喜可贺.

```java
   @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View inflate = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.rv_item_dance_indicator, parent, false);
        return new ViewHolder(inflate);
    }
```

当然学习不能不能只知其然而不知其所以然,抱着这个心态来瞄一瞄源码,这两种加载方式的区别到底在哪,以后到底该怎么使用?

**源码分析** 

``LayoutInflater.from``

```java
 /**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```

调用系统服务得到一个布局加载器.

``LayoutInflater.inflate``

```java
 public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }
        //传入布局资源id,并获得一个xml解析器,用以解析xml布局资源
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            //调用inflate开始解析xml
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```
inflate方法接收3个参数
- ``int resource ``                          需要加载的布局资源文件
- ``ViewGroup root``                      父布局
- ``boolean attachToRoot``           是否需要添加进父布局

然后调用内部的inflate方法

```
inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) 
```

```java
....
//这里只贴关键代码
// Temp is the root view that was found in the xml
                    //解析xml标签并返回一个view
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;
                   //root是最初传进来的父view
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        //如果父view不为空,则会生成相应的布局参数
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            //如果attachToRoot为false,为view设置布局参数
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    //递归加载子view
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    //如果父view不为空且attachToRoot为true,则将生成的view添加到父view中
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
```

这里可以看出你传不传root很重要,如果传了,会为加载的view设置布局参数,并且attchToRoot为ture还会将加载的布局添加到父view中.

在看一下``generateLayoutParams``方法

```java
public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new LayoutParams(getContext(), attrs);
    }
```

```java
        public LayoutParams(Context c, AttributeSet attrs) {
            TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
            setBaseAttributes(a,
                    R.styleable.ViewGroup_Layout_layout_width,
                    R.styleable.ViewGroup_Layout_layout_height);
            a.recycle();
        }
```

这里是实例化一个布局参数并读取``xml``中设置的宽高进行初始化,所以只有当你传入了父``view``,``xml``中定义``layout_width``和``layout_height``才有效.否则给的应该是一个默认值.应该都是``wrap_content``,这里只是做一个猜测.

``View.inflate()`` 

```java
 public static View inflate(Context context, @LayoutRes int resource, ViewGroup root) {
        LayoutInflater factory = LayoutInflater.from(context);
        return factory.inflate(resource, root);
    }

```

