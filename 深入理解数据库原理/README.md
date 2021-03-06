- [深入理解数据库原理](#深入理解数据库原理)
  - [前言](#前言)
  - [数据库的定义](#数据库的定义)
  - [如何学习数据库](#如何学习数据库)
  - [文章目录](#文章目录)
    - [第0章：数据库基础](#第0章数据库基础)
    - [第1章：InnoDB数据结构](#第1章InnoDB数据结构)
    - [第2章：InnoDB索引与算法](#第2章InnoDB索引与算法)
    - [第3章：InnoDB锁与事务](#第3章InnoDB锁与事务)

# 深入理解数据库原理

### 前言

开始写本文时，已经是2020年12月底了，在此之前，我写了两大篇幅文章：[深入理解JAVA虚拟机](../深入理解JAVA虚拟机)、[深入理解JAVA并发与集合](../深入理解JAVA并发与集合)，这两篇文章都是JAVA基础，固然非常重要，但个人感觉十分缺少实战经验

在不断学习的过程中，我接触到了数据库，一个是存储主要业务数据的关系型数据库MySQL（以InnoDB存储引擎为主），一个是主要负责缓存与分布式锁的非关系型数据库Redis，我相信以后还有更多的数据库等着我探索，如MongoDB。我希望把学习过程中的知识点记录下来，以后总会有用

前两篇文章写的较为详细，这也意味着要花费较多的时间，但本文篇幅的内容主要是“记录”，记录实战部分、核心理论部分，不将过多冗杂的概念写入文章篇幅中

**声明**

转载请注明出处：https://github.com/peteryuanpan/notebook/blob/master/深入理解数据库原理

本文部分内容来自鲁班学院的课程，感谢老师们以及同学们对我的帮助，这里尤其感谢一下周瑜老师

参考的书籍：《MySQL技术内幕：InnoDB存储引擎》

参考的优秀博文：[行无际的博客](https://www.cnblogs.com/itwild)

### 数据库的定义

数据库是按照数据结构来组织、存储和管理数据的仓库，是一个长期存储在计算机内的、有组织的、可共享的、统一管理的大量数据的集合

> A database is an organized collection of data, generally stored and accessed electronically from a computer system. From https://en.wikipedia.org/wiki/Database

### 如何学习数据库

学习方法
- 看书以及视频，辩证地理解清楚多个基础概念，多做笔记
- 多动手实战，多做项目，模拟调优案例，从应用中理解

### 文章目录

#### 第0章：数据库基础
- [数据库基础](数据库基础.md)
  - TODO

#### 第1章：InnoDB数据结构
- [InnoDB数据结构](InnoDB数据结构.md)
  - [文件类型](InnoDB数据结构.md#文件类型)
  - [参数文件](InnoDB数据结构.md#参数文件)
  - [日志文件](InnoDB数据结构.md#日志文件)
    - [错误日志文件](InnoDB数据结构.md#错误日志文件)
    - [慢查询日志文件](InnoDB数据结构.md#慢查询日志文件)
  - [InnoDB存储引擎文件](InnoDB数据结构.md#InnoDB存储引擎文件)
  - [分析工具](InnoDB数据结构.md#分析工具)
    - [py_innodb_page_info](InnoDB数据结构.md#py_innodb_page_info)
    - [hexdump](InnoDB数据结构.md#hexdump)
  - [索引组织表](InnoDB数据结构.md#索引组织表)
  - [InnoDB逻辑存储结构](InnoDB数据结构.md#InnoDB逻辑存储结构)
  - [InnoDB数据页结构](InnoDB数据结构.md#InnoDB数据页结构)
  - [InnoDB行记录格式](InnoDB数据结构.md#InnoDB行记录格式)
    - [Compact行记录格式](InnoDB数据结构.md#Compact行记录格式)
    - [Redundant行记录格式](InnoDB数据结构.md#Redundant行记录格式)
    - [Compressed与Dynamic行格式记录](InnoDB数据结构.md#Compressed与Dynamic行格式记录)

#### 第2章：InnoDB索引与算法
- [InnoDB索引与算法](InnoDB索引与算法.md)
  - TODO

#### 第3章：InnoDB锁与事务
- [InnoDB锁与事务](InnoDB锁与事务.md)
  - TODO
