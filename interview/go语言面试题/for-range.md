# for-range
- 没必要把每一个问题都copy过来, 直接给个超链接不好么, 认知不必悉数留存于笔记, 元认知才是属于我的财富

## practice
- https://www.topgoer.cn/docs/gomianshiti/mian2
- for-range返回的i/v都是副本而非引用, 底层实现可以见Go专家编程里的range底层
- 用go build -gcflags '-m', 可以看出val被放到了堆上, 确实被外部引用了