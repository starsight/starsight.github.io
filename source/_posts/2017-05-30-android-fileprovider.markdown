---
layout: post
title: "Android7.0完美适配——FileProvider拍照裁剪全解析"
date: 2017-05-30 14:29:42 +0800
comments: true
categories: 2017-05
tags: [android,FileProvider]
---
在做android7.0的适配时，发现拍照裁剪图片等功能莫名其妙地崩溃了。通过观察控制台的崩溃记录，原因很明显，file:// 不被允许作为一个附加的 Uri 的意图，否则会抛出 FileUriExposedException，接下来就依次适配之。<!--more-->

#### FileProvider介绍
>官方对于 FileProvider 的解释为：FileProvider 是一个特殊的 ContentProvider 子类，通过 content://Uri 代替 file://Uri 实现不同 App 间的文件安全共享。当通过包含 Content URI 的 Intent 共享文件时，需要申请临时的读写权限，可以通过 Intent.setFlags() 方法实现。
>而 file://Uri 方式需要申请长期有效的文件读写权限，直到这个权限被手动改变为止，这是极其不安全的做法。因此 Android 从 N 版本开始禁止通过 file://Uri 在不同 App 之间共享文件。
>FileProvider并不是最新出来的东西，而是以前就已经存在，由于Android的安全机制 ，一个进程默认不能影响另外一个进程的，如读取私有数据 。 那么对于进程间的文件的共享 ，出于安全考虑，用FileProvider。FileProvider会基于manifest中的定义定义的一个xml文件（xml目录 下），为所有定义的文件生成content URIs，这样外部的应用在没有权限的情况下，可以通过授予临时权限的content uri，读取相应的文件。
>谷歌做这项规定主要是针对，包含文件 URI 的 Intent 离开你的应用，换句话说，如果你的Intent中用到了Uri，这个时候你就需要提防一下了，比如说，你使用到了图片裁剪等功能。

#### 需要适配修改的地方
>Uri.parse
>Uri.fromFile
>file://
>content://
>Context.getFilesDir()、Environment.getExternalStorageDirectory()、getCacheDir()以及最终要的intent.setDataAndType(为什么需要找这个，因为这个会携带uri进行传递，这个是重头戏)

#### FileProvider基本使用流程
1.定义一个 FileProvider
2.指定共享目录
3.为文件生成有效的 Content URI
4.申请临时的读写权限
5.发送 Content URI 至其他的 App

##### 定义一个 FileProvider 
因为是ContentProvider的子类，所以也必须要在Manifest.xml中声明
```xml
<application>
    ...
    <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="${applicationId}.provider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths"/>
    </provider>
    ...
</application>
```
name的写法是固定的，不过如果你打算作为lib提供给别人可能要考虑冲突，可以继承这个类，然后不实现，以作区分。此说法来自参考链接三。

grantUriPermissions：申明为true，你才能获取临时共享权限

##### 指定共享目录
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <!--        xml文件是唯一设置分享的目录 ，不能用代码设置

         1.<files-path>        getFilesDir()  /data/data//files目录
         2.<cache-path>        getCacheDir()  /data/data//cache目录

         3.<external-path>     Environment.getExternalStorageDirectory()

         SDCard/Android/data/你的应用的包名/files/ 目录
         4.<external-files-path>     Context#getExternalFilesDir(String) Context.getExternalFilesDir(null).
         5.<external-cache-path>      Context.getExternalCacheDir().
     -->

    <!--    path :代表设置的目录下一级目录 eg：<external-path path="images/"
                整个目录为Environment.getExternalStorageDirectory()+"/images/"
            name: 代表定义在Content中的字段 eg：name = "myimages" ，并且请求的内容的文件名为default_image.jpg
                则 返回一个URI   content://com.example.myapp.fileprovider/myimages/default_image.jpg
    -->
    <!--当path 为空时 5个全配置就可以解决-->
    <!--下载apk-->
    <external-path path="" name="sdcard_files" />
    <!--相机相册裁剪-->
    <external-files-path   path="file/" name="camera_has_sdcard"/>
    <files-path path=""     name="camera_no_sdcard"/>
