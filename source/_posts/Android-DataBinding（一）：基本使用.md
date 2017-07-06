---
title: Android DataBinding（一）：基本使用
date: 2017-02-14 21:42:57
tags: Android
categories: 安卓开发

---
## MVVM设计模式
这是一种由MVP发展而来，起源于WPF开发的一种架构模式。全称为Model-View-ViewModel，其中ViewModel便是MVP对应的Presenter。其最大的特征是ViewModel实现了View和Model的双向绑定，只需要为每一个View定制一个Model，在xml中去引用Model类，强大的IDE便会自动生成ViewModel类。强大，简单，可测试，低耦合。  

## DataBinding的使用  
### 环境
首先，在gradle中开启dataBinding选项：

	android {
		...
		dataBinding {
			enabled true
		}
	}
### 基本数据绑定  
**简单数据绑定**  
先定义一个简单的Model类User：

	public class User {
    	private final String name;
    	private final String age;

    	public User(String name, String age) {
        	this.name = name;
        	this.age = age;
    	}

    	public String getName() {
       		return name;
    	}

    	public String getAge() {
        	return age;
    	}
	}
界面xml文件：

	<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
        <import type="java.lang.String"/>
        <variable
            name="user"
            type="com.stephen.learning.databinding.User" />
    </data>

    <RelativeLayout
        android:id="@+id/activity_data_binding"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/name_text_view"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{user.name}"/>
        <TextView
            android:id="@+id/age_text_view"
            android:layout_below="@id/name_text_view"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:text="@{String.valueOf(user.age)}"/>
    </RelativeLayout>
	</layout>
- **&lt;layout&gt;**标签：使用数据绑定时的根标签。
- **&lt;data&gt;**标签：数据标签，这里用来定义绑定的变量及其类型。如果需要一个类中的常量活着静态方法，可以导入类型。
- **&lt;import&gt;**标签：导入类型。
- **&lt;variable&gt;**标签：定义变量。
- **android:text="@{user.name}"**：将数据绑定到view，注意引用user的name，运行时便会调用user.getName()方法，注意命名保持一致。 
 
其中**@{}**会有更加丰富的语法，后文再讲。

Activity调用：

	public class DataBindingActivity extends AppCompatActivity {

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	ActivityDataBindingBinding binding = DataBindingUtil.setContentView(this, 
        			R.layout.activity_data_binding);
        	binding.setUser(new User("Stephen", 19));
    	}
	}
注意此时IDE生成的binding类默认是根据xml文件来命名的，笔者的xml文件时activity_data_binding，那么生成的类名便是ActivityDataBindingBinding，当然如果嫌命名太丑的话可以在**data**标签当中修改:

	<data class="BindingForActivity">
        <import type="java.lang.String"/>
        <variable
            name="user"
            type="com.stephen.learning.databinding.User" />
    </data>
对应生成的类名便是BindingForActivity。  
注意生成Binding类的方法：

	DataBindingUtil.setContentView(this, R.layout.activity_data_binding);
这是在Activity当中的一般用法，如果是要绑定ListView, RecyclerView的ItemView的话，可以这么调用：  

	ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
最后一步便是通过生成的binding类设置在xml文件中定义的变量了，在variable中定义的name时user，那么函数的名字便是setUser，注意命名。  

**Observable数据绑定**  
上面的简单数据绑定存在的问题是：当绑定之后数据发生变化时，对应的View不会更改，也就是说，没有真正的实现双向绑定。如果要实现双向绑定，就得将Model类继承BaseObservable类，并且*assigning a Bindable annotation to the getter and notifying in the setter.* 此时User类改写为：  

	public class User extends BaseObservable {
    	private String name;
    	private int age;

    	public User(String name, int age) {
        	this.name = name;
        	this.age = age;
    	}

    	@Bindable
    	public String getName() {
        	return name;
    	}

    	@Bindable
    	public int getAge() {
        	return age;
    	}

    	public void setName(String name) {
        	this.name = name;
        	notifyPropertyChanged(BR.name);
    	}

    	public void setAge(int age) {
        	this.age = age;
        	notifyPropertyChanged(BR.age);
    	}
	}
注意，如果出现BR类找不到，BR.后面的属性找不到，**那就删除所有import有关数据绑定的类的语句，并在xml中删除（加块状注释）data标签，并clean project！**  

Activity:

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        user = new User("Stephen", 19);
        BindingForActivity bindingForActivity = DataBindingUtil.setContentView(this, R.layout.activity_data_binding);
        bindingForActivity.setUser(user);

        findViewById(R.id.change_button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                user.setAge(20);
                user.setName("Stephanie");
            }
        });
    }
### 处理交互事件
数据绑定同样适合View的交互事件的处理，某个动作要处理的逻辑也可以放在引用的类当中，方法如下：  

xml文件：

	<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
        ...
        <variable
            name="handler"
            type="com.stephen.learning.databinding.DataBindingActivity"
    </data>

    <RelativeLayout
        android:id="@+id/activity_data_binding"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        ...
        <Button
            android:id="@+id/change_button"
            android:layout_below="@id/age_text_view"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:onClick="@{(view) -> handler.onButtonClick(view)}"
    </RelativeLayout>
	</layout>
