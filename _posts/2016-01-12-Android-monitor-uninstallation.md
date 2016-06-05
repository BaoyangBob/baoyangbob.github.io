---
layout: post
title: 应用卸载后的操作
category: 技术
keywords: 卸载,进程,jni
---


### 应用卸载后弹出调查网页/卸载相关联应用
需求：在卸载本应用A时卸载另一个辅助使用的应用B，减少用户操作，方便用户。
尝试：

 1. 监听卸载广播。注册BroadcastReceiver，监听"android.intent.action.PACKAGE_REMOVED"系统广播
 结果：No。因为卸载应用会首先杀死应用所在进程，此广播是在应用卸载后才由系统发出，所以应用不可能收到广播。Java端的类似方法都没有可能性。
 2. 监听相关广播，如将要卸载的广播。
 结果：No。经查阅无此类广播。
 
 3. 监控logcat日志。当用户进入本应用的卸载界面时，日志中会打印包含"android.intent.action.DELETE"和自己的包名，意味着自己将要被卸载。
结果：No。这种方法有两个问题，一是用户到了卸载界面并不意味一定会卸载，无法知道用户会不会按下卸载按钮。二是用户既然能到A应用的卸载界面，也能手动到B的卸载界面，与初衷不符。
 4. 卸载过程会删除"/data/data/包名"目录，可以用线程直接轮询这个目录是否存在，以此为依据判断自己是否被卸载。
 结果：No。与尝试1失败是相同原因，线程活不到应用卸载之后。
 5. 用C端进程轮询``/data/data/packageName``目录是否存在，监听自己是否被卸载。
 结果：OK。借助Java端进程fork出来的C端进程让init进程接管后，在应用被卸载后不会被销毁。


整理思路：在Activity/Service中调用native方法到C端操作。在C端fork出子进程，让该进程的父进程退出，使得子进程成为孤儿进程，被init进程接管，该子进程利用Linux系统的inotify机制来监控文件系统的变化。当应用所在的数据目录/data/data/packageName发生变化时，调用am start命令来打开网页或者打开卸载应用界面。
结论：在Android 4.4手机上可以达到目的，在Android 5.0手机上进程无权限调用命令。
在[google开发人员对于am命令的解释][8]中提到了两点，一是在``execpl()``中使用am命令时需要显式传入user值，否则am会使用默认值USER_CURRENT，会出现如下log,

    W/ActivityManager(  387): Permission Denial: startActivity asks to run as user -2 but is calling from user 0; this requires android.permission.INTERACT_ACROSS_USERS_FULL
所以需要在命令中加上``"--user", "0"``;二是am命令不是SDK的部分，SDK更新后不保证兼容性。

