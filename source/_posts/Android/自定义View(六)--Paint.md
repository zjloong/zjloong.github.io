---
title: 自定义View(六)--Paint
top: false
cover: true
categories: Android
tags:
  - View
  - Canvas
  - Paint
date: 2019-03-21 21:11:16
img:
coverImg:
summary: 
---

### 构造方法
1. 直接构造方法
```java
// Create a new paint with default settings
public Paint()
// 使用指定的 flags 初始化 paint. 后续也可以通过 setFlags() 去改变这些 flags
public Paint(int flags)
// 通过一个已存在的 paint, 创建一个新的 paint
public Paint(Paint paint)
```
2. 间接构造方法
```java
// 将 paint 重置为默认状态 (相当于重新 new一个, 不过效率更高)
public void reset()
// 复制 src的所有属性
public void set(Paint src)
```

### flags
给 paint 设置 flag标记, 可以选择是否开启某些特殊的效果. flags可以通过构造方法传入, 可以通过 setFlags() 设置.
```java
// 方法签名
public void setFlags(int flags)
// 可以一次设置多个 flag, 使用 | 连接
mPaint.setFlags(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
```

flags 可以取以下值:
```java
// 开启 抗锯齿
public static final int ANTI_ALIAS_FLAG     = 0x01;
// 在缩放位图上启用 双线性采样(通过缩放图片像素来减少图片占用内存大小, 绘制图片时建议开启)
public static final int FILTER_BITMAP_FLAG  = 0x02;
// 设置放抖动的 (不懂什么意思, 通过实际对比可以发现启用后色彩更柔和, 建议开启)
public static final int DITHER_FLAG         = 0x04;
// 绘制文本时, 添加下划线
public static final int UNDERLINE_TEXT_FLAG = 0x08;
// 绘制文本时, 添加贯穿线
public static final int STRIKE_THRU_TEXT_FLAG = 0x10;
// 绘制文本时, 加粗
public static final int FAKE_BOLD_TEXT_FLAG = 0x20;
// 使文本平滑线性扩展的标志 (基本没用了)
public static final int LINEAR_TEXT_FLAG    = 0x40;
// 是否打开亚像素设置来绘制文本. 
// 亚像素: 将相邻两个像素之间的距离再细分, 再插入一些像素(即亚像素). 总结就是通过程序计算的方式增加像素, 可增强文本清晰度, 但更耗性能 (红米的 4800万像素?)
public static final int SUBPIXEL_TEXT_FLAG  = 0x80;
// 绘制文本时启用位图字体的绘制标志. (来自百度翻译)
public static final int EMBEDDED_BITMAP_TEXT_FLAG = 0x400;
```

对于这些 flags, Paint也提供了一些快捷的方法去设置
```java
// 抗锯齿
mPaint.setAntiAlias(true);  == mPaint.setFlags(Paint.ANTI_ALIAS_FLAG);
// 设置双线性过滤: 优化 Bitmap 放大绘制的效果
mPaint.setFilterBitmap(true); == mPaint.setFlags(Paint.FILTER_BITMAP_FLAG);
// 设置图像的抖动: 优化色彩深度降低时的绘制效果
mPaint.setDither(true); == mPaint.setFlags(Paint.DITHER_FLAG);
// ... 一般只有这三种 flag 较常用, 其它就不一一列出了
```

### 设置填充模式
通过 paint.setStyle() 可以设置填充模式
```java
public void setStyle(Style style)
```

Paint.Style 是个枚举
```java
public enum Style {
    // 填充
    FILL            (0),
    // 描边
    STROKE          (1),
    // 填充加描边
    FILL_AND_STROKE (2);
    Style(int nativeInt) {
        this.nativeInt = nativeInt;
    }
    final int nativeInt;
}
```

以上三种取值的区别如下:
<img src="/images/paint_style.jpg" width = "500"  />

### 设置线条样式
paint提供了一些设置线条样式的方法:
```java
// 设置线条宽度, 单位px, 默认0, 绘制结果为1px, 且不受 canvas的几何变换影响
public void setStrokeWidth(float width)
// 设置线帽的形状
public void setStrokeCap(Cap cap)
// 设置线条设置拐角的形状 
public void setStrokeJoin(Join join)
// 当拐角为尖角时, 如果 (线条内外交点距离 / 线宽 > miter), 则自动切为平角
public void setStrokeMiter(float miter)
```

