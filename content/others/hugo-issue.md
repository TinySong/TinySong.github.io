---
 categories : ["hugo"]
 tags : ["hugo"]
 type : "post" 
 title : "Hugo Issue" 
 subtitle : "Hugo Issue" 
 date = "2017-12-12T22:24:59+08:00"
 draft = false 
 description= "hugo 中遇到的一些问题"

---

- [记录在使用 hugo 的一些问题](#org7270bd4)
  - [comments](#orgebee811)



<a id="org7270bd4"></a>

# 记录在使用 hugo 的一些问题


<a id="orgebee811"></a>

## comments

hugo 原生已支持 disqus comments，首选需要注册 disqus，配置时，只需要在 config.yaml 文件中添加一条记录： `disqusShortname: "myShortName"` 。 注册完 disqus 后，在 Gerneral 主页上有一个 `Shortname` 栏目，如： `Shortname Your website shortname is YourshortName.`, 注意 YourshortName 就是你 需要写入 config.yaml(or config.xx)文件中的。
