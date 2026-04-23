
kprobe使用指南：
[https://blog.csdn.net/luckyapple1028/article/details/52972315](https://blog.csdn.net/luckyapple1028/article/details/52972315)

开启方法 查看可用的函数列表（可用于kprobe）
cat /sys/kernel/debug/tracing/available_filter_functions
===不支持的可能会echo失败===


添加kprobe事件
echo 'p:my_probe queue_work' > /sys/kernel/debug/tracing/kprobe_events

开启kprobe
echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable

开启tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

开启堆栈打印
echo 1 > options/stacktrace

关闭方法

echo 0 > /sys/kernel/debug/tracing/events/kprobes/enable
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo '' > /sys/kernel/debug/tracing/kprobe_events