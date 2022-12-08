电路的输入和输出都是通过引脚来体现的
- 输出引脚是方形的![|30](https://raw.githubusercontent.com/later-3/blog-img/main/20221207212357.png)
- 输入引脚是圆形的![|30](https://raw.githubusercontent.com/later-3/blog-img/main/20221207212632.png)
- 使用手型工具点击开关或改变输出引脚的值
在logsim中电路都可以作为子电路，引脚就是模块电路之间的接口

不同颜色线缆介绍：
![](https://raw.githubusercontent.com/later-3/blog-img/main/20221207220954.png)

![|350](https://raw.githubusercontent.com/later-3/blog-img/main/20221207220954.png)


# 自动生成电路
logsim可以根据真值表、逻辑表达式生成电路，但只能是组合逻辑、输入引脚1位的、输入引脚最多8个、输出引脚最多12个


# logsim的延迟与险象
![|400](https://raw.githubusercontent.com/later-3/blog-img/main/20221208215900.png)

上面的电路中，下面的非门会有一点延迟，所以是计数器会加一。
并且用单步可以看到，输出会变为1. 后面再单步前进，会将非门的0传入到输出。
有一个缓冲器(buffer)，就是为了解决这种不同步的问题。还有一种方法是在与门的第二个引脚增加非的属性。


# logsim震荡现象

![|400](https://raw.githubusercontent.com/later-3/blog-img/main/20221208223246.png)

当上面引脚输出为1，下面引脚没又输出，整个与非门的输出为1，然后1又传递到下面引脚为1，整个与非门两个输入1，与非门输出为0，然后0传递到下面引脚，整个与非门输出又为1....


