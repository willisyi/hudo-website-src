---
title: "用纯Python就能写一个漂亮的网页-Remi"
date: 2021-07-16T10:21:32+08:00
draft: false
---

# 用纯Python就能写一个漂亮的网页-Remi

我们在写一个网站或者一个网页界面的时候，需要学习很多东西，对小白来说很困难！比如我要做一个简单的网页交互：

要懂后端，比如Python里面的Django或者Flask，或者是Java里面的SpringBoot

要懂前端，现在都叫大前端了(因为很复杂)，比如前端的框架Vue/React, 然后页面的美化框架Bootstrap ，还有html ,csss 和Javascript 三驾马车.

如果只会python, 可不可以做一个简单的网页呢， 答案是可以的， 并且有开源的库可用， 今天介绍的就是开源Python REmote Interface Library Remi.  它是个仅有100Kbytes大小的平台无关的基于web的GUI库，并且提供GUI设计工具。

首先附上项目地址： https://github.com/dddomodossola/remi， GUI 设计工具 https://github.com/dddomodossola/remi/tree/master/editor

**REMI 特点如下：**

- 跟其他GUI库区别？ Kivy，PyQT和PyGObject都需要主机操作系统的本机代码，这意味着安装或编译大型依赖项。Remi只需要一个Web浏览器即可显示您的GUI。

- 我需要懂HTML吗？ 不，只需要使用Python进行编码。

- 它是开源的吗？ 当然！Remi是根据Apache许可发布的。开源，免费！

- 我需要某种网络服务器吗？ 不，自带网络服务器。
  

**安装**

稳定版

`pip install remi`

最新feature可通过源码安装, clone 源码然后

`python setup.py install`  或者 `pip install git+https://github.com/dddomodossola/remi.git`

**运行**

启动测试脚本（https://github.com/dddomodossola/remi/blob/master/examples/widgets_overview_app.py）即可打开一个demo。

`python widgets_overview_app.py`

![widgets](/website/images/remi-widgets.png)

**初窥代码**

一个简单的helloworld只需如下代码

`

```
import remi.gui as gui
from remi import start, App

class MyApp(App):
    def __init__(self, *args):
        super(MyApp, self).__init__(*args)

    def main(self):
        container = gui.VBox(width=120, height=100)
        self.lbl = gui.Label('Hello world!')
        self.bt = gui.Button('Press me!')

        # setting the listener for the onclick event of the button
        self.bt.onclick.do(self.on_button_pressed)

        # appending a widget to another, the first argument is a string key
        container.append(self.lbl)
        container.append(self.bt)

        # returning the root widget
        return container

    # listener function
    def on_button_pressed(self, widget):
        self.lbl.set_text('Button pressed!')
        self.bt.set_text('Hi!')

# starts the web server
start(MyApp)
```

`



