---
layout: post
title: "QWebEngineView remains blank whatever I do"
tags:
- Qt
thumbnail_path: blog/2022-09-08-qwebengineview-remains-blank-whatever-i-do/pyqt.png
---

In the most recent version of PyQt5, `QWebEngineView` refuses to draw any page content. Aparently, the solution is to disable sandboxing, as mentioned in [this comment](https://github.com/vlaci/openconnect-sso/issues/69#issuecomment-1000571831):

```sh
export QTWEBENGINE_CHROMIUM_FLAGS="--no-sandbox"
```

With this flag set, the `QWebEngineView` finally starts to work properly. For example, we can render an embedded SVG:

```python
from PyQt5.QtWidgets import *
from PyQt5 import QtCore, QtGui
from PyQt5.QtGui import *
from PyQt5.QtCore import *
from PyQt5.QtWebEngineWidgets import QWebEngineView
import sys

svg = \
"""
<svg height="1000" width="1000" viewBox="-30 0 300 300" xmlns="http://www.w3.org/2000/svg">
    <path d="M 25 78  C -26 28  97 -15  98 91 C 86 34 16 33 25 78" fill="#3993c9"/>
    <path d="M 25 78  C -26 28  97 -15  98 91 C 86 34 16 33 25 78"  fill="#f3622a" transform= "rotate(90 30 64) translate(5 -14)"/>
    <path d="M 25 78  C -26 28  97 -15  98 91 C 86 34 16 33 25 78"  fill="#c1af2c" transform= "rotate(180 25 78) translate(-19 9)"/>
    <path d="M 25 78  C -26 28  97 -15  98 91 C 86 34 16 33 25 78"  fill="#499c43" transform= "rotate(-90 25 78) translate(-5 14)"/>
    <circle cx="34.5" cy="73.5" r="40"   fill="white" fill-opacity="0.3" />
</svg>
"""

class Window(QMainWindow):
 
    def __init__(self):
        super().__init__()
        webView = QWebEngineView(self)
        webView.setContent(svg.encode(), mimeType='image/svg+xml')
        self.setCentralWidget(webView)
        self.setWindowTitle("QWebEngineView SVG Test")
        self.setGeometry(0, 0, 800, 600)
        self.show()

app = QApplication(sys.argv)
window = Window()
sys.exit(app.exec())
```

![alt text](\assets\img\blog\2022-09-08-qwebengineview-remains-blank-whatever-i-do\screenshot.png)

