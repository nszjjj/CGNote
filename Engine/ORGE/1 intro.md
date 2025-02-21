# 引

本文档的内容集中在总览，即：有哪些模块、为什么要这么设计、如何联系起来。

## What is ORGE

OGRE (Object-Oriented Graphics Rendering Engine) 的名字最突出特点的就是Object-Oriented，即面向对象。不过面向对象并不是一个引擎的全部，例如：

- 插件式架构：OGRE 的许多功能（如渲染系统、场景管理器、资源加载器等）

- 分层架构：一般引擎都会分层吧...

反过来问，如果一个渲染引擎并非以面向对象为主，那它会怎样？一种可能的设计便是ECS+DOD

不过网上很多文章不是在强调他面向对象的性质，而是在强调它是面向插件的，面向插件意味着易于拓展。

# Glimpse of Project

# Design Pattern

## Plugin-oriented：Kernel and Extension

OgreMain 作为 ORGE 引擎的核心库，提供了引擎最基本的框架，它包含了场景管理、渲染、资源管理等核心模块，同时提供了与插件通信所需的类。

# Reference

[CppDepend - OGRE 引擎：OOP 原则案例研究](https://cppdepend.com/blog/ogre-engine-case-study-on-oop-principles/)
