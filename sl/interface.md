
- sort.Sort(interface Interface), 需要实现Len, Less, Swap三个方法, 比如想对[][]int排序, 可以type重定义一个新类型, 实现三个方法, 对其进行排序