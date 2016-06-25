---
layout: post
title: "Solr和Sunspot全文索引使用方法和一般问题解决办法"
date: 2014-02-16 01:20
comments: true
categories: Rails Ruby
keywords: solr, sunspot, rails, ruby, 全文索引, 分词, 全文搜索, 问题
description: 最近做的一个项目有全文搜索的需求，而Solr经过多年的发展，已经有完善的文档和社区支持，所以我就尝试用它作为搜索引擎。
---

最近做的一个项目有全文搜索的需求，而 Solr 经过多年的发展，已经有完善的文档和社区支持，所以我就尝试用它作为搜索引擎。因为使用过程中遇到了几个坑，因此想记录下来，同时也希望这篇文章能帮到遇上同样问题的人:]

 Solr && Sunspot
Solr 是一个开源的搜索服务器，Solr 使用 Java 语言开发，主要基于 HTTP 和 Apache Lucene 而实现。定制 Solr 索引的实现方法很简单，用 POST 方法向 Solr 服务器发送一个描述所有 Field 及其内容的 XML 文档就可以了。定制搜索的时候只需要发送 HTTP 的 GET 请求即可。

而在 Ruby 下使用 Sunspot 这个 Gem 就足够了，因为里面已经封装了 Solr 的底层操作，提供简单而强大的索引和查找对象功能，同时也支持各种 ORM。

---

## 安装

首先，如上所述，要运行 Solr，你要先有 Java 运行环境，还没安装的话就请移步到 Google 搜索安装方法，这里就不多说了。

毫无疑问，要在 Gemfile 中添加：

``` ruby
gem 'sunspot_rails'
gem 'sunspot_solr' # optional pre-packaged Solr distribution for use in development
```

然后毫无疑问是 Bundle：

``` ruby
bundle install
```

接着生成配置文件`config/sunspot.yml`：

``` ruby
rails generate sunspot_rails:install
```

如果安装 ```sunspot_solr``` 了，那么就运行它：

``` ruby
bundle exec rake sunspot:solr:start
```

至此，Solr就应该能跑起来了。

至于要关闭它，也只要把 ```start``` 换成 ```stop``` 即可：

``` ruby
bundle exec rake sunspot:solr:start
```

---

## 对象设置

对象设置也和安装一样简单，只要在 model 里添加一个 ```searchable``` 的 block 指明需要索引的字段和查询范围即可，例如：

``` ruby
class School < ActiveRecord::Base
  searchable do
    text :name
    
    time :found_at
  end
  
  attr_accessible :name
end
```

那么被 ```text``` 声明的字段就会作为全文索引，至于 ```time``` 或其它 Sunspot 可以使用的类型（boolean, integer, string等）就会作为查询范围时使用。

---

## 对象查找

继续用上面的 model 作为例子，当我们需要查找校名，那么可以设定这样一个方法：

``` ruby
def search_school(school_name)
  @schools = School.search do
    fulltext school_name
  end.results
  
  return @schools
end
```

很好，这就完成啦！```fulltext``` 后面跟着的就是搜索的关键词，查询的结果通过 ```#results``` 方法取出。

---

## Reindex

当我们在 School 中添加或删除记录时，sunpost 自动就会帮我们重新 reindex，但如果我们需要添加新的字段或不得不需要手动 reindex 时，可以使用如下指令：

``` ruby
rake sunspot:redinex
```

---

## 几个开发中遇到的问题

* __中文搜索__

如果想让 Solr 支持中文也十分简单，只要将设定中的 tokenizer 替换成你想使用的分词系统即可，而 Solr 内建简单的 CJK 分词系统，只要将 ```solr/conf/schema.xml``` 大概第64行换成：

``` ruby
<tokenizer class="solr.CJKTokenizerFactory"/>
```

然后重启 Solr 并且 reindex 应该就能使用了。

* __undefined method `results' for #<MetaSearch::Searches ...__

如果出现以下的问题，

``` ruby
undefined method `results' for #<MetaSearch::Searches::School:0x007fda483ef128>
```

那可能是因为你同时安装了 MetaSearch 或像 ActiveAdmin 这些用了 MetaSearch 的 Gem，而和 Solr 发生了冲突，解决办法是把搜索方法改成这样：

``` ruby
def search_school(school_name)
  @schools = School.solr_search do
    fulltext school_name
  end.results
  
  return @schools
end
```

或这样：

``` ruby
def search_school(school_name)
  @schools = Sunspot.search(School) do
    fulltext school_name
  end.results
  
  return @schools
end
```

* __Request Data: "<?xml version=\"1.0\" encoding=\"UTF-8\"?>type:Match" ...__

而如果在 production 中出现 404 xxx

``` ruby
rake sunspot:reindex
rake aborted!
RSolr::Error::Http - 404 Not Found
Error: Not Found

Request Data: "<?xml version=\"1.0\" encoding=\"UTF-8\"?>type:Match" ...
```

可以先尝试修改 ```config/sunspot.yml``` 中的：

``` ruby
production:
   solr:
...
      path: /solr/production
```

改为：

``` ruby
 production:
   solr:
...
      path: /solr/default
```

---

## 相关链接

* [Sunspot in Github](https://github.com/sunspot/sunspot)
* [Lucene的分词原理与分词系统](http://wwangcg.iteye.com/blog/1327670)

