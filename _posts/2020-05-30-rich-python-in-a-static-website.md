---
layout: post
title: "Embedding Jupyter-quailty rich visual Python into a static website"
tags:
- Building
- Software Engineering
thumbnail_path: blog/2020-05-30-rich-python-in-a-static-website/iodide.png
pyodide_jekyll_url: "https://github.com/dmikushin/pyodide-jekyll"
pyodide_jekyll_published_url: "https://pyodide-jekyll.mikush.in/"
---

We all love our CV/blog websites hosted on GitHub Pages. We also love Jupyter notebooks for revoluting the look and feel of daily data processing. Now imagine that you can put Jupyter notebook right into your blog to showcase something without the server side? Mozilla makes it possible with their awesome [Iodide](https://iodide.io/). Actually, there is a bunch of new names here to distinguish:

* **Iodide** is a client-server platform, API and UI with a server side, which authenticates you and stores the data
* **Pyodide** is a Python interpreter along with Python modules working entirely in your client web browser by means of WebAssembly
* What we present here is Iodide minus server side, but still with its rich UI and Pyodide (Pyodide alone without UI can only output into browser console); let's call it Iodide static client

![alt text](\assets\img\blog\2020-05-30-rich-python-in-a-static-website\pyodide_jekyll_window.png)

I've built a sample [GitHub repo]({{ page.pyodide_jekyll_url }}) with Jekyll-operated Iodide static client published [here]({{ page.pyodide_jekyll_published_url }}). After you press ⏩ button, the Python interpreter is being downloaded, followed by imported modules - `matplotlib` in our case. Finally, the notebook is rendered into `Report Preview` window, which can also be viewed as a whitepaper (`View as report`). So happy embedding interactive data science into your blogs!

Given that Python runs entirely in your browser, it's much better than you have expected, right? Well, it's definitely doing good to some extent, unless you try to program sockets in Python in there (that's why it feels slightly impractical atm). SciPy is reportedly partially working.

Anyway, turns out Mozilla is the new star of Data Science! Or wait: actually they still need to put LaTeX into WebAssembly as well.