Paint.Cap 指线帽的形状;  Paint.Join 指线段拐角处的形状
```java
public enum Cap {
    // 平头, 默认值
    BUTT    (0),
    // 圆头
    ROUND   (1),
    // 方形头
    SQUARE  (2);
    private Cap(int nativeInt) {
        this.nativeInt = nativeInt;
    }
    final int nativeInt;
}
...
public enum Join {
    // 尖角, 默认值
    MITER   (0),
    // 圆角
    ROUND   (1),
    // 平角
    BEVEL   (2);
    private Join(int nativeInt) {
        this.nativeInt = nativeInt;
    }
    final int nativeInt;
}
```

看看效果
<img src="/images/paint_stroke_style.jpg" width = "500"  />

### 设置路径效果
通过 paint.setPathEffect() 可以设置路径的显示效果.
```java
public PathEffect setPathEffect(PathEffect effect)
```

PathEffect本身没有什么效果, 一般使用它的子类
```java
/**
 * 将路径的转角变得圆滑
 * @param radius  圆角半径
 */
public CornerPathEffect(float radius)
/**
 * 离散路径效果: 将原路径分割为指定长度的线段, 每条线段都随机偏移一段距离
 * @param segmentLength  分割后线段的长度
 * @param deviation      线段的偏移距离
 */
public DiscretePathEffect(float segmentLength, float deviation)
/**
 * 虚线效果 
 * @param intervals  用于描述虚线的特征: 虚线长度、缝隙宽度、虚线长度、缝隙宽度、...  (数组长度最少为2,且必须为偶数)
 * @param phase      沿着路径方向, 偏移多少才开始绘制
 */
public DashPathEffect(float intervals[], float phase)
/**
 * 和虚线效果类似, 不过它的每一段组成都是 自定义的形状 shape
 * @param advance    两个 shape 之间的间隔距离
 * @param phase      沿着路径方向, 偏移多少才开始绘制
 * @param style      转角处的样式: Style.TRANSLATE(平移转角处的 shape);  Style.ROTATE(旋转转角处的 shape);  Style.MORPH(转角处的 shape 拉伸变形)
 */
public PathDashPathEffect(Path shape, float advance, float phase, Style style)
// 分别对原始路径使用 first 和 second 效果, 然后将两条路径合并输出
public SumPathEffect(PathEffect first, PathEffect second)
// 先对原始路径使用 innerpe 效果, 接着在此基础上又使用 outerpe 效果, 最后输出结果
public ComposePathEffect(PathEffect outerpe, PathEffect innerpe)
```

上图片 (图片来自网络, 第一条为原始路径)
<img src="https://images0.cnblogs.com/blog/651487/201502/222133271897419.jpg" width = "500"  />	

### 获取path (没验证过, 内容截取至网络)
获取实际的path: 指的就是 drawPath() 的绘制内容的轮廓，要算上线条宽度和设置的 PathEffect
```java
// src指原来的path, dst用于保存实际的path
public boolean getFillPath(Path src, Path dst)
```
<img src="https://img-blog.csdn.net/20171215100317180?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzA4ODkzNzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width = "500"  />

获取文本的path
```java
public void getTextPath(String text, int start, int end, float x, float y, Path path)
```
<img src="https://img-blog.csdn.net/20171215102239885?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzA4ODkzNzM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width = "500"  />

### 设置阴影
通过 paint.setShadowLayer() 可以设置阴影.
```java
/**
 * 设置阴影 
 * @param radius        阴影模糊半径
 * @param dx            阴影的水平偏移
 * @param dy            阴影的垂直偏移
 * @param shadowColor   阴影的颜色, shadowColor是半透明时, 阴影透明度为 shadowColor自己的透明度, 否则 阴影透明度为 paint的透明度
 */
public void setShadowLayer(float radius, float dx, float dy, int shadowColor)
// 清除阴影效果
public void clearShadowLayer()
```
<img src="/images/shadow_layer1.jpg" width = "500"  />	

发现除了文字外, 圆角矩形还有图片的阴影都没有生效? 这是因为只有文字阴影支持硬件加速, 所以需要关闭硬件加速才能看到其它的阴影效果.
<img src="/images/shadow_layer2.jpg" width = "500"  />

