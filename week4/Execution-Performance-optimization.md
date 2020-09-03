<!--这节课里面的内容东西过一遍就知道，没有基础的话短时间很难接受，直接开始艹作业。-->

## 课程要求    
选择issues pick-up 一个课题 7天内完成 600～3000～6000分数不等。
## 课程链接     
https://www.bilibili.com/video/BV18K4y1e7Ln
## 提供的issues选择    
|issues|number|point|link|
|-|-|-|-|
|Call For Participation: add memory trace for all the AggFuncs|#19369|600|[link](https://github.com/pingcap/tidb/issues/19369)|
|Incompatible issues about scalar functions(help wanted)|#11223|600|[link](https://github.com/pingcap/tidb/issues/11223)|
|help wanted||600|[link](https://github.com/pingcap/tidb/issues?q=is%3Aissue+is%3Aopen+help+wanted+label%3Astatus%2Fhelp-wanted)|
|Improve the performance of `WindowExec` by using sliding window or segment tree|#12967|3000|[link](https://github.com/pingcap/tidb/issues/12967)|
|Support inline projection for executors|#14428|3000|[link](https://github.com/pingcap/tidb/issues/14428)|

## 我的pick-up
|issues|number|point|link|
|-|-|-|-|
|Call For Participation: add memory trace for count agg functions|19734|600|[link]((https://github.com/pingcap/tidb/issues/19734))|

## 这个issues是解决什么问题
>This issue is used to track the memory usage of AggFunc in HashAggExec as discussed in [#14705](https://github.com/pingcap/tidb/issues/14705) and [#14103](https://github.com/pingcap/tidb/issues/14103).
We need to implement the method of the AggFunc interface and return memDelta to identify memory usage of each function.

## 已经完成的3个子项可以参考
[executor: implement `memDelta` for avg funcs to track memUsage #18901](https://github.com/pingcap/tidb/pull/18901 )    
[executor: Implement memDelta for sum functions #19376](https://github.com/pingcap/tidb/pull/19376)

## 我的issues pr
https://github.com/pingcap/tidb/pull/19770

## 出现的一些问题
1 fmt
```
git commit -m "fix stdlib "unsafe" need be group together and before non-stdlib group in ../../executor/aggfuncs/func_count.go"
```
2 对golang memory的理解
- alloc阶段会new变量产生内存消耗
- update指针引用不记内存消耗
- 在方法中如distinct需要类似hash set做存储则产生内存消耗
- 返回新的变量会产生内存消耗

## 课程提供的参考资料
[TiDB 源码阅读系列文章（十）Chunk 和执行框架简介](https://pingcap.com/blog-cn/tidb-source-code-reading-10/)
[TiDB 源码阅读系列文章（九）Hash Join](https://pingcap.com/blog-cn/tidb-source-code-reading-9/)
[TiDB 源码阅读系列文章（十五）Sort Merge Join](https://pingcap.com/blog-cn/tidb-source-code-reading-15/#tidb-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%8D%81%E4%BA%94sort-merge-join)
[TiDB 源码阅读系列文章（二十二）Hash Aggregation](https://pingcap.com/blog-cn/tidb-source-code-reading-22/#tidb-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E4%BA%8C%E5%8D%81%E4%BA%8Chash-aggregation)
[Hitting a 10x Performance forExpression Evaluation with VectorizedExecution and Community’s Help）](https://docs.qq.com/pdf/DSkNaZW9hTXNqWXVH)







