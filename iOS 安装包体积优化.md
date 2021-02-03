Android apk的体积检测方案，有开源的[Matrix-Android-ApkChecker](https://github.com/Tencent/matrix/wiki/Matrix-Android-ApkChecker)，那iOS端要如何更深入的检测ipa安装包的小大呢？我们从两个方向着手。

## LinkMap 文件

Link Map File 直译为链接映射文件，是 Xcode 生成可执行文件时一起生成的文本，用于记录链接相关信息。

### Link Map File 有什么用？

- 查看代码加载顺序

- 理解内存分段分区

- Crash 时通过 Symbols 定位源码的机制

- 分析可执行文件中类或库体积，优化包体积

这里，我们主要用到的就是第四点，分析可执行文件的类库体积，优化包体积。

### 如何获取Linkmap文件呢？

1. 打开xcode工程，点击对应的target
2. 点击build settings
3. 输入 link map过滤出设置项
4. 将 write link map file 设置为Yes，默认是No

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203115625.png)

这时，再run就会生成link map文件了，但是默认生成的路径比较深，我们可以在run之前，自行修改生成的路径，方便之后查找文件，如下图

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203115723.png)

### Linkmap文件结构

1. Path是可执行文件的路径，Arch是架构类型。

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203115803.png)

1. 上图中的Object files，这里展示的信息是链接时用到的文件，包括.o文件和dylib库。第一列的序号是类的编号，通过该编号可以对应到具体的类。

2. Symbols：

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203115927.png)

1. Sections，第一列是起始位置，第二列是Section占用内存大小，第三列是Segment类型，第四列是Section类型，这个如果要理解，需要学习更多Mach-O知识。

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203120023.png)

### 分析并获取结果

参考[LinkMapParser](https://github.com/zgzczzw/LinkMapParser)开源的脚本，针对自己应用进行修改，最后的对比结果如下：

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203120112.png)

## iPA 文件

### 分析ipa文件

除了通过linkmap文件分析之外，在和开发的沟通中，还提出了对ipa文件内容进行分析的需求，主要需要关注package、assert、bundle三个目录的大小，最后通过编写shell完成分析的过程，主要代码如下：

```PowerShell
payload="/Payload/xxx.app"
base_root_path=$1
target_root_path=$2
echo "上一版本文件的目录：${base_root_path}";
echo "当前版本文件的目录：${target_root_path}";

root_path=$PWD

package_file='Package$'
assets_car='Assets.car'
bundle_file='bundle$'

cd $1$payload
echo "开始计算..."
package_size_1=$(du -ch | grep ${package_file} | awk '{print $1}')
assets_size_1=$(ls -lh | grep ${assets_car} | awk '{print $5}')
bundle_size_kb_1=$(du -ck | grep ${bundle_file} | awk '{sum+=$1} END {print sum}')
bundle_size_mb_1=`expr ${bundle_size_kb_1} / 1024`

# 解析出来文件名
bundle_file_1=${base_root_path#*xxx_release_}
bundle_file_1=${bundle_file_1%/Payload*}"-bundle.txt"
du -ch | grep 'bundle$' > $bundle_file_1
base_bundle_list_file_path=$PWD/$bundle_file_1
echo "生成 bundle 文件列表，目录：" ${base_bundle_list_file_path}

echo "================================================================================"
echo "                                   主要文件大小变化对比                            "
echo "================================================================================"
printf "%-20s %-20s %-20s\n" FileName Pervious Current
printf "%-20s %-20s %-20s\n" Package ${package_size_1} ${package_size_2} 
printf "%-20s %-20s %-20s\n" Assets ${assets_size_1} ${assets_size_2} 
printf "%-20s %-20s %-20s\n" Bundle ${bundle_size_mb_1}M ${bundle_size_mb_2}M

```

### 分析结果

![](https://raw.githubusercontent.com/CYsuncheng/markdown-pic/master/20210203120203.png)

## 参考

[https://www.jianshu.com/p/52e0dee35830](https://www.jianshu.com/p/52e0dee35830)

[https://juejin.cn/post/6844904168096792583](https://juejin.cn/post/6844904168096792583)