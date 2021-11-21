# CPlusPlus
*C++ Notes*

muduo
-----

TCP网络编程主要处理四个事件:

* setConnectionCallback     -->  socket已连接回调

* setMessageCallback        -->  socket可读回调

* setWriteCompleteCallback  -->  socket写缓冲区完成回调

* setCloseCallback          -->  socket关闭回调

语法及STL
--------

五个不能重载的运算符: [.] [.*] [::] [: ?] [sizeof]