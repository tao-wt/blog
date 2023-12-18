---
title: hexo fluid主题安装过程的一点记录
date: 2023-12-17 12:33:55
index_img: /img/index-3.png
tags:
  - fluid
  - theme
categories:
  - [hexo, theme, fluid]
author: Tao
excerpt: 记录fluid主题使用过程中碰到的两个问题, 个对应的改动和修复步骤.
---
### 访问Googletagmanager导致页面加载时间长
---
fluid主题的web_analytics配置中虽然设置了Google Tag Manager相关参数为空，但在实际运行时还是会尝试连接，导致页面一直处于加载资源的状态：
![错误截图](/img/googletagma-error.jpg)

经初步判断, 应该是js脚本`layout/_partials/plugins/analytics.ejs`中判断有点问题:
```js
<% if (theme.web_analytics.google){ %>
  <!-- Google tag (gtag.js) -->
  <script async>
    if (!Fluid.ctx.dnt) {
      Fluid.utils.createScript("https://www.googletagmanager.com/gtag/js?id=<%= theme.web_analytics.google.measurement_id %>", function() {
        window.dataLayer = window.dataLayer || [];
        function gtag() {
          dataLayer.push(arguments);
        }
        gtag('js', new Date());
        gtag('config', '<%- theme.web_analytics.google.measurement_id %>');
      });
    }
  </script>
<% } %>
```

解决办法: 注释掉`_config.fluid.yml`中`web_analytics`的`google_tag_manager`的`measurement_id`配置：
```yaml
web_analytics:
  ...
  # Google Analytics 4 的媒体资源 ID
  # Google Analytics 4 MEASUREMENT_ID
  # See: https://support.google.com/analytics/answer/9744165#zippy=%2Cin-this-article
  google:
  # measurement_id:
```

### 网站访问计数不增长的问题
---
经过分析发现是leancloud的Counter写权限的问题，默认情况下，Counter object的权限是只读的，需要修改为可写才能正常计数。
![修改ACL](/img/counter-acl.jpg)