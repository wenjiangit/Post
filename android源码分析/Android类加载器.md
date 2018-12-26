

# Android中有哪几种ClassLoader?他们的作用是什么?

与``java`` 类似,``Android`` 中也有相应的类加载机制,只是``java`` 加载的是``class`` 字节码文件.而Android中记载的是``dex`` 字节码,继承自``ClassLoader`` 抽象类有以下几种:

- ``BootClassLoader`` ,是``ClassLoader`` 的内部类,在系统启动时用来加载一些系统相关的类

  ​

- ``PathClassLoader`` 

  官方说明:

  > 提供一个简单的``ClassLoader`` 实现，用来操作文件列表和本地文件系统中的文件和目录，但不会尝试 从网络加载类。``Android`` 使用这个类作为它的系统类加载器和其应用程序类加载器。

- ``DexClassLoader`` 

  官方说明:

  > 一个类加载器，用于加载包含一个``classes.dex`` 条目的`` .jar`` 和`` .apk`` 文件中的类,这个可以用来执行不是作为应用程序安装的代码.

``PathClassLoader`` 和``DexClassLoader`` 都继承自``BaseDexClassLoader`` ,接下来看一下源码

```java
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code DexClassLoader} that finds interpreted and native
     * code.  Interpreted classes are found in a set of DEX files contained
     * in Jar or APK files.
     *
     * <p>The path lists are separated using the character specified by the
     * {@code path.separator} system property, which defaults to {@code :}.
     *
     * @param dexPath the list of jar/apk files containing classes and
     *     resources, delimited by {@code File.pathSeparator}, which
     *     defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     *     should be written; must not be {@code null}
     * @param librarySearchPath the list of directories containing native
     *     libraries, delimited by {@code File.pathSeparator}; may be
     *     {@code null}
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
}
```

``DexClassLoader`` 只有一个构造方法,基本实现都在``BaseDexClassLoader`` 中,

参数列表

- ``dexPath``  包含``jar`` 或``apk`` 文件路径的字符串,多个文件之间用":"隔开
- ``optimizedDirectory`` : 优化后的``dex`` 存储路径.
- ``librarySearchPath`` : 本地库文件路径
- ``parent`` : 关联的父类加载器

接下来我们看一下``BaseDexClassLoader`` ,

```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    /**
     * Constructs an instance.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     * should be written; may be {@code null}
     * @param librarySearchPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath,                    optimizedDirectory);
    }
```

在``BaseDexClassLoader`` 的构造方法中实例化了一个``DexPathList`` ,接下来看看这个类是干嘛的

```java
 public DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory) {

        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath),                                                          optimizedDirectory,
                                           suppressedExceptions,                                                            definingContext);

```

调用``makeDexElements()`` 初始化``dexElements`` 

```java
private static Element[] makeElements(List<File> files, File optimizedDirectory,
                                          List<IOException>                                                               suppressedExceptions,
                                          boolean ignoreDexFiles,
                                          ClassLoader loader) {
        Element[] elements = new Element[files.size()];
        int elementsPos = 0;
        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        //遍历文件列表
        for (File file : files) {
            File zip = null;
            File dir = new File("");
            DexFile dex = null;
            String path = file.getPath();
            String name = file.getName();

            if (path.contains(zipSeparator)) {
                String split[] = path.split(zipSeparator, 2);
                zip = new File(split[0]);
                dir = new File(split[1]);
            } else if (file.isDirectory()) {
                // We support directories for looking up resources and native libraries.
                // Looking up resources in directories is useful for running libcore tests.
                elements[elementsPos++] = new Element(file, true, null, null);
            } else if (file.isFile()) {
                if (!ignoreDexFiles && name.endsWith(DEX_SUFFIX)) {
                    // Raw dex file (not inside a zip/jar).
                    try {
                      //生成DexFile
                        dex = loadDexFile(file, optimizedDirectory, loader, elements);
                    } catch (IOException suppressed) {
                        System.logE("Unable to load dex file: " + file, suppressed);
                        suppressedExceptions.add(suppressed);
                    }
                } else {
                    zip = file;

                    if (!ignoreDexFiles) {
                        try {
                            dex = loadDexFile(file, optimizedDirectory, loader, elements);
                        } catch (IOException suppressed) {
                            /*
                             * IOException might get thrown "legitimately" by the DexFile constructor if
                             * the zip file turns out to be resource-only (that is, no classes.dex file
                             * in it).
                             * Let dex == null and hang on to the exception to add to the tea-leaves for
                             * when findClass returns null.
                             */
                            suppressedExceptions.add(suppressed);
                        }
                    }
                }
            } else {
                System.logW("ClassLoader referenced unknown path: " + file);
            }

            if ((zip != null) || (dex != null)) {
              //根据zip或dex文件新建Element对象,并添加到数组中
                elements[elementsPos++] = new Element(dir, false, zip, dex);
            }
        }
        if (elementsPos != elements.length) {
            elements = Arrays.copyOf(elements, elementsPos);
        }
        return elements;
    }

```

``loadDexFile`` 是如何生成``DexFile`` 实例的?

````java
 private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
                                       Element[] elements)
            throws IOException {
        if (optimizedDirectory == null) {
          //如果优化dex路径为空,直接调用构造方法
            return new DexFile(file, loader, elements);
        } else {
          //根据优化文件夹路径和文件名生成文件路径
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
    }

````

```java
  static DexFile loadDex(String sourcePathName, String outputPathName,
        int flags, ClassLoader loader, DexPathList.Element[] elements) throws IOException {

        /*
         * TODO: we may want to cache previously-opened DexFile objects.
         * The cache would be synchronized with close().  This would help
         * us avoid mapping the same DEX more than once when an app
         * decided to open it multiple times.  In practice this may not
         * be a real issue.
         */
        return new DexFile(sourcePathName, outputPathName, flags, loader, elements);
    }
```

这里待优化,所以最后还是调用了构造函数

```
  #BaseDexClassLoader#findClass
  @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }

```

重点关注``BaseClassLoader`` 是如何加载类,通过类加载的双亲委派模型,最终会调用该``ClassLoader`` 

的``findClass`` 方法,这里又调用了``pathList`` 的``findClass`` ,

```java
#DexPathList#findClass
public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }

```

可以发现是通过遍历``dexElements`` 数组来加载类的.

由于类加载是按需加载的,现在的热修复技术基本上都是基于此,通过加载修复好的``dex`` 文件,并将其放到``dexElement`` 数组前面,下次加载就能够加载到修复好的类了.

参考:

- [类加载机制系列2——深入理解Android中的类加载器](https://www.jianshu.com/p/7193600024e7 ) 
- [Android中ClassLoader源码解析之真的是你认为的ClassLoader](http://blog.csdn.net/u010014658/article/details/52576443 )

