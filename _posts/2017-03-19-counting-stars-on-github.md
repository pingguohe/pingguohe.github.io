---

layout: post
title:  揭秘 Github 上那些开源项目的 star 数
author: Longerian

---

> 声明：转载请注明出处。

前段时间团队内开源了两个小项目：[vlayout]和[LazyScrollView]，一般我们可以通过它们的 star 数（即被其他人收藏的数量）来了解它们的受欢迎程度。但是一个项目到底集齐多少个 star 才算受欢迎呢？Github 上那些开源项目的 star 分布是怎样的？......只有了解这些才能更好的评估一个项目的流行程度。好在 Github 有开放第三方接口，提供了足够多的有用信息，我稍微整理了一下，感觉有点意思，分享一下。（以下数据统计于2017年3月18日）

# 8,316,622

搜索接口返回的数据显示，Github上共有**8316622**个开源项目，其中star 数在0-100之间的占比**99.03%**，star 数在100-1000之间的占比**0.85%**，star 数在0-1000之间总共占比**99.88%**。

> 项目数量达八百多万，还是非常惊人的，代码平铺起来都不知道能绕地球多少圈。
> 
> star 数处于0-100之间的项目占比之高，基本上是由于很多同行拿 Github 做个人代码的仓库，这些项目严格来说算不上开源项目，像我，就贡献了好多这种仓库。
> 
> 但是反过来，**如果你的一个项目集齐超过1000个 star，那么也就意味着在这方面你已经打败了地球上99.88%的人**。

# 80,495

我们假设 star 数大于100的项目算是比较有价值的项目，那么这类项目总共有**80495**个。其中的**约88%**分布在100-1000之间，剩下的1000-2000，2000-3000，3000-4000...等分布占比大概按7%，2%，1%...依次减少。

![](/images/github_stat/starsgt100.png)


# 245,775

收获 star 最多的一个项目是**[freeCodeCamp]**的项目，它是一个类似于编程训练营的组织，帮助普通人0基础学习编程。它总共收获了**245775**个 star，比排在第二名的107979高出了一倍多。

> 大概剩下那八百多万项目的 star 总和也比不过它吧，有种一览众山小，高处不胜寒的感觉。

# TOP 50

star 数超过2000的项目总共有**4550**个，可以看一下 TOP 50都是些什么项目：（搜索接口返回的 watchers 值与 stars 的值相同，并不是 github 项目主页上看到的那个 watching 数值）

