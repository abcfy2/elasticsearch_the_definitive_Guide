# 聚合
本章主要介绍ES的聚合。

直到现在,我们只是用ES来做搜索或查询。就像有时候想要在一个子集中找出符合条件的结果，这就像在草堆中寻找针！

有了聚合，我们对数据有了一个概览。取代搜索单独的文档，我们想要分析和总结我们的数据集：
- How many needles are in the haystack?
- What is the average length of the needles?
- What is the median length of the needles, broken down by manufacturer?
- How many needles were added to the haystack each month?

聚合也可以回答更微妙的问题：

- What are your most popular needle manufacturers?
- Are there any unusual or anomalous clumps of needles?
- Aggregations allow us to ask sophisticated
