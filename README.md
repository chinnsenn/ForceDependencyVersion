# 如何强制 Gradle 统一远程依赖库版本

在 Android 开发中免不了需要通过 Gradle 进行远程依赖官方库或者第三方 SDK，相信大家遇到过这样一个痛点，这个远程依赖本身也依赖了其他第三方库，而工程可能因为兼容性考虑，需要的的不是这个版本，以下举例一个场景：

## 示例
依赖 Retrofit 最新版本:
```groovy
implementation "com.squareup.retrofit2:retrofit:2.9.0"
```

通过终端输入指令:

macOS:
```Terminal
./gradlew [module_name]:dependencies
```

Windows:
```Terminal
gradlew [module_name]:dependencies
```

`以上 [module_name] 换成需要打印的模块名，比如 app`

可以打印出 Retrofit 依赖了哪些库：

```
+--- com.squareup.retrofit2:retrofit:2.9.0
     |    |    \--- com.squareup.okhttp3:okhttp:3.14.9
     |    |         \--- com.squareup.okio:okio:1.15.0
```

可以看到 Retrofit 2.9.0 依赖了 okhttp:3.14.9，但如果 App 需要兼容 Android 4.x 的机型，则需要使用 okhttp:3.12.3，通常的做法是依赖 Retrofit 时移除它的依赖，然后自己手动依赖指定版本的 okhttp ，像这样:

```groovy
implementation ("com.squareup.retrofit2:retrofit：2.9.0"){
    exclude group: ' com.squareup.okhttp3', module: 'okhttp' 
}
implementation 'com.squareup.okhttp3:okhttp:3.12.3'
```
这样写可以解决问题，但不够简洁，一个工程里会有数十个库，有类似的情况需要你手动一一排除依赖库，而且一个工程可能有很多 module ，很容易就在升级依赖库时疏忽了。

这里推荐一个能够强制统一远程依赖库的写法供大家参考:
在工程的根目录的 build.gradle 尾部中加入:

```groovy
subprojects {
	project.configurations.all {
	   //在这个例子里用不到这个语句，该语句的作用是全局移除某个依赖
		all*.exclude group: 'androidx.navigation', module: 'navigation-fragment'

       //遍历所有依赖库，通过判断 requested.group 和 requested.name 来指定使用的版本
		resolutionStrategy.eachDependency { DependencyResolveDetails details ->
			def requested = details.requested
			if (requested.group == 'com.squareup.okhttp3') {
				if (requested.module.name == 'okhttp3') {
					details.useVersion '3.12.3'
				}
			}
		}
	}
}
```

module 的 build.gradle 还是按正常依赖 Retrofit :

```groovy
implementation "com.squareup.retrofit2:retrofit:2.9.0"
```

然后再打印依赖库列表:

```
+--- com.squareup.retrofit2:retrofit:2.9.0
     |    |    \--- com.squareup.okhttp3:okhttp:3.14.9 -> 3.12.3
     |    |         \--- com.squareup.okio:okio:1.15.0 -> 2.2.2
```
如此你可以通过打印自己工程的依赖库列表，查看哪些库需要统一版本，或是通过 Android Studio 左侧目录列表栏选择 「Project」结构，展开 External Libraries 来查看当前工程依赖库列表

下图为一个真实的多 module 工程的截图，可以看到左侧出现很多同一个依赖库的不同版本

## 强制统一版本前
![统一版本前][1]
## 强制统一版本后
通过添加上述分享强制依赖版本的代码后，可以看到依赖库列表简洁了很多，除了部分硬编码写版本号（主要是因为懒），其他版本号通过统一一个文件定义版本号变量指定版本。

![统一版本后][2]

诚然，这个解决方案还是需要一点点劳动力来手动一一指定版本，但确实一劳永逸的方法。

以上是我在工作项目中的实践，也算解决了自己一个痛点，我将工程框架传到了 GitHub，还包括了统一管理依赖库名称版本的文件 config.gradle，希望对你有帮助。

[GitHub:ForceDependencyVersion][3]

[1]: https://raw.githubusercontent.com/chinnsenn/BlogFigureBed/master/blogimg%E6%88%AA%E5%B1%8F2021-08-23%20%E4%B8%8B%E5%8D%8811.00.44.png
[2]: https://raw.githubusercontent.com/chinnsenn/BlogFigureBed/master/blogimg%E6%88%AA%E5%B1%8F2021-08-23%20%E4%B8%8B%E5%8D%8810.53.19.png
[3]:https://github.com/chinnsenn/ForceDependencyVersion
