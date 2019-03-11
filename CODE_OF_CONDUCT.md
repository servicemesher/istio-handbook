# 本书写作规范

本文中指定了本书的写作规范与注意事项。

## 文档组织规则

- 所有文档使用 markdown 格式撰写；
- 文中的图片直接使用**新浪图床**保存，不用保存在GitHub 上；
- 图片必须给出标题；
- 所有引用的文章必须在文章底部的参考中给出链接，格式如 [title - domain.com]()；
- 所有代码需要指明语言；

## 流程

1. 首先加入到 [ServiceMesher](https://github.com/servicemesher) 组织中，请联系 [Jimmy Song](https://jimmysong.io/about) 加入；
2. 在 [Issues](https://github.com/servicemesher/getting-started-with-knative/issues) 中创建你想要参与的章节（issue 标题为文章路径，内容填写标题和摘要）；
3. 一次最多同时认领或提交 3 个 PR；
4. 由 owner 审核后 merge 进 master 分支；
5. 每篇文章头部的 header 请注意填写；
6. 每周将使用 GitHub pages 发布一次；

## Header

每篇文章的头部都有一个使用 YAML 格式的 header，请在每次提交 PR 的时候填写，示例：

```yaml
owners: ["rootsongjc","loverto"]
reviewers: ["fleeto","mathlsj"...]
description: "文章摘要。"
publishDate: 2019-03-02
updateDate: 2019-03-10
tags: ["tagA","tagB"...]
category: "translation|original|evolution"
```

**说明**

- owners：原则上不超过2人；
- reviewers：只要修改过文章的人都是 reviewer；
- description：本文章的摘要，不要超过200字；
- publishDate：文章第一次合并的日期；
- updateDate：最新一次修改的日期；
- tags：文章中出现的关键词；
- category：translation（翻译的文章），original（原创文章），evolution（在翻译文章的基础上重新演绎，对原文有大量更新）；

## 排版规范

- 所有的英文跟中文之间要有一个空格；
- 参阅 [Istio 网站样式指南](https://istio.io/zh/about/contribute/style-guide/)；
- 专有名词中文译名参阅 <https://github.com/servicemesher/glossary>；