阴影都出现了, 不过图片的阴影似乎有点不对? 这是因为系统在处理图片的 shadowLayer时, 是将图片复制一份, 然后再进行边缘模糊, 所以看上去和预想的有点不一样.

### 设置遮罩滤镜 (不支持硬件加速)
通过 paint.setMaskFilter() 可以对整个画面进行过滤
```java
// 注意和 setColorFilter 的区别. setColorFilter 是对每个像素进行过滤, 这里是过滤整个画面
public MaskFilter setMaskFilter(MaskFilter maskfilter)
```

MaskFilter 的两个子类
```java
/**
 * 模糊效果
 * @param radius       模糊半径 (基于高斯模糊)
 * @param style        模糊类型, 是个枚举: NORMAL(内外都模糊)、 SOLID(内部正常绘制,外部模糊)、 INNER(内部模糊,外部不绘制)、 OUTER(外部模糊,内部不绘制)
 */
public BlurMaskFilter(float radius, Blur style) 
/**
 * 浮雕效果  (过时了, 不再演示)
 * @param direction      包含3个元素的数组,指光的方向
 * @param ambient        指环境光的强度 范围是 0 ~ 1
 * @param specular       指炫光的系数
 * @param blurRadius     指应用光线的范围
 */
public EmbossMaskFilter(float[] direction, float ambient, float specular, float blurRadius)
```

下面是 BlurMaskFilter 的四种特效
<img src="/images/mask_filter.jpg" width = "500"  />			

### 色彩处理
在绘制内容时, 对色彩的处理大概可以分为三个步骤: 基本颜色 -> 色彩过滤 -> 颜色混合(合成)

#### 色彩处理第一步: 基本颜色
- 颜色的来源
    - 颜色来自直接绘制
    ```java
    canvas.drawColor();
    canvas.drawRGB()
    canvas.drawARGB()
    ```
    - 颜色来自图片
    ```java
    // 一般情况下, 绘制图片时, paint上设置的颜色对 bitmap 没有影响
    // 但如果 bitmap 中只有 alpha值, 而没有 r, g, b , 那么该图片的颜色将由 paint 决定
    // ps: bitmap.extractAlpha() 就可以以原 bitmap 为基础, 创建一个新的只包含 alpha值 的空白图片
    canvas.drawBitmap();
    ```
    - 颜色来自 paint
    ```java
    // 直接给 paint 设置颜色
    public void setColor(@ColorInt int color)
    public void setARGB(int a, int r, int g, int b)
    // 通过 Shader(着色器) 设置颜色.  -- 设置 Shader 后, setColor/setARGB 都将不再起作用
    public Shader setShader(Shader shader)
    ```
