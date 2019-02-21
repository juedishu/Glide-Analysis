# Glide分析
发展到现在Android的图片加载框架越来越成熟从一开始的UIL到后来的Glide、Picasso、fresco等。相比较而言Glide的功能更加强大，但是相应的方法数也是最多的。当然也得结合项目的需求还选择相应的框架。Glide比较吸引人的主要还是与Activity生命周期进行绑定、加载gif图片、可以配置相应的网络请求框架。看完本文你也许会知道以下几个问题:

    1.glide如何加载图片
    2.glide如何加载gif
    3.glide如何根据Activity的生命周期加载照片
    4.glide如何解决listView图片错乱问题
    5.glide的缓存策略

## 一、简单用法

    Glide.with(context).load("图片url").into(imageView)

当然这也是最简单的用法，一行代码就搞定了。比较详细的用法可以点击这个链接。
## 二、Glide的初始化

1、Glide.initializeGlide()

    private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    //获取使用注解自定义的AppGlideModule
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    //获取在manifest文件中配置的AppGlideModule
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
    manifestModules = new ManifestParser(applicationContext).parse();
    }
    //省略部分代码...
    //创建glide实例
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.registerComponents(applicationContext, glide, glide.registry);
    }
    if (annotationGeneratedModule != null) {
    annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide = glide;
    }
这里首先会获取自定义的AppGlideModule,然后在创建glide实例。这里的glide是个单例只会初始化一次。

2、GlideBuilder.bulid

    Glide build(@NonNull Context context) {
    //创建加载图片线程池
    if (sourceExecutor == null) {
    sourceExecutor = GlideExecutor.newSourceExecutor();
    }
    //创建文件缓存线程池
    if (diskCacheExecutor == null) {
    diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }
    //创建加载图片动画线程池
    if (animationExecutor == null) {
    animationExecutor = GlideExecutor.newAnimationExecutor();
    }
    //根据设备获取相应的缓存大小
    if (memorySizeCalculator == null) {
    memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
    //判断是否获取了网络请求权限
    if (connectivityMonitorFactory == null) {
    connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }
    //获取内存缓存图片池
    if (bitmapPool == null) {
    int size = memorySizeCalculator.getBitmapPoolSize();
    if (size > 0) {
    bitmapPool = new LruBitmapPool(size);
    } else {
    bitmapPool = new BitmapPoolAdapter();
    }
    }
    //对象池
    if (arrayPool == null) {
    arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }
    //内存缓存
    if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
    //磁盘缓存工厂类
    if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
    //加载图片引擎
    if (engine == null) {
    engine =
    new Engine(
    memoryCache,
    diskCacheFactory,
    diskCacheExecutor,
    sourceExecutor,
    GlideExecutor.newUnlimitedSourceExecutor(),
    GlideExecutor.newAnimationExecutor(),
    isActiveResourceRetentionAllowed);
    }

    RequestManagerRetriever requestManagerRetriever =
    new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
    context,
    engine,
    memoryCache,
    bitmapPool,
    arrayPool,
    requestManagerRetriever,
    connectivityMonitorFactory,
    logLevel,
    defaultRequestOptions.lock(),
    defaultTransitionOptions);
    }

