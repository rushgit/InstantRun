#Android Instant原理剖析

##Instant Run介绍
简单介绍一下，Instant Run是Android Studio2.0新增的一个运行机制，能显著减少第一次以后的建构和部署时间。第一次打包时全量编译,第二次及以后可以只编译修改的代码,快速应用到手机上,减少重新打包和安装的时间。

###传统编译流程
![](pic/1.png)<br>
*修改代码 -> 完整编译 -> 部署/安装 -> 重启App -> 重启Activity*<br>

###Instant Run编译流程
![](pic/2.png)<br>
*修改代码 -> 增量编译修改的代码 -> 热部署，温部署，冷部署*<br>

####热部署
Incremental code changes are applied and reflected in the app without needing to relaunch the app or even restart the current Activity. Can be used for most simple changes within method implementations.<br>
增量代码应用到app上,无需重启App和Activity,只适用方法内修改这种简单场景。

####温部署
The Activity needs to be restarted before changes can be seen and used. Typically required for changes to resources.<br>
Activity需要重启才生效,典型的场景是资源发生变化。

####冷部署
The app is restarted (but still not reinstalled). Required for any structural changes such as to inheritance or method signatures.<br>
App需要重启,但不需要重新安装。适用于代码结构和方法签名变化等场景。

##Demo分析
反编译Instant Run建构的Apk,dex文件内容如下<br>
![](pic/3.png)<br>
你会看到只有com.android.build.gradle.internal.incremental和com.android.tools2个包,并没有demo app的代码,这其实是Instant Run的框架代码,AppInfo.class中有个applicationId,值是demo app的包名,再看apk的结构,如下图<br>
![](pic/4.jpg)<br>
发现多了一个instant-run.zip文件,再看AndroidManifest.xml<br>
![](pic/5.jpg)<br>
看到App的Application被替换成了com.android.tools.fd.runtime.BootstrapApplication,那app的application在哪里呢? 继续看instant-run.zip<br>
![](pic/6.jpg)<br>
原来app被打包成了多个dex文件。

看到这里我们可以猜想大概的原理:<br>
1. Instant Run框架作为一个宿主程序<br>
2. app被编译成了多个dex文件,打包在instant-run.zip文件中。<br>
3. app启动过程中com.android.tools.fd.runtime.BootstrapApplication动态调用instant-run中的dex文件,执行具体的业务逻辑。<br>
恩,这和App加壳的原理很像,下面具体分析源码。

##Instant Run启动过程
首先看app的入口BootstrapApplication,先看attachBaseContext()

###1. attachBaseContext()
```java
    protected void attachBaseContext(Context context) {
            // As of Marshmallow, we use APK splits and don't need to rely on
            // reflection to inject classes and resources for coldswap
            //noinspection PointlessBooleanExpression
            if (!AppInfo.usingApkSplits) {
                String apkFile = context.getApplicationInfo().sourceDir;
                long apkModified = apkFile != null ? new File(apkFile).lastModified() : 0L;
                createResources(apkModified);
                setupClassLoaders(context, context.getCacheDir().getPath(), apkModified);
            }

            createRealApplication();

            // This is called from ActivityThread#handleBindApplication() -> LoadedApk#makeApplication().
            // Application#mApplication is changed right after this call, so we cannot do the monkey
            // patching here. So just forward this method to the real Application instance.
            super.attachBaseContext(context);

            if (realApplication != null) {
                try {
                    Method attachBaseContext =
                            ContextWrapper.class.getDeclaredMethod("attachBaseContext", Context.class);
                    attachBaseContext.setAccessible(true);
                    attachBaseContext.invoke(realApplication, context);
                } catch (Exception e) {
                    throw new IllegalStateException(e);
                }
            }
    }
```

依次看createResources -> setupClassLoaders -> createRealApplication -> 调用real application的attachBaseContext方法

####1.1 createResources()
```java
private void createResources(long apkModified) {
        // Look for changes stashed in the inbox folder while the server was not running
        FileManager.checkInbox();

        File file = FileManager.getExternalResourceFile();
        externalResourcePath = file != null ? file.getPath() : null;

        if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
            Log.v(LOG_TAG, "Resource override is " + externalResourcePath);
        }

        if (file != null) {
            try {
                long resourceModified = file.lastModified();
                if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Resource patch last modified: " + resourceModified);
                    Log.v(LOG_TAG, "APK last modified: " + apkModified + " " +
                            (apkModified > resourceModified ? ">" : "<") + " resource patch");
                }

                if (apkModified == 0L || resourceModified <= apkModified) {
                    if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                        Log.v(LOG_TAG, "Ignoring resource file, older than APK");
                    }
                    externalResourcePath = null;
                }
            } catch (Throwable t) {
                Log.e(LOG_TAG, "Failed to check patch timestamps", t);
            }
        }
    }
```
该方法主要判断外部资源有没有更新,并把路径存储在externalResourcePath变量上。

