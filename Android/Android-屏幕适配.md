#### 基本概念

##### 1. 屏幕尺寸

屏幕尺寸指<u>屏幕对角线长度</u>，单位是英寸，1 英寸 = 2.54 厘米

##### 2. 屏幕分辨率

屏幕分辨率指<u>在横纵方向上的像素点数</u>，单位是 `px`，常见的有 360 x 640、1080 x 1920

##### 3. 屏幕像素密度

屏幕像素密度指<u>每英寸上的像素点数</u>，单位是 `dpi (dot per inch)`，像素密度取决于屏幕尺寸与分辨率

***

#### 常见单位

##### 1. px

像素的单位

##### 2. dpi

屏幕像素密度的单位

##### 3. dip

也可写作 `dp`，密度无关像素，`density-independent pixel`，使用 `dp` 作为长度单位，可以保证不同屏幕像素密度的手机上显示相似的效果

在 160dpi(基准密度) 的屏幕上，1dp 约等于 1px，而在 320dpi 的屏幕上，1dp 约等于 2px，<u>换算公式：px = dp * (屏幕像素密度 / 160)</u>

##### 4. sp

比例无关像素，`scale-independent pixel`，用于字体单位，与 `dp` 类似，同时受系统字体大小影响：若系统字体大小为标准或默认，则 sp 与 px 的换算等同于 dp 与 px 的换算

````kotlin
/**
 * 根据当前像素密度将dp转换为px时的缩放系数
 */
public float density;

/**
 * 根据当前像素密度将sp转换为px时的缩放系数，与density相同，但是会在运行时根据系统字体大小进行调整
 */
public float scaledDensity;
````

***

#### 屏幕适配

##### 1. 单位转换

````kotlin
fun dpToPx(value: Int): Int {
    //density：根据当前像素密度将dp转换为px时的缩放系数
    val scale: Float = context.resources.displayMetrics.density
    return (value * scale + 0.5F).toInt()
}
````

##### 2. 备用位图

> 屏幕尺寸相同的情况下，如果有一张 64 x 64 的图片在 120dpi 的屏幕上显示的大小正合适，而在 240dpi 的屏幕上大小则会变为原来的二分之一，所以需要为不同 dpi 的设备提供相应的图片

![Android_屏幕适配.webp (505×201) (raw.githubusercontent.com)](https://raw.githubusercontent.com/WeYan1223/Pic/master/Android/Android_屏幕适配.webp) 

##### 3. 矢量图

