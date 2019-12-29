## 背景

公司在做SmartRouter的时候，每个模块都会有一个通过AbstractorProcessor生成一个ModuleNameMap，主要功能简单解释：urlString对应一个Activity名称，然后通过urlString进行activity跳转。

最后Application在初始化的时候会调用SmartRouter.init，它会把每个module中的map整合成RouterMap，通过它可以实现不同模块之间的跳转。

看了下整合RouterMap的过程，可以发现他是通过查找所有的dex中在编译期生成的map然后进行整合，反射加上查找类，对app启动存在比较明显的影响。

## 对比

通过之前对阿里的ARouter的理解，知道它在生成RouterMap的过程也类似：

- 遍历所有的dex。
- 通过明确的包名，查找该package中的所有ModuleMap类，添加到当前的RouterMap中。
- 通过反射生成每个类对象。

他的优化：只有在新版本第一次启动的时候会去遍历查找，后续都会通过缓存的sp来进行初始化。



## 我们的优化

首先找出问题点：Router在Application中的初始化，存在查找/反射耗时操作，这块也会影响app的启动速度。所以为了能够有效解决冷启动时候的性能，我们需要把这块耗时的操作后置或者前置。

- 后置：通常我们app都会有splash闪屏页面，我们可以在这里面进行初始化。
- 前置：在编译期间就把每个moduleMap进行整合。

## 实践：

针对前置优化，提供下解决方案的思路。参考：commitId：[441886fbd9b8d1a3a6c1987afe39cdd3c3e0f2f2](https://github.com/dndxxiangyu/MyDemo/commit/441886fbd9b8d1a3a6c1987afe39cdd3c3e0f2f2)

1. 每个模块通过AbstractProcessor来生成它自己的ModuleNameMap。
2. 使用Gradle插件，通过RouterTransform把每个模块的ModuleNameMap进行整合。

核心代码：

```java
@Override
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
    super.transform(transformInvocation)
    System.out.println("start transform --------------------------")
    // Transform的inputs有两种类型，一种是目录，一种是jar包，要分开遍历
    transformInvocation.inputs.each { input ->
        input.directoryInputs.each { directoryInput ->
            MyInject.injectToast(directoryInput.file.path, mProject)
            def dest = transformInvocation.outputProvider.getContentLocation(directoryInput.name, directoryInput.contentTypes, directoryInput.scopes, Format.DIRECTORY)
            FileUtils.copyDirectory(directoryInput.file, dest)
        }
        input.jarInputs.each { jarInput ->
            // 重命名输出文件
            MyInject.collectRouterMap(jarInput, mProject)
            def jarName = jarInput.name
            def md5Name = new DigestUtils().md5Hex(jarInput.file.getAbsolutePath())
            if (jarName.endsWith(".jar")) {
                jarName = jarName.substring(0, jarName.length() - 4)
            }
            //生成输出路径
            def dest = transformInvocation.outputProvider.getContentLocation(jarName + md5Name,
                    jarInput.contentTypes, jarInput.scopes, Format.JAR)

            //将输入内容复制到输出
            FileUtils.copyFile(jarInput.file, dest)
        }
        long startTime = System.currentTimeMillis();
        input.jarInputs.each { jarInput ->

            //针对获取到的每个模块的routermap，进行整合
            MyInject.insertRouterMap(jarInput)
            def jarName = jarInput.name
            def md5Name = new DigestUtils().md5Hex(jarInput.file.getAbsolutePath())
            if (jarName.endsWith(".jar")) {
                jarName = jarName.substring(0, jarName.length() - 4)
            }
            //生成输出路径
            def dest = transformInvocation.outputProvider.getContentLocation(jarName + md5Name,
                    jarInput.contentTypes, jarInput.scopes, Format.JAR)

            FileUtils.copyFile(jarInput.file, dest)
        }
        System.out.println("extra time: " + (System.currentTimeMillis() - startTime))
    }

}
```

我们主工程app对各个模块（比如：feed/contact），所以在生产apk的时候，feed/contact中的class是以jar包形进入到Transform中的，所以我们需要对jar包中的class进行替换。

- 首先：找到每个jar中的所有ModuleNameMap，保存到set

- 然后：找到目标RouterMap类，把刚才遍历到的set各个ModuleNameMap注入到RouteMap，简单解释：

  ```java
  /**
       * 把每个routermap，注入到SmartRouter：api中。
       * @param jarInput
       */
      static void insertRouterMap(JarInput jarInput) {
          sClassPool.importPackage('android.content.Context')
          sClassPool.makeClass('android.content.Context')
          JarFile jarFile = new JarFile(jarInput.file)
          Enumeration<JarEntry> enumeration = jarFile.entries()
          sClassPool.insertClassPath(jarInput.file.absolutePath)
  
          while (enumeration.hasMoreElements()) {
              JarEntry entry = enumeration.nextElement()
              String entryName = entry.getName()
              if (!entryName.endsWith(".class")) {
                  continue
              }
  
              if (!entryName.contains("SmartRouter")) {
                  continue
              }
  
              println("entryName: " + entryName)
              String className = entryName.substring(0, entryName.length() - 6).replaceAll("/", ".")
  
              CtClass ctClass = sClassPool.getCtClass(className)
              CtMethod ctMethod = ctClass.getDeclaredMethod("init")
              for (String routerClassName : routSet) {
                  //当前测试写入字符串
                  ctMethod.insertBefore("new ${routerClassName});")
              }
              byte[] b = ctClass.toBytecode()
              JarHandler jarHandler = new JarHandler()
              jarHandler.replaceJarFile(jarInput, b, entryName)
          }
      }
  ```

把初始化ModuleMap的代码，直接注入到SmartRouter中。

- 最后：修改后，我们在Applicaiton的init初始化，只需要调用SmartRouter.init初始化router框架，通过new来生产每个ModuleNameMap，从而完成了非反射/查找形式的初始化RouterMap。

## 现有问题：

- Transform在生成总的RouterMap过程，会比较耗时，参考数据：10个class需要90ms。
- debug下的增量编译没有适配。

## 意义：

本文主要想着可以提供一个SmartRouter框架的启动优化方向，把需要依赖反射的过程前置，把影响App启动的因素降到最低。