3、new Glide

    Glide(
    @NonNull Context context,
    @NonNull Engine engine,
    @NonNull MemoryCache memoryCache,
    @NonNull BitmapPool bitmapPool,
    @NonNull ArrayPool arrayPool,
    @NonNull RequestManagerRetriever requestManagerRetriever,
    @NonNull ConnectivityMonitorFactory connectivityMonitorFactory,
    int logLevel,
    @NonNull RequestOptions defaultRequestOptions,
    @NonNull Map<Class<?>, TransitionOptions<?, ?>> defaultTransitionOptions) {
    this.engine = engine;
    this.bitmapPool = bitmapPool;
    this.arrayPool = arrayPool;
    this.memoryCache = memoryCache;
    this.requestManagerRetriever = requestManagerRetriever;
    this.connectivityMonitorFactory = connectivityMonitorFactory;
    //解码格式
    DecodeFormat decodeFormat = defaultRequestOptions.getOptions().get(Downsampler.DECODE_FORMAT);
    bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);

    final Resources resources = context.getResources();

    registry = new Registry();
    //注册文件头解析类
    registry.register(new DefaultImageHeaderParser());

    Downsampler downsampler = new Downsampler(registry.getImageHeaderParsers(),
    resources.getDisplayMetrics(), bitmapPool, arrayPool);
    //解码类:将InputStream中解码成GIF
    ByteBufferGifDecoder byteBufferGifDecoder =
    new ByteBufferGifDecoder(context, registry.getImageHeaderParsers(), bitmapPool, arrayPool);
    //解码类:将Video中解码成bitmap
    ResourceDecoder<ParcelFileDescriptor, Bitmap> parcelFileDescriptorVideoDecoder =
    VideoDecoder.parcel(bitmapPool);
    //解码类:将ByteBuffer解码成bitmap
    ByteBufferBitmapDecoder byteBufferBitmapDecoder = new ByteBufferBitmapDecoder(downsampler);
    //解码类:将InputStreams解码成bitmap
    StreamBitmapDecoder streamBitmapDecoder = new StreamBitmapDecoder(downsampler, arrayPool);
    //解码类:通过Uri解码成Drawable
    ResourceDrawableDecoder resourceDrawableDecoder =
    new ResourceDrawableDecoder(context);
    //解码类:将资源文件转换成InputStream
    ResourceLoader.StreamFactory resourceLoaderStreamFactory =
    new ResourceLoader.StreamFactory(resources);
    //将资源文件转换成URI
    ResourceLoader.UriFactory resourceLoaderUriFactory =
    new ResourceLoader.UriFactory(resources);
    //将资源文件转换成ParcelFileDescriptor
    ResourceLoader.FileDescriptorFactory resourceLoaderFileDescriptorFactory =
    new ResourceLoader.FileDescriptorFactory(resources);
    //将资源文件转换成ParcelFileDescriptor
    ResourceLoader.AssetFileDescriptorFactory resourceLoaderAssetFileDescriptorFactory =
    new ResourceLoader.AssetFileDescriptorFactory(resources);
    //Bitmap解码类
    BitmapEncoder bitmapEncoder = new BitmapEncoder(arrayPool);
    //bitmap转btye[]类
    BitmapBytesTranscoder bitmapBytesTranscoder = new BitmapBytesTranscoder();
    //GifDrawable转btye[]类
    GifDrawableBytesTranscoder gifDrawableBytesTranscoder = new GifDrawableBytesTranscoder();

    ContentResolver contentResolver = context.getContentResolver();

    registry
    .append(ByteBuffer.class, new ByteBufferEncoder())
    .append(InputStream.class, new StreamEncoder(arrayPool))
    //添加转换成Bitmap相应解码类
    .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
    .append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
    .append(
    Registry.BUCKET_BITMAP,
    ParcelFileDescriptor.class,
    Bitmap.class,
    parcelFileDescriptorVideoDecoder)
    .append(
    Registry.BUCKET_BITMAP,
    AssetFileDescriptor.class,
    Bitmap.class,
    VideoDecoder.asset(bitmapPool))
    .append(Bitmap.class, Bitmap.class, UnitModelLoader.Factory.<Bitmap>getInstance())
    .append(
    Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
    .append(Bitmap.class, bitmapEncoder)
    //添加转换成BitmapDrawables相应解码类
    .append(
    Registry.BUCKET_BITMAP_DRAWABLE,
    ByteBuffer.class,
    BitmapDrawable.class,
    new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
    .append(
    Registry.BUCKET_BITMAP_DRAWABLE,
    InputStream.class,
    BitmapDrawable.class,
    new BitmapDrawableDecoder<>(resources, streamBitmapDecoder))
    .append(
    Registry.BUCKET_BITMAP_DRAWABLE,
    ParcelFileDescriptor.class,
    BitmapDrawable.class,
    new BitmapDrawableDecoder<>(resources, parcelFileDescriptorVideoDecoder))
    .append(BitmapDrawable.class, new BitmapDrawableEncoder(bitmapPool, bitmapEncoder))
    //添加转换成GIFs相应解码类
    .append(
    Registry.BUCKET_GIF,
    InputStream.class,
    GifDrawable.class,
    new StreamGifDecoder(registry.getImageHeaderParsers(), byteBufferGifDecoder, arrayPool))
    .append(Registry.BUCKET_GIF, ByteBuffer.class, GifDrawable.class, byteBufferGifDecoder)
    .append(GifDrawable.class, new GifDrawableEncoder())
    /* GIF Frames */
    // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
    //添加将GIFs解码成Bitmap
    .append(
    GifDecoder.class, GifDecoder.class, UnitModelLoader.Factory.<GifDecoder>getInstance())
    .append(
    Registry.BUCKET_BITMAP,
    GifDecoder.class,
    Bitmap.class,
    new GifFrameResourceDecoder(bitmapPool))
    //添加转换成Drawables相应解码类
    .append(Uri.class, Drawable.class, resourceDrawableDecoder)
    .append(
    Uri.class, Bitmap.class, new ResourceBitmapDecoder(resourceDrawableDecoder, bitmapPool))
    //添加文件处理类
    .register(new ByteBufferRewinder.Factory())
    .append(File.class, ByteBuffer.class, new ByteBufferFileLoader.Factory())
    .append(File.class, InputStream.class, new FileLoader.StreamFactory())
    .append(File.class, File.class, new FileDecoder())
    .append(File.class, ParcelFileDescriptor.class, new FileLoader.FileDescriptorFactory())
    // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
    .append(File.class, File.class, UnitModelLoader.Factory.<File>getInstance())
    //添加转换类(将任意复杂的数据模型转化为一个具体的数据类型,然后通过DataFetcher处理得到相应的可用资源)
    .register(new InputStreamRewinder.Factory(arrayPool))
    //通过资源文件转化成InputStream
    .append(int.class, InputStream.class, resourceLoaderStreamFactory)
    //通过资源文件转化成ParcelFileDescriptor
    .append(
    int.class,
    ParcelFileDescriptor.class,
    resourceLoaderFileDescriptorFactory)
    .append(Integer.class, InputStream.class, resourceLoaderStreamFactory)
    .append(
    Integer.class,
    ParcelFileDescriptor.class,
    resourceLoaderFileDescriptorFactory)
    //通过资源文件转化成Uri
    .append(Integer.class, Uri.class, resourceLoaderUriFactory)
    .append(
    int.class,
    AssetFileDescriptor.class,
    resourceLoaderAssetFileDescriptorFactory)
    .append(
    Integer.class,
    AssetFileDescriptor.class,
    resourceLoaderAssetFileDescriptorFactory)
    .append(int.class, Uri.class, resourceLoaderUriFactory)
    //通过字符串转化成InputStream
    .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
    //通过Uri转化成InputStream
    .append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
    //通过String转化成InputStream
    .append(String.class, InputStream.class, new StringLoader.StreamFactory())
    //通过String转化成ParcelFileDescriptor
    .append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
    //通过String转化成AssetFileDescriptor
    .append(
    String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
    //通过网络Uri转化成InputStream
    .append(Uri.class, InputStream.class, new HttpUriLoader.Factory())
    //通过资产目录Uri转化成InputStream
    .append(Uri.class, InputStream.class, new AssetUriLoader.StreamFactory(context.getAssets()))
    //通过Uri转化成ParcelFileDescriptor
    .append(
    Uri.class,
    ParcelFileDescriptor.class,
    new AssetUriLoader.FileDescriptorFactory(context.getAssets()))
    //通过image Uri转化成InputStream
    .append(Uri.class, InputStream.class, new MediaStoreImageThumbLoader.Factory(context))
    //通过video Uri转化成InputStream
    .append(Uri.class, InputStream.class, new MediaStoreVideoThumbLoader.Factory(context))
    //通过Uri转化成InputStream
    .append(
    Uri.class,
    InputStream.class,
    new UriLoader.StreamFactory(contentResolver))
    //通过Uri转化成ParcelFileDescriptor
    .append(
    Uri.class,
    ParcelFileDescriptor.class,
    new UriLoader.FileDescriptorFactory(contentResolver))
    //通过Uri转化成AssetFileDescriptor
    .append(
    Uri.class,
    AssetFileDescriptor.class,
    new UriLoader.AssetFileDescriptorFactory(contentResolver))
    //通过http/https Uris转化成InputStream
    .append(Uri.class, InputStream.class, new UrlUriLoader.StreamFactory())

    //通过  java.net.URL转化成InputStream
    .append(URL.class, InputStream.class, new UrlLoader.StreamFactory())
    //通过多媒体文件uri转化成文件
    .append(Uri.class, File.class, new MediaStoreFileLoader.Factory(context))
    //通过http/https url转化成InputStream
    .append(GlideUrl.class, InputStream.class, new HttpGlideUrlLoader.Factory())
    //通过数组转化成ByteBuffer
    .append(byte[].class, ByteBuffer.class, new ByteArrayLoader.ByteBufferFactory())
    //通过数组转化成InputStream
    .append(byte[].class, InputStream.class, new ByteArrayLoader.StreamFactory())

    .append(Uri.class, Uri.class, UnitModelLoader.Factory.<Uri>getInstance())
    .append(Drawable.class, Drawable.class, UnitModelLoader.Factory.<Drawable>getInstance())
    .append(Drawable.class, Drawable.class, new UnitDrawableDecoder())
    //注册转码类
    //bitmap转码成BitmapDrawable
    .register(
    Bitmap.class,
    BitmapDrawable.class,
    new BitmapDrawableTranscoder(resources))
    //bitmap转码成byte[]
    .register(Bitmap.class, byte[].class, bitmapBytesTranscoder)
    //Drawable转码成byte[]
    .register(
    Drawable.class,
    byte[].class,
    new DrawableBytesTranscoder(
    bitmapPool, bitmapBytesTranscoder, gifDrawableBytesTranscoder))
    //GifDrawable转码成byte[]
    .register(GifDrawable.class, byte[].class, gifDrawableBytesTranscoder);

    ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
    glideContext =
    new GlideContext(
    context,
    arrayPool,
    registry,
    imageViewTargetFactory,
    defaultRequestOptions,
    defaultTransitionOptions,
    engine,
    logLevel);
    }

glide的初始化话还是比较复杂的,主要就是配置相应的设置和加载注册将相应的数量类型转换成可用的数据。至此glide的初始化就完成了。

## 三、加载图片
初始化完之后就开始加载图片了,调用with方法就会创建一个RequestManager,这里以with(Activity)为例：</br>

1、创建RequestManager</br>
1.1、Glide.with()

    public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
    }

    这里获取初始化Glide时设置的RequestManagerRetriever：
    public RequestManager get(@NonNull Activity activity) {
    //判断当前线程是否是主线程
    if (Util.isOnBackgroundThread()) {
    //如果不在主线程中否则则创建一个绑定application生命周期的RequestManager
    return get(activity.getApplicationContext());
    } else {
    //排查当前Activity没有被销毁
    assertNotDestroyed(activity);
    //获取fragmentManager
    android.app.FragmentManager fm = activity.getFragmentManager();
    //创建一个与fragment生命周期绑定的RequestManager(RequestManager实现了LifecycleListener接口
    //并将自己添加到了RequestManagerFragment的ActivityFragmentLifecycle队列中,
    //当RequestManagerFragment执行生命周期方法时就会调用LifecycleListener接口实现类,
    //从而做出相应的处理暂停请求或移除请求)
    return fragmentGet(
    activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }

    }
1.2、RequestManagerRetriever.fragmentGet()

    private RequestManager fragmentGet(@NonNull Context context,
    @NonNull android.app.FragmentManager fm,
    @Nullable android.app.Fragment parentHint,
    boolean isParentVisible) {
    //创建一个RequestManagerFragment并添加到当前Activity中
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    //判断与其fragment绑定的RequestManager是否已经创建,否则则创建
    if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
    factory.build(
    glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
    }
    return requestManager;
    }

2、创建RequestBuilder</br>
2.1、RequestManager.load()</br>
这里以字符串为参数,因为没有设置将对应资源转成具体的数据类型(bitmap、GIF、drawable),所以会默认转换成drawable。
    public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
    }

    RequestManager.asDrawable()->
    public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
    }

    RequestManager.as()->
    创建RequestBuilder实例
    public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
    }

