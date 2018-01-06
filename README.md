# better-scroll V1.6.3源码阅读笔记

better-scroll 是一款重点解决移动端（未来可能会考虑 PC 端）各种滚动场景需求的插件。由于工作中需要使用better-scroll进行开发，然而better-scroll有着非常多的配置选项和可配置的功能，因此希望通过阅读它的源码，从而在开发中能够得心应手使用。

该笔记基于better-scroll V1.6.3版本。由于better-scroll一直在更新，可能会有部分与最新版本不同，但核心应该都是一样的。不影响大家理解better-scroll的整体实现过程。

在阅读源码之前，你最好已经使用过better-scroll，查看过better-scroll的相关文档，清楚better-scroll大致能做哪些事情。同时，better-scroll的相关demo代码都放在源码项目example目录下，通常情况下，通过阅读和模仿demo代码中的better-scroll使用方式，可以满足平时的大多数需求，因此推荐大家多看demo代码。当然，如果你在开发中遇到了问题或者对better-scroll的实现方式感兴趣，那么希望我的文章能帮助到你。文章主要关注better-scroll的关键实现过程，没有去扣细节，适合大多数初学者阅读。

提醒：better-scroll有很多配置项，阅读过程中如果有哪些配置项不清楚，直接去看[better-scroll文档](https://ustbhuangyi.github.io/better-scroll/doc/zh-hans/options.html#flicklimittime)

[better-scroll源码阅读（一）：初始化过程](https://github.com/tank0317/beter-scroll-source-code-reading/issues/1)

[better-scroll源码阅读（二）：滚动相关核心代码](https://github.com/tank0317/beter-scroll-source-code-reading/issues/2)

[better-scroll源码阅读（三）：上拉下拉与点击事件](https://github.com/tank0317/beter-scroll-source-code-reading/issues/3)

[better-scroll源码阅读（四）：轮播图slide相关代码](https://github.com/tank0317/beter-scroll-source-code-reading/issues/4)
