---
title: Android自定义字体解决方案
date: 2017-02-26 16:04:26
tags: Android
categories: 安卓开发

---
在安卓开发中我们通常需要自定义字体，下面笔者给出两种解决方案，以供参考。
# 利用反射
这种方案通常情况下适用于替换整个app的默认字体，方法是利用反射更改**Typeface**类的静态**DEFAULT**成员，如下所示：

	public class MyApplication extends Application {
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	
	        Typeface typeface = Typeface.createFromAsset(getAssets(), "fonts/CourierNewBold.ttf");
	        try {
	            Field field = Typeface.class.getDeclaredField("MONOSPACE");
	            field.setAccessible(true);
	            field.set(null, typeface);
	        } catch (NoSuchFieldException | IllegalAccessException e) {
	            e.printStackTrace();
	            Log.e("Application", "Error happened!");
	        }
	    }
	}
style文件：

	<style name="AppTheme" parent="android:Theme.Holo.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="android:typeface">monospace</item>
    </style>
运行效果如下：
![](http://ok34fi9ya.bkt.clouddn.com/Screenshot_1488099698.png)
这种方法的局限也很明显，就是在Android 5.0默认主题之上并不能运行，需要更换一个旧的主题，如果主题不是很重要的话，这种方法就比较合适了。
# 利用DataBinding
这是笔者认为最快速，最便捷的方法了，用**BindingAdapters**注解标注方法即可在xml文件中指定**Typeface**，方法如下：

MyApplication:

	public class MyApplication extends Application {

	    public static final Map<String, Typeface> FONTS_MAP = new HashMap<>();
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	
	        FONTS_MAP.put(getString(R.string.courier_new_bold),
	                Typeface.createFromAsset(getAssets(), "fonts/" +
	                        getString(R.string.courier_new_bold)));
	        FONTS_MAP.put(getString(R.string.lao_sangam_mn),
	                Typeface.createFromAsset(getAssets(), "fonts/" +
	                        getString(R.string.lao_sangam_mn)));
	        FONTS_MAP.put(getString(R.string.microsoft_sans_serif),
	                Typeface.createFromAsset(getAssets(), "fonts/" +
	                        getString(R.string.microsoft_sans_serif)));
	    }
	}
FontsUtils:

	public class FontsUtils {
	    @BindingAdapter({"fonts"})
	    public static void setTextViewFonts(TextView tv, String fontName) {
	        tv.setTypeface(MyApplication.FONTS_MAP.get(fontName));
	    }
	}
xml:

	<layout xmlns:app="http://schemas.android.com/apk/res-auto">
	    <data>
	        <import type="com.stephen.customfontdemo.FontsUtils"/>
	    </data>
	    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	        xmlns:tools="http://schemas.android.com/tools"
	        android:id="@+id/activity_main"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:padding="16dp"
	        android:orientation="vertical"
	        tools:context="com.stephen.customfontdemo.MainActivity">
	
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="测试字体1"
	            app:fonts="@{@string/courier_new_bold}"/>
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="测试字体2"
	            app:fonts="@{@string/lao_sangam_mn}"/>
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="测试字体3"
	            app:fonts="@{@string/microsoft_sans_serif}"/>
	    </LinearLayout>
	</layout>
运行效果：![](http://ok34fi9ya.bkt.clouddn.com/Screenshot_1488100937.png)
额……可能2和3看不出效果，不过确实已经更改了。

需要注意的是： **Typeface**的加载要放在Application类当中，或者放在另一个线程当中进行，因为它是一个比较耗时的操作，如果大量的**TextView**用到自定义字体会造成卡顿。

这样在需要自定义字体时，只需要在**FONTS_MAP**和strings.xml当中注册，在xml中一行代码即可指定！