# db-notes

#### TiDB 源码阅读

- [优化器的整体结构](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E4%BC%98%E5%8C%96%E5%99%A8%E7%9A%84%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84.md)
- 执行计划的数据结构
  - [执行计划的结构体类型](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E7%9A%84%E7%BB%93%E6%9E%84%E4%BD%93%E7%B1%BB%E5%9E%8B.md)
  - [执行计划类型实现的接口](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E7%B1%BB%E5%9E%8B%E5%AE%9E%E7%8E%B0%E7%9A%84%E6%8E%A5%E5%8F%A3.md)
- [逻辑执行计划节点](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md)
- [逻辑执行计划的构造方法](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E7%9A%84%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95.md)
  - [构造逻辑执行计划：处理 WITH 子句](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20WITH%20%E5%AD%90%E5%8F%A5.md)
  - [构造逻辑执行计划：处理 FROM 子句](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20FROM%20%E5%AD%90%E5%8F%A5.md)
  - [构造逻辑执行计划：通配符展开](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E9%80%9A%E9%85%8D%E7%AC%A6%E5%B1%95%E5%BC%80.md)
  - [构造逻辑执行计划：输出列命名](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E8%BE%93%E5%87%BA%E5%88%97%E5%91%BD%E5%90%8D.md)
- [逻辑优化的架构设计](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E4%BC%98%E5%8C%96%E7%9A%84%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1.md)
  - 逻辑优化规则：谓词下推