2.2、RequestBuilder.load()

    public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
    }

3、加载图片</br>
3.1构建请求</br>
3.1.1、RequestBuilder.into()

    public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    //确保是在主线程中运行
    Util.assertMainThread();
    //检测当前ImageView是否为null
    Preconditions.checkNotNull(view);
    //配置请求 设置图片的缩放类型(ImageView 默认是FIT_CENTER)
    RequestOptions requestOptions = this.requestOptions;
    if (!requestOptions.isTransformationSet()
    && requestOptions.isTransformationAllowed()
    && view.getScaleType() != null) {
    switch (view.getScaleType()) {
    case CENTER_CROP:
    requestOptions = requestOptions.clone().optionalCenterCrop();
    break;
    case CENTER_INSIDE:
    requestOptions = requestOptions.clone().optionalCenterInside();
    break;
    case FIT_CENTER:
    case FIT_START:
    case FIT_END:
    requestOptions = requestOptions.clone().optionalFitCenter();
    break;
    case FIT_XY:
    requestOptions = requestOptions.clone().optionalCenterInside();
    break;
    case CENTER:
    case MATRIX:
    default:
    // Do nothing.
    }
    }

    return into(
    glideContext.buildImageViewTarget(view, transcodeClass),
    /*targetListener=*/ null,
    requestOptions);
    }