</paths>
```
可以看出，这五种子元素基本涵盖内外存储空间所有目录路径，包含应用私有目录。同时，每个子元素都拥有 name 和 path 两个属性。
path 属性用于指定当前子元素所代表目录下需要共享的子目录名称。注意：path 属性值不能使用具体的独立文件名，只能是目录名。path只能添加一个路径，如果需要共享多个则指定多个即可。
name 属性用于给 path 属性所指定的子目录名称取一个别名。后续生成 content:// URI 时，会使用这个别名代替真实目录名。这样做的目的，很显然是为了提高安全性。

##### 生成有效的 Content URI
这边简略叙述，后面实战时详细介绍。

在 Android 7.0 出现之前，我们通常使用 Uri.fromFile() 方法生成一个 File URI。这里，我们需要使用 FileProvider 类提供的公有静态方法 getUriForFile 生成 Content URI。
>文件配置完成后还需要生成可以被其他 App 访问的 Content URI，可以直接调用 FileProvider 提供的 getUriForFile(File file) 方法，顾名思义，传入文件名称就可以得到相应的 Content URI 。需要访问该文件的 App 可以通过 ContentResolver.openFileDescriptor 得到一个 ParcelFileDescriptor 对象。
>
>假定你想要共享一个图片文件，文件存放的位置为手机内部存储空间下的 images 文件夹，图片文件名字为 default_name.jpg ，那么生成 Content URI 方式如下：


>```
File imagePath = new File(getContext().getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = getUriForFile(getContext(), "com.mydomain.provider", newFile);
```
>最后生成的 Content URI 为
>```
content://com.domain.example.provider/images/default_image.jpg.
```

##### 申请临时的读写权限
>生成 Content URI 对象后，需要对其授权访问权限。授权方式有两种：
>第一种方式，使用 Context 提供的 grantUriPermission(package, Uri, mode_flags) 方法向其他应用授权访问 URI 对象。三个参数分别表示授权访问 URI 对象的其他应用包名，授权访问的 Uri 对象，和授权类型。其中，授权类型为 Intent 类提供的读写类型常量：

>FLAG_GRANT_READ_URI_PERMISSION

>FLAG_GRANT_WRITE_URI_PERMISSION

>或者二者同时授权。这种形式的授权方式，权限有效期截止至发生设备重启或者手动调用 revokeUriPermission() 方法撤销授权时。

>第二种方式，配合 Intent 使用。通过 setData() 方法向 intent 对象添加 Content URI。然后使用 setFlags() 或者 addFlags() 方法设置读写权限，可选常量值同上。这种形式的授权方式，权限有效期截止至其它应用所处的堆栈销毁，并且一旦授权给某一个组件后，该应用的其它组件拥有相同的访问权限。

##### 发送 Content URI
>拥有授予权限的 Content URI 后，便可以通过 startActivity() 或者 setResult() 方法启动其他应用并传递授权过的 Content URI 数据。当然，也有其他方式提供服务。

>如果你需要一次性传递多个 URI 对象，可以使用 intent 对象提供的 setClipData() 方法，并且 setFlags() 方法设置的权限适用于所有 Content URIs。

举个例子:
```java
Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
intent.addCategory(Intent.CATEGORY_OPENABLE);
intent.setType("image/*");
Uri uriForFile = FileProvider.getUriForFile(this,"com.wenjiehe.android_study.fileprovider", mGalleryFile);
intent.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
startActivityForResult(intent, SELECT_PIC_NOUGAT);
```

[官方文档地址](https://developer.android.com/reference/android/support/v4/content/FileProvider.html)

#### 两种过程对比

为什么在 Android Nougat 下 file:// 不被允许？
其实背后有一个很好的理由，如果文件路径被发送到目标应用程序（在这种情况下，是相机应用程序），文件将在访问相机应用程序的过程中被完全访问，而不仅仅只有发起者能收到。（原文：If file path is sent to the target application (Camera app in this case), file will be fully accessed through the Camera app's process not the sender one.）实际上就是我们把控制权交给了相机程序，而作为这个file的拥有者的我们完全丧失了控制权。

{% img /images/fileprovider/1.jpg%} 

但让我们考虑一下，实际上是由我们的应用程序去启动摄像头拍照，并保存作为我们的应用程序的代表文件。因此，该文件的访问权限应该是我们的应用程序而不是摄像头应用程序本身。这就是为什么现在 file:// 在 targetSdkVersion 24 中要求每一位开发者都去完成这个任务。

FileProvider的解决方案

{% img /images/fileprovider/2.jpg%} 

使用FileProvider，其实就是收回控制权，通过赋予相机程序临时的读写权限，掌握File文件的绝对控制！

#### FileProvider实战

##### 手机拍照
在 Android N 之前的版本调用相机获取图片可以用如下代码实现
```java
// 指定调用相机拍照后照片的储存路径
File imgFile = new File(imgPath);
Uri imgUri = null;
imgUri = Uri.fromFile(imgFile);
Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
intent.putExtra(MediaStore.EXTRA_OUTPUT,imgUri);
startActivityForResult(intent, takePhotoRequestCode);
```

这边主要修改fromFile的调用
```java
String path = Environment.getExternalStorageDirectory()+"/Android/data/com.wenjiehe.android_study/files/";
File mCameraFile = new File(path, "IMAGE_FILE_NAME.jpg");//照相机的File对象
Intent intentFromCapture = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {//7.0及以上
    Uri uriForFile = FileProvider.getUriForFile(this, "com.wenjiehe.android_study.fileprovider", mCameraFile);
    intentFromCapture.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
    intentFromCapture.addFlags(FLAG_GRANT_READ_URI_PERMISSION);
    intentFromCapture.addFlags(FLAG_GRANT_WRITE_URI_PERMISSION);
//  mCameraFile -> /storage/emulated/0/Android/data/com.wenjiehe.android_study/files/IMAGE_FILE_NAME.jpg
} else {
    intentFromCapture.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(mCameraFile));
}
startActivityForResult(intentFromCapture, CAMERA_REQUEST_CODE);
```
这边我是放在了外部存储的私有目录。值得说明的是，这个目录的访问也是不需要访问SD卡的权限的，外部私有目录。

```java
case CAMERA_REQUEST_CODE: {//照相后返回
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
        Uri inputUri = FileProvider.getUriForFile(this, "com.wenjiehe.android_study.fileprovider", mCameraFile);//通过FileProvider创建一个content类型的Uri
// inputUri -> content://com.wenjiehe.android_study.fileprovider/sdcard_files/Android/data/com.wenjiehe.android_study/files/IMAGE_FILE_NAME.jpg
        startPhotoZoom(inputUri);//设置输入类型
    } else {
        Uri inputUri = Uri.fromFile(mCameraFile);
        startPhotoZoom(inputUri);
    }
    break;
}
```


##### 相册
```java
Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
intent.addCategory(Intent.CATEGORY_OPENABLE);
intent.setType("image/*");
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {//如果大于等于7.0使用FileProvider
    Uri uriForFile = FileProvider.getUriForFile
            (this, "com.wenjiehe.android_study.fileprovider", mGalleryFile);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
    intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    startActivityForResult(intent, SELECT_PIC_NOUGAT);
} else {
    //intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(mGalleryFile));
    startActivityForResult(intent, IMAGE_REQUEST_CODE);
}
```

```java
case IMAGE_REQUEST_CODE: {//版本<7.0  图库后返回
    if (data != null) {
        // 得到图片的全路径
        Uri uri = data.getData();
        //crop(uri);
        startPhotoZoom(uri);
    }
    break;
}
case SELECT_PIC_NOUGAT://版本>= 7.0
    File imgUri = new File(GetImagePath.getPath(this, data.getData()));
    Uri dataUri = FileProvider.getUriForFile
            (this, "com.wenjiehe.android_study.fileprovider", imgUri);
    // Uri dataUri = getImageContentUri(data.getData());
    startPhotoZoom(dataUri);
    break;
