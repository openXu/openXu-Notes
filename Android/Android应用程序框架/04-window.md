
[这一次，彻底搞懂Android 中的Window](https://mp.weixin.qq.com/s/Q9HeT39w0LXGoR_ifyd9bg)


window机制就是为了解决屏幕上的view的显示逻辑问题，让所有view都按照秩序来显示，比如Activity上显示dialog、popupWindow、Toast等的层序问题。

Android中的window不像Windows系统中的window(每个window都对应一个应用程序，实实在在的显示能被看见)，它本身并不存在，我们无法感知，我们看到的只是view。window机制的操作单位是view树，view是window的存在形式，window是view的载体。在android中window是一个虚拟的概念，它并不存在，所以从源码中很难找到它的踪迹，如果要说它在源码中的存在形式，目前的认知就是WindowManagerService中每个view对应一个windowStatus（view的当前状态）。













