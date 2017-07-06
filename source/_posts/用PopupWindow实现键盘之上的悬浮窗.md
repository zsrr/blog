---
title: 用PopupWindow实现键盘之上的悬浮窗
date: 2017-04-08 20:22:23
tags: Android
categories: 安卓开发

---
最近遇到一个写评论的需求，就是点击一个写评论的按钮跳出软键盘，软键盘之上有一个输入框和发送按钮，想了好半天决定采用悬浮窗，其主要想法来自：[模仿微信，QQ评论输入框](http://www.itdadao.com/articles/c15a380342p0.html)，其效果图如下：
![](http://ok34fi9ya.bkt.clouddn.com/Screenshot_1491657613.png)

# 主要代码
## activity_main:

	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    tools:context="com.stephen.popupwindowedittext.MainActivity"
	    android:id="@+id/root">

	    <LinearLayout
	        android:id="@+id/comment_part"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:orientation="horizontal"
	        android:padding="12dp">
	
	        <TextView
	            android:id="@+id/comment"
	            android:layout_width="0dp"
	            android:layout_height="wrap_content"
	            android:layout_weight="1"
	            android:textStyle="bold"
	            android:text="评论"
	            android:textSize="14sp"/>
	        <Button
	            android:id="@+id/write_comment_button"
	            android:background="@drawable/bg_write_comment_button"
	            android:layout_width="48dp"
	            android:layout_height="24dp"
	            android:text="写评论"
	            android:textSize="10sp"
	            android:textColor="@android:color/white"/>
	    </LinearLayout>
	
	    <android.support.v7.widget.RecyclerView
	        android:id="@+id/comments_list"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:layout_below="@id/comment_part"/>
	</RelativeLayout>
## window_edittext

	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:orientation="horizontal"
	    android:layout_width="match_parent"
	    android:layout_height="wrap_content"
	    android:paddingLeft="12dp"
	    android:paddingRight="12dp">
	
	    <EditText
	        android:id="@+id/main_edit_text"
	        android:layout_width="0dp"
	        android:layout_height="wrap_content"
	        android:layout_weight="1"/>
	    <Button
	        android:background="@drawable/bg_send_button"
	        android:id="@+id/send_button"
	        android:layout_width="52dp"
	        android:layout_height="24dp"
	        android:layout_marginLeft="8dp"
	        android:text="发送"
	        android:textColor="@android:color/white"
	        android:textSize="12sp"/>
	</LinearLayout>
## MainActivity:

	public class MainActivity extends AppCompatActivity implements View.OnClickListener {
	
	    private List<String> data = new ArrayList<>();
	    private RecyclerView rv;
	    private InputMethodManager imm;
	
	    private View commentPatternView;
	    private PopupWindow commentPatternWindow;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        imm = (InputMethodManager) getSystemService(Context.INPUT_METHOD_SERVICE);
	
	        initView();
	    }
	
	    private void initView() {
	        findViewById(R.id.write_comment_button).setOnClickListener(this);
	        rv = (RecyclerView) findViewById(R.id.comments_list);
	        rv.setLayoutManager(new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false));
	        rv.setAdapter(new CommentsListAdapter(data));
	    }
	
	    @Override
	    public void onClick(View v) {
	        switch (v.getId()) {
	            case R.id.write_comment_button: {
	                showCommentPattern();
	                break;
	            }
	            case R.id.send_button: {
	                sendComment();
	                if (commentPatternWindow != null) {
	                    commentPatternWindow.dismiss();
	                }
	                break;
	            }
	        }
	    }
	
	    private void sendComment() {
	        String content = ((EditText) commentPatternView.findViewById(R.id.main_edit_text)).getText().toString();
	        data.add(content);
	        rv.getAdapter().notifyItemInserted(data.size() - 1);
	    }
	
	    private void showCommentPattern() {
	        View parent = findViewById(R.id.root);
	
	        if (commentPatternView == null) {
	            commentPatternView = getLayoutInflater().inflate(R.layout.window_edit_text, null);
	            commentPatternView.findViewById(R.id.send_button).setOnClickListener(this);
	        }
	
	        if (commentPatternWindow == null) {
	            commentPatternWindow = new PopupWindow(commentPatternView, WindowManager.LayoutParams.MATCH_PARENT,
	                    WindowManager.LayoutParams.WRAP_CONTENT, true);
	            commentPatternWindow.setBackgroundDrawable(new ColorDrawable(Color.WHITE));
	            commentPatternWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
	        }
	
	        commentPatternWindow.showAtLocation(parent, Gravity.BOTTOM, 0, 0);
	        commentPatternWindow.setOnDismissListener(new PopupWindow.OnDismissListener() {
	            @Override
	            public void onDismiss() {
	                imm.toggleSoftInput(InputMethodManager.HIDE_IMPLICIT_ONLY, 0);
	            }
	        });
	
	        Timer timer = new Timer();
	        timer.schedule(new TimerTask() {
	            @Override
	            public void run() {
	                imm.showSoftInput(commentPatternView.findViewById(R.id.main_edit_text), InputMethodManager.SHOW_FORCED);
	            }
	        }, 100);
	    }
	}

上面两个布局文件没什么好说的，主要是第三个文件，首先要注意的是要调用**PopupWindow**的**setBackgroundDrawable**方法，否则会有各种意想不到的bug；其次要调用**setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE)**，这是为了让**PopupWindow**出现在软键盘之上。

软键盘的调用方式也是值得注意的地方，在**showAtLocation**方法调用完成之后不能立即使软键盘弹出，得有一个延时的过程，软键盘消失的事件与window消失的事件绑定在一起即可。

# 缺陷
上述方法有一个缺陷，就是不能禁止掉**outsideTouchable**，因为要接受软键盘的输入，所以**focusable**必须为**true**，而**setOutsideTouchable(false)**方法必须在**focusable**为**false**，**touchable**为**true**时才生效，所以上述方法不能保证点击其余空白区域悬浮窗不消失。