```

##### 裁剪
```java
public void startPhotoZoom(Uri inputUri) {
    if (inputUri == null) {
        Log.e("error","The uri is not exist.");
        return;
    }

    Intent intent = new Intent("com.android.camera.action.CROP");
    //sdk>=24
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
        Uri outPutUri = Uri.fromFile(mCropFile);
// outPutUri -> file:///storage/emulated/0/Android/data/com.wenjiehe.android_study/files/PHOTO_FILE_NAME.jpg
        intent.setDataAndType(inputUri, "image/*");
        intent.putExtra(MediaStore.EXTRA_OUTPUT, outPutUri);
        intent.addFlags(FLAG_GRANT_READ_URI_PERMISSION);
        intent.addFlags(FLAG_GRANT_WRITE_URI_PERMISSION);
    } else {
        Uri outPutUri = Uri.fromFile(mCropFile);
        if (Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.KITKAT) {
            String url = GetImagePath.getPath(this, inputUri);//这个方法是处理4.4以上图片返回的Uri对象不同的处理方法
            intent.setDataAndType(Uri.fromFile(new File(url)), "image/*");
        } else {
            intent.setDataAndType(inputUri, "image/*");
        }
        intent.putExtra(MediaStore.EXTRA_OUTPUT, outPutUri);
    }


    // 设置裁剪
    intent.putExtra("crop", "true");
    // aspectX aspectY 是宽高的比例
    intent.putExtra("aspectX", 1);
    intent.putExtra("aspectY", 1);
    // outputX outputY 是裁剪图片宽高
    intent.putExtra("outputX", 250);
    intent.putExtra("outputY", 250);
    intent.putExtra("return-data", false);
    intent.putExtra("noFaceDetection", false);//去除默认的人脸识别，否则和剪裁匡重叠
    intent.putExtra("outputFormat", "JPEG");
    //intent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());// 图片格式
    startActivityForResult(intent, RESULT_REQUEST_CODE);//这里就将裁剪后的图片的Uri返回了
}
```
版本高于7.0的，显然要是用FileProvider
版本高于4.4的，图片路径有一个分水岭问题，所以需要转换一下

#### 终章

##### 源码

```java
public class Button3Activity extends AppCompatActivity implements View.OnClickListener{

