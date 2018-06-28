1、首先我们先探讨下为什么需要监控？我们的app做好后，性能到底怎么样？发生了性能问题，我们能否快速定位到问题发生在哪里？这几个问题就是我们需要监控的目的。

2、移动监控，业界做的比较有名的是听云、听云、博睿、OneAPM、阿里百川、datatrace。其中国内的这些产品大部分都抄袭自国外newrelic。采用JVM --》 agent  --》instrument+asm 实现编译阶段动态插码来实现。今天我所讲的是通过groovy脚本+ Javassist 来实现。其中asm 和Javassist 都是操作字节码工具。asm 操作起来比较复杂，Javassist 相对简单，但是局限性也大点。

3、我们需要采集的指标数据如下：

交互类指标：

•APP启动时长
•界面交互（界面个功能时长，交互轨迹，卡顿检测）
•网络交互（调用时长、DNS解析时长、状态码、首包时间、流量）
•
•
资源类指标：

•CPU、内存占用率，线程数、电量、帧率
•维度信息：设备类型、OS版本、应用版本、地域、运营商、网络接入类型。


稳定性能指标：

•卡顿/ANR次数
•Crash次数

•错误次数（HTTP/文件I/O等）



4、移动监控的技术架构如下：





5、通过架构，我们SDK 分2部分一部分为goovy实现的插件，类扫描器。主要功能实现编译阶段插码。

    另外一部分为具有采集功能的sdk，在扫描器工作的时候，会把这部分代码织入；同时对采集到的数据触发      上报策略。如果在弱网络环境或者无网环境采用文件缓存，强网络环境后台上报分析。

    1）插件实现，groovy脚本

    



 

首先得实现插件plug，需要实现groovy 的plgin 接口

public class JavassistPlugin  implements Plugin<Project>{

    void apply(Project project) {

        def log = project.logger

 

        def android = project.extensions.findByType(AppExtension)

        android.registerTransform(new JavassistTransform(project))

    }

}

第二步实现继承transform

写出关键部分代码：

@Override

    void transform(Context context, Collection<TransformInput> inputs,

                   Collection<TransformInput> referencedInputs,

                   TransformOutputProvider outputProvider, boolean isIncremental)

            throws IOException, TransformException, InterruptedException {

        // Transform的inputs有两种类型，一种是目录，一种是jar包，要分开遍历

        inputs.each { TransformInput input ->

            try {

                input.jarInputs.each {

                    ScannerClass.injectDir(it.file.getAbsolutePath(), "com\\huawei", project)

                    String outputFileName = it.name.replace(".jar", "") + '-' + it.file.path.hashCode()

                    def output = outputProvider.getContentLocation(outputFileName, it.contentTypes, it.scopes, Format.JAR)

                    FileUtils.copyFile(it.file, output)

                }

            } catch (Exception e) {

                project.logger.err e.getMessage()

            }

            //对类型为“文件夹”的input进行遍历

            input.directoryInputs.each { DirectoryInput directoryInput ->

                //文件夹里面包含的是我们手写的类以及R.class、BuildConfig.class以及R$XXX.class等

                ScannerClass.injectDir(directoryInput.file.absolutePath, "com\\huawei", project)

                // 获取output目录

                def dest = outputProvider.getContentLocation(directoryInput.name,

                        directoryInput.contentTypes, directoryInput.scopes,

                        Format.DIRECTORY)

 

                // 将input的目录复制到output指定目录

                FileUtils.copyDirectory(directoryInput.file, dest)

            }

        }

        ClassPool.getDefault().clearImportedPackages();

    }

第三部分就是类扫描器及其采集模块代码代码太长了copy的话版面太乱了就不粘贴了。下来直接讲原理了。

 

扫描器，直接在dex包之前扫描所有生成的class字节码，在关键部分，插入采集模块代码。

1、sdk 采集模块，通过TraceMachine 去采集各种指标数据，下边各种trace 都继承自TraceMachine ，实现各自的TraceMachine 



a、讲解下APP启动耗时时间。



 

b、性能采集中最主要的是UI界面用户操作业务功能的性能监控。我们希望监控到用户点击一个按钮，这个按钮调用了哪些方法，及其方法调用的整个调用链中的性能。如果发生性能问题，我们能快速知道，到底是调用链中哪一个方法导致性能问题的，针对性就比较明确。下图为调用链示意图，需要三个参数来保证调用链的生成。 

采集逻辑图如下：



网络请求监控示意图：

c、html5的监控



在扫描过程中发现如下方法就去织入代码，如果没有实现这些代码，那么就去添加上这些代码即可。

d、卡顿监控方案。可以通过Looper中的println 方法来实现，我们只需要重写其方法就可以拿到UI线程的时间。

Android里面的Activity，Fragment之类的其实都是系统通过aidl方式最终把启动，暂停，销毁Activity之类的封装成一个Message发给UI线程的Looper里的MessageQueue去挨个取出来执行操作。


Loop 源码：

Printer logging = me.mLogging;

 if (logging != null) { logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + ": " + msg.what); }

          msg.target.dispatchMessage(msg);

if (logging != null) { logging.println("<<<<< Finished to " + msg.target + " " + msg.callback); }

主要介绍上边几种主要采集方案，其它的都大同小异，就不赘述了。

这种方案是有优缺点的。

优点：灵活性比较强，任何想要数据都能采集到位。

确点:   会使得apk包增大
