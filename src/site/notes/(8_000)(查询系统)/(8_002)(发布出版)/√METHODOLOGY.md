---
{"dg-publish":true,"permalink":"/8-000/8-002/methodology/"}
---


## 数据质量工程
+ 数据异常定位：<strong><font color=#70f3ff>如果XX指标发生了波动（上升或下降），需要你去定位原因，你的分析思路是什么？</font></strong>
    + 通过<strong><font color=#FF4500>数据异常检测</font></strong>确认业务所说的波动是否属于异常波动；
    + <strong><font color=#FF4500>根据外部因素和内部因素分别进行排查</font></strong>；其中外部采用PEST分析法，内部因素按照数据生产关系分为生产者、参与者、加工者，在对每个层级分别排查定位问题。
        + ![外部因素定位](http://mmbiz.qpic.cn/mmbiz_png/icHOSb47jqpWt6Q8KIlW3hxro7XXoRDJd9Pxe9Ld70EVnOntbNnQQZbawdCVMcEK3icJj8TaLqIQ1FbKwgicSCpXg/640?mprfK=http%3A%2F%2Fmp.weixin.qq.com%2F)
        + ![内部因素定位](http://mmbiz.qpic.cn/mmbiz_png/icHOSb47jqpWt6Q8KIlW3hxro7XXoRDJd0zUib56YZzwvo1HRMnyS7Pc5r8vcmZM2E9wlKDL06XeUgzgD2LB2dSg/640?mprfK=http%3A%2F%2Fmp.weixin.qq.com%2F)
    + 用AB实验的思想进行<strong><font color=#FF4500>数据异常归因</font></strong>。
    + ![思路总结](http://mmbiz.qpic.cn/mmbiz_png/icHOSb47jqpWt6Q8KIlW3hxro7XXoRDJdt7Wf4B0btswmlk3je8596oOHdaqFiac5CEcdCIhWCACQvIricvvKfa3Q/640?mprfK=http%3A%2F%2Fmp.weixin.qq.com%2F)






















