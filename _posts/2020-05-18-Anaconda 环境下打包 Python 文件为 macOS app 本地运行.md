---
title: Anaconda 环境下打包 Python 文件为 macOS app 本地运行
description: 
categories:
- Python
tags:
- macOS
- Python
- Anaconda
---

## 环境

- 操作系统：macOS
- Anaconda
- 已安装 `py2app` ：

   ```python
   pip install py2app
   ```

## 文件结构

- XX Project
  - Pyfiles
    - xx.py
  - Resources
    - xx.ui
    - AppIcon.icns
  - Data
    - xx.json

## 步骤

1. `cd` 至 Python 文件所在文件夹：

   ```python
   cd /Users/dandy/Documents/XX Project/Pyfiles
   ```

2. 生成 `setup.py` ：

   ```python
   py2applet --make-setup xx.py
   ```
   
3. 将 `xx.py` 文件中调用的其他文件写入 `setup.py` 文件中的 `DATA_FILES` ：

   ```python
   DATA_FILES = ['../Resources/xx.ui', '../Data/xx.json']
   ```

4. 修改 `setup.py` 文件中的 `OPTIONS` 为如下内容：

```python
OPTIONS = {'argv_emulation': True,
           'iconfile': '../Resources/AppIcon.icns',
           'plist':{'PyRuntimeLocations':['/Users/dandy/opt/anaconda3/lib/libpython3.7m.dylib']
                   }
          }
```

5. 创建 app ：

   ```python
   python3 setup.py py2app -A
   ```

6. 将 `DATA_FILES` 中写入的文件拷贝到 `app/Contents` 中对应的各处，替换掉替身文件，以免源文件变动而影响 app 的运行。