3.1.2、

    glideContext.buildImageViewTarget(view, transcodeClass)
    ImageViewTargetFactory.buildTarget:
    public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
    @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
    //作为Bitmap加载
    return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
    //作为Drawable加载
    return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
    throw new IllegalArgumentException(
    "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
    }

根据相应想要加载类型返回ViewTarget。
3.1.3、

    RequestBuilder.into( Y target,RequestListener<TranscodeType> targetListener,RequestOptions options)
    @NonNull
    private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @NonNull RequestOptions options) {
    Util.assertMainThread();
    Preconditions.checkNotNull(target);
    //判断是否调用了load方法
    if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
    }
    //克隆一份请求设置，因为得保证每一个ImageView对应一个options
    options = options.autoClone();
    //创建请求 见3.1.4
    Request request = buildRequest(target, targetListener, options);
    //获取当前ImageView对应的请求
    Request previous = target.getRequest();
    //若当前ImageView对应的请求已经设置(即为ImageView设置了tag)
    if (request.isEquivalentTo(previous)
    && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();
    if (!Preconditions.checkNotNull(previous).isRunning()) {
    previous.begin();
    }
    return target;
    }
    //先清除当前View的tag
    requestManager.clear(target);
    //设置当前View tag(即setTag)
    target.setRequest(request);
    //添加执行request
    requestManager.track(target, request);

    return target;
    }

    3.1.4、RequestBuilder.buildRequest()
    ->RequestBuilder.buildRequestRecursive():
    private Request buildRequestRecursive(
    Target<TranscodeType> target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @Nullable RequestCoordinator parentCoordinator,
    TransitionOptions<?, ? super TranscodeType> transitionOptions,
    Priority priority,
    int overrideWidth,
    int overrideHeight,
    RequestOptions requestOptions) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    //判断是否设置了当前请求错误的错误请求
    if (errorBuilder != null) {
    errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
    parentCoordinator = errorRequestCoordinator;
    }
    //构建请求(这里会根据是否设置了获取缩略图请求或缩略比例来创建SingleRequest。这里SingleRequest是从SingleRequestPool中获取的 详见SingleRequest类)
    Request mainRequest =
    buildThumbnailRequestRecursive(
    target,
    targetListener,
    parentCoordinator,
    transitionOptions,
    priority,
    overrideWidth,
    overrideHeight,
    requestOptions);
    //没有设置错误时的错误请求时则直接返回正常请求
    if (errorRequestCoordinator == null) {
    return mainRequest;
    }
    //省略部分代码
    Request errorRequest = errorBuilder.buildRequestRecursive(
    target,
    targetListener,
    errorRequestCoordinator,
    errorBuilder.transitionOptions,
    errorBuilder.requestOptions.getPriority(),
    errorOverrideWidth,
    errorOverrideHeight,
    errorBuilder.requestOptions);
    //设置错误时的错误请求时则添加errorRequest
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
    }

