---
layout: post
category: "java"
title:  "使用脚本移动鼠标，避免锁屏"
tags: [java]
---
## 配置方法:

由于公司安全策略的问题，所有机器自动设置屏幕自动设置3min钟屏保
java实现，通过判定当前是否活动状态，如果处于活跃状态，自动移动鼠标
```
package com.shsnc.mouse;

import java.awt.AWTException;
import java.awt.MouseInfo;
import java.awt.Point;
import java.awt.Robot;
import java.util.HashSet;
import java.util.Queue;
import java.util.Set;
import java.util.concurrent.ArrayBlockingQueue;

/**
 * 一个后台线程监控鼠标的移动，如果连续T时间间隔内都没有变化，那么启动移动鼠标的命令
 * @author gongp
 *
 */

public class V4 {
	//1s 检查一次
	public static int time_interval = 1000;
	//检查次数 ，默认20次
	public static int times = 20;
	
	public static boolean needMove = false;

	 public static void main(String[] args) throws AWTException {
		 
		 
		 
		 Thread moveThread = new Thread(new Runnable() {
				@Override
				public void run() {
					Robot robot = null;
					try {
						robot = new Robot();
					} catch (AWTException e) {
						e.printStackTrace();
					}
					boolean flag = true;
					while(true) {
						if(needMove) {
							if(flag) {
								robot.mouseMove(10,20);
							}else {
								robot.mouseMove(30, 40);
							}
							flag=!flag;
						}
						try {
							Thread.sleep(time_interval);
						} catch (InterruptedException e) {
							// TODO Auto-generated catch block
							e.printStackTrace();
						}
					}
					
				}
			});
			
			moveThread.start();
		 		 
		 
		 
		 
		 //连续10s 鼠标没有移动
		 Thread watcher = new Thread(new Runnable() {
			Queue<Integer> xs = new ArrayBlockingQueue<Integer>(times);
			@Override
			public void run() {
				//循环判断
				int i = 0 ;
				while(true) {
//					System.out.println(xs.size());
					Point mousepoint = MouseInfo.getPointerInfo().getLocation();
					if(i<20) {
						i++;
					}
					if(i>10) {
						xs.poll();
					}
					xs.add(mousepoint.x);
					//查看队列中有无不一样的值
					Set<Object> myset = new HashSet<>();
					Object[] array = xs.toArray();
					for (Object item : array) {
						myset.add(item);
					}
					if(myset.size()>3) {
						needMove=false;
					}else {
						needMove=true;
					}
		            System.out.println(mousepoint.x + "  " + mousepoint.y);
		            
		            try {
						Thread.sleep(time_interval);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
		            
				}
			}
		});
		 watcher.setDaemon(true); 
		 watcher.start(); 
		//设置为后台启动
	    }

}

```