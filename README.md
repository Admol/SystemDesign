# 系统设计面试：内幕指南（中文翻译）

原名：《System Design Interview: An Insider’s Guide》

作者：Alex Xu

译者：精灵王 [@Admol](https://github.com/Admol)



## 目录
- [第 1 章：从零到数百万用户的规模](CHAPTER%201：SCALE%20FROM%20ZERO%20TO%20MILLIONS%20OF%20USERS.md)
- [第 2 章：粗略判断](CHAPTER%202：BACK-OF-THE-ENVELOPE%20ESTIMATION.md)
- [第 3 章：系统设计面试框架](CHAPTER%203：A%20FRAMEWORK%20FOR%20SYSTEM%20DESIGN%20INTERVIEWS.md)
- [第 4 章：设计一个分布式限流器](CHAPTER%204：DESIGN%20A%20RATE%20LIMITER.md)
- [第 5 章：一致性哈希设计](CHAPTER%205：DESIGN%20CONSISTENT%20HASHING.md)
- [第 6 章：设计一个 key-value 存储系统](CHAPTER%206：DESIGN%20A%20KEY-VALUE%20STORE.md)
- [第 7 章：设计一个分布式系统唯一ID生成器](CHAPTER%207：DESIGN%20A%20UNIQUE%20ID%20GENERATOR%20IN%20DISTRIBUTED%20SYSTEMS.md)
- [第 8 章：设计一个短网址系统](CHAPTER%208：DESIGN%20A%20URL%20SHORTENER.md)
- [第 9 章：设计一个网络爬虫系统](CHAPTER%209：DESIGN%20A%20WEB%20CRAWLER.md)
- [第 10章：设计一个通知系统](CHAPTER%2010：DESIGN%20A%20NOTIFICATION%20SYSTEM.md)
- [第 11章：设计一个新闻提要系统](CHAPTER%2011：DESIGN%20A%20NEWS%20FEED%20SYSTEM.md)
- [第 12章：设计一个聊天系统](CHAPTER%2012：DESIGN%20A%20CHAT%20SYSTEM.md)
- [第 13章：设计一个搜索自动完成系统](CHAPTER%2013：DESIGN%20A%20SEARCH%20AUTOCOMPLETE%20SYSTEM.md)
- [第 14章：设计 YouTube](CHAPTER%2014：DESIGN%20YOUTUBE.md)
- [第 15章：设计 Google Drive](CHAPTER%2015：DESIGN%20GOOGLE%20DRIVE.md)
---


# 声明
译者纯粹出于 **学习目的** 与 **个人兴趣** 翻译本书，不追求任何经济利益。

译者保留对此版本译文的署名权，其他权利以原作者和出版社的主张为准。

本译文只供学习研究参考之用，不得公开传播发行或用于商业用途。

有能力阅读英文书籍者请至[亚马逊](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)购买正版支持。