3.2、执行请求</br>
至此请求就已经构建好了，接下来就是执行请求了。</br>
3.2.1、RequestManager.track()

    void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
    }

3.2.2、TargetTracker.track()

    public void track(@NonNull Target<?> target) {
    targets.add(target);
    }

因为TargetTracker实现了LifecycleListener,当回调RequestManager的生命周期方法时同时也会执行TargetTracker的相对应方法。
3.2.2、requestTracker.runRequest()

    public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
    request.begin();
    } else {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
    Log.v(TAG, "Paused, delaying request");
    }
    pendingRequests.add(request);
    }
    }

这里比较简单主要就是先判断是否进入了Paused状态,若没有则直接开始请求,若处于Paused状态时则加入到等待请求队列中。这里的ruquest是在3.1.4创建的SingleRequest。
3.2.3、

    SingleRequest.begin()
    @Override
    public void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    //判断加载的内容模型是否为空
    if (model == null) {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    width = overrideWidth;
    height = overrideHeight;
    }
    int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
    onLoadFailed(new GlideException("Received null model"), logLevel);
    return;
    }
    //当前请求正在执行中...
    if (status == Status.RUNNING) {
    throw new IllegalArgumentException("Cannot restart a running request");
    }
    //当前请求已经执行完成
    if (status == Status.COMPLETE) {
    onResourceReady(resource, DataSource.MEMORY_CACHE);
    return;
    }
    //设置请求图片的宽高
    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    //开始请求
    onSizeReady(overrideWidth, overrideHeight);
    } else {
    //获取当前View的宽高之后 再调用onSizeReady方法
    target.getSize(this);
    }
    //正在请求时 设置PlaceholderDrawable图片
    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
    && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
    }
    }

