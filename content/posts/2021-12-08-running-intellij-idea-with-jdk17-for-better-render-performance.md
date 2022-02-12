---
title: Running IntelliJ IDEA with JDK 17 for Better Render Performance with Metal
date: 2021-12-08
--- 

Over the years, although a tremendously valuable tool, IntelliJ IDEA started to feel slowed down, even on these latest M1 Pro machines. It's become frustrating. I disabled as many plugins as not needed. However, it's still not sufficient. I have discovered that you can improve the performance by switching IDEA to the latest JDK. [JetBrains has its runtime](https://github.com/JetBrains/JetBrainsRuntime) with its patches for HiDPI supports, performance improvements, and bugfixes for running the IntelliJ family more smoothly. My issue probably relates to using an external 4K monitor in scaled mode, where [many others have the same problem](https://youtrack.jetbrains.com/issue/JBR-745), but the lags, unfortunately, persist in default display mode as well. I feel every new tool, not just IntelliJ, has some interaction lag, regardless of substantial improvements to hardware, but this is another discussion topic.

On Mac, as I learned just now, [java2d uses OpenGL to render](https://youtrack.jetbrains.com/issue/JBR-745), which openGL has been deprecated on Mac in favor of [Metal](https://developer.apple.com/metal/). The latest JBR17 builds have Metal support which has substantial improvements. IntelliJ started an initiative [JEP-382 - Project Lenai](https://blog.jetbrains.com/platform/2020/11/metal-for-intellij-platform/) to bring Metal to JDK for 2D rendering pipeline. And it's in [JDK 17 since b17](https://openjdk.java.net/jeps/382). 

Following flags needs to be added to your `vmoptions` because Java 17 [changed encapsualtion](https://www.infoq.com/news/2021/06/internals-encapsulated-jdk17/) preventing access to internal modules. We also force Metal to be used. You can search the `Edit Custom VM Options` action.

```sh
  --illegal-access=warn
  -Dsun.java2d.metal=true
  --add-opens=java.desktop/java.awt.event=ALL-UNNAMED
  --add-opens=java.desktop/sun.font=ALL-UNNAMED
  --add-opens=java.desktop/java.awt=ALL-UNNAMED
  --add-opens=java.desktop/sun.awt=ALL-UNNAMED
  --add-opens=java.base/java.lang=ALL-UNNAMED
  --add-opens=java.base/java.util=ALL-UNNAMED
  --add-opens=java.desktop/javax.swing=ALL-UNNAMED
  --add-opens=java.desktop/sun.swing=ALL-UNNAMED
  --add-opens=java.desktop/javax.swing.plaf.basic=ALL-UNNAMED
  --add-opens=java.desktop/java.awt.peer=ALL-UNNAMED
  --add-opens=java.desktop/javax.swing.text.html=ALL-UNNAMED
  --add-exports=java.desktop/sun.font=ALL-UNNAMED
  --add-exports=java.desktop/com.apple.eawt=ALL-UNNAMED
  --add-exports=java.desktop/com.apple.laf=ALL-UNNAMED
  --add-exports=java.desktop/com.apple.eawt.event=ALL-UNNAMED
  --add-opens=java.desktop/sun.lwawt.macosx=ALL-UNNAMED
```

Then download the latest release from https://github.com/JetBrains/JetBrainsRuntime/releases. Beware of your platform. I have downloaded file [jbr_jcef-17_0_1-osx-aarch64-b164.8.tar.gz](https://cache-redirector.jetbrains.com/intellij-jbr/jbr_jcef-17_0_1-osx-aarch64-b164.8.tar.gz) for an M1 Mac, which is an `osx-aarch64` platform. Note that if you untar this file using Mac UI instead of the `tar` command directly since the files are not signed, they will not run per new protections by Mac. Use `xattr -rd com.apple.quarantine <DIRECTORY>` to remove the quarantine. 

On Idea, open up the Actions to find `Choose Boot Java Runtime for the IDE` and point to the extracted folder. It'll restart. If you face any problems, rename or delete the extracted folder, and IntelliJ will default to the bundled runtime. I noticed some improvements, but it's hard to be sure when you are not measuring them correctly, but it won't (probably) hurt to try. At [Resmo](https://www.resmo.com), we are using Kotlin, and IntelliJ is invaluable to us, and it feels great to have a responsive IDE to be productive.

*Update: 24/12/2021:* If you are having problems with debug mode, please update to latest version of IDEA. 2021.3 works for me with occasional, non-blocking issues, but the experience is worth it.

*Update: 13/02/2021:* There is a [new JBR 17 release](https://github.com/JetBrains/JetBrainsRuntime/releases/tag/jbr17_0_2b315.1). Also add `--add-opens=java.desktop/sun.lwawt.macosx=ALL-UNNAMED` to options if you are having problems with menus in MacOS.