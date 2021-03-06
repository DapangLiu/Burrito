
# Burrito

__Video!: You could click [here](https://youtu.be/Lp3gkcTTYyk) to have a look how the app runs so smoothly with our strategy.__

![](https://s3.amazonaws.com/artceleration/Burrito.png)

## Development Goals

1. Swift development based on appropriate __Modularity__, including needed Abstraction and MVC project structure.
2. High-Efficiency threads control and smart service progress lifecycle design. 
3. Smooth user experience and friendly UI.
4. Several available filter, based on algorithms specially designed. 
5. Social Network features, which enable users to share their edited images on social media!

## Highly-Abstract Architecture

![](https://s3.amazonaws.com/artceleration/Ass2.png)

Above is the project structure in high level. All activities are marked with circles and classes are marked with rounded rectangles. We could see that ArtLib is the core for the whole structure. Besides the helper classes and needed supported classes provided by professor, I created three other classes to implement service and thread. 

![](https://s3.amazonaws.com/artagfinal/artTransform-lib_new.png)

Above is the NDK structure and NEON intrinsics is included. ArtTransformHandler declares the native functions and call them in the Message Handler part.

I implemented five fancy filters:

1. Gaussian Blur (Java)
2. Color Filter (Java)
3. Bright (NDK C)
4. Lomo (NDK/NEON)
5. Noir (NDK C)

## Message Queue & Separate Thread… How does it work? 

![](https://s3.amazonaws.com/artceleration/truck_whole.png)

Above is a big picture of the whole story. We assumes __the artTransformService__ to be a burrito making service. For convenience, we provide a truck for this service, such as we provide our __artTransformService__ with a __separate thread__. The process of making burrito is like a __runnable__ event, and instead of doing by ourselves, we hire a guy to handle all the burrito things, like receiving orders and making burritos. 

“Receiving orders” is just like “receiving message”. __Message__, in our app, is sent when the ArtLib needs the __artTransformService__ (with all the data). Similarly, in the burrito truck, the cook receives orders/messages from the customers who need this snack service. A better way to handle these trivial orders is to hire another guy who can help us loop through the order in a FIFO manner, as professor mentioned.

![](https://s3.amazonaws.com/artceleration/truck.png)

Now, we hire a girl to help us do the “Looper” job. Her name is “__Looper__”. What she does is: Every time she notices the cook is about to finish an order, she grabs the next order from the “__Message Queue__” and gives it to the cook. That’s basically a __FIFO manner__, because of the existing of message pool.

## AsyncTask + ThreadPool = Ultra Smooth!

![](https://s3.amazonaws.com/artceleration/AsyncTask.png)

After we figure out how the combination of thread, looper and handler works, we need to think about how to imporve the efficiency of our app, especially because all the ArtTransform should run without disturbing each other, which means we need to use parallel threads.

A thread pool is a good idea. Android provides some defined threadpools, like "Executors.newCachedThreadPool()" or "Executors.newFixedThreadPool()". More detailed information could be found [here](https://developer.android.com/reference/java/util/concurrent/Executors.html).

Another thing we need think a little bit is how to queue how task. We may make several ArtTransform requests at the same time, or at least, due to the processing time, there would be some requests processing simultaneously.
Android also introduces some great tools for us to handle the multi-thread task. It's called "__AsyncTask__".

>**Definition from Android Docs:**
AsyncTask is designed to be a helper class around Thread and Handler and does not constitute a generic threading framework. AsyncTasks should ideally be used for short operations (a few seconds at the most.) If you need to keep threads running for long periods of time, it is highly recommended you use the various APIs provided by the java.util.concurrent package such as Executor, ThreadPoolExecutor and FutureTask.

So my strategy is, everytime the service receives message from the client (here, it's ArtLib.class.), it would call a AsyncTask, where we retrieve data from message, do the transform and send back the message including the proceeded image to the client. The client would implement the listener method of activity which would notify the main activity to update its UI elements.  

## Corresponding Code Strategy
### ArtTransformService
```java
	public class ArtTransformService extends Service{

    private static final String TAG = ArtTransformService.class.getSimpleName();
    
    public Messenger mMessenger = new Messenger(new ArtTransformHandler());
    private ArtTransformHandler mArtTransformHandler;
    public ArtTransformService() {
    }


    @Override
    public void onCreate() {

        // Put service into a separate Thread named "ArtTransformThread"
        ArtTransformThread thread = new ArtTransformThread();
        thread.setName("ArtTransformThread");
        thread.start();

        // Make sure Handler is available
        while (thread.mArtTransformHandler == null) {

        }
        mArtTransformHandler = thread.mArtTransformHandler;
        mArtTransformHandler.setService(this);
    }
    ...
```

In @Overriding OnCreat() Method, we initiate a separate thread for our artTransformService. The while() {} loop is needed because we should make sure out handler is available before we give the reference to the handler to the thread class.

### ArtTransformThread
```java
public class ArtTransformThread extends Thread{
    private static final String TAG = ArtTransformThread.class.getSimpleName();
    public ArtTransformHandler mArtTransformHandler;

    @Override
    public void run() {
        Looper.prepare();
        mArtTransformHandler = new ArtTransformHandler();
        Looper.loop();
    }
}
```

Nothing fancy. Actually there are two ways to create a separate thread. Besides above, we could also announce a new thread in the Manifest file.

### ArtTransformHandler
```java
public class ArtTransformHandler extends Handler {
    private ArtTransformService mService;
    static ArrayList<Messenger> mClients = new ArrayList<>();
    static Messenger targetMessenger;
    List<ArtTransformAsyncTask> mArtTransformAsyncTasks;


    @Override
    public void handleMessage(Message msg) {

        targetMessenger = msg.replyTo;
        mArtTransformAsyncTasks = new ArrayList<>();
        Bundle dataBundle = msg.getData();


        switch (msg.what) {
            case 0:
                Log.d("AsyncTask", "Gaussian_Blur");

                try {
                    new ArtTransformAsyncTask().executeOnExecutor(Executors.newCachedThreadPool(), dataBundle);

                } finally {
                    Log.d("AsyncTask", "Gaussian_Blur Finished");
                }

                break;
            case 1:
                Log.d("AsyncTask", "Neon_Edges");

                try {
                    new ArtTransformAsyncTask().executeOnExecutor(Executors.newCachedThreadPool(), dataBundle);

                } finally {
                    Log.d("AsyncTask", "Neon_Edges Finished");
                }
                break;
            case 2:
                Log.d("AsyncTask", "Color_Filter");

                try {
                    new ArtTransformAsyncTask().executeOnExecutor(Executors.newCachedThreadPool(), dataBundle);

                } finally {
                    Log.d("AsyncTask", "Color_Filter Finished");
                }

                break;
            default:
                break;
        }

    }
    ...
}
```
Here we handle the message from the client, and based on "msg.what", we start different AsyncTask. The point is, we use ThreadPool to handle all the Asynctasks, which could definitely speed up our processing.

### ArtTransformAsyncTask
```java
public class ArtTransformAsyncTask extends AsyncTask<Bundle, Void, Void> {

        private Bitmap rawBitmap;

        @Override
        protected void onPreExecute() {
            mArtTransformAsyncTasks.add(this);
        }

        @Override
        protected Void doInBackground(Bundle... params) {

            Log.d("Message", String.valueOf(params[0]));

                switch (params[0].getInt("index")) {

                    case 0:
                        try {
                            rawBitmap = changeSaturation(loadImage(params[0]));
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        break;
                    case 1:
                        try {
                            rawBitmap = changeHSL(loadImage(params[0]));
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        break;
                    case 2:
                        try {
                            rawBitmap = changeLight(loadImage(params[0]));
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        break;
                    default:
                        break;

                }
            return null;
        }

        @Override
        protected void onPostExecute(Void aVoid) {

            mArtTransformAsyncTasks.remove(this);
            if (mArtTransformAsyncTasks.size() == 0) {
                Log.d("AsyncTask", "All Tasks Finished");
            }
            notifyArtLib(rawBitmap);

        }

    }
```
and mentioned methods partly are:
```java
private Bitmap changeLight(Bitmap img) {
        ColorMatrix colorMatrixchangeLight = new ColorMatrix();
        ColorMatrix allColorMatrix = new ColorMatrix();

        colorMatrixchangeLight.reset();
        colorMatrixchangeLight.setScale(1.5f, 1.5f, 1.5f, 1);

        allColorMatrix.reset();
        allColorMatrix.postConcat(colorMatrixchangeLight);

        Bitmap newBitmap = Bitmap.createBitmap(img.getWidth(), img.getHeight(), Bitmap.Config.ARGB_8888);

        Canvas canvas = new Canvas(newBitmap);
        Paint paint = new Paint();
        paint.setAntiAlias(true);

        paint.setColorFilter(new ColorMatrixColorFilter(allColorMatrix));
        canvas.drawBitmap(img, 0, 0, paint);
        return newBitmap;
    }
```
Above is one of three my custom filters. This one could change the brightness of the image. 
## Challenge

In fact, there's nothing too challenging. Maybe the debug progress is something frustrating.

![](https://s3.amazonaws.com/artceleration/threadpool.png)

That is DDMS to make sure we manage to create a separate thread.

![](https://s3.amazonaws.com/artceleration/FIFO.png)

Above is debug log. Click different option could send different message to handler. After processing, the onTransformProcessed() Method would be triggered.

Another challenge is the NEON implements.

```java
void
line_pixel_processing_intrinsics (argb * new, argb * old, uint32_t width){

    int i;
    uint8x8_t rfac = vdup_n_u8(255 * 0.22);
    uint8x8_t gfac = vdup_n_u8(255 * 0.44);
    uint8x8_t bfac = vdup_n_u8(255 * 0.88);
    width/=8;

    for (i=0; i<width; i++)
    {
        uint16x8_t  temp;
        uint8x8x3_t rgb;
        rgb.val[0] = vld1_u8(old->red);
        rgb.val[1] = vld1_u8(old->green);
        rgb.val[2] = vld1_u8(old->blue);

        uint8x8_t result;

        temp = vmull_u8 (rgb.val[0],      rfac);
        temp = vmlal_u8 (temp,rgb.val[1], gfac);
        temp = vmlal_u8 (temp,rgb.val[2], bfac);

        result = vshrn_n_u16 (temp, 8);
        vst1_u8 (new, result);
        old  += 8*3;
        new += 8;
    }
}
```

## Improvement/Potential Extra Credits 

### Service Connection Management
I think we could do something more on the service connection management, because service also has its lifecycle. If we do not consider about when the service should connect, when disconnect, it would always stay in our users' background progresses and consume device resource all the time. 

So we have tried some mechanisms like:

```java
...
@Override
    public int onStartCommand(Intent intent, int flags, int startId) {

        Message message = Message.obtain();
        message.arg1 = startId;
        mArtTransformHandler.sendMessage(message);
        return Service.START_REDELIVER_INTENT;
    }
...
```

We assume value of "arg1" in message to be starId, which is a parameter recording the unique ID of service progress everytime the service is called.

```java
...
@Override
    public void handleMessage(Message msg) {
        
        ...

        // Stop the service progress based on unique ID 
        mService.stopSelf(msg.arg1);
    }
...
```

Above is in the Handler class. It enables out service progress could stop by itself after all its message queue has been handled yet. The keypoint is, if our user kill the service progress for some reasons, the service would automatically continue its undo queue handle job!

### Threadpool Management

When using our app, you would not feel any lag, because we managed to handle all the threads in a perfect way.

```java
public class ArtTransformThreadPool {

    private static ArtTransformThreadPool sArtTransformThreadPoolIns = null;

    private final BlockingQueue<Runnable> mArtTransformRequestQueue;
    private List<Future> mDoingArtTransform;
    private final ExecutorService mArtTransformThreadPool;
    private static final TimeUnit KEEP_ALIVE_TIME_UNIT;
    private static final int KEEP_ALIVE_TIME = 1;
    
    // Get the currently available max number of cores.
    private static int NUMBER_OF_CORES = Runtime.getRuntime().availableProcessors();
    
    static {
        sArtTransformThreadPoolIns = new ArtTransformThreadPool();
        KEEP_ALIVE_TIME_UNIT = TimeUnit.SECONDS;
    }

    private ArtTransformThreadPool() {
        mArtTransformRequestQueue = new LinkedBlockingQueue<Runnable>();
        mDoingArtTransform = new ArrayList<>();
	
	// Creat a Thread Pool with NUMBER_OF_CORES threads
        mArtTransformThreadPool = Executors.newFixedThreadPool(NUMBER_OF_CORES, new BackgroundThreadFactory());
    }

    public static ArtTransformThreadPool getInstance() {
        return sArtTransformThreadPoolIns;
    }

    // Add a task
    public void addArtTransformTask(Callable callable) {
        Future future = mArtTransformThreadPool.submit(callable);
        mDoingArtTransform.add(future);
    }

    public void cancelAll() {
        synchronized (this) {
            mArtTransformRequestQueue.clear();
            for (Future task : mDoingArtTransform) {
                if (!task.isDone()) {
                    task.cancel(true);
                }
            }
            mDoingArtTransform.clear();
        }
    }
    
    private static class BackgroundThreadFactory implements ThreadFactory {
        private static int sTag = 1;

        @Override
        public Thread newThread(Runnable runnable) {
            Thread thread = new Thread(runnable);
            thread.setName("CustomThread" + sTag);
            thread.setPriority(Process.THREAD_PRIORITY_BACKGROUND);

            // A exception handler is created to log the exception from threads
            thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
                @Override
                public void uncaughtException(Thread thread, Throwable ex) {
                    Log.e("BackgroundThread", thread.getName() + " encountered an error: " + ex.getMessage());
                }
            });
            return thread;
        }
    }
}
```

Above is the core class for creating a custom thread pool which makes the device spare no effort to accomplish all the tasks. In our case, the AsyncTasks.

That's all. Thanks.