Activity:  

	public class DataBindingActivity extends AppCompatActivity {

    	@Override
    	protected void onCreate(Bundle savedInstanceState) {
        	...
        	bindingForActivity.setHandler(this);
    	}
    	
    	public void onButtonClick(View v) {
        	Log.e("DataBinding", String.valueOf(v.getId()));
        	Toast.makeText(this, "onClick!", Toast.LENGTH_SHORT).show();
    	}
	}
android:onClick后面是一个lambda表达式，参数是一个view，便是其自身，后面是点击之后将要做的事，这里调用了Activity中定义好的函数。  

除了onClick之外，还有onLongClick，onSearchClick(SearchView)等等，用法相同，这里不再细谈。
### 神奇的花括号
xml中的{}里面支持各种各样的操作，支持的操作符有这些（来自官方文档）：  
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-15%20%E4%B8%8A%E5%8D%8812.54.14.png)
用法很多，比如根据model选择view相应的状态：

	android:visibility="@{age < 13 ? View.GONE : View.VISIBLE}"
	android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
	android:background="@{isError ? @drawable/error : @color/white}"
调用相应的函数：

	<data>
    	<import type="com.example.MyStringUtils"/>
    	<variable name="user" type="com.example.User"/>
	</data>
	…
	<TextView
   		android:text="@{MyStringUtils.capitalize(user.lastName)}"
   		android:layout_width="wrap_content"
   		android:layout_height="wrap_content"/>  
 获取数组中的特定元素：
 
 	<data>
    	<import type="android.util.SparseArray"/>
    	<import type="java.util.Map"/>
    	<import type="java.util.List"/>
    	<variable name="list" type="List&lt;String&gt;"/>
    	<variable name="sparse" type="SparseArray&lt;String&gt;"/>
    	<variable name="map" type="Map&lt;String, String&gt;"/>
    	<variable name="index" type="int"/>
    	<variable name="key" type="String"/>
	</data>
	…
	android:text="@{list[index]}"
	…
	android:text="@{sparse[index]}"
	…
	android:text="@{map[key]}"
### 动态变量的绑定
有很多时候我们并不能提前知道所要的数据是什么，比如RecyclerView的adapter中的数据有可能是经过网络请求得来的，这里展示一下用法：  
item_view.xml:

	<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data class="ViewHolderBinding">
        <variable
            name="dataStr"
            type="String"/>
    </data>

    <LinearLayout
        android:orientation="vertical" android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
            android:id="@+id/test_text_view"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{dataStr}"/>
    </LinearLayout>
	</layout>
RecyclerView.Adapter:

	public class Adapter extends RecyclerView.Adapter<Adapter.ViewHolder> {

    	private List<String> mData;
    	private LayoutInflater inflater;

    	public Adapter(List<String> mData, Context context) {
        	this.mData = mData;
        	inflater = LayoutInflater.from(context);
    	}

    	@Override
    	public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        	return new ViewHolder(inflater.inflate(R.layout.item_view, parent, false));
    	}

    	@Override
    	public void onBindViewHolder(ViewHolder holder, int position) {
        	holder.binding.setDataStr(mData.get(position));
    	}

    	@Override
    	public int getItemCount() {
        	return mData.size();
    	}

    	static class ViewHolder extends RecyclerView.ViewHolder {
        	ViewHolderBinding binding;

        	public ViewHolder(View itemView) {
           	super(itemView);
            	binding = DataBindingUtil.bind(itemView);
        	}
    	}
	}
非常简单，不再细谈。
### 自定义Setter
有时候可以让我们的xml变得更加灵活，例如想要在xml里面设置一张图片的imageUrl就可以显示图片，实现的方式便是自定义Setter，实现过程如下：  
xml文件：

	<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <data class="ImageViewBinding">
        <import type="com.stephen.learning.databinding.DataBindingActivity"/>
    </data>

    <RelativeLayout
        android:id="@+id/activity_data_binding"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingTop="@dimen/activity_vertical_margin">

        <ImageView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:imageUrl='@{"http://imgsrc.baidu.com/baike/pic/item/7aec54e736d12f2ee289bffe4cc2d5628435689b.jpg"}'/>
    </RelativeLayout>
	</layout>
然后在Activity里面定义相应的方法，用BindAdapter标注：

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        DataBindingUtil.setContentView(this, R.layout.activity_data_binding);
    }

    @BindingAdapter({"imageUrl"})
    public static void loadImage(ImageView imageView, String imageUrl) {
        Picasso.with(imageView.getContext()).load(imageUrl).into(imageView);
    }
 要注意的两点一是在xml文件中要import static方法所在的类，第二点是方法要用BindAdaper备注，必须是一个public static方法，参数的顺序第一个必须是对应的View，往后的按照注解的参数来即可。
 
 
  **是不是感觉开启了新世界的大门呢？**