3.2.4、SingleRequest.onSizeReady()

    -->Engine.load()
    public <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    //更新相应属性创建EngineKey
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
    resourceClass, transcodeClass, options);
    //尝试从活跃的资源内存缓存中中获取
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
    cb.onResourceReady(active, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
    }
    //尝试从内存缓存中获取
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
    cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return null;
    }
    //判断当前请求是在在队列中
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
    current.addCallback(cb);
    if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Added to existing load", startTime, key);
    }
    return new LoadStatus(cb, current);
    }
    //创建请求job
    EngineJob<R> engineJob =
    engineJobFactory.build(
    key,
    isMemoryCacheable,
    useUnlimitedSourceExecutorPool,
    useAnimationPool,
    onlyRetrieveFromCache);
    //创建解码job
    DecodeJob<R> decodeJob =
    decodeJobFactory.build(
    glideContext,
    model,
    key,
    signature,
    width,
    height,
    resourceClass,
    transcodeClass,
    priority,
    diskCacheStrategy,
    transformations,
    isTransformationRequired,
    isScaleOnlyOrNoTransform,
    onlyRetrieveFromCache,
    options,
    engineJob);
    //保存当前请求job
    jobs.put(key, engineJob);

    engineJob.addCallback(cb);
    //开始请求
    engineJob.start(decodeJob);

    return new LoadStatus(cb, engineJob);
    }

    3.2.5、EngineJob.start()
    public void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor = decodeJob.willDecodeFromCache()
    ? diskCacheExecutor
    : getActiveSourceExecutor();
    executor.execute(decodeJob);
    }

    DecodeJob实现了Runnable接口,这里将其添加到线程池中。接下来看下run方法 。
    3.2.5、DecodeJob.run()
    @Override
    public void run() {
    GlideTrace.beginSectionFormat("DecodeJob#run(model=%s)", model);
    DataFetcher<?> localFetcher = currentFetcher;
    try {
    //判断是否已经被取消
    if (isCancelled) {
    notifyFailed();
    return;
    }
    runWrapped();
    } catch (Throwable t) {
    if (stage != Stage.ENCODE) {
    throwables.add(t);
    notifyFailed();
    }
    if (!isCancelled) {
    throw t;
    }
    } finally {
    if (localFetcher != null) {
    localFetcher.cleanup();
    }
    GlideTrace.endSection();
    }
    }

    3.2.6、DecodeJob.runWrapped()
    private void runWrapped() {
    switch (runReason) {
    case INITIALIZE:
    stage = getNextStage(Stage.INITIALIZE);
    //根据当前流程步骤获取相应获取数据类(刚开始肯定为INITIALIZE初始化状态)
    currentGenerator = getNextGenerator();
    runGenerators();
    break;
    case SWITCH_TO_SOURCE_SERVICE:
    runGenerators();
    break;
    case DECODE_DATA:
    decodeFromRetrievedData();
    break;
    default:
    throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
    }

3.2.7、DecodeJob.getNextGenerator()

    private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
    case RESOURCE_CACHE:
    //从内存缓存中获取
    return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
    //从文件缓存中获取
    return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
    //从相应的资源中获取
    return new SourceGenerator(decodeHelper, this);
    case FINISHED:
    return null;
    default:
    throw new IllegalStateException("Unrecognized stage: " + stage);
    }
    }

3.2.8、DecodeJob.runGenerators()

    private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    //依次从内存、文件及相应资源中获取。
    while (!isCancelled && currentGenerator != null
    && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();
    if (stage == Stage.SOURCE) {
    reschedule();
    return;
    }
    }
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
    }
    }

这里会依次从文件缓存、相应资源(比如网络、uri等)处获取图片。因为是图片是第一次加载,所以接下来我们看下SourceGenerator的startNext()方法。
3.2.9、SourceGenerator.startNext()

    public boolean startNext() {
    //判断当前请求是否成功，成了调用DataCacheGenerator的next方法,从而生成相应的图片
    if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
    }
    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
    }
    sourceCacheGenerator = null;
    loadData = null;
    boolean started = false;
    //依次从注册了处理该资源的实例
    while (!started && hasNextModelLoader()) {
    loadData = helper.getLoadData().get(loadDataListIndex++);
    if (loadData != null
    && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
    || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
    started = true;
    //开始加载资源
    loadData.fetcher.loadData(helper.getPriority(), this);
    }
    }
    return started;
    }

    这里会从registry中匹配相应的Modeloader来加载数据。这里是HttpUrlFetcher。
    3.2.10、HttpUrlFetcher.loadData()
    @Override
    public void loadData(@NonNull Priority priority,
    @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
    //连接网络获取图片流
    InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
    //成功回调
    callback.onDataReady(result);
    } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
    Log.d(TAG, "Failed to load data for url", e);
    }
    callback.onLoadFailed(e);
    } finally {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
    Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
    }
    }
    }

