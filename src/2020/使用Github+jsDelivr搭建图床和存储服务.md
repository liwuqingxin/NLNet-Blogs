# 使用元素

<tag>我的博客</tag><cnt>[NLNet](https://www.cnblogs.com/liwuqingxin/)</cnt> 并未搭建自己的博客，使用博客园（cnblogs），自定义了主题[NLNet-Theme](https://github.com/liwuqingxin/NLNet-Themes)。

<tag>写作工具</tag><cnt>[Typora](https://www.typora.io/)</cnt> 优秀的Markdown编辑器。参考[NLNet-Theme](https://github.com/liwuqingxin/NLNet-Themes)，我会不定期持续更新自定义的Typora的主题[nlnet](https://github.com/liwuqingxin/NLNet-Themes)。

<tag>网络图床</tag><cnt>[Github](https://github.com/liwuqingxin/nlnet-blogs)</cnt> 考虑到图床的长久稳定性和钱包，以及我的博客的可怜的访问量，我将图片托管在了Github上。

<tag>图片上传</tag><cnt>[VSCode](https://code.visualstudio.com/)</cnt> 我并没有使用热门的PicGo工具，个人感觉用起来别扭。

我的用法是：使用本地图片写Markdown，完成后用我自己的正则替换工具，将图片本地url替换成网络url。最后将Markdown文件和本地图片一起push到Github上，任务完成。

<tag>CDN</tag><cnt><img src="https://cdn.jsdelivr.net/www.jsdelivr.com/2e32e3fa6bdf98d47bd71e78b3b9b051e3119f8f/img/logo-horizontal.svg" style="height:24px;margin:0 0 4px 0;vertical-align: middle;"/></cnt> 没有梯子的小伙伴访问Github的速度简直不能忍受，使用JSDELIVR作为cdn，cdn访问格式：

```
格式：https://cdn.jsdelivr.net/gh/你的用户名/你的仓库名@发布的版本号或者主分支名/文件路径
示例：https://cdn.jsdelivr.net/gh/liwuqingxin/NLNet-Themes@main/src/cnblogs/nlnet.js
更新：https://purge.jsdelivr.net/gh/liwuqingxin/NLNet-Themes@main/src/cnblogs/nlnet.js
```

<tag>主题文件</tag><cnt>aliyun.oss</cnt> 收费产品不给链接了。由于自己近期会频繁更新主题，JSDELIVR刷新CDN不方便，因此把主题相关的文件放到了阿里云的oss上，保存+拖放+F5，即可看到最新的主题效果，赞<i id="smile" class="icon" size="32"> </i>。