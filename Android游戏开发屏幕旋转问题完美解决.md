#Android游戏开发屏幕旋转问题完美解决

做游戏Android上的游戏开发也有一年多了，虽然没有什么高深的见解，但就遇到的问题及解决方案做下记录

我们的游戏，横屏的棋牌游戏，一直是锁定横屏的一个方向运行，有一天，项目经理找到我，想让游戏像一些IPhone游戏一样，在旋转屏幕的时候按照重力进行旋转，在锁定方向的时候又能固定一个方向，也就是能按照用户选择在横屏方向进行锁定和旋转

当然，我信心满满的答应了，感觉对于一个以Android起家的我来说这并不算什么，五分钟搞定

结果一个小时过去了，并没有找到完美的解决方案，在Android的manifest.xml做配置是不可行的

经过我一番百度和看Android文档，我才发现，Android对旋转屏，特别是只有横屏或者竖屏虽重力旋转的支持是到Android4.3.1才有完美支持的

```
unspecified - 默认值，由系统选择显示方向
landscape   - 橫向
portrait    - 纵向
reverseLandscape    - 反横向(API >= 9)
reversePortrait     - 反纵向(API >= 9)
user        - 用户当前的首选方向
behind      - 与Activity堆栈下的方向相同
sensor      - 根据物理传感器方向3/4个方向(取决于设备)
fullSensor  - 根据物理传感器方向4个方向
nosensor    - 不按照物理传感器方向，除此之外与"unspecified"无区别
sensorLandscape     - 按照物理传感器，只在横向(2个方向)进行翻转(API >= 9)
sensorPortrait      - 按照物理传感器，只在纵向(2个方向)进行翻转(API >= 9)
userLandscape       - 按照用户选择，锁定一个横向，或者按照物理传感器进行横向的翻转(API >= 18)
userPortrait        - 按照用户选择，锁定一个纵向，或者按照物理传感器进行纵向的翻转(API >= 18)
fullUser    - 如果用户锁定了屏幕，它与"user"作用一致，如果是解锁了旋转，它与"fullSensor"作用一致(API >= 18)
locked      - 锁定了屏幕当前方向(API >= 18)
```

但由于游戏是要适配各个系统版本的，只在AndroidManifest.xml里配置显然只能满足部分系统需求，于是我写了下面的代码

```java
public class TestOrientationActivity extends Activity{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        super.onCreate(savedInstanceState);
        resetScreenOrientation();
        Uri rotationUri = Settings.System.getUriFor(Settings.System.ACCELEROMETER_ROTATION);
        ContentResolver resolver = getApplication().getContentResolver();
        // 此处注册监听旋转设置变化
        resolver.registerContentObserver(rotationUri, true, mContentConfigObserver);
    }

    /**
     * 用于监听旋转变化
     */
    private ContentObserver mContentConfigObserver = new ContentObserver(new Handler()) {
        @Override
        public void onChange(boolean selfChange) {
            resetScreenOrientation();
        }
    };

    @Override
    protected void onRestart() {
        // TODO Auto-generated method stub
        super.onRestart();
        resetScreenOrientation();
    }

    private void resetScreenOrientation() {
        // TODO Auto-generated method stub
        int orientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE;
        int sdkInt = android.os.Build.VERSION.SDK_INT;  
        if (sdkInt >= Build.VERSION_CODES.JELLY_BEAN_MR2 /*18*/) {
            //大于JELLY_BEAN_MR2(4.3.1)版本的直接支持按照用户选择进行屏幕旋转
            //Field requires API level 18 (current min is 8): 
            orientation = ActivityInfo.SCREEN_ORIENTATION_USER_LANDSCAPE;
        } else if (sdkInt >= Build.VERSION_CODES.GINGERBREAD) {
            int flag = Settings.System.getInt(getContentResolver(),Settings.System.ACCELEROMETER_ROTATION, 0);
            if (0 == flag) {
                // 屏幕旋转已经关闭，那么固定屏幕在某一个方向
                orientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE;
            } else {
                // 屏幕旋转打开，屏幕按照sensor的参数进行旋转
                // 此参数只在GINGERBREAD(2.3.3)以上的系统支持
                //Field requires API level 18 (current min is 8): 
                orientation = ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE;
            }
        } else {
            // 小于GINGERBREAD(2.3.3)版本的系统不支持屏幕旋转
            orientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE;
        }
        int curOrientation = getRequestedOrientation();
        if (orientation != curOrientation) {
            setRequestedOrientation(orientation);
        }
    }
}
```

除了在2.3.3-4.3.1的部分手机监听设置改变时候会出现延迟以外，别的都是完美解决