请求成功之后回调了DataCallback方法。最终会回调到DecodeJob的onDataFetcherReady方法,之后在进行解码操作：

    private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
    logWithTimeAndKey("Retrieved data", startFetchTime,
    "data: " + currentData
    + ", cache key: " + currentSourceKey
    + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
    //进行解码(从registery中匹配相应的解码实例类Downsampler进行解码)
    resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
    e.setLoggingDetails(currentAttemptingKey, currentDataSource);
    throwables.add(e);
    }
    if (resource != null) {
    //解码成功
    notifyEncodeAndRelease(resource, currentDataSource);
    } else {
    //解码失败进行下一步操作
    runGenerators();
    }

    }
解码成功之后,回调EngineJob中的onResourceReady方法：

    public void onResourceReady(Resource<R> resource, DataSource dataSource) {
    this.resource = resource;
    this.dataSource = dataSource;
    MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
    }

    通过handler切换至主线程
    private static class MainThreadCallback implements Handler.Callback {
    @Synthetic
    @SuppressWarnings("WeakerAccess")
    MainThreadCallback() { }

    @Override
    public boolean handleMessage(Message message) {
    EngineJob<?> job = (EngineJob<?>) message.obj;
    switch (message.what) {
    case MSG_COMPLETE:
    job.handleResultOnMainThread();
    break;
    case MSG_EXCEPTION:
    job.handleExceptionOnMainThread();
    break;
    case MSG_CANCELLED:
    job.handleCancelledOnMainThread();
    break;
    default:
    throw new IllegalStateException("Unrecognized message: " + message.what);
    }
    return true;
    }

    }
3.2.11、EngineJob.handleResultOnMainThread()

    @Synthetic
    void handleResultOnMainThread() {
    stateVerifier.throwIfRecycled();
    if (isCancelled) {
    resource.recycle();
    release(false /*isRemovedFromQueue*/);
    return;
    } else if (cbs.isEmpty()) {
    throw new IllegalStateException("Received a resource without any callbacks to notify");
    } else if (hasResource) {
    throw new IllegalStateException("Already have resource");
    }
    engineResource = engineResourceFactory.build(resource, isCacheable);
    hasResource = true;

    // Hold on to resource for duration of request so we don't recycle it in the middle of
    // notifying if it synchronously released by one of the callbacks.
    engineResource.acquire();
    listener.onEngineJobComplete(this, key, engineResource);

    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = cbs.size(); i < size; i++) {
    ResourceCallback cb = cbs.get(i);
    if (!isInIgnoredCallbacks(cb)) {
    engineResource.acquire();
    cb.onResourceReady(engineResource, dataSource);
    }
    }
    // Our request is complete, so we can release the resource.
    engineResource.release();

    release(false /*isRemovedFromQueue*/);

    }
    将Bitmap设置给ViewImage。至此就完成了一张图片的加载。
    四、加载GIF
    不管是加载Bitmap、Drawable还是Gif,解码成功并转化成相应的Resource之后都会调用SingleRequest.onResourceReady()。
    1、SingleRequest.onResourceReady()
    private void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    boolean isFirstResource = isFirstReadyResource();
    status = Status.COMPLETE;
    this.resource = resource;
    isCallingCallbacks = true;
    try {
    //是否设置了加载成功的监听并且判断返回值是否为true
    if ((requestListener == null
    || !requestListener.onResourceReady(result, model, target, dataSource, isFirstResource))
    && (targetListener == null
    || !targetListener.onResourceReady(result, model, target, dataSource, isFirstResource))) {
    //加载动画
    Transition<? super R> animation =
    animationFactory.build(dataSource, isFirstResource);
    //为target设置值
    target.onResourceReady(result, animation);
    }
    } finally {
    isCallingCallbacks = false;
    }
    //通知加载成功
    notifyLoadSuccess();
    }

    2、DrawableImageViewTarget.onResourceReady()
    @Override
    public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    if (transition == null || !transition.transition(resource, this)) {
    //给ImageView设置drawable(相当于是gif的第一帧图片)
    setResourceInternal(resource);
    } else {
    //开始GIF动画
    maybeUpdateAnimatable(resource);
    }
    }

    这里我们看的是加载GIF,所以我们重点看下gif相关的两个重点类GifDrawable和GifFrameLoader。
    2、GifDrawable.start()
    public void start() {
    isStarted = true;
    resetLoopCount();
    if (isVisible) {
    startRunning();
    }
    }

    开始动画
    private void startRunning() {
    Preconditions.checkArgument(!isRecycled, "You cannot start a recycled Drawable. Ensure that"
    + "you clear any references to the Drawable when clearing the corresponding request.");
    if (state.frameLoader.getFrameCount() == 1) {
    invalidateSelf();
    } else if (!isRunning) {
    isRunning = true;
    //订阅GifFrameLoader切换下一帧图片事件
    state.frameLoader.subscribe(this);
    invalidateSelf();
    }

    }
