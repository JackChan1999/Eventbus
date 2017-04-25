## ![AndroidEventBus Logo](http://img.blog.csdn.net/20150203120217873) AndroidEventBus

如果你不知道事件总线是什么，那么没有关系，下面我们先来看这么一个场景：

> 你是否在开发的过程中遇到过想在Activity-B中回调Activity-A中的某个函数，但Activity又不能手动创建对象来设置一个Listener什么的？ 你是否想在某个Service中想更新Activity或者Fragment中的界面? 等等之类的组件之间的交互问题……

一经思考，你会发现[Android](http://lib.csdn.net/base/android)中的Activity，Fragment， Service之间的交互是比较麻烦的，可能我们第一想到的是使用广播接收器来在它们之间进行交互。例如上述所说在Activity-B中发一个广播，在Activity-A中注册一个广播接收器来接受该广播。但使用广播接收器稍显麻烦，如果你要将一个实体类当做数据在组件之间传递，那么该实体类还得实现序列化接口，这个成本实在有点高啊！如下所示 :

```java
    // ActivityA中注册广播接收器
    class ActivityA extends Activity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            registerReceiver(new BroadcastReceiver() {

                @Override
                public void onReceive(Context context, Intent intent) {
                    User person = intent.getParcelableExtra("user") ;
                }
            }, new IntentFilter("my_action")) ;
        }
        // ......
    }


    // ActivityB中发布广播
    class ActivityB extends Activity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);


            Intent intent  = new Intent("my_action") ;
            intent.putExtra("user", new User("mr.simple")) ;
            sendBroadcast(intent);
        }
        // ......
    }


    // 实体类实现序列化
    class User implements Parcelable {
        String name ;
        public User(String aName) {
            name = aName ;
        }

       // 代码省略

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeString(name);
        }
    }
```

是不是有很麻烦的感觉！

我们再来看一个示例，在开发过程中，我们经常要在子线程中做一些耗时操作，然后将结果更新到UI线程，除了AsyncTask之外，Thread加Handler是我们经常用的手段。我们看看如下示例:

```java
    class MyActivity extends Activity {

        Handler mHandler = new Handler () {
            public void handleMessage(android.os.Message msg) {
                if ( msg.what == 1 ) {
                    User user = (User)msg.obj ;
                    // do sth
                }
            };
        } ;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            // code ......

            new Thread(
                new Runnable() {
                    public void run() {
                        // do sth
                        User newUser = new User("simple") ;
                        Message msg = mHandler.obtainMessage() ;
                        msg.what = 1 ;
                        msg.obj = newUser ;
                        mHandler.sendMessage(msg) ;
                    }
            }).start();
        }
    }
```

是不是又有相当麻烦的感觉！！！此时，你应该冷静下来思考一下，还有哪些情况适合使用事件总线。例如穿透多个类只为一个回调的Listener......

事件总线框架就是为了简化这些操作而出现的，并且降低组件之间的耦合而出现的，到底如何解决呢？咱们继续看下去吧。

AndroidEventBus是一个Android平台轻量级的事件总线框架，它简化了Activity、Fragment、Service等组件之间的交互，很大程度上降低了它们之间的耦合，使得我们的代码更加简洁，耦合性更低，提升我们的代码质量。

在往下看之前，你可以考虑这么一个场景，两个Fragment之间的通信你会怎么实现？

按照Android官方给的建议的解决方法如下: [Communicating with the Activity](http://developer.android.com/intl/zh-cn/guide/components/fragments.html#CommunicatingWithActivity)，思路就是Activity实现某个接口，然后在Fragment-A关联上Activity之后将Activity强转为接口类型，然后在某个时刻Fragment中回调这个接口，然后再从Activity中调用Fragment-B中方法。这个过程是不是有点复杂呢？ 如果你也这么觉得，那也就是你继续看下去的理由了。

## 1. 基本结构

![结构图](http://img.blog.csdn.net/20150203125508110)
AndroidEventBus类似于观察者模式，通过register函数将需要订阅事件的对象注册到事件总线中，然后根据@Subscriber注解来查找对象中的订阅方法，并且将这些订阅方法和订阅对象存储在map中。当用户在某个地方发布一个事件时，事件总线根据事件的参数类型和tag找到对应的订阅者对象，最后执行订阅者对象中的方法。这些订阅方法会执行在用户指定的线程模型中，比如mode=ThreadMode.ASYNC则表示该订阅方法执行在子线程中，更多细节请看下面的说明。

## 2. 与greenrobot的EventBus的不同

2.1 greenrobot的[EventBus](https://github.com/greenrobot/EventBus)是一个非常流行的开源库，但是它在使用体验上并不友好，例如它的订阅函数必须以onEvent开头，并且如果需要指定该函数运行的线程则又要根据规则将函数名加上执行线程的模式名，这么说很难理解，比如我要将某个事件的接收函数执行在主线程，那么函数名必须为onEventMainThread。那如果我一个订阅者中有两个参数名相同，且都执行在主线程的接收函数呢? 这个时候似乎它就没法处理了。而且规定死了函数命名，那就不能很好的体现该函数的功能，也就是函数的自文档性。AndroidEventBus使用注解来标识接收函数，这样函数名不受限制，比如我可以把接收函数名写成updateUserInfo(Person info)，这样就灵活得多了。

2.2 另一个不同就是AndroidEventBus增加了一个额外的tag来标识每个接收函数可接收的事件的tag，这类似于Broadcast中的action，比如每个Broadcast对应一个或者多个action，当你发广播时你得指定这个广播的action，然后对应的广播接收器才能收到.greenrobot的EventBus只是根据函数参数类型来标识这个函数是否可以接收某个事件，这样导致只要是参数类型相同，任何的事件它都可以接收到，这样的投递原则就很局限了。比如我有两个事件，一个添加用户的事件， 一个删除用户的事件，他们的参数类型都为User，那么greenrobot的EventBus大概是这样的:

```java
private void onEventMainThread(User aUser) {
    // code
}
```

如果你有两个同参数类型的接收函数，并且都要执行在主线程，那如何命名呢 ？ 即使你有两个符合要求的函数吧，那么我实际上是添加用户的事件，但是由于EventBus只根据事件参数类型来判断接收函数，因此会导致两个函数都会被执行。

AndroidEventBus的策略是为每个事件添加一个tag，参数类型和tag共同标识一个事件的唯一性，这样就确保了事件的精确投递。

这就是AndroidEventBus和greenrobot的EventBus的不同，当然greenrobot出于性能的考虑这么处理也可以理解，但是我们在应用中发布的事件数量是很有限的，性能差异可以忽略，但使用体验上却是很直接的。另外由于本人对greenrobot的EventBus并不是很了解，很可能上述我所说的有误，如果是那样，欢迎您指出。

AndroidEventBus起初只是为了学习，但是在学习了EventBus的实现之后，发现它在使用上有些不便之处，我想既然我有这些感觉，应该也是有同感之人，在开发群里交流之后，发现确实有这样的情况。因此才将正式地AndroidEventBus以开源库的形式推出来，希望能够帮助到一些需要的人。当然，这个库的成长需要大家的支持与[测试](http://lib.csdn.net/base/softwaretest)，欢迎大家发 pull request。

## 3. 使用AndroidEventBus

你可以按照下面几个步骤来使用AndroidEventBus.

### 3.1 注册事件接收对象

```java
public class YourActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main_activity);
        // 注册对象
        EventBus.getDefault().register(this);
    }

    @Override
    protected void onDestroy() {
        // 不要忘记注销！！！！
        EventBus.getDefault().unregister(this);
        super.onDestroy();
    }
}
```

### 3.2 标识接收方法

通过Subscriber注解来标识事件接收对象中的接收方法

```java
public class YourActivity extends Activity {
    // code ......

    // 接收方法,默认的tag,执行在UI线程
    @Subscriber
    private void updateUser(User user) {
        Log.e("", "### update user name = " + user.name);
    }

    // 含有my_tag,当用户post事件时,只有指定了"my_tag"的事件才会触发该函数,执行在UI线程
    @Subscriber(tag = "my_tag")
    private void updateUserWithTag(User user) {
        Log.e("", "### update user with my_tag, name = " + user.name);
    }

    // 含有my_tag,当用户post事件时,只有指定了"my_tag"的事件才会触发该函数,
    // post函数在哪个线程执行,该函数就执行在哪个线程
    @Subscriber(tag = "my_tag", mode=ThreadMode.POST)
    private void updateUserWithMode(User user) {
        Log.e("", "### update user with my_tag, name = " + user.name);
    }

    // 含有my_tag,当用户post事件时,只有指定了"my_tag"的事件才会触发该函数,执行在一个独立的线程
    @Subscriber(tag = "my_tag", mode = ThreadMode.ASYNC)
    private void updateUserAsync(User user) {
        Log.e("", "### update user async , name = " + user.name + ", thread name = " + Thread.currentThread().getName());
    }
}
```

接收函数使用tag来标识可接收的事件类型，与BroadcastReceiver中指定action是一样的，这样可以精准的投递消息。mode可以指定目标函数执行在哪个线程，默认会执行在UI线程，方便用户更新UI。目标方法执行耗时操作时，可以设置mode为ASYNC，使之执行在子线程中。

### 3.3 发布事件

在其他组件，例如Activity， Fragment，Service中发布事件

```java
    EventBus.getDefault().post(new User("android"));

    // post a event with tag, the tag is like broadcast's action
    EventBus.getDefault().post(new User("mr.simple"), "my_tag");
```

发布事件之后，注册了该事件类型的对象就会接收到响应的事件

## 4. Github链接

欢迎star和fork， [AndroidEventBus框架](https://github.com/bboyfeiyu/AndroidEventBus)

## 5. jar文件下载

[AndroidEventBus下载](https://raw.githubusercontent.com/bboyfeiyu/AndroidEventBus/master/lib/androideventbus-1.0.2.jar)

# **什么是EventBus?**
eventBus是GreenRobot公司出品的一个基于【publish/subscribe】模型的开发库

1、GreenRobot公司除了出品eventBus还出品了比较有名的greenDao(sqlite数据库的orm开发框架)

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204506o00fkgmff82f9mfj.png.thumb.jpg)

可以看出GreenDao的star数为4581 而EventBus的star数更高 9564 (star数是一个框架/项目影响力的指标)

2、publish/subscribe是一个以发送消息与接收消息模型。以前咱们学习过相似的知识点有

- sendBroadcast与BroadcastReceiver是一个典型
- handler.sendMessage与handleMessage(Message msg)也是一个典型
- 所以该模型主要包含两个方面 一个是发送消息 一个是接收消息
 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204506npog0kmrodrf5e0o.png.thumb.jpg)

# **优点**
- 代码简洁、层次清晰，大大提高了代码的可读性和可维护性。
- 可以通过简单的代码把数据传递给Activities, Fragments, Threads, Services等等

# **下载EventBus官方SDK**
Github下载地址为：https://github.com/greenrobot/EventBus

# **EventBus3.0的使用**

可以更简单的在 app/build.gradle中，如下配置即可
![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204508lzpcpsjswtplf5ts.png.thumb.jpg)

然后运行gradle脚本

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204509bdd06xjek82m68ox.png.thumb.jpg)

EventBus的方法非常简洁只要掌握以下方法基本就可以完成企业开发了

## **1、注册与移除**
|方法|功能描述|
|---|---|
|EventBus.getDefault().register(this)|注册|
|EventBus.getDefault().unregister(this)|解除|
<br>
以上两个方法使用上跟广播接收者的注册与使用是类似的有注册必须在使用后移除，以Activity为例:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    EventBus.getDefault().register(this);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    EventBus.getDefault().unregister(this);
}
```
## **2、发送消息**
发布消息的对象即为【publisher】发布者

EventBus.getDefault().post(new MainMessage("Hello EventBus"));

Post参数类型为普通javaBean

```java
public class MainMessage {
    private String msg;
    public MainMessage(String msg) {
        this.msg = msg;
    }
    public String getMsg() {
        return msg;
    }
}
```

## **3、消息的接收处理(建议使用onEventXXX写法)**
接收消息的对象即为【subscriber】 订阅者

```java
//主线程中执行
    @Subscribe(threadMode = ThreadMode.MainThread)
    public void onMainEventBus(MainMessage msg) {
        Log.d(TAG, "onEventBus() returned: " + Thread.currentThread());
}
```

@Subscribe在案例中会细细地讲解 因为是一个大重点

## **4、案例分析：代替请求码与响应码**
MainActivity请求一个NextPageActivity要求该NextPageActivity关闭时把数据传回给MainActivity

startActivityForResult请求代码略

**在zaowu.itheima.com.eventbusdemo.demo1.FistEvent创建消息类(给post作为发送内容)**

```java
public class FistEvent {
    private String msg;
    public FistEvent(String msg) {
        this.msg = msg;
    }
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

zaowu.itheima.com.eventbusdemo.demo1.MainActivity

加载布局：layout/activity_main.xml

![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204509reb74b00pp8fkktf.png.thumb.jpg)

**注册与移除 订阅者**

```java
public class MainActivity extends AppCompatActivity {
    private  TextView textView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView= (TextView) findViewById(R.id.main_activity_tv);
        //注册事件
        EventBus.getDefault().register(this);
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        //移除注册
        EventBus.getDefault().unregister(this);
    }
    public void open(View view) {
        startActivity(new Intent(this, NextPageActivity.class));
    }
```

**编写订阅者接收消息方法**

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onEventMainThread(FistEvent event) {
    Log.i("itheima", "onEventMainThread:"+event.getMsg()+Thread.currentThread());
    textView.setText(event.getMsg());
}
```

- @Subscribe设置订阅者接收处理消息的方法
- 可以给该注解配置线程参数@Subscribe(threadMode = ThreadMode.MAIN)
- ThreadMode用来指定线程

**zaowu.itheima.com.eventbusdemo.demo1.NextPageActivity点击发送消息并关闭**

```java
public class NextPageActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_next_page);
    }
    public void event1(View view) {
        //发送消息
        FistEvent fistMessage = new FistEvent("NextPageActivity-FistEvent");
        EventBus.getDefault().post(fistMessage);
    }
}
```

**运行前后**

![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204510nil8db4hiz00v8lh.png.thumb.jpg) -->![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204511yeiv3k5ycksenv9h.png.thumb.jpg)

控件制台显示:消息已经由NextPageActivity传给MainActivity了。显示main主线程

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204511hl8mcm58vqs2vrlm.png.thumb.jpg)
不过首先注意一点eventBus将消息传给参数类型一致的方法

eventBus底层post出消息后，会查找作为subscriber订阅者的参数类型一致的方法然后向这些方法传递post出来的消息。

所以当订阅者有两个以上参数类型一致的方法时都会接收到消息。

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204511dl4ulusuafoka81z.png.thumb.jpg)

建议开发者创建不同的javaBean类来代表不同的消息类型

##**ThreadMode原理**

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204512c75oyc88yv5o0eq6.png.thumb.jpg)

|方法|功能描述|
|---|---|
|ThreadMode.MAIN|标注方法运行在主线程，可以更新UI|
|ThreadMode.BACKGROUND|标注方法运行在子线程，不可以更新UI但可以执行耗时代码|
|ThreadMode.POSTING|标注方法 运行在跟post一样的线程。如果post在主线程此时相当于MAIN,反之相当于BACKGROUND|
|ThreadMode.ASYNC|标注方法运行在子程。但是跟BACKGROUND有区别。举个例子如果ASYNC标注的方法将要运行时有一个BACKGROUND标注的方法正在运行(延时10秒)，前者标注的方法不会等待10秒结束再运行，而是另外开一个线程运行|
<br>
以下是EventBus的源代码分析

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/205653oese749xbwbkw9ce.png.thumb.jpg)

#**案例分析：代替广播接收者**
如果掌握了EventBus就可以不用广播接收者来发送消息接收消息了，用EventBus更简洁。

## 1、创建不同的消息对象
zaowu.itheima.com.eventbusdemo.demo2.MainEvent
zaowu.itheima.com.eventbusdemo.demo2.BackGroundEvent
zaowu.itheima.com.eventbusdemo.demo2.AsyncEvent
zaowu.itheima.com.eventbusdemo.demo2.PostingEvent

```java
public class MainEvent {
    private String msg;

    public MainEvent(String msg) {
        this.msg = msg;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```
## **2、发送不同的消息**

layout/activity_threadmode.xml
zaowu.itheima.com.eventbusdemo.demo2.ThreadModeActivity

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204513rfze2h4onbre5fbe.png.thumb.jpg)
```java
public void event1(View view) {
    //发送消息
    MainEvent mainEvent = new MainEvent("MainEvent");
    EventBus.getDefault().post(mainEvent);
    Log.i("itheima", "post(mainEvent);"+System.currentTimeMillis());
}
public void event2(View view) {
    //发送消息
    BackGroundEvent backGroundEvent = new BackGroundEvent("BackGroundEvent");
    EventBus.getDefault().post(backGroundEvent);
    Log.i("itheima", "post(backGroundEvent);"+System.currentTimeMillis());
}
public void event3(View view) {
    //发送消息
    AsyncEvent asyncEvent = new AsyncEvent("AsyncEvent");
    EventBus.getDefault().post(asyncEvent);
    Log.i("itheima", "post(asyncEvent);"+System.currentTimeMillis());
}
public void event4(View view) {
    //发送消息
    PostingEvent postingEvent = new PostingEvent("PostingEvent");
    EventBus.getDefault().post(postingEvent);
    Log.i("itheima", "post(postingEvent);"+System.currentTimeMillis());
}
```

## **3、接收消息**

zaowu.itheima.com.eventbusdemo.demo2.ThreadModeActivity

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMainEvent(MainEvent event) {
    Log.i("itheima", "onMainEvent接收事件:"+event.getMsg()+Thread.currentThread()+":"+System.currentTimeMillis());
}
@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void onBackGroundEvent(BackGroundEvent event) {
    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    Log.i("itheima", "onBackGroundEvent接收事件:"+event.getMsg()+Thread.currentThread()+":"+System.currentTimeMillis());
}
@Subscribe(threadMode = ThreadMode.ASYNC)
public void onAsyncEvent(AsyncEvent event) {
    Log.i("itheima", "AsyncEvent接收事件:"+event.getMsg()+Thread.currentThread()+":"+System.currentTimeMillis());
}
@Subscribe(threadMode = ThreadMode.POSTING)
public void onPostingEvent(PostingEvent event) {
    Log.i("itheima", "onPostingEvent接收事件:"+event.getMsg()+Thread.currentThread()+":"+System.currentTimeMillis());
}
```
运行结果【按点击顺序】

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204513x0o0ct3tkont609v.png.thumb.jpg)

【1】的运行结果为 ThreadMode.MAIN

![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204514af5b0p99sbkerhlr.png.thumb.jpg)

【2】的运行结果为ThreadMode.BACKGROUND
消息发送到接收到经过了10秒 且运行在子线程

![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/205654tbogbvaejttxgjlj.png.thumb.jpg)

【3】的运行结果为ThreadMode.POSTING 与post在一个线程

![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204514iw0zddndp3sz3d0d.png.thumb.jpg)

【4】测试ThreadMode.ASYNC时按以下顺序

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204515rlhex5dm7uistqyr.png.thumb.jpg)

运行结果为

 ![这里写图片描述](http://bbs.itheima.com/data/attachment/forum/201605/10/204515jfedctt0tfacj22o.png.thumb.jpg)
说明与ThreadMode.BACKGROUND都是子线程。但是如果后台有BACKGROUND在运行，ASYCN消息不会等待BACKGOUND完成，而是直接运行。

# **总结**
有了EeventBus代码会更简洁。同时它比handler,广播,内容观察者的实现都简洁统一还支持消息接收的线程指定。开发者重点握：ThreadMode变量

> 原文出处：http://bbs.itheima.com/thread-299831-1-1.html

# EventBus 源码解析

项目：[EventBus](https://github.com/greenrobot/EventBus)，分析者：[Trinea](https://github.com/trinea)，校对者：[扔物线](https://github.com/rengwuxian)

> 本文为 [Android 开源项目源码解析](http://a.codekk.com/) 中 EventBus 部分
> 项目地址：[EventBus](https://github.com/greenrobot/EventBus)，分析的版本：[ccc2771](https://github.com/greenrobot/EventBus/commit/ccc2771199f958a34bd4ea6c90d0a8c671c2e70a)，Demo 地址：[EventBus Demo](https://github.com/android-cn/android-open-project-demo/tree/master/event-bus-demo)
> 分析者：[Trinea](https://github.com/trinea)，校对者：[扔物线](https://github.com/rengwuxian)，校对状态：完成

### 1. 功能介绍

#### 1.1 EventBus

EventBus 是一个 Android 事件发布/订阅框架，通过解耦发布者和订阅者简化 Android 事件传递，这里的事件可以理解为消息，本文中统一称为事件。事件传递既可用于 Android 四大组件间通讯，也可以用户异步线程和主线程间通讯等等。
传统的事件传递方式包括：Handler、BroadCastReceiver、Interface 回调，相比之下 EventBus 的优点是代码简洁，使用简单，并将事件发布和订阅充分解耦。

#### 1.2 概念

**事件(Event)：**又可称为消息，本文中统一用事件表示。其实就是一个对象，可以是网络请求返回的字符串，也可以是某个开关状态等等。`事件类型(EventType)`指事件所属的 Class。
事件分为一般事件和 Sticky 事件，相对于一般事件，Sticky 事件不同之处在于，当事件发布后，再有订阅者开始订阅该类型事件，依然能收到该类型事件最近一个 Sticky 事件。

**订阅者(Subscriber)：**订阅某种事件类型的对象。当有发布者发布这类事件后，EventBus 会执行订阅者的 onEvent 函数，这个函数叫`事件响应函数`。订阅者通过 register 接口订阅某个事件类型，unregister 接口退订。订阅者存在优先级，优先级高的订阅者可以取消事件继续向优先级低的订阅者分发，默认所有订阅者优先级都为 0。

**发布者(Publisher)：**发布某事件的对象，通过 post 接口发布事件。

### 2. 总体设计

本项目较为简单，总体设计请参考`3.1 订阅者、发布者、EventBus 关系图`及`4.1 类关系图`。

### 3. 流程图

#### 3.1 订阅者、发布者、EventBus 关系图

![eventbus img](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/relation-flow-chart.png)
EventBus 负责存储订阅者、事件相关信息，订阅者和发布者都只和 EventBus 关联。

#### 3.2 事件响应流程

![eventbus img](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/event-response-flow-chart.png)
订阅者首先调用 EventBus 的 register 接口订阅某种类型的事件，当发布者通过 post 接口发布该类型的事件时，EventBus 执行调用者的事件响应函数。

### 4. 详细设计

#### 4.1 类关系图

![eventbus img](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/class-relation.png)
以上是 EventBus 主要类的关系图，从中我们也可以看出大部分类都与 EventBus 直接关联。上部分主要是订阅者相关信息，中间是 EventBus 类，下面是发布者发布事件后的调用。具体类的功能请看下面的详细介绍。

#### 4.2 类详细介绍

##### 4.2.1 EventBus.java

EventBus 类负责所有对外暴露的 API，其中的 register()、post()、unregister() 函数配合上自定义的 EventType 及事件响应函数即可完成核心功能，见 3.2 图。
EventBus 默认可通过静态函数 getDefault 获取单例，当然有需要也可以通过 EventBusBuilder 或 构造函数新建一个 EventBus，每个新建的 EventBus 发布和订阅事件都是相互隔离的，即一个 EventBus 对象中的发布者发布事件，另一个 EventBus 对象中的订阅者不会收到该订阅。
EventBus 中对外 API，主要包括两类：
**(1) register 和 unregister**
分别表示订阅事件和取消订阅。register 最底层函数有三个参数，分别为订阅者对象、是否是 Sticky 事件、优先级。

```
private synchronized void register(Object subscriber, boolean sticky, int priority)

```

PS：在此之前的版本 EventBus 还允许自定义事件响应函数名称，这版本中此功能已经被去除。
register 函数流程图如下： ![eventbus img](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/register-flow-chart.png)
register 函数中会先根据订阅者类名去`subscriberMethodFinder`中查找当前订阅者所有事件响应函数，然后循环每一个事件响应函数，依次执行下面的 subscribe 函数：

**(2) subscribe**
subscribe 函数分三步
第一步：通过`subscriptionsByEventType`得到该事件类型所有订阅者信息队列，根据优先级将当前订阅者信息插入到订阅者队列`subscriptionsByEventType`中；
第二步：在`typesBySubscriber`中得到当前订阅者订阅的所有事件队列，将此事件保存到队列`typesBySubscriber`中，用于后续取消订阅；
第三步：检查这个事件是否是 Sticky 事件，如果是则从`stickyEvents`事件保存队列中取出该事件类型最后一个事件发送给当前订阅者。

**(3) post、cancel 、removeStickyEvent**
post 函数用于发布事件，cancel 函数用于取消某订阅者订阅的所有事件类型、removeStickyEvent 函数用于删除 sticky 事件。
post 函数流程图如下： ![eventbus img](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/post-flow-chart.png)
post 函数会首先得到当前线程的 post 信息`PostingThreadState`，其中包含事件队列，将当前事件添加到其事件队列中，然后循环调用 postSingleEvent 函数发布队列中的每个事件。

postSingleEvent 函数会先去`eventTypesCache`得到该事件对应类型的的父类及接口类型，没有缓存则查找并插入缓存。循环得到的每个类型和接口，调用 postSingleEventForEventType 函数发布每个事件到每个订阅者。

postSingleEventForEventType 函数在`subscriptionsByEventType`查找该事件订阅者订阅者队列，调用 postToSubscription 函数向每个订阅者发布事件。

postToSubscription 函数中会判断订阅者的 ThreadMode，从而决定在什么 Mode 下执行事件响应函数。ThreadMode 共有四类：

1. `PostThread`：默认的 ThreadMode，表示在执行 Post 操作的线程直接调用订阅者的事件响应方法，不论该线程是否为主线程（UI 线程）。当该线程为主线程时，响应方法中不能有耗时操作，否则有卡主线程的风险。适用场景：**对于是否在主线程执行无要求，但若 Post 线程为主线程，不能耗时的操作**；

2. `MainThread`：在主线程中执行响应方法。如果发布线程就是主线程，则直接调用订阅者的事件响应方法，否则通过主线程的 Handler 发送消息在主线程中处理——调用订阅者的事件响应函数。显然，`MainThread`类的方法也不能有耗时操作，以避免卡主线程。适用场景：**必须在主线程执行的操作**；

3. `BackgroundThread`：在后台线程中执行响应方法。如果发布线程**不是**主线程，则直接调用订阅者的事件响应函数，否则启动**唯一的**后台线程去处理。由于后台线程是唯一的，当事件超过一个的时候，它们会被放在队列中依次执行，因此该类响应方法虽然没有`PostThread`类和`MainThread`类方法对性能敏感，但最好不要有重度耗时的操作或太频繁的轻度耗时操作，以造成其他操作等待。适用场景：*操作轻微耗时且不会过于频繁*，即一般的耗时操作都可以放在这里；

4. `Async`：不论发布线程是否为主线程，都使用一个空闲线程来处理。和`BackgroundThread`不同的是，`Async`类的所有线程是相互独立的，因此不会出现卡线程的问题。适用场景：*长耗时操作，例如网络访问*。

**(4) 主要成员变量含义**

1. `defaultInstance`默认的 EventBus 实例，根据`EventBus.getDefault()`函数得到。
2. `DEFAULT_BUILDER`默认的 EventBus Builder。
3. `eventTypesCache`事件对应类型及其父类和实现的接口的缓存，以 eventType 为 key，元素为 Object 的 ArrayList 为 Value，Object 对象为 eventType 的父类或接口。
4. `subscriptionsByEventType`事件订阅者的保存队列，以 eventType 为 key，元素为`Subscription`的 ArrayList 为 Value，其中`Subscription`为订阅者信息，由 subscriber, subscriberMethod, priority 构成。
5. `typesBySubscriber`订阅者订阅的事件的保存队列，以 subscriber 为 key，元素为 eventType 的 ArrayList 为 Value。
6. `stickyEvents`Sticky 事件保存队列，以 eventType 为 key，event 为元素，由此可以看出对于同一个 eventType 最多只会有一个 event 存在。
7. `currentPostingThreadState`当前线程的 post 信息，包括事件队列、是否正在分发中、是否在主线程、订阅者信息、事件实例、是否取消。
8. `mainThreadPoster`、`backgroundPoster`、`asyncPoster`事件主线程处理者、事件 Background 处理者、事件异步处理者。
9. `subscriberMethodFinder`订阅者响应函数信息存储和查找类。
10. `executorService`异步和 BackGround 处理方式的线程池。
11. `throwSubscriberException`当调用事件处理函数异常时是否抛出异常，默认为 false，建议通过

```java
EventBus.builder().throwSubscriberException(true).installDefaultEventBus()

```

打开。
12. `logSubscriberExceptions`当调用事件处理函数异常时是否打印异常信息，默认为 true。
13. `logNoSubscriberMessages`当没有订阅者订阅该事件时是否打印日志，默认为 true。
14. `sendSubscriberExceptionEvent`当调用事件处理函数异常时是否发送 SubscriberExceptionEvent 事件，若此开关打开，订阅者可通过

```java
public void onEvent(SubscriberExceptionEvent event)

```

订阅该事件进行处理，默认为 true。
15. `sendNoSubscriberEvent`当没有事件处理函数对事件处理时是否发送 NoSubscriberEvent 事件，若此开关打开，订阅者可通过

```java
public void onEvent(NoSubscriberEvent event)

```

订阅该事件进行处理，默认为 true。
16. `eventInheritance`是否支持事件继承，默认为 true。

##### 4.2.2 EventBusBuilder.java

跟一般 Builder 类似，用于在需要设置参数过多时构造 EventBus。包含的属性也是 EventBus 的一些设置参数，意义见`4.2.1 EventBus.java`的介绍，build 函数用于新建 EventBus 对象，installDefaultEventBus 函数将当前设置应用于 Default EventBus。

##### 4.2.3 SubscriberMethodFinder.java

订阅者响应函数信息存储和查找类，由 HashMap 缓存，以 ${subscriberClassName} 为 key，SubscriberMethod 对象为元素的 ArrayList 为 value。findSubscriberMethods 函数用于查找订阅者响应函数，如果不在缓存中，则遍历自己的每个函数并递归父类查找，查找成功后保存到缓存中。遍历及查找规则为：
a. 遍历 subscriberClass 每个方法；
b. 该方法不以`java.`、`javax.`、`android.`这些 SDK 函数开头，并以`onEvent`开头，表示可能是事件响应函数继续，否则检查下一个方法；
c. 该方法是否是 public 的，并且不是 ABSTRACT、STATIC、BRIDGE、SYNTHETIC 修饰的，满足条件则继续。其中 BRIDGE、SYNTHETIC 为编译器生成的一些函数修饰符；
d. 该方法是否只有 1 个参数，满足条件则继续；
e. 该方法名为 `onEvent` 则 threadMode 为`ThreadMode.PostThread`；
该方法名为 `onEventMainThread` 则 threadMode 为`ThreadMode.MainThread`；
该方法名为 `onEventBackgroundThread` 则 threadMode 为`ThreadMode.BackgroundThread`；
该方法名为 `onEventAsync` 则 threadMode 为`ThreadMode.Async`；
其他情况且不在忽略名单 (skipMethodVerificationForClasses) 中则抛出异常。
f. 得到该方法唯一的参数即事件类型 eventType，将这个方法、threadMode、eventType 一起构造 SubscriberMethod 对象放到 ArrayList 中。
g. 回到 b 遍历 subscriberClass 的下一个方法，若方法遍历结束到 h；
h. 回到 a 遍历自己的父类，若父类遍历结束回到 i；
i. 若 ArrayList 依然为空则抛出异常，否则会将 ArrayList 做为 value，${subscriberClassName} 做为 key 放到缓存 HashMap 中。 对于事件函数的查找有两个小的性能优化点：

```
a. 第一次查找后保存到了缓存中，即上面介绍的 HashMap
b. 遇到 java. javax. android. 开头的类会自动停止查找

```

类中的 skipMethodVerificationForClasses 属性表示跳过哪些类中非法以 `onEvent` 开头的函数检查，若不跳过则会抛出异常。
PS：在此之前的版本 EventBus 允许自定义事件响应函数名称，缓存的 HashMap key 为 ${subscriberClassName}.${eventMethodName}，这版本中此功能已经被去除。

##### 4.2.4 SubscriberMethod.java

订阅者事件响应函数信息，包括响应方法、线程 Mode、事件类型以及一个用来比较 SubscriberMethod 是否相等的特征值 methodString 共四个变量，其中 methodString 为 ${methodClassName}#${methodName}(${eventTypeClassName}。

##### 4.2.5 Subscription.java

订阅者信息，包括 subscriber 对象、事件响应方法 SubscriberMethod、优先级 priority。

##### 4.2.6 HandlerPoster.jva

事件主线程处理，对应`ThreadMode.MainThread`。继承自 Handler，enqueue 函数将事件放到队列中，并利用 handler 发送 message，handleMessage 函数从队列中取事件，invoke 事件响应函数处理。

##### 4.2.7 AsyncPoster.java

事件异步线程处理，对应`ThreadMode.Async`，继承自 Runnable。enqueue 函数将事件放到队列中，并调用线程池执行当前任务，在 run 函数从队列中取事件，invoke 事件响应函数处理。

##### 4.2.8 BackgroundPoster.java

事件 Background 处理，对应`ThreadMode.BackgroundThread`，继承自 Runnable。enqueue 函数将事件放到队列中，并调用线程池执行当前任务，在 run 函数从队列中取事件，invoke 事件响应函数处理。与 AsyncPoster.java 不同的是，BackgroundPoster 中的任务只在同一个线程中依次执行，而不是并发执行。

##### 4.2.9 PendingPost.java

订阅者和事件信息实体类，并含有同一队列中指向下一个对象的指针。通过缓存存储不用的对象，减少下次创建的性能消耗。

##### 4.2.10 PendingPostQueue.java

通过 head 和 tail 指针维护一个`PendingPost`队列。HandlerPoster、AsyncPoster、BackgroundPoster 都包含一个此队列实例，表示各自的订阅者及事件信息队列，在事件到来时进入队列，处理时从队列中取出一个元素进行处理。

##### 4.2.11 SubscriberExceptionEvent.java

当调用事件处理函数异常时发送的 EventBus 内部自定义事件，通过 post 发送，订阅者可自行订阅这类事件进行处理。

##### 4.2.12 NoSubscriberEvent.java

当没有事件处理函数对事件处理时发送的 EventBus 内部自定义事件，通过 post 发送，订阅者可自行订阅这类事件进行处理。

##### 4.2.13 EventBusException.java

封装于 RuntimeException 之上的 Exception，只是覆盖构造函数，相当于一个标记，标记是属于 EventBus 的 Exception。

##### 4.2.14 ThreadMode.java

线程 Mode 枚举类，表示事件响应函数执行线程信息，包括`ThreadMode.PostThread`、`ThreadMode.MainThread`、`ThreadMode.BackgroundThread`、`ThreadMode.Async`四种。

### 5. 与 Otto 对比

等 Otto 分析完成
