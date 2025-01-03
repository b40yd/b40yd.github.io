+++
title = "Python Progress Bar"
date = 2024-12-26
lastmod = 2025-01-03T15:23:30+08:00
tags = ["python", "progress", "bar"]
categories = ["python", "progress", "bar"]
draft = false
author = "B40yd"
+++

## 小工具分享 {#小工具分享}

进度条，表格美化显示等，推荐使用[rich](https://github.com/Textualize/rich)库。

经常需要用python处理一些小任务，比如数据迁移，数据处理，数据分析。在一些特定环境，无法安装第三方包，推荐以下实现的实用轮子。

-   定义 ProgressBar 类:
    -   支持指定进度条长度。
    -   支持进度条刷新。
    -   支持多任务进度。
    -   支持 with 操作。

-   进度显示信息：
    -   当前进度百分比。
    -   已完成的数量和总数量。
    -   速度（可选）。
    -   剩余时间（可选）。
    -   减少刷新频率：不要在每次更新时都刷新进度条，可以设置一个最小的刷新间隔。

<!--listend-->

```python
#encoding: utf-8
from threading import Lock
import random
import math
import time
import sys

class ProgressBar(object):
    def __init__(self, total, length=40, min_update_interval=0.1):
        self.total = total
        self.length = length
        self.current = 0
        self.lock = Lock()
        self.start_time = time.time()
        self.min_update_interval = min_update_interval
        self.last_update_time = 0

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        pass

    def update(self, advance=1):
        with self.lock:
            self.current += advance
            current_time = time.time()
            if current_time - self.last_update_time >= self.min_update_interval:
                self._render()
                self.last_update_time = current_time
            if self.current == self.total:
                self._render()
                sys.stdout.write("\n")
                sys.stdout.flush()

    def _render(self):
        elapsed_time = time.time() - self.start_time
        progress = self.current / float(self.total)
        block = int(round(self.length * progress))
        progress_percent = round(progress * 100, 2)
        if block > self.length:
            block = self.length
            bar = "▓" * block
        else:
            bar = "▓" * block + "-" * (self.length - block)

        speed = self.current / elapsed_time if elapsed_time > 0 else 0
        remaining_time = (self.total - self.current) / speed if speed > 0 else 0
        total_time = time.time() - self.start_time

        show_extend = [
            "{:.2f} items/s".format(speed),
            "ETA: {:.2f}s".format(remaining_time),
            "Total time: {:.2f}s".format(total_time),
        ]
        out_template = "\r{{0:4.2f}}% [{{1}}] {{2:>{0}}}/{{3:<{0}}} {{4}}".format(len(repr(self.total)))
        output = out_template.format(
            progress_percent, bar, self.current, self.total, " - ".join(show_extend))

        # Ensure the length of the output is consistent to avoid flickering
        sys.stdout.write(output)
        sys.stdout.flush()

def test_case1():
    page = int(math.ceil(9030000/10000))
    read_progres = ProgressBar(page)
    for n in range(0, page):
        read_progres.update(1)

def test_case2():
    with ProgressBar(100000000, length=50) as progress:
        for n in range(0, 100000000, 1):
            progress.update(1)
if __name__ == "__main__":
    test_case1()
    test_case2()
```

进度条效果：

```nil
84.4% [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓--------] 84401905/100000000 - 1743830.63 items/s - ETA: 8.94s - Total time: 48.40s
```