核心代码：

    #include <string.h>
    #include <jni.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <android/log.h>
    #include <unistd.h>
    #include <sys/inotify.h>

    /* 宏定义begin */
    //清0宏
    #define MEM_ZERO(pDest, destSize) memset(pDest, 0, destSize)
    
    //LOG宏定义
    #define LOG_INFO(tag, msg) __android_log_write(ANDROID_LOG_INFO, tag, msg)
    #define LOG_DEBUG(tag, msg) __android_log_write(ANDROID_LOG_DEBUG, tag, msg)
    #define LOG_WARN(tag, msg) __android_log_write(ANDROID_LOG_WARN, tag, msg)
    #define LOG_ERROR(tag, msg) __android_log_write(ANDROID_LOG_ERROR, tag, msg)
    
    
     #ifdef __cplusplus
     extern "C" {
    #endif
      
     /* 内全局变量begin */
    static char TAG[] = "BootReceiver.init";
     static jboolean b_IS_COPY = JNI_TRUE;
     /* 内全局变量 */
     
     /*
      * Class:     com_realvnc_androidsampleserver_receiver_BootReceiver
      * Method:    init
      * Signature: ()V
      */
     JNIEXPORT void JNICALL Java_com_realvnc_androidsampleserver_service_MonitorService_init(JNIEnv *env, jobject obj)
     {
    	 jstring tag = (*env)->NewStringUTF(env, TAG);
         //初始化log
         LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
                    , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "init OK"), &b_IS_COPY));

     //fork子进程，以执行轮询任务
      pid_t pid = fork();
     if (pid < 0){
            //出错log
            LOG_ERROR((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
                    , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "fork failed !!!"), &b_IS_COPY));
            exit(1);
     }else if (pid == 0){
        	//子进程再创建子进程
        	printf("I am the first child process.pid:%d\tppid:%d\n",getpid(),getppid());
        	pid = fork();
        	if (pid < 0){
        	    perror("fork error:");
        	    exit(1);
        	}
        	//第一个子进程退出
        	else if (pid >0){
        	    printf("first procee is exited.\n");
        	    exit(0);
        	}
        	//第二个子进程
        	//睡眠3s保证第一个子进程退出，这样第二个子进程的父亲就是init进程里
        	sleep(3);
        	printf("I am the second child process.pid: %d\tppid:%d\n",getpid(),getppid());
        	setsid();
        	//子进程注册"/data/data/com.realvnc.androidsampleserver"目录监听器
        	int fileDescriptor = inotify_init();
        	if (fileDescriptor < 0){
        	        LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                        , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "inotify_init failed !!!"), &b_IS_COPY));

        	        exit(1);
        	}

        	 int watchDescriptor;
        	 watchDescriptor = inotify_add_watch(fileDescriptor, "/data/data/com.realvnc.androidsampleserver", IN_DELETE);
        	 if (watchDescriptor < 0){
        	                LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                        , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "inotify_add_watch failed !!!"), &b_IS_COPY));

        	                exit(1);
        	 }

        	            //分配缓存，以便读取event，缓存大小=一个struct inotify_event的大小，这样一次处理一个event
        	            void *p_buf = malloc(sizeof(struct inotify_event));
        	if (p_buf == NULL){
        	                LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                        , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "malloc failed !!!"), &b_IS_COPY));

        	                exit(1);
        	}
        	            //开始监听
        	            LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                        , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "start observer"), &b_IS_COPY));
        	            size_t readBytes = read(fileDescriptor, p_buf, sizeof(struct inotify_event));

        	            //read会阻塞进程，走到这里说明收到目录被删除的事件，注销监听器
        	            free(p_buf);
        	            inotify_rm_watch(fileDescriptor, IN_DELETE);

        	            //目录不存在log
        	            LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                        , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "uninstalled"), &b_IS_COPY));

        	            //执行命令am start -a android.intent.action.VIEW -d http://shouji.360.cn/web/uninstall/uninstall.html
        	           // execlp("am", "am", "start", "-a", "android.intent.action.VIEW", "-d", "http://shouji.360.cn/web/uninstall/uninstall.html", (char *)NULL);
        	            //4.2以上的系统由于用户权限管理更严格，需要加上 --user 0
        	         // execlp("am", "am", "start","--user", "0" ,"-a", "android.intent.action.VIEW", "-d", "http://www.baidu.com", (char *)NULL);

        	            //adb shell am start -a android.intent.action.DELETE -d package:<your app package>
        	            //adb shell pm uninstall -k com.packagename
        	           // execlp("pm", "pm", "uninstall","--user", "0" , "com.realvnc.android.remote", (char *)NULL);
        	            //execlp("am", "am", "start","--user", "0" ,"-a", "android.intent.action.DELETE", "-d", "package:com.realvnc.android.remote", (char *)NULL);
        	            LOG_DEBUG((*env)->GetStringUTFChars(env, tag, &b_IS_COPY)
        	                                 , (*env)->GetStringUTFChars(env, (*env)->NewStringUTF(env, "uninstalled1111"), &b_IS_COPY));
        	            execlp("am", "am", "start","--user", "0" ,"-a", "android.intent.action.DELETE", "-d", "\"package:com.realvnc.android.remote\"", (char *)NULL);


        	         exit(0);
        	     }
        	     //父进程处理第一个子进程退出
        	     if (waitpid(pid, NULL, 0) != pid)
        	     {
        	         perror("waitepid error:");
        	         exit(1);
        	     }
        	     exit(0);


    //        else
    //        {
    //            //父进程直接退出，使子进程被init进程领养，以避免子进程僵死
    //        }
        }



     #ifdef __cplusplus
     }
     #endif




相关问题：

[Why does getRuntime().exec(“/system/bin/am”) always return “Operation not permitted”?][3]

[Android应用如何监听自己是否被卸载及卸载反馈功能的实现1][4]，

[Android应用如何监听自己是否被卸载及卸载反馈功能的实现2][5]，

[Android应用如何监听自己是否被卸载及卸载反馈功能的实现3][6],

[Android不被kill的Service与卸载之后跳转出反馈页面][7]


 
  [3]: http://stackoverflow.com/questions/29509826/why-does-getruntime-exec-system-bin-am-always-return-operation-not-permit
  [4]: http://www.cnblogs.com/zealotrouge/p/3157126.html
  [5]: http://www.cnblogs.com/zealotrouge/p/3159772.html
  [6]: http://www.cnblogs.com/zealotrouge/p/3182617.html
  [7]: http://blog.csdn.net/jimmylopez/article/details/41015337#comments772.html
  [8]: https://code.google.com/p/android/issues/detail?id=39801
