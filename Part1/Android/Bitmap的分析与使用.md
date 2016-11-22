## **Bitmap**的分析与使用
 - Bitmap的创建
    - 创建Bitmap的时候，Java不提供`new Bitmap()`的形式去创建，而是通过`BitmapFactory`中的静态方法去创建,如:`BitmapFactory.decodeStream(is);//通过InputStream去解析生成Bitmap`(这里就不贴`BitmapFactory`中创建`Bitmap`的方法了，大家可以自己去看它的源码)，我们跟进`BitmapFactory`中创建`Bitmap`的源码，最终都可以追溯到这几个native函数
    ```
        private static native Bitmap nativeDecodeStream(InputStream is, byte[] storage,
                    Rect padding, Options opts);
        private static native Bitmap nativeDecodeFileDescriptor(FileDescriptor fd,
                Rect padding, Options opts);
        private static native Bitmap nativeDecodeAsset(long nativeAsset, Rect padding, Options opts);
        private static native Bitmap nativeDecodeByteArray(byte[] data, int offset,
                int length, Options opts);
    ```
     而`Bitmap`又是Java对象，这个Java对象又是从native，也就是C/C++中产生的，所以，在Android中Bitmap的内存管理涉及到两部分，一部分是*native*，另一部分是*dalvik*，也就是我们常说的java堆(如果对java堆与栈不了解的同学可以戳)，到这里基本就已经了解了创建Bitmap的一些内存中的特性(大家可以使用``adb shell dumpsys meminfo``去查看Bitmap实例化之后的内存使用情况)。
 
 - Bitmap的使用
    - 我们已经知道了`BitmapFactory`是如何通过各种资源创建`Bitmap`了，那么我们如何合理的使用它呢？以下是几个我们使用`Bitmap`需要关注的点
        1. **Size**
            - 这里我们来算一下，在Android中，如果采用`Config.ARGB_8888`的参数去创建一个`Bitmap`，[这是Google推荐的配置色彩参数](https://developer.android.com/reference/android/graphics/Bitmap.Config.html)，也是Android4.4及以上版本默认创建Bitmap的Config参数(``Bitmap.Config.inPreferredConfig``的默认值)，那么每一个像素将会占用4byte，如果一张手机照片的尺寸为1280×720，那么我们可以很容易的计算出这张图片占用的内存大小为 1280x720x4 = 3686400(byte) = 3.5M，一张未经处理的照片就已经3.5M了! 显而易见，在开发当中，这是我们最需要关注的问题，否则分分钟OOM!
            - *那么，我们一般是如何处理Size这个重要的因素的呢？*，当然是调整`Bitmap`的大小到适合的程度啦！辛亏在`BitmapFactory`中，我们可以很方便的通过`BitmapFactory.Options`中的`options.inSampleSize`去设置`Bitmap`的压缩比，官方给出的说法是 
            > If set to a value > 1, requests the decoder to subsample the original image, returning a smaller image to save memory....For example, inSampleSize == 4 returns
            an image that is 1/4 the width/height of the original, and 1/16 the
            number of pixels. Any value <= 1 is treated the same as 1.
            
             很简洁明了啊！也就是说，只要按计算方法设置了这个参数，就可以完成我们Bitmap的Size调整了。那么，应该怎么调整姿势才比较舒服呢？下面先介绍其中一种通过``InputStream``的方式去创建``Bitmap``的方法，上一段从Gallery中获取照片并且将图片Size调整到合适手机尺寸的代码：
        ```
            static final int PICK_PICS = 9;
            
            public void startGallery(){
                Intent i = new Intent();
                i.setAction(Intent.ACTION_PICK);
                i.setType("image/*");
                startActivityForResult(i,PICK_PICS);
            }
            
             private int[] getScreenWithAndHeight(){
                WindowManager wm = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
                DisplayMetrics dm = new DisplayMetrics();
                wm.getDefaultDisplay().getMetrics(dm);
                return new int[]{dm.widthPixels,dm.heightPixels};
            }
            
            /**
             *
             * @param actualWidth 图片实际的宽度，也就是options.outWidth
             * @param actualHeight 图片实际的高度，也就是options.outHeight
             * @param desiredWidth 你希望图片压缩成为的目的宽度
             * @param desiredHeight 你希望图片压缩成为的目的高度
             * @return
             */
            private int findBestSampleSize(int actualWidth, int actualHeight, int desiredWidth, int desiredHeight) {
                double wr = (double) actualWidth / desiredWidth;
                double hr = (double) actualHeight / desiredHeight;
                double ratio = Math.min(wr, hr);
                float n = 1.0f;
                //这里我们为什么要寻找 与ratio最接近的2的倍数呢？
                //原因就在于API中对于inSimpleSize的注释：最终的inSimpleSize应该为2的倍数，我们应该向上取与压缩比最接近的2的倍数。
                while ((n * 2) <= ratio) {
                    n *= 2;
                }
        
                return (int) n;
            }

            @Override
            protected void onActivityResult(int requestCode, int resultCode, Intent data) {
                if(resultCode == RESULT_OK){
                    switch (requestCode){
                        case PICK_PICS:
                            Uri uri = data.getData();
                            InputStream is = null;
                            try {
                                is = getContentResolver().openInputStream(uri);
                            } catch (FileNotFoundException e) {
                                e.printStackTrace();
                            }
        
                            BitmapFactory.Options options = new BitmapFactory.Options();
                            //当这个参数为true的时候,意味着你可以在解析时候不申请内存的情况下去获取Bitmap的宽和高
                            //这是调整Bitmap Size一个很重要的参数设置
                            options.inJustDecodeBounds = true;
                            BitmapFactory.decodeStream( is,null,options );
        
                            int realHeight = options.outHeight;
                            int realWidth = options.outWidth;
        
                            int screenWidth = getScreenWithAndHeight()[0];
                            
                            int simpleSize = findBestSampleSize(realWidth,realHeight,screenWidth,300);
                            options.inSampleSize = simpleSize;
                            //当你希望得到Bitmap实例的时候，不要忘了将这个参数设置为false
                            options.inJustDecodeBounds = false;
        
                            try {
                                is = getContentResolver().openInputStream(uri);
                            } catch (FileNotFoundException e) {
                                e.printStackTrace();
                            }
        
                            Bitmap bitmap = BitmapFactory.decodeStream(is,null,options);
        
                            iv.setImageBitmap(bitmap);
        
                            try {
                                is.close();
                                is = null;
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
        
                            break;
                    }
                }
                super.onActivityResult(requestCode, resultCode, data);
            }
        ```
        
     我们来看看这段代码的功效：
     压缩前：![压缩前](https://leanote.com/api/file/getImage?fileId=578d9ed8ab644135ea01684c)
     压缩后：![压缩后](https://leanote.com/api/file/getImage?fileId=578d9f76ab644135ea016851)
     **对比条件为：1080P的魅族Note3拍摄的高清无码照片**
            
        2. **Reuse**
        上面介绍了``BitmapFactory``通过``InputStream``去创建`Bitmap`的这种方式，以及``BitmapFactory.Options.inSimpleSize`` 和 ``BitmapFactory.Options.inJustDecodeBounds``的使用方法，但将单个Bitmap加载到UI是简单的，但是如果我们需要一次性加载大量的图片，事情就会变得复杂起来。`Bitmap`是吃内存大户，我们不希望多次解析相同的`Bitmap`，也不希望可能不会用到的`Bitmap`一直存在于内存中，所以，这个场景下，`Bitmap`的重用变得异常的重要。
        *在这里只介绍一种``BitmapFactory.Options.inBitmap``的重用方式，下一篇文章会介绍使用三级缓存来实现Bitmap的重用。*
        
              根据官方文档[在Android 3.0 引进了BitmapFactory.Options.inBitmap](https://developer.android.com/reference/android/graphics/BitmapFactory.Options.html#inBitmap)，如果这个值被设置了，decode方法会在加载内容的时候去重用已经存在的bitmap. 这意味着bitmap的内存是被重新利用的，这样可以提升性能, 并且减少了内存的分配与回收。然而，使用inBitmap有一些限制。特别是在Android 4.4 之前，只支持同等大小的位图。
            我们看来看看这个参数最基本的运用方法。
            
            ```
            new BitmapFactory.Options options = new BitmapFactory.Options();
            //inBitmap只有当inMutable为true的时候是可用的。
            options.inMutable = true;
            Bitmap reusedBitmap = BitmapFactory.decodeResource(getResources(),R.drawable.reused_btimap,options);
            options.inBitmap = reusedBitmap;
            ```
            
              这样，当你在下一次decodeBitmap的时候，将设置了`options.inMutable=true`以及`options.inBitmap`的`Options`传入，Android就会复用你的Bitmap了，具体实例：
            
            ```
            @Override
            protected void onCreate(@Nullable Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(reuseBitmap());
            }
    
            private LinearLayout reuseBitmap(){
                LinearLayout linearLayout = new LinearLayout(this);
                linearLayout.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                linearLayout.setOrientation(LinearLayout.VERTICAL);
        
                ImageView iv = new ImageView(this);
                iv.setLayoutParams(new ViewGroup.LayoutParams(500,300));
        
                options = new BitmapFactory.Options();
                options.inJustDecodeBounds = true;
                //inBitmap只有当inMutable为true的时候是可用的。
                options.inMutable = true;
                BitmapFactory.decodeResource(getResources(),R.drawable.big_pic,options);
                
                //压缩Bitmap到我们希望的尺寸
                //确保不会OOM
                options.inSampleSize = findBestSampleSize(options.outWidth,options.outHeight,500,300);
                options.inJustDecodeBounds = false;
        
                Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.big_pic,options);
                options.inBitmap = bitmap;
        
                iv.setImageBitmap(bitmap);
        
                linearLayout.addView(iv);
        
                ImageView iv1 = new ImageView(this);
                iv1.setLayoutParams(new ViewGroup.LayoutParams(500,300));
                iv1.setImageBitmap( BitmapFactory.decodeResource(getResources(),R.drawable.big_pic,options));
                linearLayout.addView(iv1);
        
                ImageView iv2 = new ImageView(this);
                iv2.setLayoutParams(new ViewGroup.LayoutParams(500,300));
                iv2.setImageBitmap( BitmapFactory.decodeResource(getResources(),R.drawable.big_pic,options));
                linearLayout.addView(iv2);
        
        
                return linearLayout;
            }
            ```
            
             以上代码中，我们在解析了一次一张1080P分辨率的图片，并且设置在`options.inBitmap`中，然后分别decode了同一张图片，并且传入了相同的`options`。最终只占用一份第一次解析`Bitmap`的内存。
            
        3. **Recycle**
        一定要记得及时回收Bitmap，否则如上分析，你的native以及dalvik的内存都会被一直占用着，最终导致OOM
        
        
        ```
        // 先判断是否已经回收
        if(bitmap != null && !bitmap.isRecycled()){
            // 回收并且置为null
            bitmap.recycle();
            bitmap = null;
        }
        System.gc();
        ```
        
    - Enjoy Android  :) 如果有误，轻喷，欢迎指正。