| name                        | owner         | stars  | watchers | language     | forks |
|-----------------------------|---------------|--------|-----------------|--------------|-------|
| freeCodeCamp                | freeCodeCamp  | 245775 | 245775          | JavaScript   | 10160 |
| bootstrap                   | twbs          | 107979 | 107979          | JavaScript   | 49463 |
| free-programming-books      | vhf           | 79925  | 79925           | None         | 20018 |
| react                       | facebook      | 62156  | 62156           | JavaScript   | 11443 |
| d3                          | d3            | 61687  | 61687           | JavaScript   | 16378 |
| angular.js                  | angular       | 55097  | 55097           | JavaScript   | 27494 |
| awesome                     | sindresorhus  | 54791  | 54791           | None         | 6773  |
| You-Dont-Know-JS            | getify        | 53500  | 53500           | JavaScript   | 8501  |
| tensorflow                  | tensorflow    | 51073  | 51073           | C++          | 23884 |
| oh-my-zsh                   | robbyrussell  | 50744  | 50744           | Shell        | 12064 |
| javascript                  | airbnb        | 49141  | 49141           | JavaScript   | 9444  |
| Font-Awesome                | FortAwesome   | 49058  | 49058           | HTML         | 8543  |
| gitignore                   | github        | 47754  | 47754           | None         | 20034 |
| vue                         | vuejs         | 47121  | 47121           | JavaScript   | 6108  |
| react-native                | facebook      | 45801  | 45801           | JavaScript   | 10567 |
| jquery                      | jquery        | 43891  | 43891           | JavaScript   | 12269 |
| electron                    | electron      | 43174  | 43174           | C++          | 5219  |
| linux                       | torvalds      | 43032  | 43032           | C            | 16384 |
| docker                      | docker        | 40776  | 40776           | Go           | 12288 |
| animate.css                 | daneden       | 40126  | 40126           | CSS          | 9024  |
| coding-interview-university | jwasham       | 37563  | 37563           | None         | 8401  |
| swift                       | apple         | 37315  | 37315           | C++          | 5518  |
| meteor                      | meteor        | 36896  | 36896           | JavaScript   | 4632  |
| html5-boilerplate           | h5bp          | 36839  | 36839           | JavaScript   | 9110  |
| node-v0.x-archive           | nodejs        | 36547  | 36547           | None         | 8194  |
| atom                        | atom          | 35543  | 35543           | CoffeeScript | 6174  |
| rails                       | rails         | 34928  | 34928           | Ruby         | 14249 |
| reveal.js                   | hakimel       | 33418  | 33418           | JavaScript   | 10126 |
| node                        | nodejs        | 32865  | 32865           | JavaScript   | 6190  |
| Semantic-UI                 | Semantic-Org  | 32159  | 32159           | JavaScript   | 3618  |
| three.js                    | mrdoob        | 31521  | 31521           | JavaScript   | 11179 |
| socket.io                   | socketio      | 31243  | 31243           | JavaScript   | 5952  |
| impress.js                  | impress       | 31085  | 31085           | JavaScript   | 6544  |
| nw.js                       | nwjs          | 30999  | 30999           | C++          | 3497  |
| awesome-python              | vinta         | 30689  | 30689           | Python       | 5834  |
| express                     | expressjs     | 30634  | 30634           | JavaScript   | 5607  |
| laravel                     | laravel       | 30591  | 30591           | PHP          | 10052 |
| moment                      | moment        | 30390  | 30390           | JavaScript   | 4376  |
| the-art-of-command-line     | jlevy         | 30002  | 30002           | None         | 2881  |
| legacy-homebrew             | Homebrew      | 29192  | 29192           | Ruby         | 13693 |
| redux                       | reactjs       | 29142  | 29142           | JavaScript   | 5605  |
| jekyll                      | jekyll        | 29066  | 29066           | Ruby         | 6471  |
| Chart.js                    | chartjs       | 28740  | 28740           | JavaScript   | 7537  |
| httpie                      | jakubroztocil | 28704  | 28704           | Python       | 1897  |
| AFNetworking                | AFNetworking  | 28665  | 28665           | Objective-C  | 9063  |
| material-design-icons       | google        | 28607  | 28607           | CSS          | 5479  |
| ionic                       | driftyco      | 28504  | 28504           | TypeScript   | 7028  |
| brackets                    | adobe         | 27085  | 27085           | JavaScript   | 5803  |
| resume.github.com           | resume        | 27080  | 27080           | JavaScript   | 711   |
| hacker-scripts              | NARKOZ        | 26633  | 26633           | JavaScript   | 4876  |

> 满屏都是 JS 项目；Facebook 有2个项目上榜，Google 也有两个，国内的项目还没看到上榜的。

# 按编程语言汇总

在前4550个项目里，汇总了一批主流的语言。

![](/images/github_stat/classify_by_language.png)

![](/images/github_stat/classify_by_language_histogram.png)

> JS 项目 仍然是占绝大多数，近几年前端的火爆程度也由此可见一斑。其次才是 JAVA、Python、OC 之类的项目。

# 按公司汇总

在前4550个项目里，挑几个国内外的开源大户来看看吧，其实这里有个人，有商业公司，也有正经的开源基金会。

![](/images/github_stat/classify_by_owner.png)

![](/images/github_stat/classify_by_owner_histogram.png)

> 可以看到国外的机构里， Google，Facebook 等都是开源大户，连微软也是积极拥抱开源的。国内方面，阿里也是贡献较多的，其实腾讯也有几个项目排名在前列，但是数量较少，之前他们在国内 csdn 上也有托管过源代码，还有一些可能不是以腾讯公司名义托管的，那就没有统计在其中。

# 数据源
最后献上几段流水账脚本[github_stat]，有兴趣的可以拿去挖掘更多数据。

# What's more
接下来一段时间内我们团队也会继续推出新的开源项目，敬请关注苹果核。

[vlayout]:<https://github.com/alibaba/vlayout>
[lazyScrollView]:<https://github.com/alibaba/lazyscrollview>
[freeCodeCamp]:<https://www.freecodecamp.com/>
[github_stat]:<https://github.com/longerian/github_stat>
