# python 性能问题
## 字典、列表、元组初始化用 `{} [] ()` 还是 `dict() list() tuple()`
```
import timeit
print(timeit.timeit(stmt='{}', number=10000000))
print(timeit.timeit(stmt='dict()', number=10000000))

输出：
0.1654788
0.83084
```

```
import timeit
print(timeit.timeit(stmt='[]', number=10000000))
print(timeit.timeit(stmt='list()', number=10000000))

输出：
0.1816867
0.8409157
```

```
import timeit
print(timeit.timeit(stmt='()', number=10000000))
print(timeit.timeit(stmt='tuple()', number=10000000))

输出：
0.1089527
0.5617243
```

# Python 语法
- 字典直接根据下标取值，如果下标对应的值不存在，会报错（和C++不同）