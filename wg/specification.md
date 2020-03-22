# 写作规范

本文中指定了本书的写作规范与注意事项。

## 文档组织规则

- 所有文档使用 markdown 格式撰写；
- 文中的图片请保存在本书的 GitHub 上；
- 图片必须给出标题；
- 所有引用的文章必须在文章底部的参考中给出链接，格式如 `title - domain.com`；
- 所有代码需要指明语言；

## 流程

1. 首先加入到本书的[协作群](https://github.com/servicemesher/istio-handbook/issues/42)
2. 在 [Issues](https://github.com/servicemesher/getting-started-with-knative/issues) 中认领你想要参与的章节（issue 标题为文章路径，内容填写标题和摘要）；
3. 一次最多同时认领 3 个 issue；
4. 由[编委会](editorial-board.md)成员审核后 merge 进 master 分支；
6. 合并后会立即发布到 <https://www.servicemesher.com/istio-handbook> 上；

## Header

每篇文章的头部都有一个使用 YAML 格式的 header，请在每次提交 PR 的时候填写，示例：

```yaml
authors: ["rootsongjc","malphi"]
reviewers: ["gorda","mathlsj"...]
```

**说明**

- authors：GitHub 账号，本文的主要作者，可以为一到两人；
- reviewers：GitHub 账号，只要修改过文章的人都是 reviewer；

## 排版规范

- 所有的英文跟中文之间要有一个空格；
- 参阅 [Istio 网站样式指南](https://istio.io/zh/about/contribute/style-guide/)；
- 专有名词中文译名参阅 <https://github.com/servicemesher/glossary>；
- 所有代码文件需要在 Markdown 格式中指明代码语言；
- 如果文中引用了外部参考资料，需要将参考资料的标题和链接放到文末的「参考」中；

## 注意事项

1. 请注意引用内容的版权，需标明出处；
1. 请不要发布涉及关于国家统一、宗教自由、民族相关的内容；
1. 请不要使用人像图片；
1. 请不要使用过分口语化或俚语表达；
1. 请勿在正文中引用带有具体公司名称的案例或评论；

