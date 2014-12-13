---           
layout: post
title: "Distributing third party jars in Mesos"
date : 2014-12-13
categories: mesos scala
---

Every distributed application needs an effective way of distributing third party jars. Hadoop uses DistributedCache and Spark uses similar approaches to do same. When you are building your own distributed system, you also have to build a mechanism which allows you to dynamically distribute third part libraries.

This post talks how to implement this distribution on mesos. This implementation is inspired by Apache Spark implementation. 

This post extends code discussed in this [post](). If you are new to mesos please go through that post before continuing.

tl;dr Access the complete code here.

### Third Party libraries in distributed Applications

Almost every piece of code we write today uses one or more third party library. In distributed systems third party libraries become tricky to handle.As we don't have control over instantiation of our tasks, the underlying system should allow us to express the dependencies so that it can add those to the classpath when our task is run.

There are many ways to handle this problem. The following are the following options

## Creating an uber jar
We can create a big jar which contains our code and all the dependencies. Build tools like maven,ant makes this very easy to do. Though it's an attractive solution it has its own quirks

The following are few of the con's of creating a uber jar

* ####Jar Size 

We will be updating our code faster than changing dependencies. So every time when we make a change we have distribute a big uber jar rather than distributed only changed our code. This effects the startup time and lot's of network bandwidth.

* #### Version

If we mix user code with dependencies upgrading third libraries become challenging. Whenever we upgrade any dependence we have to recompile all code and re distribute again. This is not feasible to big code bases.

So it's not wise to mix user code with libraries. We will be needing an independent way of distributing jars.



## Using Distributed file system

One of the alternative to uber jar is using a distributed file system for distributing libraries. Normally any distributed applications run on distributed file system like HDFS. Though it's a good approach for most of applications it does comes with dependency that a distributed file system has to be available. Also the individual library jar size is small which performs poorly in file system like HDFS.



## Localized distribution over HTTP

This is the approach we are going to follow in this post. We are going to host our dependencies in a HTTP server and ask the slaves to pull the jars from server. We create this HTTP server on demand and configure tasks to read and configure using them.

The following are the steps to distribute the third party libraries.


### HTTP server at master side

 We run a server at master when we start executing task. The following code shows a http server created using jetty server.

 {% highlight scala %}
  class HttpServer(resourceBase: File) {
  var server: Server = null
  var port: Int = -1

  def start() = {
    if (server != null) {
      throw new RuntimeException("server already running")
    }
    else {

      val threadPool = new QueuedThreadPool()
      threadPool.setDaemon(true)

      server = new Server(0)
      server.setThreadPool(threadPool)

      //serve the files in the folder as the path is indicated
      val resourceHandler = new ResourceHandler
      resourceHandler.setResourceBase(resourceBase.getAbsolutePath)

      val handlerList = new HandlerList
      handlerList.setHandlers(Array(resourceHandler, new DefaultHandler))

      //start the server with handler and capture the port
      server.setHandler(handlerList)
      server.start()
      port = server.getConnectors()(0).getLocalPort
    }

  }   
 {% endhighlight   %}

 We call a start method to start which serves files from *resourceBase* folder.


### Integrating server with Scheduler


 The above server has to be started dynamically whenever we require to distribute the jars.


{% highlight scala %}

override def registered(driver: SchedulerDriver, frameworkId: FrameworkID, masterInfo: MasterInfo): Unit = {
    //create the jar server if there are any jars have to be distributed
    if(jars.size > 0 )  createJarServer()
  }

{% endhighlight   %}

We override registered method,so that whenever scheduler comes up we start the server.

{% highlight scala %}
 def createJarServer() = {
    val dirFile = Utils.createTempDir()
    println("jar directory is" + dirFile.getAbsolutePath)
    val fileNames = new ArrayBuffer[String]()
    for ((path, index) <- jars.zipWithIndex) {
      val file = new File(path)
      val fileName = index + "_" + file.getName
      copyFile(file, new File(dirFile, fileName))
      fileNames += fileName
    }

    jarServer = new HttpServer(dirFile)
    jarServer.start()
    val uri = jarServer.uri
    println("jar server started at " + uri)
    jarUris = fileNames.map(f => uri + "/" + f).mkString(",")
  }
{% endhighlight   %}

*createServer* copies specified jar from it's path a temporary location. Also it populates *jarUris* string which contains all the jar URI's in a comma separated manner.

### Passing jarUris to executor

We have to pass this uri's to the executor so that it can download the jars and add them to class path before it starts running tasks. We pass it as the command line argument to the executor script as follows.

{% highlight scala %}
 def getExecutorInfo(d: SchedulerDriver): ExecutorInfo = {
    val scriptPath = System.getProperty("executor_script_path","~/run-executor.sh")
    ExecutorInfo.newBuilder().
      setCommand(CommandInfo.newBuilder().setValue("" +
      "/bin/sh "+scriptPath+ s" $jarUris"))
      .setExecutorId(ExecutorID.newBuilder().setValue("1234"))
      .build()
  }
{% endhighlight   %}

### Accessing jars in Executor

In the executor side, we download this jars from specified jar uris. The following code shows that.

{% highlight scala %}
 def getClassLoader(): ClassLoader = {
        var loader = this.getClass.getClassLoader
        val localfiles = new ArrayBuffer[String]()

        //if jars is specified then split by comma to get all paths
        if (args.length > 0) {
          for (uri <- args(0).split(",").filter(_.size > 0)) {
            val url = new URL(uri)
            val fileName = url.getPath.split("/").last
            downloadFile(url, fileName)
            localfiles += fileName
          }

          //use the URL classloader to add it to the classpath
          if (localfiles.size > 0) {
            val urls = localfiles.map(file => new File(file).toURI.toURL).toArray
            loader = new URLClassLoader(urls, loader)
          }
        }
        loader
      }
{% endhighlight   %}

Once it downloads the jars, it adds to the classpath using URLClassLoader. We set this classloader on the thread which executes the task.

### Mysql task

The following is one of example which dynamically loads the mysql driver which passed as library jar.


{% highlight scala %}

def mysqlTask() = {
    new FunctionTask[Unit](
      () => {
        try {
          Thread.currentThread.getContextClassLoader.loadClass("com.mysql.jdbc.Driver")
          println("successfully loaded")
        }
        catch {
          case e: Exception => {
            e.printStackTrace()
            throw e
          }
        }
      }
    )
 }

{% endhighlight   %}

### Running

* Step 1 : Download code from []()
* Step 2 : Build the code using *mvn clean install*
* Step 3 : Update run-executor.sh file in src/main/resources to point to the right directory.

* Step 4 : Run *CustomTasks* main method with mesos master url , path to the run-executor.sh shell script and path to the mysql jar file. 

More detail steps for running you can find [here]().

### Output

The output should be available in mesos logs as specified [here](/mesos-helloworld-scala#output).

If everything goes right you should be able to see "successfully loaded" message in the logs.