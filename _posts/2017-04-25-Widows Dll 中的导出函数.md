# Widows Dll 中的导出函数

在程序中有如下的函数定义：

```c++
             _declspec(dllexport) void            Fun_default_default(int e){}
             _declspec(dllexport) void _stdcall   Fun_default_stdcall(int e){}
             _declspec(dllexport) void _cdecl     Fun_default_cdecl(int e){}
 
extern "C"   _declspec(dllexport) void            Fun_C_default(int e){}
extern "C"   _declspec(dllexport) void _stdcall   Fun_C_stdcall(int e){}
extern "C"   _declspec(dllexport) void _cdecl     Fun_C_cdecl(int e){}
 
extern "C++" _declspec(dllexport) void            Fun_CPP_default(int e){}
extern "C++" _declspec(dllexport) void _stdcall   Fun_CPP_stdcall(int e){}
extern "C++" _declspec(dllexport) void _cdecl     Fun_CPP_cdecl(int e){}
```

编译成dll后的结果如下：

![mx32DFB](/data/mx32DFB.png)

总结：

1. 函数会不会被重命名同时受导出方式和调用约定的控制；
2. 所有以 `extern "C++"` 方式导出的函数，都会被重命名。而且在不同编译器下的命名方式有可能不一样，因为C++标准没有规定要如何命名，因为C++有重载，因此需要重命名。
3. 所有以 `extern "C"` 方式导出的函数，需不需要重命名依赖于函数的调用约定，如果是 _stdcall 方式的话，就会被重命名，与C++不同的是，C标准规定了重命名的方式，因此，在不同编译器之间是通用的，而  _cdecl 方式不会导致重命名。
4. 默认情况下，函数的导出方式为 `extern "C++"`
5. 默认情况下，函数的调用约定为 `_cdecl`
6. extern "C++" 和 _stdcall 都会导致导出函数被重命名。
7. 如果想要以 _stdcall 方式导出，并且保持导出的函数不被重命名，可以使用def文件定义导出函数。



PS1: 如果出现链接错误，可以使用 `lib` 和 dumpbin 命令查看lib和obj文件的导出符号信息。

PS2: 链接最经常的错误是找不到符号，找不到符号有以下四种可能：

1. 链接时没有找到包含符号的lib文件；
2. lib文件中不包含符号所在的obj；（lib命令）
3. obj中不包含要查找的符号；（dumpbin命令）
4. 要查找的符号的名字不对；（dumpbin命令）