####1.2 setupClassLoaders()
```java
private static void setupClassLoaders(Context context, String codeCacheDir, long apkModified) {
        List<String> dexList = FileManager.getDexList(context, apkModified);

        // Make sure class loader finds these
        @SuppressWarnings("unused") Class<Server> server = Server.class;
        @SuppressWarnings("unused") Class<MonkeyPatcher> patcher = MonkeyPatcher.class;

        if (!dexList.isEmpty()) {
            if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                Log.v(LOG_TAG, "Bootstrapping class loader with dex list " + join('\n', dexList));
            }

            ClassLoader classLoader = BootstrapApplication.class.getClassLoader();
            String nativeLibraryPath;
            try {
                nativeLibraryPath = (String) classLoader.getClass().getMethod("getLdLibraryPath")
                                .invoke(classLoader);
                if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Native library path: " + nativeLibraryPath);
                }
            } catch (Throwable t) {
                Log.e(LOG_TAG, "Failed to determine native library path " + t.getMessage());
                nativeLibraryPath = FileManager.getNativeLibraryFolder().getPath();
            }
            IncrementalClassLoader.inject(
                    classLoader,
                    nativeLibraryPath,
                    codeCacheDir,
                    dexList);
        } else {
            Log.w(LOG_TAG, "No instant run dex files added to classpath");
        }
    }
```
调用了IncrementalClassLoader#inject()方法,IncrementalClassLoader源码如下:
```java
/**
 * A class loader that loads classes from any .dex file in a particular directory on the SD card.
 * <p>
 * <p>Used to implement incremental deployment to Android phones.
 */
public class IncrementalClassLoader extends ClassLoader {
    /** When false, compiled out of runtime library */
    public static final boolean DEBUG_CLASS_LOADING = false;

    private final DelegateClassLoader delegateClassLoader;

    public IncrementalClassLoader(
            ClassLoader original, String nativeLibraryPath, String codeCacheDir, List<String> dexes) {
        super(original.getParent());

        // TODO(bazel-team): For some mysterious reason, we need to use two class loaders so that
        // everything works correctly. Investigate why that is the case so that the code can be
        // simplified.
        delegateClassLoader = createDelegateClassLoader(nativeLibraryPath, codeCacheDir, dexes,
                original);
    }

    @Override
    public Class<?> findClass(String className) throws ClassNotFoundException {
        try {
            Class<?> aClass = delegateClassLoader.findClass(className);
            //noinspection PointlessBooleanExpression,ConstantConditions
            if (DEBUG_CLASS_LOADING && Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                Log.v(LOG_TAG, "Incremental class loader: findClass(" + className + ") = " + aClass);
            }

            return aClass;
        } catch (ClassNotFoundException e) {
            //noinspection PointlessBooleanExpression,ConstantConditions
            if (DEBUG_CLASS_LOADING && Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                Log.v(LOG_TAG, "Incremental class loader: findClass(" + className + ") : not found");
            }
            throw e;
        }
    }

    /**
     * A class loader whose only purpose is to make {@code findClass()} public.
     */
    private static class DelegateClassLoader extends BaseDexClassLoader {
        private DelegateClassLoader(
                String dexPath, File optimizedDirectory, String libraryPath, ClassLoader parent) {
            super(dexPath, optimizedDirectory, libraryPath, parent);
        }

        @Override
        public Class<?> findClass(String name) throws ClassNotFoundException {
            try {
                Class<?> aClass = super.findClass(name);
                //noinspection PointlessBooleanExpression,ConstantConditions
                if (DEBUG_CLASS_LOADING && Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Delegate class loader: findClass(" + name + ") = " + aClass);
                }

                return aClass;
            } catch (ClassNotFoundException e) {
                //noinspection PointlessBooleanExpression,ConstantConditions
                if (DEBUG_CLASS_LOADING && Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Delegate class loader: findClass(" + name + ") : not found");
                }
                throw e;
            }
        }
    }

    private static DelegateClassLoader createDelegateClassLoader(
            String nativeLibraryPath, String codeCacheDir, List<String> dexes,
            ClassLoader original) {
        String pathBuilder = createDexPath(dexes);
        return new DelegateClassLoader(pathBuilder, new File(codeCacheDir),
                nativeLibraryPath, original);
    }

    @NonNull
    private static String createDexPath(List<String> dexes) {
        StringBuilder pathBuilder = new StringBuilder();
        boolean first = true;
        for (String dex : dexes) {
            if (first) {
                first = false;
            } else {
                pathBuilder.append(File.pathSeparator);
            }

            pathBuilder.append(dex);
        }

        if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
            Log.v(LOG_TAG, "Incremental dex path is "
                    + BootstrapApplication.join('\n', dexes));
        }
        return pathBuilder.toString();
    }

    private static void setParent(ClassLoader classLoader, ClassLoader newParent) {
        try {
            Field parent = ClassLoader.class.getDeclaredField("parent");
            parent.setAccessible(true);
            parent.set(classLoader, newParent);
        } catch (IllegalArgumentException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        }
    }

    public static ClassLoader inject(
            ClassLoader classLoader, String nativeLibraryPath, String codeCacheDir,
            List<String> dexes) {
        IncrementalClassLoader incrementalClassLoader =
                new IncrementalClassLoader(classLoader, nativeLibraryPath, codeCacheDir, dexes);
        setParent(classLoader, incrementalClassLoader);

        return incrementalClassLoader;
    }
}
```
这里需要先介绍一下ClassLoader的双亲委派模型(Parent Delegation Model),其工作过程是这样的:<br>
如果一个类加载器收到了类加载的请求,它首先不会自己去尝试加载这个类,优先委派给父加载器去加载,每一层都向上委派,直到顶层的类加载器(在Android里是BootClassLoader),只有当父类无法完成加载时,子类才会尝试去加载。<br><br>
App的类加载器是PathClassLoader,其父加载器为BootClassLoader,Instant Run通过inject,改变PathClassLoader的父加载器为IncrementalClassLoader,从这里可以知道App的dex最终是通过IncrementalClassLoader来加载的,inject前后结构如下图所示:<br>
![](pic/7.png)<br>

