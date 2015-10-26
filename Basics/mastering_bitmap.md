# Bitmap使用详解

by Cyandev

`Bitmap`是在Android开发中较为常见的一个类，顾名思义，该类会提供与位图相关的功能并为位图提供存储空间。然而`Bitmap`类并不提供位图文件解析加载的功能，也不会提供绘制相关的接口，因此本文除了介绍`Bitmap`类本身外还会提及与其相关的`BitmapFactory`和`BitmapDrawable`类，同时本文也会略谈Bitmap使用中的一些坑及需要注意的一些点。

## 读取Bitmap的方式

### 1. 从文件流中读取

首先准备位图文件的输入流，整个文件读取操作应当包裹在`try`语句块中：

```java
try {
	FileInputStream in = new FileInputStream("/sdcard/Download/sample.png");
} catch (FileNotFoundException e) {
	e.printStackTrace();
}
```

下面使用`BitmapFactory`类直接解析位图文件的`Stream`即可获得该`Bitmap`：

```java
Bitmap bitmap = BitmapFactory.decodeStream(in);
```

值得注意的是，当从文件创建`Bitmap`失败时，Android并不会抛出相关异常，而是会在`BitmapFactory#decodeStream`方法中返回`null`，因此在使用之前应该检查其是否为`null`

### 2. 从资源中读取

`BitmapFactory`类中同样提供了一个`decodeResource`方法，用于直接读取资源文件，使用方法非常简单：

```java
Bitmap bitmap = BitmapFactory.decodeResource(getContext().getResources(), R.drawable.sample);
```

从资源中读取图片时需要用到`Context`中的`Resources`类。

另外还有一种比较tricky的方式可以不使用到`R`类：

```java
Bitmap bitmap = BitmapFactory.decodeStream(getClass().getResourceAsStream("/res/drawable/sample.jpg"));
```

这种方式并不是最佳实践，不推荐经常使用。

### 3. 读取位图时的一点注意事项

当图片尺寸较大时，在主线程读取图片会有较长的耗时，此时可以通过降低采样率来获得类似缩略图的小尺寸`Bitmap`，从而提升加载速度：

```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = 2;  // 长和宽均为原来的1/2，即图片尺寸为原来的1/4
```

## 在UI中显示Bitmap

`Bitmap`终究是存储位图数据的类，它并不提供绘制的功能，若要使用`View`或`Canvas`显示该`Bitmap`我们就需要一个`Drawable`对象，用于封装`Bitmap`的`Drawable`类是`BitmapDrawable`：

```java
BitmapDrawable drawable = new BitmapDrawable(getResources(), bitmap);
```

Android中还提供了一个`BitmapDrawable`的构造方法不需要Resources但并现已被弃用，因为Android会通过Resources推断所加载位图的尺寸和像素密度。

## 裁剪和变换

`Bitmap#createBitmap`可以由一个原始Bitmap创建裁剪和变换后的新Bitmap，变换操作需要使用到`Matrix`，本文暂不展开解释，举例如下：

```java
Matrix mat = new Matrix();
mat.postScale(1.5f, 1.5f);

Bitmap newBitmap = Bitmap.createBitmap(bitmap, 100, 100, 100, 100, mat, true);
```

## 关于内存管理

### 1. inJustDecodeBounds的使用

在从文件构造Bitmap时，我们可以为`BitmapFactory`提供一个`Options`，并将`BitmapFactory.Options#inJustDecodeBounds`属性设置为`true`，这样，BitmapFactory并不会分配内存，而只会读取图片的大小。这样，根据屏幕大小和图片大小，我们就可以确定一个采样率来压缩读取图片的大小。

```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;

Bitmap bitmap = BitmapFactory.decodeXXX(xxx, options);
int width = options.outWidth;
int height = options.outHeight;

WindowManager wm = getWindowManager();
Display display = wm();
int screenWidth = display.getWidth();
int screenHeight = display.getHeight();

if (width > height) {
	if (width > screenWidth) {
		options.inSampleSize = width / screenWidth;
	}
} else {
	if (height > screenHeight) {
		options.inSampleSize = height / screenHeight;
	}
}

options.inJustDecodeBounds = false;

bitmap = BitmapFactory.decodeXXX(xxx, options);
```

### 2. 软引用

当应用中会使用到较多图片时，可以使用软引用在应用内做类似缓存的处理。关于软引用的使用，本文不展开赘述。简单说说思路：

用一个Map盛放`SoftReference<Bitmap>`，当需要调用图片时查询Map中是否含有这个图片的软引用，如果有，则从软引用中获取实例，判断实例是否为`null`，若不为`null`则直接使用该实例，否则使用文中上述方法加载图片并加入Map中。

### 3. 显式回收

在`Activity`中使用到的`Bitmap`在`onDestroy()`后不会马上回收，因此一定要显式调用`Bitmap#recycle`来回收，而`Bitmap#recycle`方法也不会马上回收内存，而只是将其标记为死亡对象，还需要系统通过GC来回收内存空间。可以调用

```java
System.gc();
```

来立刻回收位图所占用的内存空间。
