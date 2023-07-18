# python 引用踩坑



python  引用的时候出现的一些问题

python 在引入兄弟模块的时候就容易出问题，比如有一个兄弟模块如下图

```

|____first_dir
| |______init__.py
| |____one.py
|____second_dir
| |______init__.py
| |____two.py

```

假如在 second_dir 这个模块中的 two.py 中引入 first_dir 模块

```
from first import one
```

当 two.py 直接运行就会报错

```py
ModuleNotFoundError: No module named 'first'
```

为什么会这样呢？ 我的理解下原因可能是：

1. python 是被设计的一个脚步语言，也就是每个 py 文件，既可以直接运行，又可以当作模块，为了支持这种特性。就需要防止其解释的文件系统混乱，肯定默认是以程序入口的 py 文件为根路径，这样的话也就无法引入兄弟包了，因为其根本就不在文件系统路径中。要解决也很简单就是手动将其加入文件系统中
2. 参考资料很好的解释的这个问题

参考资料：

1、Relative imports in python3: https://stackoverflow.com/questions/16981921/relative-imports-in-python-3