在代码中验证一下:
```java
public void onCreate() {
        super.onCreate();
        ClassLoader loader = getClassLoader();
        Log.i("TAG", "class loader: " + loader.getClass().getName());
        while ((loader = loader.getParent()) != null) {
            Log.i("TAG", "parent class loader: " + loader.getClass().getName());
        }
    }
```
运行结果:
```java
TAG     : class loader: dalvik.system.PathClassLoader
TAG     : parent class loader: com.android.tools.fd.runtime.IncrementalClassLoader
TAG     : parent class loader: java.lang.BootClassLoader
```
继续分析createRealApplication()

####1.3 createRealApplication()
```java
private void createRealApplication() {
        if (AppInfo.applicationClass != null) {
            if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                Log.v(LOG_TAG, "About to create real application of class name = " +
                        AppInfo.applicationClass);
            }

            try {
                @SuppressWarnings("unchecked")
                Class<? extends Application> realClass =
                        (Class<? extends Application>) Class.forName(AppInfo.applicationClass);
                if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Created delegate app class successfully : " + realClass +
                            " with class loader " + realClass.getClassLoader());
                }
                Constructor<? extends Application> constructor = realClass.getConstructor();
                realApplication = constructor.newInstance();
                if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Created real app instance successfully :" + realApplication);
                }
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        } else {
            realApplication = new Application();
        }
    }
```
该方式通过反射的方法创建出realApplication类对象,此真实类名记录在上面反编译截图看到的AppInfo.class的applicationClass字段里,如果这个字段为空,说明app没有自定义Application类,会创建系统的Application对象。

####1.4 real application的attachBaseContext方法
attachBaseContext的最后一步是反射调用realApplication的attachBaseContext。<br>
BootstrapApplication的attachBaseContext方法分析结束,下面分析onCreate方法

