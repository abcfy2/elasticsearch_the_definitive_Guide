# 项目简介
[《Elasticsearch权威指南》](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)一书的学习笔记。

本书按照[`GitBook`](https://www.gitbook.com/)格式编写，可以使用[`gitbook-cli`](https://github.com/GitbookIO/gitbook-cli)工具生成本地HTML预览。

即:
- `README.md`文件会被编译为对应目录下的`index.html`
- `/SUMMARY.md`文件会被编译为目录

## 本书约定
本书编写过程中遵循以下规范:
- 所有文件及路径统统使用小写ASCII字符，不得出现任何中文字符以及特殊字符(如`'`,`"`,`>`,`<`,`?`,`*`等有特殊含义的字符)
- 如有特殊字符，一律使用`_`字符替换。如标题为`I'm fine.`的文档，重命名为`i_m_fine.md`
- 单词之间使用`_`字符连接。如`getting_started`
- 目录层级关系与文件夹层级关系保持一致。如目录层级`chapter01/stage01/summary`对应的文件路径为`chapter01/stage01/summary.md`
- 如果层级本身有内容，应放入对应文件夹的`README.md`中。如`chapter01/stage01/`对应的文件路径为`chapter01/stage01/README.md`

举个例子。比如这样的目录结构:
```
$ tree .
.
├── 01_getting_started
│   ├── README.md
│   └── 01_you_know_for_search
│       ├── installing_and_running_elasticsearch.md
│       └── README.md
├── README.md
└── SUMMARY.md
```

那么对应的`SUMMARY.md`应为:

```markdown
* [Getting Started](01_getting_started/README.md)
    * [You know, For search](01_getting_started/01_you_know_for_search/README.md)
        * [Installing and Running Elasticsearch](01_getting_started/01_you_know_for_search/installing_and_running_elasticsearch.md)
```
