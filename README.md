# PKMS 学习 之 权限申请框架 

> AspectJ 实现的权限申请框架

AOP是一个老生常谈的话题，全称"Aspect Oriented Programming"，表示面向切面编程。由于面向对象的编程思想推崇高内聚、低耦合的架构风格，使得模块间代码的可见性变差，这使得实现下面的需求变得十分复杂：统计埋点、日志输出、权限拦截等等，如果手动编码，代码侵入性太高且不利于扩展，AOP技术应运而生。

通常来说，AOP都是为一些相对基础且固定的需求服务，实际常见的场景大致包括：

- 统计埋点
- 日志打印/打点
- 数据校验
- 行为拦截
- 性能监控
- 动态权限控制

本demo就是基于AOP实现的动态权限申请框架，实现原理为Android平台，通过Gradle Transform，在class文件生成后至dex文件生成前，遍历并匹配所有符合AspectJ文件中声明的切点，然后将事先声明好的代码在切点前后织入，会增加一定的编译时长，但几乎不会影响程序的运行时效率。

## 1.0 环境配置

在Android平台，我们通常使用上文提到的Aspectjx插件来配置AspectJ环境，具体使用是通过AspectJ注解完成。

1. 在项目根目录的build.gradle里依赖AspectJX



```bash
dependencies {
    classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.4'
}
```

1. 在需要支持AspectJ的module的build.gradle文件中声明插件。



```bash
apply plugin: 'android-aspectjx'
```

在编译阶段AspectJ会遍历工程中所有class文件（包括第三方类库的class）寻找符合条件的切入点，为加快这个过程或缩小代码织入范围，我们可以使用exclude排除掉指定包名的class。



```ruby
# app/build.gradle
aspectjx {
    //排除所有package路径中包含`android.support`的class文件及库（jar文件）
    exclude 'android.support'
}
```

在debug阶段我们更注重编译速度，可以关闭代码织入。



```cpp
# app/build.gradle
aspectjx {
    //关闭AspectJX功能
    enabled false
}
```

![飞书20220610-000039](https://user-images.githubusercontent.com/30100887/172892586-41a47914-d54a-4dd0-a2ac-ef231ca36729.png)


## 2.0 传统方式申请权限

> 耦合度太高

```java
// 传统方式申请权限的方式， 耦合度太高
public class MainActivity extends AppCompatActivity {

    private static final String TAG = MainActivity.class.getSimpleName();
    private static final int  MY_PERMISSIONS_REQUEST_READ_CONTACTS = 999;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 申请 危险权限
        requestPermission();
    }

    // 第二步：封装了一个requestPermission方法来动态检查和申请权限
    private void requestPermission() {

        Log.i(TAG,"requestPermission");

        // 注意：先检查之前有没有申请过这个READ_CONTACTS权限，如果么有申请过，才申请
        // Here, thisActivity is the current activity
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) {
            Log.i(TAG,"checkSelfPermission");

            // Should we show an explanation?
            if (ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.READ_CONTACTS)) {
                Log.i(TAG,"shouldShowRequestPermissionRationale 原来你个货，之前拒绝过申请权限");

                // Show an expanation to the user *asynchronously* -- don't block
                // this thread waiting for the user's response! After the user
                // sees the explanation, try again to request the permission.
                // 申请 联系人读取权限
                ActivityCompat.requestPermissions(this,
                        new String[]{Manifest.permission.READ_CONTACTS},
                        MY_PERMISSIONS_REQUEST_READ_CONTACTS);

            } else {
                Log.i(TAG,"requestPermissions 之前没有拒绝过，正常的申请权限");
                // No explanation needed, we can request the permission.
                // 申请 联系人读取权限
                ActivityCompat.requestPermissions(this,
                        new String[]{Manifest.permission.READ_CONTACTS},
                        MY_PERMISSIONS_REQUEST_READ_CONTACTS);
                // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
                // app-defined int constant. The callback method gets the
                // result of the request.
            }
        }
    }

    // 第三步：重写onRequestPermissionsResult方法根据用户的不同选择做出响应。
    @Override
    public void onRequestPermissionsResult(int requestCode,
                                           String permissions[], int[] grantResults) {
        switch (requestCode) {
            case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
                // If request is cancelled, the result arrays are empty.
                if (grantResults.length > 0
                        && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    Log.i(TAG,"onRequestPermissionsResult granted");
                    // permission was granted, yay! Do the
                    // contacts-related task you need to do.

                } else {
                    Log.i(TAG,"onRequestPermissionsResult denied");
                    // permission denied, boo! Disable the
                    // functionality that depends on this permission.
                    showWaringDialog();
                }
                return;
            }

            // other 'case' lines to check for other
            // permissions this app might request
        }
    }

    // 如果点击 拒绝，就会弹出这个 提示框
    private void showWaringDialog() {
        new AlertDialog.Builder(this)
                .setTitle("警告！")
                .setMessage("请前往设置->应用->PermissionDemo->权限中打开相关权限，否则功能无法正常运行！")
                .setPositiveButton("确定", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        // 一般情况下如果用户不授权的话，功能是无法运行的，做退出处理
                        finish();
                    }
                }).show();
    }
}
```



## 3.0 权限申请框架的方式

使用 权限申请框架  申请权限的方式， 只需要注解姐可以完成权限申请， 打打简化了开发的工作 ， *但是苦逼的就是架构师了*

要写权限申请库？
权限申请这一件事情，应该独立处理，隔离出来。不要让Activity或Fragment有着高耦合度。

使用者：
只需要关心，用一个注解，把具体权限传递到注解中就OK了，其他什么都不需要管

```java
package com.example.myapplication;

/**

 要写权限申请库？
 权限申请这一件事情，应该独立处理，隔离出来。
 不要让Activity或Fragment有着高耦合度。

 使用者：
 只需要关心，用一个注解，把具体权限传递到注解中就OK了，其他什么都不需要管

 */

/**
 * TODO 用户的角度上  use 我们写的权限申请库
 */
public class PermissionFrameActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void test(View view) {}

    // 点击事件
    public void permissionRequestTest(View view) {
        testRequest();
    }

    // 申请权限  函数名可以随意些
    @Permission(value = Manifest.permission.READ_EXTERNAL_STORAGE, requestCode = 200)
    public void testRequest() {
        Toast.makeText(this, "权限申请成功...", Toast.LENGTH_SHORT).show();
        // 100 行
    }

    // 权限被取消  函数名可以随意些
    @PermissionCancel
    public void testCancel() {
        Toast.makeText(this, "权限被拒绝", Toast.LENGTH_SHORT).show();
    }

    // 多次拒绝，还勾选了“不再提示”
    @PermissionDenied
    public void testDenied() {
        Toast.makeText(this, "权限被拒绝(用户勾选了 不再提示)，注意：你必须要去设置中打开此权限，否则功能无法使用", Toast.LENGTH_SHORT).show();
    }
}

```