2、GifFrameLoader.start()

    private void start() {
    if (isRunning) {
    return;
    }
    isRunning = true;
    isCleared = false;

    loadNextFrame();
    }

    GifFrameLoader.loadNextFrame():
    private void loadNextFrame() {
    if (!isRunning || isLoadPending) {
    return;
    }
    if (startFromFirstFrame) {
    Preconditions.checkArgument(
    pendingTarget == null, "Pending target must be null when starting from the first frame");
    gifDecoder.resetFrameIndex();
    startFromFirstFrame = false;
    }
    if (pendingTarget != null) {
    DelayTarget temp = pendingTarget;
    pendingTarget = null;
    onFrameReady(temp);
    return;
    }
    isLoadPending = true;
    int delay = gifDecoder.getNextDelay();
    long targetTime = SystemClock.uptimeMillis() + delay;

    gifDecoder.advance();
    next = new DelayTarget(handler, gifDecoder.getCurrentFrameIndex(), targetTime);
    //继续加载下一帧图片,加载成功之后一样的会调用 DelayTarget的onResourceReady方法   
    requestBuilder.apply(signatureOf(getFrameSignature())).load(gifDecoder).into(next);
    }

3、DelayTarget.onResourceReady()

    @Override
    public void onResourceReady(@NonNull Bitmap resource,
    @Nullable Transition<? super Bitmap> transition) {
    this.resource = resource;
    Message msg = handler.obtainMessage(FrameLoaderCallback.MSG_DELAY, this);
    handler.sendMessageAtTime(msg, targetTime);
    }

    这里的handle是GifFrameLoader中设置的handler,我们看下是如何处理这个消息的：
    @Override
    public boolean handleMessage(Message msg) {
    if (msg.what == MSG_DELAY) {
    GifFrameLoader.DelayTarget target = (DelayTarget) msg.obj;
    onFrameReady(target);
    return true;
    } else if (msg.what == MSG_CLEAR) {
    GifFrameLoader.DelayTarget target = (DelayTarget) msg.obj;
    requestManager.clear(target);
    }
    return false;
    }

这里又回调了GifFrameLoader中的onFrameReady方法
3、GifFrameLoader. onFrameReady()

    @VisibleForTesting
    void onFrameReady(DelayTarget delayTarget) {
    if (onEveryFrameListener != null) {
    onEveryFrameListener.onFrameReady();
    }
    isLoadPending = false;
    //如果取消了则停止更新imageView的drawable
    if (isCleared) {
    handler.obtainMessage(FrameLoaderCallback.MSG_CLEAR, delayTarget).sendToTarget();
    return;
    }
    if (!isRunning) {
    pendingTarget = delayTarget;
    return;
    }
    if (delayTarget.getResource() != null) {
    //回收上一帧图片
    recycleFirstFrame();
    DelayTarget previous = current;
    //将加载好的图片赋值给当前需要加载的图片
    current = delayTarget;
    for (int i = callbacks.size() - 1; i >= 0; i--) {
    FrameCallback cb = callbacks.get(i);
    //刷新给imageView设置的drawable
    cb.onFrameReady();
    }
    if (previous != null) {
    handler.obtainMessage(FrameLoaderCallback.MSG_CLEAR, previous).sendToTarget();
    }
    }
    //接着进行下一帧图片的加载(然后又会从新执行这个方法,如此循环往复)
    loadNextFrame();
    }

至此我们就知道了Glide加载Gif图片的原理了,就是将gif根据每一帧解析成很张图片,然后在依次设置给ImageView。

## 五、总结
1、Glide实现了内存缓存和磁盘缓存，且都设置了相应的大小，并根据lrus算法进行更新和删除。</br>
2、Glide将加载了的图片添加到Map(Key, ResourceWeakReference)中实现内存缓存。</br>
3、Glide的加载顺序是先从内存中获取,若没有则从磁盘中获取,在没有则通过网络获取。从网络上获取到了图片之后若设置了可缓存,则会缓存到磁盘中;然后在通过相应的加载参数进行解码压缩裁剪等操作之后得到图片设置给ImageView,同时缓存到内存中。</br>
4、Glide通过为当前Activity添加一个fragment来监听相应的生命周期方法,从而实现加载与Activity生命周期绑定,onstart时进行加载、onstop时停止加载等。</br>
5、Glide在开始加载图片时,会给当前ViewTarget设置对应的Request,同时也会ViewTarget中的View设置tag为Request。在listView复用时有了这个Tag值,从而解决错乱的问题</br>

