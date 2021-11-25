**C++学习笔记**

# GDB

常用命令:

print settings

    (gdb) set p vtbl on
    (gdb) set p pretty on
    (gdb) set p obj on
    (gdb) set p elem 3

print vtbl

    (gdb) p /a *0x4106f0 @12

debug multi-threads

    (gdb) i thr         --> 查看所有线程
    (gdb) thr 5         --> 切换到线程5
    (gdb) bt            --> 查看当前线程调用栈
    (gdb) f 8           --> 切换到调用栈8
    (gdb) i loc         --> 查看本地变量
    (gdb) p *this       --> 打印当前对象实例

others

    (gdb) r
    (gdb) c
    (gdb) b 5
    (gdb) i b           --> 查看断点信息
    (gdb) n 5           --> 单步前进5步
    (gdb) s             --> 单步前进并且进入函数
    (gdb) finish        --> 完成当前函数调用

# muduo

TCP网络编程主要处理四个事件:

* setConnectionCallback     -->  socket已连接回调

* setMessageCallback        -->  socket可读回调

* setWriteCompleteCallback  -->  socket写缓冲区完成回调

* setCloseCallback          -->  socket关闭回调

# 语法和STL

五个不能重载的运算符: [.] [.*] [::] [: ?] [sizeof]