###2. onCreate
源码如下:
```java
public void onCreate() {
        // As of Marshmallow, we use APK splits and don't need to rely on
        // reflection to inject classes and resources for coldswap
        //noinspection PointlessBooleanExpression
        if (!AppInfo.usingApkSplits) {
            MonkeyPatcher.monkeyPatchApplication(
                    BootstrapApplication.this, BootstrapApplication.this,
                    realApplication, externalResourcePath);
            MonkeyPatcher.monkeyPatchExistingResources(BootstrapApplication.this,
                    externalResourcePath, null);
        } else {
            // We still need to set the application instance in the LoadedApk etc
            // such that getApplication() returns the new application
            MonkeyPatcher.monkeyPatchApplication(
                    BootstrapApplication.this, BootstrapApplication.this,
                    realApplication, null);
        }
        super.onCreate();

        // Start server, unless we're in a multiprocess scenario and this isn't the
        // primary process
        if (AppInfo.applicationId != null) {
            try {
                boolean foundPackage = false;
                int pid = Process.myPid();
                ActivityManager manager = (ActivityManager) getSystemService(
                        Context.ACTIVITY_SERVICE);
                List<RunningAppProcessInfo> processes = manager.getRunningAppProcesses();

                boolean startServer;
                if (processes != null && processes.size() > 1) {
                    // Multiple processes: look at each, and if the process name matches
                    // the package name (for the current pid), it's the main process.
                    startServer = false;
                    for (RunningAppProcessInfo processInfo : processes) {
                        if (AppInfo.applicationId.equals(processInfo.processName)) {
                            foundPackage = true;
                            if (processInfo.pid == pid) {
                                startServer = true;
                                break;
                            }
                        }
                    }
                    if (!startServer && !foundPackage) {
                        // Safety check: If for some reason we didn't even find the main package,
                        // start the server anyway. This safeguards against apps doing strange
                        // things with the process name.
                        startServer = true;
                        if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                            Log.v(LOG_TAG, "Multiprocess but didn't find process with package: "
                                    + "starting server anyway");
                        }
                    }
                } else {
                    // If there is only one process, start the server.
                    startServer = true;
                }

                if (startServer) {
                    Server.create(AppInfo.applicationId, BootstrapApplication.this);
                }
            } catch (Throwable t) {
                if (Log.isLoggable(LOG_TAG, Log.VERBOSE)) {
                    Log.v(LOG_TAG, "Failed during multi process check", t);
                }
                Server.create(AppInfo.applicationId, BootstrapApplication.this);
            }
        }

        if (realApplication != null) {
            realApplication.onCreate();
        }
    }
```
我们依次分析 monkeyPatchApplication -> monkeyPatchExistingResources -> start server -> 调用realApplication#onCreate()