- Shader 的子类
```java
/**
 * 线性渐变: 其坐标是相对于当前 View的, 坐标更多的作用是设置线性渐变的角度, '线' 本身是无限宽的
 * @param (x0, y0), (x1, y1)        线性渐变的起点坐标 和 终点坐标
 * @param color0, color1            线性渐变的起点颜色 和 终点颜色    
 * @param tile   绘制区域超出线性渐变所指定的范围时, 所采用的重复策略: CLAMP(用边缘色彩填充多余空间);  MIRROR(镜像重复);  REPEAT(平铺重复)
*/
public LinearGradient(float x0, float y0, float x1, float y1, @ColorInt int color0, @ColorInt int color1,@NonNull TileMode tile)
/**
 * @param colors      渐变颜色的数组
 * @param positions   与 colors数组对应, 指定每个颜色在整个渐变中所占的百分比.  取值范围 0 ~ 1
*/
public LinearGradient(float x0, float y0, float x1, float y1, @NonNull @ColorInt int colors[], @Nullable float positions[], @NonNull TileMode tile)
/**
 * 辐射渐变
 * @param centerX, centerY          渐变的中心点
 * @param radius                    渐变的半径
 * @param centerColor, edgeColor    渐变的中心颜色 和 边缘颜色
 */
public RadialGradient(float centerX, float centerY, float radius, @ColorInt int centerColor, @ColorInt int edgeColor, @NonNull TileMode tileMode) 
public RadialGradient(float centerX, float centerY, float radius, @NonNull @ColorInt int colors[], @Nullable float stops[], @NonNull TileMode tileMode)
/**
 * 扫描式渐变
 * @param cx, cy           扫描中心点坐标
 * @param color0, color1   起始颜色 和 终点颜色
 */
public SweepGradient(float cx, float cy, @ColorInt int color0, @ColorInt int color1)
public SweepGradient(float cx, float cy, @NonNull @ColorInt int colors[], @Nullable float positions[])
/**
 * 图片着色器:    BitmapShader 会从当前控件的左上角开始摆放图片, 如果图片尺寸不够, 则按照对应方向上的 TileMode 处理空白区域
 * @param bitmap           要在着色器中使用的位图. 使用BitmapShader后, paint就可以用bitmap去绘制指定的图形 
 * @param tileX, tileY     水平 和 垂直方向上的重复模式
 */
public BitmapShader(@NonNull Bitmap bitmap, @NonNull TileMode tileX, @NonNull TileMode tileY)
/**
 * 混合着色器.        ComposeShader 会把 shaderA 和 shaderB 按照 mode 进行混合
 * @param shaderA    ComposeShader 会把它当做   目标图 -- "dst"
 * @param shaderB    ComposeShader 会把它当做   源图 -- "src" 
 * @param mode       一般使用其子类 PorterDuffXfermode 
 */
public ComposeShader(@NonNull Shader shaderA, @NonNull Shader shaderB, @NonNull Xfermode mode)
public ComposeShader(@NonNull Shader shaderA, @NonNull Shader shaderB, @NonNull PorterDuff.Mode mode)
``` 
- PorterDuffXfermode & PorterDuff.Mode
```java
public PorterDuffXfermode(PorterDuff.Mode mode)
public enum PorterDuff.Mode {
    CLEAR, SRC, DST, SRC_OVER, DST_OVER, SRC_IN, DST_IN, 
    SRC_OUT, DST_OUT, SRC_ATOP, DST_ATOP, XOR, DARKEN, 
    LIGHTEN, MULTIPLY, SCREEN, ADD, OVERLAY
}
```
这是 Google 的演示效果图 (注意图片周围的透明范围)
<img src="/images/porter_duff.webp" width = "500"  />
这是实践后的效果图 (去掉了透明区域)
<img src="/images/porter_duff.jpg" width = "500"  />
PorterDuffXfermode的逻辑是这样的: 先绘制 DST中与SRC没有相交的部分, 而对于两者相交的区域, 会先清除该部分的图片, 然后将采用PorterDuff.Mode计算后的结果绘制上去.

- 给 Shader 设置矩阵变换
```java
// Matrix类中包含一个 3x3 的矩阵, 可用于转换坐标
// 给 Shader 设置 Matrix 后, 通过不断转换 Matrix 坐标, 可以实现一些动画效果
public void setLocalMatrix(@Nullable Matrix localM)
```
- SweepGradient + matrix.setRotate() 实现雷达效果
<img src="/images/sweep_radient.gif" width = "500"  />
- LinearGradient + matrix.setTranslate() 实现文字发光效果 
<img src="/images/linear_radient.gif" width = "500"  />	
- BitmapShader + canvas.drawCircle() 实现圆形图片		
<img src="/images/bitmap_radient.gif" width = "500"  />	

#### 色彩处理第二步: 过滤
通过 paint.setColorFilter() 可以对基础颜色进行过滤
```java
// 使用 ColorFilter 对paint当前的颜色进行过滤
public ColorFilter setColorFilter(ColorFilter filter) 
```

看看 ColorFilter 的几个子类
```java
/**
 * LightingColorFilter 会将目标颜色的 RGB通道乘以一种颜色(mul), 然后再加上另一种颜色(add).   公式如下:
 *  新R = R * mul.R / 0xff + add.R
 *  新G = G * mul.G / 0xff + add.G
 *  新B = B * mul.B / 0xff + add.B
 */
public LightingColorFilter(@ColorInt int mul, @ColorInt int add)
// 以paint当前颜色为 dst, 以这里的 color 为 scr, 按照 mode 进行混合
public PorterDuffColorFilter(@ColorInt int color, @NonNull PorterDuff.Mode mode)
/**
 * ColorMatrixColorFilter 使用一个颜色矩阵(ColorMatrix)进行颜色过滤. 
 * ColorMatrix 通过一个长度为20的数组构建, 该数组表示一个 4 * 5 的颜色矩阵.
 */
public ColorMatrixColorFilter(@NonNull ColorMatrix matrix)
public ColorMatrixColorFilter(@NonNull float[] array)

```

