+++
title = "如何看待HashiCorp 产品禁止中国公司使用？"
date = 2020-06-05T00:00:00+08:00
tags = ["FreeSoftware", "OpenSource"]
categories = ["FreeSoftware", "OpenSource"]
draft = false
author = "7ym0n"
+++

![](/hashicorp/terms-of-evaluation.jpg)
国外知名 DevOps 服务商 HashiCorp 的官网[相关条款页面](https://www.hashicorp.com/terms-of-evaluation)明确声明禁止中国公司使用其 Vault 企业版产品。不过此次限制的是企业版，此事件一度引发国内开源界广泛关注。


## 对使用开源产品的担忧？ {#对使用开源产品的担忧}

  国内厂商对开源产品的使用是否需要担忧？其实还是有一定的担忧的，比如 HashiCorp 该公司的产品，属于开源软件而不是自由软件，这两者区别很大；如果不了解情况，请移步阅读我的另一篇，
[为什么需要了解自由软件，而不只是开源软件](https://www.scanbuf.net/post/why-you-need-to-know-freesoftware/)。目前来讲该公司的策略应该属于中美贸易、科技战的美国国家政策要求，所以，如果国内厂商在产品中使用大量的开源软件，那么在现在中美冲突越演越烈的情况下，并不能排除未来这些开源软件迫于国家政策要求进行闭源，如果真到这个时候，那么可以肯定的是，国内厂商将大面积受影响，当然存在一定的缓冲时间，互联网行业也就大概 1-2
年的时间，传统行业可能会更长一点，但是，如果真是这样，那么很多小公司将在这场冲突中成为牺牲品。


## 对未来开源软件持悲观态度 {#对未来开源软件持悲观态度}

  如果在世界政治环境没有变化的情况下，开源软件是未来发展趋势之一，并且有利于行业发展，但以目前全球化的经济衰退导致全球政治不稳定，中美冲突等，那么就有可能会受到政治影响，目前来看，已经有一些影响了。因为它的许可问题，如果在公司的手中，那么公司可能会因为迫于政治压力，而选择闭源。但愿各大开源软件社区不会受政治影响，否则就不是阵痛了。


## 自由软件的未来 {#自由软件的未来}

  推荐阅读[中国自由软件的未来](https://nalaginrut.com/archives/2019/09/21/%E4%B8%AD%E5%9B%BD%E8%87%AA%E7%94%B1%E8%BD%AF%E4%BB%B6%E7%9A%84%E6%9C%AA%E6%9D%A5)。很多人误解自由软件是免费软件或者反商业的软件，其实本质上自由软件根本没表现出反商业的情况（不然 redhat 早就没了），自由软件是保护你不会因为付费而影响到你的四项基本自由[^fn:1]。况且为了更好的商业行为还专门定制了一个 LGPL 的协议。


## 如何避免未来可预见性和灾难性的问题？ {#如何避免未来可预见性和灾难性的问题}

其实，有三条路选择:

-   第一、采用自由软件许可的产品替代，并积极参与维护。
-   第二、在第二选择，实在解决不了的情况下还是可选择由社区维护的开源产品替代。
-   第三、自主研发

大多数情况下，只有基础软件设施的通用性较强，如果跟业务相关，除了专业商业软件，大多数只有自主研发。比如数据库，在开源版本中，中小企业完全足够，但遇到特殊行业业务就可能满足不了需求。

  在这三条路中，如果厂商有实力可选择第三种，毕竟自己的东西随便外界怎么变化，都不会影响自己（除了技术落后）。第一种就是有一定的实力，但是没有资源做，不用担心闭源问题。第二种就是没办法的办法了，入坑太深，又没资源，又没实力。

[^fn:1]: <https://www.gnu.org/philosophy/free-sw.zh-cn.html>