    @BindView(R.id.button30)
    Button button30;

    @BindView(R.id.button31)
    Button button31;

    @BindView(R.id.iv_photo)
    ImageView iv_photo;


    String path = Environment.getExternalStorageDirectory()+"/Android/data/com.wenjiehe.android_study/files/";
    File mCameraFile = new File(path, "IMAGE_FILE_NAME.jpg");//照相机的File对象
    File mCropFile = new File(path, "PHOTO_FILE_NAME.jpg");//裁剪后的File对象
    File mGalleryFile = new File(path, "IMAGE_GALLERY_NAME.jpg");//相册的File对象


    private static final int IMAGE_REQUEST_CODE = 100;
    private static final int SELECT_PIC_NOUGAT = 101;
    private static final int RESULT_REQUEST_CODE = 102;
    private static final int CAMERA_REQUEST_CODE = 104;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_button3);

        ButterKnife.bind(this);

        button30.setOnClickListener(this);
        button31.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()){
            case R.id.button30:{


                Intent intentFromCapture = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {//7.0及以上
                    Uri uriForFile = FileProvider.getUriForFile(this, "com.wenjiehe.android_study.fileprovider", mCameraFile);
                    intentFromCapture.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
                    intentFromCapture.addFlags(FLAG_GRANT_READ_URI_PERMISSION);
                    intentFromCapture.addFlags(FLAG_GRANT_WRITE_URI_PERMISSION);
                } else {
                    intentFromCapture.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(mCameraFile));
                }
                startActivityForResult(intentFromCapture, CAMERA_REQUEST_CODE);

                break;
            }
            case R.id.button31:{
                Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
                intent.addCategory(Intent.CATEGORY_OPENABLE);
                intent.setType("image/*");
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {//如果大于等于7.0使用FileProvider
                    Uri uriForFile = FileProvider.getUriForFile
                            (this, "com.wenjiehe.android_study.fileprovider", mGalleryFile);
                    intent.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
                    intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
                    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                    startActivityForResult(intent, SELECT_PIC_NOUGAT);
                } else {
                    //intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(mGalleryFile));
                    startActivityForResult(intent, IMAGE_REQUEST_CODE);
                }
                break;
            }
        }
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {

        switch (requestCode){
            case CAMERA_REQUEST_CODE: {//照相后返回
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                    Uri inputUri = FileProvider.getUriForFile(this, "com.wenjiehe.android_study.fileprovider", mCameraFile);//通过FileProvider创建一个content类型的Uri

                    startPhotoZoom(inputUri);//设置输入类型
                } else {
                    Uri inputUri = Uri.fromFile(mCameraFile);
                    startPhotoZoom(inputUri);
                }
                break;
            }
            case IMAGE_REQUEST_CODE: {//版本<7.0  图库后返回
                if (data != null) {
                    // 得到图片的全路径
                    Uri uri = data.getData();
                    //crop(uri);
                    startPhotoZoom(uri);
                }
                break;
            }
            case SELECT_PIC_NOUGAT://版本>= 7.0
                File imgUri = new File(GetImagePath.getPath(this, data.getData()));
                Uri dataUri = FileProvider.getUriForFile
                        (this, "com.wenjiehe.android_study.fileprovider", imgUri);
                // Uri dataUri = getImageContentUri(data.getData());
                startPhotoZoom(dataUri);
                break;
            case RESULT_REQUEST_CODE:{
                Uri inputUri = FileProvider.getUriForFile(this, "com.wenjiehe.android_study.fileprovider", mCropFile);//通过FileProvider创建一个content类型的Uri
                Bitmap bitmap = null;
                try {
                    bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(inputUri));
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                }
                //Bitmap bitmap = data.getParcelableExtra("data");
                iv_photo.setImageBitmap(bitmap);
                break;
            }

        }
    }

    private void crop(Uri uri) {
        // 裁剪图片意图
        Intent intent = new Intent("com.android.camera.action.CROP");
        intent.setDataAndType(uri, "image/*");
        intent.putExtra("crop", "true");
        // 裁剪框的比例，1：1
        intent.putExtra("aspectX", 1);
        intent.putExtra("aspectY", 1);
        // 裁剪后输出图片的尺寸大小
        intent.putExtra("outputX", 250);
        intent.putExtra("outputY", 250);
        // 图片格式
        intent.putExtra("outputFormat", "JPEG");
        intent.putExtra("noFaceDetection", true);// 取消人脸识别
        intent.putExtra("return-data", true);// true:不返回uri，false：返回uri
        intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        startActivityForResult(intent, RESULT_REQUEST_CODE);
    }

    /**
     * 裁剪图片方法实现
     *
     * @param inputUri
     */
    public void startPhotoZoom(Uri inputUri) {
        if (inputUri == null) {
            Log.e("error","The uri is not exist.");
            return;
        }

        Intent intent = new Intent("com.android.camera.action.CROP");
        //sdk>=24
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {

            Uri outPutUri = Uri.fromFile(mCropFile);
            intent.setDataAndType(inputUri, "image/*");
            intent.putExtra(MediaStore.EXTRA_OUTPUT, outPutUri);
            intent.addFlags(FLAG_GRANT_READ_URI_PERMISSION);
            intent.addFlags(FLAG_GRANT_WRITE_URI_PERMISSION);

        } else {
            Uri outPutUri = Uri.fromFile(mCropFile);
            if (Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.KITKAT) {
                String url = GetImagePath.getPath(this, inputUri);//这个方法是处理4.4以上图片返回的Uri对象不同的处理方法
                intent.setDataAndType(Uri.fromFile(new File(url)), "image/*");
            } else {
                intent.setDataAndType(inputUri, "image/*");
            }
            intent.putExtra(MediaStore.EXTRA_OUTPUT, outPutUri);
        }


        // 设置裁剪
        intent.putExtra("crop", "true");
        // aspectX aspectY 是宽高的比例
        intent.putExtra("aspectX", 1);
        intent.putExtra("aspectY", 1);
        // outputX outputY 是裁剪图片宽高
        intent.putExtra("outputX", 250);
        intent.putExtra("outputY", 250);
        intent.putExtra("return-data", false);
        intent.putExtra("noFaceDetection", false);//去除默认的人脸识别，否则和剪裁匡重叠
        intent.putExtra("outputFormat", "JPEG");
        //intent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());// 图片格式
        startActivityForResult(intent, RESULT_REQUEST_CODE);//这里就将裁剪后的图片的Uri返回了
    }


}
```

布局就不贴了，两个button，一个imageview。这代码没写权限管理的，所以要先赋好。


##### 拍照流程
给出Content Uri，调用相机应用程序，拍摄完成（在SD卡中能找到），再根据Content Uri，转入裁剪程序，裁剪有两个Uri，输入和输出，这里输入是Content Uri的，输出是File Uri的，输出必须是File Uri。

##### 相册流程
给出Content Uri，调用相册程序，图片问题同样有分水岭问题（之前拍照不需要是因为图片路径是已知的，我们自己指定的），转化之后与拍照的流程一致。

以上所说流程均是Android 7.0

#### 注意事项

##### 注意一：
>仔细的看下这个方法：
>```java
File imgUri = new File(GetImagePath.getPath(getContext(), data.getData()));
Uri dataUri = FileProvider.getUriForFile(getActivity(), "com.renwohua.conch.fileprovider", imgUri);
startPhotoZoom(dataUri);
```
>来讲下原理：打开相机这个应用，开始拍照，然后自己提供一个储存的路径，并创建一个共享ContentUri，来将照片放在你这个ContentUri里面，这个东西只是一个虚拟的，拍的照片并没有保存在手机里面(lz注：应该是存的，不过控制权在我们这，且路径固定)，而相册不同，你是去访问相册里面的本身自带的照片，它本身已经存储了路径，你在
>```java
Uri uriForFile = FileProvider.getUriForFile(getActivity(), "com.renwohua.conch.fileprovider", mGalleryFile); intent.putExtra(MediaStore.EXTRA_OUTPUT, uriForFile);
```
>这里定义了输出到这个ContentUri里面，但是从onActivityForResult中像相机一样使用
>```java
Uri inputUri = FileProvider.getUriForFile(getActivity(), "com.renwohua.conch.fileprovider", mCameraFile);//通过FileProvider创建一个content类型的Uri startPhotoZoom(inputUri);//设置输入类型`
```
>这种方式，获取的ContentUri里面并没有获取到相应的图片，只能通过data.getData()获取，因为这是相册本身自带的，它有自己的定义的输出目录，那就是data.getData(),同时，假如你的手机是7.0,你返回的Uri也面临着4.4图片分水岭的问题，所以也需要使用GetImagePath.getPath(getContext(), data.getData());去获取Uri，然后再通过FileProvider去做裁剪的动作,接下来就和上面的相机的裁剪一样的了。

##### 注意二：
>看我们前面的xml里面external-path,也就是外部路径,我们知道,现在的Android手机都把内置了储存空间,也是相当于一个SD卡,如果你有第二块SD呢,如三星的机型,如果传入的SD卡中的图片路径就会报错,why?因为我们我们前面写的external-path,表示的外部储存的路径,也就是你的机身储存那么路径又要怎么表示?前面我们翻译了文档,发现上面并没有说明这类情况,那么我们该怎么解决?看FileProvide的源码. 
>{% img /images/fileprovider/3.jpg%}
>发现里面除了文档上面说的那五类中路径没还有一个就是root-path,也就是整个手机的根路径,那就好办了
>{% img /images/fileprovider/4.jpg%}