ColorMatrixColorFilter 内部依赖一个 ColorMatrix 来完成颜色过滤. ColorMatrix的构造方法如下:
```java
public ColorMatrix(float[] src)
public ColorMatrix()
```

ColorMatrix 使用一个 4 * 5 的矩阵进行颜色换算, 公式如下:
```java
// 假设是这样一个矩阵:
[ a, b, c, d, e,
  f, g, h, i, j,
  k, l, m, n, o,
  p, q, r, s, t ]
// 那么颜色的计算公式如下 
新R = a*R + b*G + c*B + d*A + e;
新G = f*R + g*G + h*B + i*A + j;
新B = k*R + l*G + m*B + n*A + o;
新A = p*R + q*G + r*B + s*A + t;
```

直接使用 ColorMatrixColorFilter(@NonNull float[] array) 或者 ColorMatrix(float[] src) 肯定是比较困难的. 因此通常可以直接使用 ColorMatrix() 初始化 ColorMatrix 实例, 其默认矩阵如下:
```java
[ 1, 0, 0, 0, 0,
  0, 1, 0, 0, 0,
  0, 0, 1, 0, 0,
  0, 0, 0, 1, 0 ]
```

套用上面的计算公式, 可以知道默认的矩阵不会影响原来的颜色. ColorMatrix 中封装了一些矩阵变换的快捷方法, 可以帮助我们进行颜色过滤.
```java
// 将矩阵还原到默认状态 
public void reset()
// 色彩缩放.  四个参数分别表示对应颜色通道的缩放倍数
public void setScale(float rScale, float gScale, float bScale, float aScale) 
/**
 * 围绕某个颜色轴进行旋转
 * @param axis      0 绕R轴旋转;  1 绕G轴旋转;  2 绕B轴旋转
 * @param degrees   旋转的角度
 */
public void setRotate(int axis, float degrees)
/**
 * 相关矩阵相乘:   matA * matB
 */
public void setConcat(ColorMatrix matA, ColorMatrix matB)
// 等价于 setConcat(this, prematrix)
public void preConcat(ColorMatrix prematrix)
// 等价于 setConcat(postmatrix, this)
public void postConcat(ColorMatrix postmatrix)
// 设置整体的饱和度:   0 无色彩;  1 原色彩;  > 1 饱和度增加
public void setSaturation(float sat)
// Set the matrix to convert RGB to YUV
public void setRGB2YUV()
// Set the matrix to convert from YUV to RGB
public void setYUV2RGB()
```
看看 ColorMatrixColorFilter 的作用
<img src="/images/color_matrix.gif" width = "500"  />

#### 色彩处理第三步: 颜色混合
相关方法
```java
// 以当前图层中已有的内容为 目标图片(dst), 以下一步要绘制的内容为 源图片(src), 按照 xfermode 进行混合
public Xfermode setXfermode(Xfermode xfermode)
```

关于 Xfermode 可以回去看看上面的图片. 先来个简单的案例	
```java
private Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
private Xfermode xfermode = new PorterDuffXfermode(PorterDuff.Mode.SRC_IN);
private String text = "CHOICE";
private Paint.FontMetrics metrics = new Paint.FontMetrics();
private float textWidth;
private float textHeight;
{
    paint.setTextSize(200);
    paint.setStyle(Paint.Style.FILL_AND_STROKE);
    paint.setTextAlign(Paint.Align.CENTER);
    paint.getFontMetrics(metrics);
    textWidth = paint.measureText(text);
    textHeight = metrics.bottom - metrics.top;
}
@Override
protected void onDraw(Canvas canvas) {
    float halfHeight = getHeight() * 1f / 2;
    // 计算文字原点, 让其居中显示
    float x = getWidth() * 1f / 2;
    float y = halfHeight + textHeight / 2 - metrics.bottom;
    int layer = canvas.saveLayer(null, null, Canvas.ALL_SAVE_FLAG);
    paint.setColor(0xfff25555);
    canvas.drawText(text, x, y, paint);
    paint.setXfermode(xfermode);
    paint.setColor(0xff18a2ff);
    canvas.drawRect(x - textWidth / 2, halfHeight, x + textWidth / 2, y + metrics.bottom, paint);
    paint.setXfermode(null);
    canvas.restoreToCount(layer);
}
```
<img src="/images/xfer_mode_text.jpg" width = "500"  />

