---
layout: post
title:  "unix操作系统设计-第五章文件系统的系统调用笔记"
date:   2020-08-13 21:33:00 +0700
categories: [unix]
---

## 学习目标
学习和文件系统相关的系统调用,包括*open、read、write、lseek、close*(访问现有文件),和*create、mknod*(创建新文件)和*chdir、chroot、chown、chmod、stat、fstat*(改变文件的inode)和一些其他的高级调用.

## 核心词汇
文件描述符表
文件表
内存索引节点表

## 概念解释
![[Pasted image.png]]
如图所示,文件描述符表存储着被进程打开的文件的标示符,其指向文件表,文件表中存储的是文件系统中每个被打开的文件的信息,其指向索引节点表,索引节点表存储的是每个文件的索引节点.

### 算法open步骤
fd=open(pathname,flags,modes)
1. 首先通过namei获得上锁的索引节点
2. 给file table分配表项,指向索引节点,初始化引用数和offset,默认是0(read和write开始的地方)
3. 分配用户描述符表项,设置指向file table表项的指针
4. inode解锁
5. 返回fd

文件描述符的0、1、2分别是标准输入、标准输出、标准异常

### 算法read步骤
number=read(fd,buf,count)

1. 从fd拿到file table表项
2. 校验文件是否可访问
3. 设置u区的IO参数
![[Pasted image 1.png]]
4. 从file table中拿到inode
5. 锁住inode
6. loop:当读取的字节数未到count
	1. 使用bmap将offset转换为block中要读取多少数据,如果为0说明到达文件末尾返回
	2. 否则读取block中的数据到内核缓冲区(bread),再将数据拷贝至用户指定的buf
	3. 更新u区的参数
	4. 释放bread锁住的内核缓冲区
7. 解锁inode
8. 更新file table的offset,表示下次读取/写入从哪里开始
9. 返回读取了多少字节

### 算法write步骤
write的过程中如果数据块不够,可能要动态分配(alloc算法),在动态分配的过程中,
还要修改inode的磁盘块记录.












