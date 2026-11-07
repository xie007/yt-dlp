### 启动项目

你刚才写的代码是 yt-dlp 内置的爱奇艺 extractor，通过 `python -m yt_dlp` 调用即可，不需要单独运行一个程序。

在 PowerShell 中执行：

```bash
cd D:\my_code\Python\yt-dlp
```

先列出视频格式：

```bash
python -m yt_dlp --ignore-config --proxy "" -v -F "https://www.iqiyi.com/v_19rrkhw0rc.html"
```

正常时应看到类似输出：

```bash
100 mp4 384x224   | m3u8 | 240P
200 mp4 640x368   | m3u8 | 360P
300 mp4 896x520   | m3u8 | 480P
500 mp4 1248x720  | m3u8 | 720P
```

再实际下载测试：

```bash
python -m yt_dlp --ignore-config --proxy "" -f 100 "https://www.iqiyi.com/v_19rrkhw0rc.html"
```

运行仓库测试：

```bash
python -m devscripts.run_tests Iqiyi
```

测试会下载一小段视频并自动清理。也可以运行完整生成测试：

```bash
python -m unittest test.test_download.TestDownload.test_Iqiyi_all
```

如果出现“Geo-restricted to China”，通常是代理节点位于中国大陆以外。关闭代理或明确使用直连：

```bash
python -m yt_dlp --ignore-config --proxy "" -F "https://www.iqiyi.com/v_19rrkhw0rc.html"
```

如果 [iqiyi] 能识别 URL、列出 100/200/300/500 格式并成功下载，说明新 extractor 工作正常

### 为什么 python -m yt_dlp 会使用源码

Python 执行 -m 时，会把当前工作目录放在模块搜索路径的前面。当前目录下存在：

D:\my_code\Python\yt-dlp\yt_dlp\__init__.py

因此 Python 查找 yt_dlp 时，会先找到这个本地包：

D:\my_code\Python\yt-dlp\yt_dlp\__init__.py

找到后就不会继续查找：

C:\Users\Jerry\AppData\Local\Programs\Python\Python314\Lib\site-packages\yt_dlp

随后执行的是本地包中的：

D:\my_code\Python\yt-dlp\yt_dlp\__main__.py

这个文件调用本地 yt_dlp.main()，此后导入的 extractor、utils 等模块也都来自源码目录。因此修改 iqiyi.py 后不需要重新 pip install，下一个 Python 进程就会读取新代码。

### 不同启动方式的区别

| 启动方式 | 使用的代码 |
|---|---|
| 在仓库根目录运行 python -m yt_dlp | 本地源码 |
| 在仓库外运行 python -m yt_dlp | site-packages 中的 pip 版本 |
| 运行 yt-dlp 命令 | Scripts\yt-dlp.exe，通常使用 pip 安装版本 |
| 在仓库根目录运行 .\yt-dlp.cmd | 本地源码 |

刚才实测结果是：

yt-dlp --version          -> 2026.10.21
python -m yt_dlp --version -> 2026.10.26

从 D:\my_code\Python 运行时也能确认 pip 包被选中：

module = C:\Users\Jerry\...\site-packages\yt_dlp\__init__.py
version = 2026.10.21

所以可以这样理解：两个包都叫 yt_dlp，但 python -m yt_dlp 从仓库根目录启动时，本地目录先被找到，pip 安装的同名包就被遮蔽了。

### 为什么我没有找到对应的测试代码？

这段测试命令 `python -m unittest test.test_download.TestDownload.test_Iqiyi_all` 对应的是哪部分源码？为什么我没有找到对应的测试代码？

这个测试名是运行时动态生成的，源码中不会出现字面量 test_Iqiyi_all。我核对一下测试收集函数和生成器所在位置，把调用链具体对应到文件和行号。

test_Iqiyi_all 不是一个直接写在源码文件里的固定测试函数，而是 test/test_download.py 在导入时动态生成并挂载到 TestDownload 类上的。

对应关系如下：

1. iqiyi.py (yt_dlp/extractor/iqiyi.py:40) 中的 IqiyiIE._TESTS 提供测试数据：URL、期望的视频 ID、标题、格式、跳过原因等。
2. helper.py (test/helper.py:88) 中的 gettestcases() 扫描所有 extractor：

```py
    def gettestcases(include_onlymatching=False):
        for ie in yt_dlp.extractor.gen_extractors():
            yield from ie.get_testcases(include_onlymatching)
```

   因此它会读取 IqiyiIE._TESTS。
3. test_download.py (test/test_download.py:270) 中的 inject_tests() 为每条测试数据动态创建方法：

```py
    test_method = generator(test_case, tname)
    test_method.__name__ = tname
    setattr(TestDownload, test_method.__name__, test_method)
```

生成的名称类似：

test_Iqiyi
test_Iqiyi_1
test_Iqiyi_2
...

4. test_download.py (test/test_download.py:288) 中的 batch_generator() 再创建一个批量入口：

```py
def test_template(self):
    for label, num_tests in tests_counter[name].items():
        for i in range(num_tests):
            test_name = join_nonempty('test', name, label, i, delim='_')
            try:
                getattr(self, test_name)()
            except unittest.SkipTest:
                print(f'Skipped {test_name}')
```

5. 最下面的循环把它命名为 test_Iqiyi_all：

```py
    test_method.__name__ = f'test_{name}_all'
    setattr(TestDownload, test_method.__name__, test_method)
```

所以执行：

python -m unittest test.test_download.TestDownload.test_Iqiyi_all

时，实际流程是：

导入 test/test_download.py
    ↓
调用 gettestcases()
    ↓
读取 IqiyiIE._TESTS
    ↓
inject_tests() 动态创建 test_Iqiyi、test_Iqiyi_1...
    ↓
batch_generator() 创建 test_Iqiyi_all
    ↓
依次调用所有 Iqiyi 测试，并捕获 SkipTest

你搜索不到 test_Iqiyi_all，因为源码中只有生成逻辑：

f'test_{name}_all'

没有这个完整字符串。可以这样查看运行时生成的测试方法：

python -c "from test.test_download import TestDownload; print('\n'.join(x for x in dir(TestDownload) if x.startswith('test_Iqiyi')))"

当前会看到：

test_Iqiyi
test_Iqiyi_1
test_Iqiyi_2
test_Iqiyi_3
test_Iqiyi_4
test_Iqiyi_5
test_Iqiyi_all

其中 `test_Iqiyi_all` 是批量运行器；真正执行下载和断言的是前面那些动态生成的 test_template 方法，测试数据则来自 iqiyi.py (yt_dlp/extractor/iqiyi.py:40) 的 `_TESTS`。