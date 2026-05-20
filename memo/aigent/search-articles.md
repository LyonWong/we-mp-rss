> 我想通过API接口,搜索文章, 但是想增加一些过滤条件,比如文章的状态,标题中包含或者不包含某些关键词,我注意到有 api/v1/wx/articles 这个接口,
  也注意到数据库中有 articles 这个表.问题: 1.api接口中的status有哪些合法值,和数据库中的status字段如何对应;2.数据表中的art_type, show_type,
  ,service_type,is_read等字段,能否作为搜索条件;3.控制台中的删除是直接物理删除吗,能否标记删除 .
  总的来说,我的目的是想更灵活的过滤搜索,如果现有功能不能直接满足,评估改造或者新增接口的难度并给出建议

> 为 api/v1/wx/articles 增加文章内容查找:
  - 我已为 articles 表增加了 content 字段的全文索引
  - 在 api 接口中增加一个 match 字段, 接受一个字符串, 使用 MATCH(content) AGAINST(? IN NATURAL LANGUAGE MODE) 进行全文检索
  - 实现时参考 search 条件的处理方式
  先给出方案,待我确认
