---
title: Builder模式
date: 2016-09-06 21:25
tags: 设计模式
categories: 设计模式
---

又是一个开学季，新生们都纷纷入学了。上了大学，买电脑便是每位学生要考虑的事情，作为计算机学院的一员，自然想买配置高点的电脑用来编码，这样会使用得舒服很多。可是手头又紧，想买配置好的电脑有时就会考虑自己去组装电脑。
那么，在这里就用Builder（建造者）模式来描述一下组装电脑这件事。先贴个UML图：
![这里写图片描述](http://img.blog.csdn.net/20160906204739992)
<!--more-->
首先，需要一个抽象类——电脑：
```
public abstract class Computer {
	private String mCPU;
	private String mDisplay;
	private String mMemory;
	
	//配置CPU
	public void setmCPU(String mCPU) {
		this.mCPU = mCPU;
	}
	
	//配置显示器
	public void setmDisplay(String mDisplay) {
		this.mDisplay = mDisplay;
	}
	
	//配置内存
	public void setmMemory(String mMemory) {
		this.mMemory = mMemory;
	}
	
	public String listHardware() {
		return "CPU:" + mCPU + ",显示器:" + mDisplay + ",内存:" + mMemory;
	}
}

```
然后接着写一个子类MyComputer：

```
public class MyComputer extends Computer{
	protected MyComputer() {};
}

```
好了，这时候已经和卖家沟通好要组装一部电脑，那么准备组装，首先需要一个抽象类Builder用来组装电脑：

```
public abstract class Builder {
	//组装CPU
	public abstract Builder buildCPU(String cpu);
	//组装显示器
	public abstract Builder buildDisplay(String display);
	//组装内存
	public abstract Builder buildMemory(String memory);
	//完成组装
	public abstract Computer create();
}
```
接着，需要一个MyComputerBuilder来实现具体组装：

```
public class MyComputerBuiler extends Builder{
	private Computer mComputer = new MyComputer();

	@Override
	public Builder buildCPU(String cpu) {
		mComputer.setmCPU(cpu);
		return this;
	}

	@Override
	public Builder buildDisplay(String display) {
		mComputer.setmDisplay(display);
		return this;
	}

	@Override
	public Builder buildMemory(String memory) {
		mComputer.setmMemory(memory);
		return this;
	}

	@Override
	public Computer create() {
		return mComputer;
	}
	
}
```

一切准备就绪，就能我们说需要配置哪些硬件了：

```
public class Buyer {
	public static void main(String[] args) {
		//初始化建造器
		Builder builer = new MyComputerBuiler();
		//需要的配置为Intel CPU，AOC显示器，8G内存
		Computer computer = builer.buildCPU("Intel").buildDisplay("AOC").buildMemory("8G").create();
		System.out.println(computer.listHardware());
	}
}
```
看看结果，一个电脑组装出来了！

> CPU:Intel,显示器:AOC,内存:8G

上述便是Builder模式的一个简单的例子。

Builder模式在Android中的应用
---------------------
相信这个应用大家都不会陌生，那就是：

```
AlertDialog.Builder builder = new AlertDialog.Builder(this);
builder.setTitle("TestDialog")
                .setMessage("This is a dialog!")
                .setPositiveButton("OK", null)
                .setNegativeButton("Cancel", null)
                .create().show();
```
在Android中，我们比较常用的AlertDialog就使用了建造者模式，具体看看是如何实现的：

```
public class AlertDialog extends AppCompatDialog implements DialogInterface { 
	//AlertController 为弹窗的控制器，在该类里面具体实现了设置dialog
    private AlertController mAlert;

    AlertDialog(Context context, int theme, boolean createThemeContextWrapper) {
        super(context, resolveDialogTheme(context, theme));
        mAlert = new AlertController(getContext(), this, getWindow());
    }
	
	
    protected AlertDialog(Context context, boolean cancelable, OnCancelListener cancelListener) {
        super(context, resolveDialogTheme(context, 0));
        setCancelable(cancelable);
        setOnCancelListener(cancelListener);
        //实例化AlertController 
        mAlert = new AlertController(context, this, getWindow());
    }
    
    @Override
	public void setTitle(CharSequence title) {
		//调用AlertController 中的setTitle方法
	    mAlert.setTitle(title);
	}
	
    public static class Builder {
	    //属性P为用户设置的各种参数
		private final AlertController.AlertParams P;
        private int mTheme;

		public Builder(Context context) {
            this(context, resolveDialogTheme(context, 0));
        }
		//构造器
        public Builder(Context context, int theme) {
            P = new AlertController.AlertParams(new ContextThemeWrapper(
                    context, resolveDialogTheme(context, theme)));
            mTheme = theme;
        }
        
	     
	    //设置确认按钮
	    public Builder setPositiveButton(CharSequence text, final OnClickListener listener) {
	            P.mPositiveButtonText = text;
	            P.mPositiveButtonListener = listener;
	            return this;
	    } 
	        
	    //设置取消按钮
	    public Builder setNegativeButton(CharSequence text, final OnClickListener listener) {
	            P.mNegativeButtonText = text;
	            P.mNegativeButtonListener = listener;
	            return this;
	    } 
	    
	    
	    public AlertDialog create() {
		    //构造一个AlertDialog，并将设置的参数传入
            final AlertDialog dialog = new AlertDialog(P.mContext, mTheme, false);
            //将P中的属性应用到AlertController 
            P.apply(dialog.mAlert);
            //设置各种按钮监听接口
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }

		//展示窗口，不深入具体细节
        public AlertDialog show() {
            AlertDialog dialog = create();
            dialog.show();
            return dialog;
        }
	}
}
```
可以看到上述通过apply方法来实现了设置窗口：

```
 public void apply(AlertController dialog) {
			 //应用各种属性
            if (mCustomTitleView != null) {
                dialog.setCustomTitle(mCustomTitleView);
            } else {
                if (mTitle != null) {
                    dialog.setTitle(mTitle);
                }
                if (mIcon != null) {
                    dialog.setIcon(mIcon);
                }
                if (mIconId != 0) {
                    dialog.setIcon(mIconId);
                }
                if (mIconAttrId != 0) {
                    dialog.setIcon(dialog.getIconAttributeResId(mIconAttrId));
                }
            }
            if (mMessage != null) {
                dialog.setMessage(mMessage);
            }
            if (mPositiveButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_POSITIVE, mPositiveButtonText,
                        mPositiveButtonListener, null);
            }
            //省略部分代码
        }
```
这样，一个dialog就展示出来了。
![这里写图片描述](http://img.blog.csdn.net/20160906212241024)

Builder模式的优缺点
-------------

 - 优点
	 -体现了良好封装性，容易扩展。
 - 缺点
	 -产生了多余的Builder对象，消耗内存。
 