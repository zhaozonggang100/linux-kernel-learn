参考：http://www.mysixue.com/?p=119

### 1、简介

page owner是跟踪谁分配了page，可以用来debug内核泄漏和内存越界，当分配page发生时，会保存当前的分配调用栈到page_ext结构中。

### 2、内核doc

`Documentation/vm/page_owner.txt`

### 3、sys查看

`cat /sys/kernel/debug/page_owner`

