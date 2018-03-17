---

layout: post
title:  Github 上那些开源项目的 star 数
author: Longerian

---

掐指一算，一年时间过去了，去年的这个时候，我还捞了一下 github 上的开源项目数据，分析了一下 github 上的项目分布、受欢迎程度等，还是由一些小小的有意思的发现（[原文在此](http://pingguohe.net/2017/03/19/counting-stars-on-github.html)）。前几天看到[《GitHub预测2018年开源项目趋势》](http://www.infoq.com/cn/articles/open-source-project-trends-for-2018)一文，感觉是时候简单回顾下这一年来开源项目的变化了。

# 39,919,570 与 110,512
+ 搜索接口返回的数据显示，Github上共有 **39,919,570** 个开源项目，当然这里面大部分是个人存储的代码仓库，如果把 star 数大于 100 的的项目认为是真正的开源项目，那么有 **110,512** 个，较去年增长了 **37%**。

![](https://gw.alicdn.com/tfs/TB1nJNxdntYBeNjy1XdXXXXyVXa-852-874.png)

# 291,714

收获 star 最多的一个项目仍然是 **[freeCodeCamp]** 的项目，比上年增长 **18%** 左右，从这个数据也反应出又有不少新人入行学习编程，计算机仍然是当今最热门的就业领域之一。

# TOP 50

star 数超过 2000 的项目总共增长到了 **6688** 个，比去年增长 **47%**，同样可以看一下 TOP 50都是哪些项目：

| name                        | owner           | stars  | language   | forks |
|-----------------------------|-----------------|--------|------------|-------|
| freeCodeCamp                | freeCodeCamp    | 291714 | JavaScript | 13576 |
| bootstrap                   | twbs            | 122600 | CSS        | 58131 |
| free-programming-books      | EbookFoundation | 102321 | None       | 26000 |
| tensorflow                  | tensorflow      | 92090  | C++        | 59339 |
| react                       | facebook        | 90293  | JavaScript | 17053 |
| vue                         | vuejs           | 86275  | JavaScript | 12638 |
| awesome                     | sindresorhus    | 79962  | None       | 10511 |
| You-Dont-Know-JS            | getify          | 77256  | None       | 14026 |
| d3                          | d3              | 73206  | JavaScript | 18874 |
| javascript                  | airbnb          | 67512  | JavaScript | 12907 |
| oh-my-zsh                   | robbyrussell    | 66957  | Shell      | 14202 |
| gitignore                   | github          | 62669  | None       | 28530 |
| react-native                | facebook        | 60960  | JavaScript | 13960 |
| coding-interview-university | jwasham         | 58934  | None       | 16104 |
| angular.js                  | angular         | 58125  | JavaScript | 28840 |
| electron                    | electron        | 57573  | C++        | 7524  |
| linux                       | torvalds        | 56216  | C          | 20709 |
| Font-Awesome                | FortAwesome     | 55287  | CSS        | 9542  |
| animate.css                 | daneden         | 49723  | CSS        | 10739 |
| jquery                      | jquery          | 48267  | JavaScript | 14670 |
| moby                        | moby            | 47961  | Go         | 14143 |
| awesome-python              | vinta           | 46590  | Python     | 9021  |
| node                        | nodejs          | 46291  | JavaScript | 9706  |
| vscode                      | Microsoft       | 45704  | TypeScript | 6178  |
| create-react-app            | facebook        | 44825  | JavaScript | 8776  |
| atom                        | atom            | 43863  | JavaScript | 8757  |
| developer-roadmap           | kamranahmedse   | 43710  | None       | 6111  |
| swift                       | apple           | 42921  | C++        | 6760  |
| laravel                     | laravel         | 40977  | PHP        | 12797 |
| Semantic-UI                 | Semantic-Org    | 40003  | JavaScript | 4329  |
| html5-boilerplate           | h5bp            | 39963  | JavaScript | 9592  |
| three.js                    | mrdoob          | 39919  | JavaScript | 14853 |
| socket.io                   | socketio        | 39796  | JavaScript | 7506  |
| meteor                      | meteor          | 39379  | JavaScript | 4965  |
| reveal.js                   | hakimel         | 39266  | JavaScript | 11652 |
| rails                       | rails           | 38916  | Ruby       | 15748 |
| redux                       | reactjs         | 38875  | JavaScript | 9178  |
| go                          | golang          | 38843  | Go         | 5270  |
| webpack                     | webpack         | 38499  | JavaScript | 4802  |
| axios                       | axios           | 37561  | JavaScript | 2619  |
| express                     | expressjs       | 37084  | JavaScript | 6635  |
| node-v0.x-archive           | nodejs          | 35958  | None       | 7964  |
| moment                      | moment          | 35867  | JavaScript | 5360  |
| Chart.js                    | chartjs         | 35627  | JavaScript | 8559  |
| resume.github.com           | resume          | 35150  | JavaScript | 924   |
| youtube-dl                  | rg3             | 34679  | Python     | 6370  |
| httpie                      | jakubroztocil   | 34379  | Python     | 2331  |
| thefuck                     | nvbn            | 34183  | Python     | 1693  |
| public-apis                 | toddmotto       | 34153  | Python     | 3206  |
| the-art-of-command-line     | jlevy           | 34000  | None       | 3385  |

有几个值得注意的点：JavaScript 类项目仍然是非常受欢迎；**TensorFlow** 上升最快，说明人工智能领域仍然是全球范围内最受关注的领域，虽然 2017 年被誉为区块链元年，在资本市场区块链项目也大受欢迎，但还没有迎来真正爆发式增长的阶段。我个人觉得相比之下，在应用场景中，人工智能能给终端消费者带来直接的体验提升，而区块链强调的去中心化可能是一种美好的想象，在有竞争机制、计算力不平等的情况下很难做到平等竞争，必然会产生新的局部中心，倒是不可窜改这一点的应用值得期待。最后跨平台的开发技术也受欢迎，像 react、electron 相关项目；另外有几个 Python 类项目上升也很快，个人不太熟悉。

# 按编程语言汇总

![](https://gw.alicdn.com/tfs/TB1oK8GdntYBeNjy1XdXXXXyVXa-1068-1066.png)

不用看，其实也是能猜个大概分布。

# 按作者汇总

![](https://gw.alicdn.com/tfs/TB17tBJdntYBeNjy1XdXXXXyVXa-1290-1108.png)

较去年比较，可以看到，变化最大的是**腾讯**，**腾讯**在过去一年里大力推进了开源工作，新开放了大量优秀的项目。

# 我们的开源项目

我们团队在 2017 年也推出过自己的开源项目 Tangram 系列，它是用来做页面结构动态化和组件动态化的一个方案。其中 vlayout 项目，受到了不小的欢迎，虽然没有排上上述榜单，但在整个大盘里也属于靠前的。

### Android

+ [Tangram-Android](https://github.com/alibaba/Tangram-Android)
+ [Virtualview-Android](https://github.com/alibaba/Virtualview-Android)
+ [vlayout](https://github.com/alibaba/vlayout)
+ [UltraViewPager](https://github.com/alibaba/UltraViewPager)

### iOS

+ [Tangram-iOS](https://github.com/alibaba/Tangram-iOS)
+ [Virtualview-iOS](https://github.com/alibaba/VirtualView-iOS)
+ [LazyScrollView](https://github.com/alibaba/lazyscrollview)

# 数据说明

本文的数据通过 github 接口抓取，统计截止 2018-03-11。
