<span id="busuanzi_container_page_pv">本文总阅读量<span id="busuanzi_value_page_pv"></span>次</span>

## GMP模型
https://gitee.com/wangbinzjut/interview-go/blob/master/base/go-gpm.md


## golang垃圾回收
https://www.xhyonline.com/?p=1185

### 总结:
- GoV1.3- 普通标记清除法，整体过程需要启动STW，效率极低。
    - 启动STW -> Mark标记 -> 停止STW -> Sweep清除
- GoV1.5- 三色标记法， 堆空间启动写屏障，栈空间不启动，全部扫描之后，需要重新扫描一次栈(需要STW)，效率普通
    - 插入写屏障：堆空间节点下插入节点，新加的节点标记为灰色。
    - 删除写屏障：被删除的对象为灰色或者白色，那么将其标记为灰色。
- 插入写屏障和删除写屏障的短板：
    - 插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
    - 删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。
- GoV1.8-三色标记法，混合写屏障机制，栈空间不启动，堆空间启动。整个过程几乎不需要STW，效率较高。
    - 1、GC开始：扫描栈区，将可达对象全部标记为黑(之后不再进行第二次重复扫描，无需STW)，
    - 2、GC期间，任何在栈上创建的新对象，均为黑色。
    - 3、被删除的对象标记为灰色。
    - 4、被添加的对象标记为灰色。

## golang面试题

Go 语言笔试面试题(基础语法)
https://geektutu.com/post/qa-golang-1.html

Go 语言笔试面试题(实现原理)
https://geektutu.com/post/qa-golang-2.html

Go 语言笔试面试题(并发编程)
https://geektutu.com/post/qa-golang-3.html

Go 语言笔试面试题(代码输出)
https://geektutu.com/post/qa-golang-c1.html