####2.1 monkeyPatchExistingResources
```java
public static void monkeyPatchApplication(@Nullable Context context,
                                              @Nullable Application bootstrap,
                                              @Nullable Application realApplication,
                                              @Nullable String externalResourceFile) {
        // BootstrapApplication is created by reflection in Application#handleBindApplication() ->
        // LoadedApk#makeApplication(), and its return value is used to set the Application field in all
        // sorts of Android internals.
        //
        // Fortunately, Application#onCreate() is called quite soon after, so what we do is monkey
        // patch in the real Application instance in BootstrapApplication#onCreate().
        //
        // A few places directly use the created Application instance (as opposed to the fields it is
        // eventually stored in). Fortunately, it's easy to forward those to the actual real
        // Application class.
        try {
            // Find the ActivityThread instance for the current thread
            Class<?> activityThread = Class.forName("android.app.ActivityThread");
            Object currentActivityThread = getActivityThread(context, activityThread);

            // Find the mInitialApplication field of the ActivityThread to the real application
            Field mInitialApplication = activityThread.getDeclaredField("mInitialApplication");
            mInitialApplication.setAccessible(true);
            Application initialApplication = (Application) mInitialApplication.get(currentActivityThread);
            if (realApplication != null && initialApplication == bootstrap) {
                mInitialApplication.set(currentActivityThread, realApplication);
            }

            // Replace all instance of the stub application in ActivityThread#mAllApplications with the
            // real one
            if (realApplication != null) {
                Field mAllApplications = activityThread.getDeclaredField("mAllApplications");
                mAllApplications.setAccessible(true);
                List<Application> allApplications = (List<Application>) mAllApplications
                        .get(currentActivityThread);
                for (int i = 0; i < allApplications.size(); i++) {
                    if (allApplications.get(i) == bootstrap) {
                        allApplications.set(i, realApplication);
                    }
                }
            }

            // Figure out how loaded APKs are stored.

            // API version 8 has PackageInfo, 10 has LoadedApk. 9, I don't know.
            Class<?> loadedApkClass;
            try {
                loadedApkClass = Class.forName("android.app.LoadedApk");
            } catch (ClassNotFoundException e) {
                loadedApkClass = Class.forName("android.app.ActivityThread$PackageInfo");
            }
            Field mApplication = loadedApkClass.getDeclaredField("mApplication");
            mApplication.setAccessible(true);
            Field mResDir = loadedApkClass.getDeclaredField("mResDir");
            mResDir.setAccessible(true);

            // 10 doesn't have this field, 14 does. Fortunately, there are not many Honeycomb devices
            // floating around.
            Field mLoadedApk = null;
            try {
                mLoadedApk = Application.class.getDeclaredField("mLoadedApk");
            } catch (NoSuchFieldException e) {
                // According to testing, it's okay to ignore this.
            }

            // Enumerate all LoadedApk (or PackageInfo) fields in ActivityThread#mPackages and
            // ActivityThread#mResourcePackages and do two things:
            //   - Replace the Application instance in its mApplication field with the real one
            //   - Replace mResDir to point to the external resource file instead of the .apk. This is
            //     used as the asset path for new Resources objects.
            //   - Set Application#mLoadedApk to the found LoadedApk instance
            for (String fieldName : new String[]{"mPackages", "mResourcePackages"}) {
                Field field = activityThread.getDeclaredField(fieldName);
                field.setAccessible(true);
                Object value = field.get(currentActivityThread);

                for (Map.Entry<String, WeakReference<?>> entry :
                        ((Map<String, WeakReference<?>>) value).entrySet()) {
                    Object loadedApk = entry.getValue().get();
                    if (loadedApk == null) {
                        continue;
                    }

                    if (mApplication.get(loadedApk) == bootstrap) {
                        if (realApplication != null) {
                            mApplication.set(loadedApk, realApplication);
                        }
                        if (externalResourceFile != null) {
                            mResDir.set(loadedApk, externalResourceFile);
                        }

                        if (realApplication != null && mLoadedApk != null) {
                            mLoadedApk.set(realApplication, loadedApk);
                        }
                    }
                }
            }
        } catch (Throwable e) {
            throw new IllegalStateException(e);
        }
    }
```
该方法把app中所有的applicaton替换为realApplication,包括:<br>
**1.替换ActivityThread的mInitialApplication和mAllApplications为realApplication**<br>
**2.替换ActivityThread的mPackages和mResourcePackages中的mLoaderApk中的mAllApplication为realApplication**

