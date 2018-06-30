---
title: UITableView使用指南
date: 2018-02-04 19:55:27
tags: iOS
categories: iOS

---
本人正式从做安卓的转成做iOS的了……好久没有写博客。iOS和Android开发对比起来呢，主要有几点的不同：

- Android Studio比Xcode不知道好用了多少倍……用后者编码效率不知道下降了多少倍，可能现在还是不熟练吧，坐等App Code推广成为主流IDE。
- iOS的UI不知道比Android高到哪里去了，可能我写上10年也写不出iOS的UI库……
- iOS出了Bug很难修，因为看不到源码，还有就是更新的手段比较绝……

好了不扯淡了，现在进入主题。

<!--more-->

本文不打算介绍它的基本使用，我只想说明几个非常蛋疼的问题，而且现在网上没有我满意的解决的方法。

# UITableView reloadData方法/insertRows方法调用之后bounds和contentOffset突变
解决这个Bug花费了我一天的时间，最后的原因是设置了**rowHeight**这个属性。这个属性官方说的是为了性能优化，不必每次调用**tableView(_:heightForRowAt:)**，就可以设置这个属性，说白了就是一些静态的cell，静态不光是cell高度是不变的，其绑定的数据也不应该变，如果设置了这个属性而不去实现**tableView(_:heightForRowAt:)**, 那在reloadData方法调用完后整个TableView的bounds和contentOffset会改变，所以如果是为了展现变化的数据，实现上拉加载/下拉刷新，还是老老实实实现**tableView(_:heightForRowAt:)**方法，再去设置**estimatedRowHeight**属性吧。

# 自适应高度
老生常谈的问题，但还是要注意一点，就是数据源绑定的时机。网上大多数都在说在willDisplayCell方法当中实现数据绑定，但是在自适应高度的情况下，要在cellForRowAtIndexPath中实现数据绑定。因为UITableView对于自适应高度的计算是在cellForRowAtIndexPath和willDisplayCell两者之间的。也就是说如果在后者当中实现数据绑定，就不能改变cell的高度了。