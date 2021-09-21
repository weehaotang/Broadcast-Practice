# Broadcast-Practice
**广播的练习：聊天软件的强制下线:**

**思路**

1. 对于聊天软件登录界面，只实现了简单的功能：

首先将LoginActivity的继承结构改成继承自BaseActivity，然后在登录按钮的点击事件里对输入的账号和密码进行判断：如果账号是admin并且密码是123456，就认为登录成功并跳转到MainActivity，否则就提示用户账号或密码错误。
![截屏2021-09-21 00.59.26](/Users/weehaotang/Library/Application Support/typora-user-images/截屏2021-09-21 00.59.26.png)

![截屏2021-09-21 00.59.44](/Users/weehaotang/Library/Application Support/typora-user-images/截屏2021-09-21 00.59.44.png)

2. 创建一个按钮用于触发强制下线功能。

在按钮的点击事件里发送了一条广播，广播的值为`com.example.broadcastbestpractice.FORCE_OFFLINE`，这条广播就是用于通知程序强制用户下线的。也就是说，强制用户下线的逻辑并不是写在MainActivity里的，而是应该写在接收这条广播的BroadcastReceiver里。这样强制下线的功能就不会依附于任何界面了，不管是在程序的任何地方，只要发出这样一条广播，就可以完成强制下线的操作了。

![截屏2021-09-21 01.00.13](/Users/weehaotang/Library/Application Support/typora-user-images/截屏2021-09-21 01.00.13.png)

**问题：**

1.弹出的下线通知，点击Back按键可以继续使用程序

**解决方案**：使用AlertDialog.Builder构建下线对话框时调用setCancelable()

![截屏2021-09-21 00.45.51](/Users/weehaotang/Library/Application Support/typora-user-images/截屏2021-09-21 00.45.51.png)



强制下线功能需要先关闭所有的Activity，然后回到登录界面。先创建一个`ActivityCollector`类用于管理所有的Activity：

```kotlin
object ActivityCollector {

    private val activities = ArrayList<Activity>()

    fun addActivity(activity: Activity) {
        activities.add(activity)
    }

    fun removeActivity(activity: Activity) {
        activities.remove(activity)
    }

    fun finishAll() {
        for (activity in activities) {
            if (!activity.isFinishing) {
                activity.finish()
            }
        }
        activities.clear()
    }

}
```

创建`BaseActivity`类作为所有Activity的父类:

```kotlin
open class BaseActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ActivityCollector.addActivity(this)
    }

    override fun onDestroy() {
        super.onDestroy()
        ActivityCollector.removeActivity(this)
    }

}
```

创建一个`LoginActivity`来作为登录界面，编辑布局文件activity_login.xml

```xml
<!--使用LinearLayout编写了一个登录布局，最外层是一个纵向的LinearLayout，里面包含了3行直接子元素。第一行是一个横向的LinearLayout，用于输入账号信息；第二行也是一个横向的LinearLayout，用于输入密码信息；第三行是一个登录按钮。-->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="60dp">
        <TextView
            android:layout_width="90dp"
            android:layout_height="wrap_content"
            android:layout_gravity="center_vertical"
            android:textSize="18sp"
            android:text="Account:" />

        <EditText
            android:id="@+id/accountEdit"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:layout_gravity="center_vertical" />
    </LinearLayout>

    <LinearLayout
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="60dp">
        <TextView
            android:layout_width="90dp"
            android:layout_height="wrap_content"
            android:layout_gravity="center_vertical"
            android:textSize="18sp"
            android:text="Password:" />

        <EditText
            android:id="@+id/passwordEdit"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:layout_gravity="center_vertical"
            android:inputType="textPassword" />
    </LinearLayout>

    <Button
        android:id="@+id/login"
        android:layout_width="200dp"
        android:layout_height="60dp"
        android:layout_gravity="center_horizontal"
        android:text="Login" />

</LinearLayout>
```

修改LoginActivity中的代码:

```kotlin
class LoginActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
        login.setOnClickListener {
            val account = accountEdit.text.toString()
            val password = passwordEdit.text.toString()
            // 如果账号是admin且密码是123456，就认为登录成功
            if (account == "admin" && password == "123456") {
                val intent = Intent(this, MainActivity::class.java)
                startActivity(intent)
                finish()
            } else {
                Toast.makeText(this, "account or password is invalid",
                          Toast.LENGTH_SHORT).show()
            }
        }
    }

}
```

修改activity_main.xml：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <Button
        android:id="@+id/forceOffline"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Send force offline broadcast" />

</LinearLayout>
```

修改MainActivity中的代码:

```kotlin
class MainActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        forceOffline.setOnClickListener {
            val intent = Intent("com.example.broadcastbestpractice.FORCE_OFFLINE")
            sendBroadcast(intent)
        }
    }

}
```

在BaseActivity中动态注册一个BroadcastReceiver：

```kotlin
open class BaseActivity : AppCompatActivity() {

    lateinit var receiver: ForceOfflineReceiver

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ActivityCollector.addActivity(this)
    }

    override fun onResume() {
        super.onResume()
        val intentFilter = IntentFilter()
        intentFilter.addAction("com.example.broadcastbestpractice.FORCE_OFFLINE")
        receiver = ForceOfflineReceiver()
        registerReceiver(receiver, intentFilter)
    }

    override fun onPause() {
        super.onPause()
        unregisterReceiver(receiver)
    }

    override fun onDestroy() {
        super.onDestroy()
        ActivityCollector.removeActivity(this)
    }

    inner class ForceOfflineReceiver : BroadcastReceiver() {

        override fun onReceive(context: Context, intent: Intent) {
            AlertDialog.Builder(context).apply {
                setTitle("Warning")
                setMessage("You are forced to be offline. Please try to login again.")
                setCancelable(false)
                setPositiveButton("OK") { _, _ ->
                    ActivityCollector.finishAll() // 销毁所有Activity
                    val i = Intent(context, LoginActivity::class.java)
                    context.startActivity(i) // 重新启动LoginActivity
                }
                show()
            }
        }

    }

}
```

修改AndroidManifest.xml：

```xml
<!--将主Activity设置为LoginActivity-->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.example.broadcastbestpractice">
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".LoginActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <activity android:name=".MainActivity">
        </activity>
    </application>
</manifest>
```

