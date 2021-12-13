## 1. 使用外部包(package)后运行报错



```xml
Error: Cannot run with sound null safety, because the following dependencies
don't support null safety:

 - package:english_words
```

[启用无效空安全](https://dart.dev/null-safety#enable-null-safety)

如果您确定要以无效的null安全性运行应用程序，则可以使用`flutter run --no-sound-null-safety`命令，`--no-sound-null-safety`选项未在本文中进行记录，但是，最近几个月我没有遇到任何问题（尤其是自从整个Flutter框架已迁移到null安全性以来，尤其如此）。

要在您选择的IDE中进行设置，可以在IntelliJ中使用“编辑配置”（在**运行**配置中）→“其他运行args”，然后在您的IDE中搜索“其他Args”（带有“飞镖” /“ Flutter”） VSCode中的用户设置。

在Android Studio中：运行->编辑配置->添加其他运行参数-> --no-sound-null-safety