####2.2 monkeyPatchExistingResources
```java
public static void monkeyPatchExistingResources(@Nullable Context context,
                                                    @Nullable String externalResourceFile,
                                                    @Nullable Collection<Activity> activities) {
        if (externalResourceFile == null) {
            return;
        }

        try {
            // Create a new AssetManager instance and point it to the resources installed under
            // /sdcard
            AssetManager newAssetManager = AssetManager.class.getConstructor().newInstance();
            Method mAddAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
            mAddAssetPath.setAccessible(true);
            if (((Integer) mAddAssetPath.invoke(newAssetManager, externalResourceFile)) == 0) {
                throw new IllegalStateException("Could not create new AssetManager");
            }

            // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
            // in L, so we do it unconditionally.
            Method mEnsureStringBlocks = AssetManager.class.getDeclaredMethod("ensureStringBlocks");
            mEnsureStringBlocks.setAccessible(true);
            mEnsureStringBlocks.invoke(newAssetManager);

            if (activities != null) {
                for (Activity activity : activities) {
                    Resources resources = activity.getResources();

                    try {
                        Field mAssets = Resources.class.getDeclaredField("mAssets");
                        mAssets.setAccessible(true);
                        mAssets.set(resources, newAssetManager);
                    } catch (Throwable ignore) {
                        Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                        mResourcesImpl.setAccessible(true);
                        Object resourceImpl = mResourcesImpl.get(resources);
                        Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                        implAssets.setAccessible(true);
                        implAssets.set(resourceImpl, newAssetManager);
                    }

                    Resources.Theme theme = activity.getTheme();
                    try {
                        try {
                            Field ma = Resources.Theme.class.getDeclaredField("mAssets");
                            ma.setAccessible(true);
                            ma.set(theme, newAssetManager);
                        } catch (NoSuchFieldException ignore) {
                            Field themeField = Resources.Theme.class.getDeclaredField("mThemeImpl");
                            themeField.setAccessible(true);
                            Object impl = themeField.get(theme);
                            Field ma = impl.getClass().getDeclaredField("mAssets");
                            ma.setAccessible(true);
                            ma.set(impl, newAssetManager);
                        }

                        Field mt = ContextThemeWrapper.class.getDeclaredField("mTheme");
                        mt.setAccessible(true);
                        mt.set(activity, null);
                        Method mtm = ContextThemeWrapper.class.getDeclaredMethod("initializeTheme");
                        mtm.setAccessible(true);
                        mtm.invoke(activity);

                        if (SDK_INT < 24) { // As of API 24, mTheme is gone (but updates work
                                            // without these changes
                            Method mCreateTheme = AssetManager.class
                                    .getDeclaredMethod("createTheme");
                            mCreateTheme.setAccessible(true);
                            Object internalTheme = mCreateTheme.invoke(newAssetManager);
                            Field mTheme = Resources.Theme.class.getDeclaredField("mTheme");
                            mTheme.setAccessible(true);
                            mTheme.set(theme, internalTheme);
                        }
                    } catch (Throwable e) {
                        Log.e(LOG_TAG, "Failed to update existing theme for activity " + activity,
                                e);
                    }

                    pruneResourceCaches(resources);
                }
            }

            // Iterate over all known Resources objects
            Collection<WeakReference<Resources>> references;
            if (SDK_INT >= KITKAT) {
                // Find the singleton instance of ResourcesManager
                Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
                Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance");
                mGetInstance.setAccessible(true);
                Object resourcesManager = mGetInstance.invoke(null);
                try {
                    Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
                    fMActiveResources.setAccessible(true);
                    @SuppressWarnings("unchecked")
                    ArrayMap<?, WeakReference<Resources>> arrayMap =
                            (ArrayMap<?, WeakReference<Resources>>) fMActiveResources.get(resourcesManager);
                    references = arrayMap.values();
                } catch (NoSuchFieldException ignore) {
                    Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
                    mResourceReferences.setAccessible(true);
                    //noinspection unchecked
                    references = (Collection<WeakReference<Resources>>) mResourceReferences.get(resourcesManager);
                }
            } else {
                Class<?> activityThread = Class.forName("android.app.ActivityThread");
                Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
                fMActiveResources.setAccessible(true);
                Object thread = getActivityThread(context, activityThread);
                @SuppressWarnings("unchecked")
                HashMap<?, WeakReference<Resources>> map =
                        (HashMap<?, WeakReference<Resources>>) fMActiveResources.get(thread);
                references = map.values();
            }
            for (WeakReference<Resources> wr : references) {
                Resources resources = wr.get();
                if (resources != null) {
                    // Set the AssetManager of the Resources instance to our brand new one
                    try {
                        Field mAssets = Resources.class.getDeclaredField("mAssets");
                        mAssets.setAccessible(true);
                        mAssets.set(resources, newAssetManager);
                    } catch (Throwable ignore) {
                        Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                        mResourcesImpl.setAccessible(true);
                        Object resourceImpl = mResourcesImpl.get(resources);
                        Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                        implAssets.setAccessible(true);
                        implAssets.set(resourceImpl, newAssetManager);
                    }

                    resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
                }
            }
        } catch (Throwable e) {
            throw new IllegalStateException(e);
        }
    }
```
该方法是替换app中所有AssetManager为newAssetManager,具体如下:
**1.当资源文件(resource.ap_)有变化时,新建一个newAssetManager对象,替换当前Resource,Resource.Theme中的mAssets成员变量为newAssetManager**<br>
**2.遍历所有已启动的Activity,把其mResources成员变量的mAssets替换为newAssetManager对象**

####2.3 start server
判断Server是否启动,如未启动,则启动一个Server用于监听IDE的消息。

####2.4 调用realApplication的onCreate方法
Application和AssetManager均已替换,调用realApplication的onCreate方法。<br>
到这一步为止,app已经正常启动了,下一步分析Server是如何监听并处理热部署、温部署和冷部署的。

###3. Server监听和处理热部署、温部署和冷部署

##参考文档
[1].[Instant Run: How Does it Work?!](https://medium.com/google-developers/instant-run-how-does-it-work-294a1633367f#.9q7cddaie)<br>
[2].[Android Instant Run原理分析](https://github.com/nuptboyzhb/AndroidInstantRun)<br>
[3].[Instant Run原理解析](http://www.jianshu.com/p/0400fb58d086)<br>