---
layout:     post
title:      AsyncTask源码分析
subtitle:   AsyncTask是Android很有特色的多线程切换框架，我们今天看看源码
date:       2021-12-14
author:     Zejian Zhan
header-img: img/art-Anaconda-TensorFlow.jpg
catalog: true
tags:
    - Android
    - 线程池
    - 多线程
---


# AsyncTask是Android很有特色的多线程切换框架，我们今天看看源码
--

首先，我们粗略知道AsyncTask的主要用法，它主要用于在后台线程请求任务，在执行前和执行后可以在主线程进行相关页面的刷新，因此涉及到[线程切换]和[线程池]两个较重要的关注点。

  private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

[线程切换]自然是要用到Handler，可以看到的是AsyncTask类中有个静态内部类InternalHandler继承了Handler，主要也就是根据msg的what信息去回调onProgressUpdate或者finish函数。
[线程池]方面，AsyncTask内部有两个线程池，可以分为核心线程池和备用线程池，我们先看以下代码：

  private static final int CORE_POOL_SIZE = 1;  //核心线程池的核心线程数
  private static final int MAXIMUM_POOL_SIZE = 20;  //核心线程池的最大线程数
  private static final int BACKUP_POOL_SIZE = 5;  //备用线程池的核心线程和最大线程数
  private static final int KEEP_ALIVE_SECONDS = 3;  //线程存活时间

核心线程池代码如下：
  public static final Executor THREAD_POOL_EXECUTOR;

  static {
      ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
              CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
              new SynchronousQueue<Runnable>(), sThreadFactory);
      threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
      THREAD_POOL_EXECUTOR = threadPoolExecutor;
  }

核心线程池的特点是核心线程数为1，最大线程为20个，使用了SynchronousQueue也就是不会将任务加入BlockingQueue等待而是直接交给线程执行，超过最大线程数后执行拒绝策略，我们再看下拒绝策略部分实现：

  private static ThreadPoolExecutor sBackupExecutor;
    private static LinkedBlockingQueue<Runnable> sBackupExecutorQueue;

    private static final RejectedExecutionHandler sRunOnSerialPolicy =
            new RejectedExecutionHandler() {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            android.util.Log.w(LOG_TAG, "Exceeded ThreadPoolExecutor pool size");
            // As a last ditch fallback, run it on an executor with an unbounded queue.
            // Create this executor lazily, hopefully almost never.
            synchronized (this) {
                if (sBackupExecutor == null) {
                    sBackupExecutorQueue = new LinkedBlockingQueue<Runnable>();
                    sBackupExecutor = new ThreadPoolExecutor(
                            BACKUP_POOL_SIZE, BACKUP_POOL_SIZE, KEEP_ALIVE_SECONDS,
                            TimeUnit.SECONDS, sBackupExecutorQueue, sThreadFactory);
                    sBackupExecutor.allowCoreThreadTimeOut(true);
                }
            }
            sBackupExecutor.execute(r);
        }
    };

拒绝策略中可以看到使用了备用线程池，最大和核心线程数都为5，没有任务执行时存活3秒(注意这里设置了核心线程也可以退出)，使用了无上界的BlockingQueue
---

## Install Anaconda
>[Anaconda](https://www.anaconda.com/) is a free and open-source distribution of the Python and R programming languages for scientific computing, that aims to simplify package management and deployment.   

**You can download anaconda [here](https://www.anaconda.com/distribution/#download-section).**

One of the advantage of anaconda is that it can create **isolated environment** in your device, and you can configure any libraries and toolkits in the 'env' without affect other environment. Once you are nor satisfied of your configuration, you can simplily delete the environment.

Note that in you are in **China**, download anaconda might take a long time due to some resons that cannot say. Instead, you can download it from [**Tsinghua mirror**](https://mirror.tuna.tsinghua.edu.cn/help/anaconda/), and install it **manually**.  

After downloading this successfully, try to run the installation file.
For example, if you use ubuntu, you can cd to the path of the sh file and run the following command:

```bash
./Anaconda3-5.3.1-Linux-x86_64.sh
```
***Attention that you should change the command above to your own installation file name.***

Then you will successfully install Anaconda!

## Create new environment by conda

If you are unwilling to create conda environment (maybe because of lazy), you can skip this section. However, I strongly reconmend you to create this **for the convience in the future**.  

Run the command below:
```bash
conda create -n tf
```
![picture1](/img/20190328post.jpg)

'tf' is the name of your new conda environment, you can try other names as your own interest.

For other management you conda env, you can read [this](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html?highlight=environment).

## Install Tensorflow

First, you need to change to the env you have just built by conda:
```bash
source activete tf
```
![picture2](/img/20190328post2.jpg)  

For Chinese users, before starting the installation, you may change the source of conda as the same reason before. For more details, read the webcite of [Tsinghua Mirror](https://mirror.tuna.tsinghua.edu.cn/help/anaconda/).
Chinese users should type in this:
```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --set show_channel_urls yes
```


Afterwards, type in the command to install TensorFlow you need:
```bash
conda install tensorflow-gpu
```
![picture3](/img/20190328post3.jpg)  

If you want to install a specific version of tensorflow-gpu or cpu veison, you can change the command like this:
```bash
conda install tensorflow-gpu=1.10.0  #if you want to install 1.10.0 version
conda install tensorflow  #if you want to install cpu version
```
After anaconda solve the environment, you just need to type in 'y' to confirm the installation.  

Anaconda will **automatically** install other libs and toolkits needed by tensorflow(e.g. CUDA, and cuDNN), so you have no need to worry about this.
--

Type in `python` to enter the python environment.
```python
import tensorflow as tf
tf.__version__
```
When you see the version of tensorflow, such as 1.10.0, you have successfully install it.

That's all, Thank you.

If you encounter any problems, you can open an issue in the **Comment area**.
