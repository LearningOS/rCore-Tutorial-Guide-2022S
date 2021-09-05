.. rCore-Tutorial-Book-v3 documentation master file, created by
   sphinx-quickstart on Thu Oct 29 22:25:54 2020.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

rCore-Tutorial-Book 2021 秋季学期
==================================================

.. toctree::
   :maxdepth: 2
   :caption: 正文
   :hidden:
   
   0setup-devel-env
   chapter1/index
   chapter2/index
   chapter3/index
   chapter4/index
   chapter5/index
   chapter6/index
   chapter7/index
   
.. toctree::
   :maxdepth: 2
   :caption: 附录
   :hidden:

   appendix-a/index
   appendix-b/index
   appendix-c/index
   appendix-d/index

.. toctree::
   :maxdepth: 2
   :caption: 开发注记
   :hidden:

   setup-sphinx
   rest-example


项目简介
---------------------

本教程展示了如何 **从零开始** 用 **Rust** 语言写一个基于 **RISC-V** 架构的 **类 Unix 内核** 。

用于 2021 年秋季学期操作系统课堂教学。

导读
---------------------
 
请先阅读第零章 :doc:`0setup-devel-env` 完成环境配置。

本学期的操作系统实验共分 4 次，分布于第三章、第四章、第六章和第七章。

以下是读者为了完成实验需掌握的技术：

- 阅读和修改简单的 Makefile 文件；
- 阅读简单的 RISC-V 汇编代码；
- git 的基本功能，解决 git merge 冲突的办法；
- Rust 基本语法和一些进阶语法，包括函数式编程，Unsafe Rust 和错误处理的手段。

你可以在实操中一点点熟悉它们。

.. attention::

   虽然实验本身在总评中占比有限，但根据往届经验，考试中可能大量出现与编程作业、思考题、代码实现思路直接相关的题目。


.. danger::

  清华大学学生纪律处分管理规定实施细则 第六章 第二十一条 
  
  有下列违反课程学习纪律情形之一的，给予警告以上、留校察看以下处分：

  （一）课程作业抄袭严重的；

  （四）在课程学习过程中严重弄虚作假的其他情形。

项目协作
----------------------

- :doc:`/setup-sphinx` 介绍了如何基于 Sphinx 框架配置文档开发环境，之后可以本地构建并渲染 html 或其他格式的文档；
- :doc:`/rest-example` 给出了目前编写文档才用的 ReStructuredText 标记语言的一些基础语法及用例；
- 时间仓促，本项目还有很多不完善之处，欢迎大家积极在每一个章节的评论区留言，或者提交 Issues 或 Pull Requests，让我们
  一起努力让这本书变得更好！


鸣谢
----------------------
