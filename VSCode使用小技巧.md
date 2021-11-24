## VS Code实用小技巧

1. vscode如果提示消失，确实可以通过快捷键让提示继续

方法这里介绍:[VSCode里面代码提示快捷键](https://www.jianshu.com/p/3a278e15adbc)

`preference` => `keyboard shortcut`

![image-20211125011559597](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125011559597.png)

2. vscode使用F12跳转到(函数 或 变量)定义处;

3. vscode使用如下这些快捷方式返回刚才的跳转

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125011705524.png" alt="image-20211125011705524" style="zoom:67%;" />

4. [快捷键大全](https://juejin.im/post/5d34fdfff265da1b897b0c8d)

   比如:

   - `alt`+`f12` 只是查看定义而不跳转;

   - `shift`+`f12` 查看所有引用;

   - `cmmand`+`opetion`+左/右 可以在当前窗口的多个打开文件间切换。

   - `ctrl` + \` 能打开和关闭终端;

5. 设置终端选择后复制到粘贴板，右键则粘贴

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125011850180.png" alt="image-20211125011850180" style="zoom:67%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125011942150.png" alt="image-20211125011942150" style="zoom:67%;" />

6. [三分钟教你同步 Visual Studio Code 设置](https://juejin.im/post/5b9b5a6f6fb9a05d22728e36)

7. 编写c++时，如何设置`-std=c++11`

   报错情况:`non-aggregate type 'vector<int>' cannot be initialized with an initializer list`

   参考文章:[Visual Studio Code 支持C++11](https://www.jianshu.com/p/e1bc046edecc)

   设置C的标准:

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125012602565.png" alt="image-20211125012602565" style="zoom:50%;" />

   设置C++的标准:

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125012700911.png" alt="image-20211125012700911" style="zoom:50%;" />

   `build task`的配置:

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125012807618.png" alt="image-20211125012807618 " style="zoom:50%;" />

8. vs code调试C/C++程序，如何设置`LD_PRELOAD`?

   参考文档: [miDebuggerArgs to enable LD_PRELOAD doesn't seem to have any effect ](https://github.com/microsoft/vscode-cpptools/issues/4567)

   ```json
   			"environment": [
   					{
   						"name":"LD_LIBRARY_PATH",
   						"value":"/data/home/xxxx/code/xxxx/deps/so"
   					},{
   					 	"name":"LD_PRELOAD",
   					 	"value":"/data/home/xxx/code/xxxx/deps/so/libjemalloc.so"
   					}
   			],
   ```

   <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125013329176.png" alt="image-20211125013329176" style="zoom:50%;" />

9. 关于vscode中 debug console 打印变量

   [在VS Code中开启gdb的pretty-printer功能](https://blog.csdn.net/yanxiangtianji/article/details/80579236)

   - 执行命令用: `-exec print vec`;

   - 开启`pretty-printing`: `-exec -enable-pretty-printing`;

   - 调试 c/c++时，查看int、size_t等类型都是显示的 16进制值，如何显示 10进制值？

     参考文章: [VSCode display hex values for variables in debug mode](https://stackoverflow.com/questions/42645761/vscode-display-hex-values-for-variables-in-debug-mode)、 [How to display hex value in watch in vscode?](https://stackoverflow.com/questions/39973214/how-to-display-hex-value-in-watch-in-vscode)

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125014526577.png" alt="image-20211125014526577" style="zoom:50%;" />

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125014403109.png" alt="image-20211125014403109 " style="zoom:50%;" />

     <img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211125014655548.png" alt="image-20211125014655548 " style="zoom:50%;" />