# shlab

write a sample shell in linux

首先考虑相关要写的代码。

eval：解析和解释命令行的主例程（70 lines）<br>
builtin_cmd：识别和解释内置命令（25 lines）<br>
do_bgfg: 实现 bg 和 fg 内置命令（50 lines）<br>
waitfg：等待前台工作完成（20 lines）<br>
sigchld_handler：处理 SIGCHILD 信号（80 lines）<br>
sigint_handler：处理 SIGINT 信号（15 lines）<br>
sigtstp_handler：处理 SIGTSTP 信号（15 lines）<br>
