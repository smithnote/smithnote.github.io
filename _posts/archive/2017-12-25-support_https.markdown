---
layout: archive
title: github page 支持https
date: 2017-12-25 10:00
categories: tool
tag: https

---

在github上搭建了github page博客后，默认如果不自定义域名的话，可以[直接支持https](https://help.github.com/articles/securing-your-github-pages-site-with-https/)，一键设置后
便可生效，但是如果自己有自定义域名绑定后,github就不能直接支持了，这个时候就需要另寻它法了，
至于为什么要让自己网站支持https的原因，主要是为了在手机上浏览时不被运营上劫持。如下图
中的红色圆圈部分就是http被运营商劫持后的添加的推广信息。
<center>
<img src="/assets/images/support_https-http_hack.png" style="width:50%; height:50%;">
</center>
我们可以借助cloudflare的支持来进行对https的支持，详细步骤如下：  
1. 注册[cloudflare](https://www.cloudflare.com/a/login)
2. cloudflare setup添加自己的域名扫描。continue
3. 添加dns解析。我查资料的时候都是说直接出来dns解析信息，而我使用的时候并没有所以我就自行添
dns解析记录，添加后的结果如图![](/assets/images/support_https-dns_record.png)
4. 选择套餐，当然是"Free Website"啦. continue
5. 修改nameserver，以我使用万网为例， 如图，我是直接修改好了，就不再演示重原有的dns改成
cloudflare提供的dns了 ![](/assets/images/support_https-aliyun_change_nameserver.png)
6. 选择ssl选项flexible ![](/assets/images/support_https-ssl_flexible.png)
7. 添加page rule, 这样的话，当别人没有输入https的时候，自动装换为https链接访问
![](/assets/images/support_https-create_page_rule.png)
8. (选项) 对于我是使用[Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/)
主题的，所以当http转换为https的时候，还要修改_config.yml文件中的**url : "http://smithnote.com"** 修改成 **url : "https://smithnote.com"**.

至此，你的github page就可以正式支持https了

**注**: 如果你使用有google console支持网站优化，你必须要将https://yoursite.com添加进去，
因为goolge认为http://yoursite.com和https://yoursite.com是两个不同的网站
