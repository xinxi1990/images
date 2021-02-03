# 计算 Android FPS

> FPS（Frame Per Second）是图像领域中的定义，是指画面每秒传输帧数，通俗来讲就是指动画或视频的画面数。

一般的屏幕刷新包括三个步骤：**CPU 计算屏幕数据、GPU 进一步处理和缓存、最后 display 再将缓存中（buffer）的屏幕数据显示出来**。

## Android View 的绘制

每一帧画面的正常绘制过程大概如下图所示：

![Display](https://cdn.nlark.com/yuque/0/2020/png/125389/1585300937639-e2d06f36-2b7b-4a65-8654-dcf4e334493b.png)

Display 这一行可以理解成屏幕，所以可以看到，底层是以固定的频率发出 VSync 信号的，而这个固定频率就是我们常说的每 16.6ms 发送一个 VSync 信号。

16ms = 1000ms/60，意思就是 1 秒 60 帧。

每一个 View 的绘制过程都必须经历三个最主要的过程，也就是 measure（计算）、layout（布局）和 draw（绘制）。所以上面提到的 16ms，也是这三个步骤所用的耗时。

CPU 这块的耗时其实就是我们 app 绘制当前 View 树的时间，如果你的布局很复杂，层次嵌套很多，每一帧内需要刷新的 View 又很多时，那么每一帧的绘制耗时自然就会多一点，然后，可能每一秒内没有绘制 60 帧，而掉帧越多，用户看到的画面就越是不连贯。也就是越卡顿。

### Jank

如果发生掉帧后，可能就是下图的样子：

![Jank](https://cdn.nlark.com/yuque/0/2020/png/125389/1585300975874-6f441276-3f45-4672-a8a0-8f619722c9dc.png)

当 VSync 信号来时，因为缓冲 B 的数据还没准备好，屏幕只能继续显示之前缓冲 A 的那一帧，此时缓冲 A 的数据也不能被清空和交换数据。这种情况被Android开发组命名为“Jank”，就是所谓的“丢帧”；

当需要显示的数据准备完成后，它并不会马上被显示，而是要等待下一个 VSync，屏幕下次刷新后，用户才看到画面的更新。

## 如何计算 FPS

根据上面的描述，如果可以拿到每一秒的掉帧数据，然后就可以知道每一秒实际绘制了多少帧，这样就可以知道应用整体的 FPS 情况，也基本能表示应用整体的流畅度如何。那如何取得掉帧情况呢？

1. 首先需要了解一下 Choreographer 这个类，简单理解，这个类里面会有一个 looper，来处理 VSYNC 信号，然后执行 doFrame 方法，来刷新 UI。

2. 其实丢帧的数据，系统默认是一直有日志输出的，可以使用命令`adb shell logcat -v time -d Choreographer:I *:S`，但是因为SKIPPED_FRAME_WARNING_LIMIT 值默认设置为 30。导致UI线程 doFrame 时，只要丢帧不高于 30 帧，就不会通过 log 输出警告。

3. 所以获得我们想要的数据，只需要新建一个类，通过反射的方式来修改默认值。

代码如下：

```Java
public class FPSFrameCallback implements Choreographer.FrameCallback {
  private static final int SKIPPED_FRAME_WARNING_LIMIT = 1;
  public void setWarningLimit(){
        if (Integer.parseInt(android.os.Build.VERSION.SDK)<16){
            return;
        }
        try {
            Class<?> onwChoreographer =Class.forName("android.view.Choreographer");
            Field mSkipped = onwChoreographer.getDeclaredField("SKIPPED_FRAME_WARNING_LIMIT");
            mSkipped.setAccessible(true);
            mSkipped.set(null,SKIPPED_FRAME_WARNING_LIMIT);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

之后需要再 Mainactivity 的 onCreate 方法中，加一行代码

```Java
FPSFrameCallback.setWarningLimit();
```

可以看到输出的 log 如下：

![](https://cdn.nlark.com/yuque/0/2020/png/125389/1585300902038-6acfb53d-b36b-41c6-a2a4-c33808c58c5f.png#align=left&display=inline&height=684&margin=%5Bobject%20Object%5D&originHeight=684&originWidth=1530&size=0&status=done&style=none&width=1530)

由上图可以看出，会有几种情况需要处理，解决思路如下：

1. 将属于同一秒的内的掉帧数据相加，得到每秒内的掉帧总数。

2. 会有未掉帧的秒数，即掉帧为 0，需要对未掉帧的秒数，进行补充数据。

3. 某一秒掉帧总数大于 60，这种情况比较特殊，目前采取的校正方式是掉帧总数除以 60，取余。用 60 减去某一秒的掉帧数据，得出的值就是这一秒内显示的总帧数。用这个值来体现流畅程度。

以上的步骤，都是准备工作，下面是涉及流畅度计算部分的代码：

开启线程

```Java
public void callableForSM() {
  fpsCS.submit(new Callable<ArrayList<Integer>>() {
  
  ArrayList<Integer> jankArrayList = new ArrayList<>();
  
  public ArrayList<Integer> call() throws Exception {
      int interval = 5; //取值间隔
      long last_time = System.currentTimeMillis();
      String jankString; //存储logcat输出的内容
      String janknum = null; //存储掉帧数
      String lastTime; //上一行时间
      String currentTime = null; //当前行时间
      String pid = exec.exec("adb shell ps | grep com.dedao | awk \\'{print $2}\\' | sed -n \\'2p\\'").toString();
```

循环取初始值

```Java
while (startSM) {
  long current_time = System.currentTimeMillis();
  if (current_time - last_time > interval * 1000) { //循环取值间隔
     last_time = current_time;
     jankString = exec.exec("adb shell logcat -v time -d  Choreographer:I *:S|grep \\'Choreographer\\'|awk \\'{print $2,$"+p+"}\\'").toString();
     if (jankString.length() > 0) {
         logger.debug("fps start get value");
         lastTime = jankString.substring(0, 8);
     } else {
         logger.debug("!!! jankString.length() > 0");
         continue;
     }
```

根据掉帧数分别处理

```Java
Integer sum = 0;
while (jankString.length() > 14) {
   currentTime = jankString.substring(0, 8);
   if (jankString.charAt(14) == '\\n') {//掉帧为个位数
       janknum = jankString.substring(13, 14);
       logger.debug("fps frame jank ========= " + janknum);
       if (jankString.length() == 15) {
           jankString = " ";
       } else {
           jankString = jankString.substring(15);
       }
   } else if (jankString.charAt(15) == '\\n') {//掉帧为十位数
       janknum = jankString.substring(13, 15);
       logger.debug("fps frame jank ========= " + janknum);
       if (jankString.length() == 16) {
           jankString = " ";
       } else {
           jankString = jankString.substring(16);
       }
   }
```

计算相同秒数内的掉帧总数

```Java
if (lastTime.equals(currentTime)) {
       sum = sum + Integer.parseInt(janknum);//同一秒内掉帧之和
   } else {
       if (sum < 60) {
           jankArrayList.add(60 - sum);//当丢帧<60时，流畅度SM =60-丢帧数
       } else {
           int num = sum/60;
           jankArrayList.add(60 - sum % 60);//当丢帧frame>60时，流畅度SM = 60-丢帧数%60
           for (int i = 0; i < num; i++) {
               jankArrayList.add(0);
           }
       }
       sum = Integer.parseInt(janknum);
       lastTime = currentTime;
   }
}
```

处理同一秒内掉帧总数大于60的情况

```Java
if (sum != 0) {
   if (sum < 60)
       jankArrayList.add(60 - sum);
   else {
       int num = sum/60;
       jankArrayList.add(60 - sum % 60);
       for (int i = 0; i < num; i++) {
           jankArrayList.add(0);
       }
```

填充没有掉帧的秒数的数据

```Java
int SM_60_Seconds = interval - jankArrayList.size();
   logger.debug("不掉帧的秒数为：" + SM_60_Seconds);
   for (int i = 0; i < SM_60_Seconds; i++) {
       jankArrayList.add(60);
   }
```

拿到处理好的数据之后，分别计算每一秒的 FPS 值，再将所有的 FPS 值， 根据区间（例如 5050）计算出各自百分比，在报告中展示。

Choreographer.FrameCallback 这个接口，还可以做更多的事情，例如：我们可以重写 doFrame 函数，来输出跳帧所耗费的时间：

```Java
@Override
public void doFrame(long frameTimeNanos) {
  if (mLastFrameTimeNanos == 0) {
        mLastFrameTimeNanos = frameTimeNanos;
    }
    // 开始执行doFrame的时间
    long startNanos = System.nanoTime();
    // 接收VSYNC任务和实际开始执行的时间差
    final long jitterNanos = startNanos - frameTimeNanos;
    // 时间差大于16ms
    if (jitterNanos > mFrameIntervalNanos) {
        Log.d(TAG, "The jitterNanos is " + jitterNanos * 0.000001f + "ms");
    }
    mLastFrameTimeNanos = frameTimeNanos;
    mLastStartNanos = startNanos;
    // 注册下一帧的回调
    Choreographer.getInstance().postFrameCallback(this);
}
```

### 结果展示

![](https://cdn.nlark.com/yuque/0/2020/png/125389/1585300902081-b3ff244a-273b-4d07-94dd-3663fb5fe898.png#align=left&display=inline&height=540&margin=%5Bobject%20Object%5D&originHeight=540&originWidth=1060&size=0&status=done&style=none&width=1060)