再看一个案例
```java
public class RoundImageView extends AppCompatImageView {
    ...
    // SCR_IN
    private Xfermode xfermode = new PorterDuffXfermode(PorterDuff.Mode.SRC_IN);
    private Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
    {
        paint.setStyle(Paint.Style.FILL_AND_STROKE);
        paint.setStrokeWidth(0F);
    }
    @Override
    protected void onDraw(Canvas canvas) {
        int saveLayer = canvas.saveLayer(null, null, Canvas.ALL_SAVE_FLAG);  // 新建一个空白图层
        float x = getWidth() * 1f/ 2;
        float y = getHeight() * 1f/ 2;
        canvas.drawCircle(x, y, Math.min(x, y), paint);        // 画一个圆形, 作为 DST
        paint.setXfermode(xfermode);                           // 以这里为分界. 之前的步骤是绘制 DST; 以下则是绘制 SRC
        canvas.saveLayer(null, paint, Canvas.ALL_SAVE_FLAG);   // 注意一定要传入 paint, 才会让后面的绘制受 Xfermode 影响
        super.onDraw(canvas);                                  // 绘制 SRC
        paint.setXfermode(null);                               // 清除 Xfermode
        canvas.restoreToCount(saveLayer);                      
    }
}
```
<img src="/images/xfer_mode_cirlce.jpg" width = "500"  />

### 硬件加速
1. android中, 硬件加速是指将绘制的计算工作交给GPU来处理. 
    - 未开启硬件加速时: 调用 canvas.drawXX()时, 由CPU将绘制的内容, 转换为具体的像素信息, 保存在 bitmap 中, 最后在渲染到屏幕上.
    - 开启硬件加速时: 调用 canvas.drawXX()时, 由CPU将绘制的内容, 转换为GPU操作保存下来, 最后由GPU来完成渲染.
2. 硬件加速的优点 
    - 未开启硬件加速时: 绘制的内容直接被CPU转换为像素信息保存到bitmap中. 因此当某个View重绘时, 为了正确的计算出bimap中的像素, 会导致整个页面的重绘.
    - 开启硬件加速时: 绘制的内容被转换为GPU操作, 最后由GPU转换为具体的像素. 因此当某个View重绘时, 只会更新其对应的那部分GPU操作, 而不会引起整个页面的重绘. 
3. 硬件加速的限制: 受GPU绘制方式的限制, 某些API会在开启硬件加速的时候失效, 此时需要我们主动关闭硬件加速. 下面是一些API支持硬件加速的情况:
<img src="/images/hardware_accelerated.webp" width = "500"  />	
4. 关闭硬件加速的方式
    - 针对整个应用程序
    ```java
    // AndroidManifest.xml文件为application标签
    <application android:hardwareAccelerated="true" ...> 
    ```
    - 针对当前页面
    ```java
    <activity android:hardwareAccelerated="false" /> 
    ```
    - 针对具体的View
    ```java
    // 方式一: 在布局属性中设置
    <MyView xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layerType="software"
        ... >
    // 方式二: 代码设置
    view.setLayerType(View.LAYER_TYPE_SOFTWARE, null);  
    ```
5. **setLayerType** 真正的作用是设置 "离屏缓冲", 只是当参数为 View.LAYER_TYPE_SOFTWARE 时, 顺便又可以关闭硬件加速. "离屏缓冲" 是指用一个单独的地方, 将最终要渲染到屏幕上的像素信息缓存下来, 然后再由它渲染到屏幕上. "离屏缓冲" 的优点是可以缓存像素信息, 针对某些像素不发生改变的刷新场景(比如动画平移、旋转)可以利用缓存提升效率; 缺点是在需要重绘的场景下, 会因为增加额外的工作而影响效率. 因此一定要慎用 setLayerType(), 且在开启 "离屏缓存后", 在必要时一定要及时关闭.

setLayerType() 可接受以下参数:
```java
// 使用一个 bitmap 来缓冲, 同时会关闭硬件加速
public static final int LAYER_TYPE_SOFTWARE = 1;
// 在硬件加速开启的情况下, 会使用 GPU 来缓冲; 在已经关闭硬件加速的情况下, 并不会开启硬件加速, 而且此时该参数的作用和 LAYER_TYPE_SOFTWARE 一样
public static final int LAYER_TYPE_HARDWARE = 2;
// 不使用缓冲
public static final int LAYER_TYPE_NONE = 0;
```
