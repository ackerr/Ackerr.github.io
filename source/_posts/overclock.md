---
title: 装机实录之超频篇 
date: 2021-01-10 23:51:41
tags:
---

经历一天多次黑屏重启，初步完成了首次超频之旅。本文记录一下作为萌新超频的整个过程，就当图一乐，真正学技术还得看贴吧。

## 前言

先介绍一下平台

- CPU：AMD Ryzen 5800x
- 主板：微星B550-M
- 显卡：微星RTX 3070
- 内存：芝奇幻光戟3200MHz c16 

本篇文章中使用到的软件及在本文中的用途

- [AIDA64](https://www.aida64.com/downloads)：硬件检测工具，对CPU进行压力测试，判断CPU稳定性
- [CPU-Z](https://www.cpuid.com/softwares/cpu-z.html)：侦察CPU信息，对CPU进行单核/多核评分和稳定性测试
- [AMD Ryzen Master](https://www.amd.com/zh-hans/technologies/ryzen-master)：AMD官方提供的锐龙系CPU超频工具
- [GPU-Z](https://www.techpowerup.com/download/gpu-z/)：侦察显卡信息
- [3DMark](https://benchmarks.ul.com/3dmark)：通过Time spy压力测试，判断显卡稳定性
- [Afterburner](https://www.msi.com/Landing/afterburner)：显卡超频工具，监控硬件图标
- [RTSS](https://www.guru3d.com/files-details/rtss-rivatuner-statistics-server-download.html)：显示Afterburner中监控的硬件指标

## CPU超频

CPU下的超频需要主板和CPU同时支持，Intel这边需要型号末尾为K，Z系列主板，而AMD锐龙全系列支持，除部分A系列主板外。

通过CPU-Z查看默频下的cpu信息，顺便记一下默频跑分（单核661.5/多核6584.4），如下图所示

![默频跑分](https://picture.wzmmmmj.com/overclocking-1.png)

大部分教程中，会通过BIOS修改参数进行超频，但此方法需要来回重启电脑进入BIOS，许多主板产商会提供相关的超频工具，这里我们用AMD官方的超频工具AMD Ryzen Master进行超频。

打开AMD Ryzen Master，面板中可看到当前CPU核心频率和电压以及它们的最大值，如下图

![默频](https://picture.wzmmmmj.com/overclocking-2.png)

如果想要超频，只需调整上图右侧控制模式为手动，微调频率和电压即可。这里推荐核心频率每次增加200MHz，点击运行和测试，如果测试过程中出现死机或重启的情况，适当的上调电压，再重复上述过程，直达达到预期频率。至于CPU的最佳频率请自行搜索。如下图所示，CPU核心频率从默频3.8GHz提高到了4.6GHz，并通过测试。

![超频后](https://picture.wzmmmmj.com/overclocking-3.png)

接着再通过CPU-Z测试一下当前频率的跑分（单核631.1/多核6681.0），对比默频下的跑分，发现虽然多核分数提高了，但单核分数反而降低了。这里可能是因为CPU默频，在高负载下会自动睿频到4.8Ghz，从而提升了单核性能。

![超频后跑分](https://picture.wzmmmmj.com/overclocking-4.png)

通过Ryzen Master以及CPU-Z的粗略的压力测试后，再通过AIDA64中的进行稳定性测试，勾选Stress FPU，点击Start，观察CPU-Z中的频率以及电压看是否稳定，CPU温度是否在合理范围内。在稳定跑半个小时左右，基本宣告超频完成，如果途中出现黑屏重启或死机，适当调高电压，最好不超过1.45v，不然你懂得。如果希望进一步提升频率，重复上述步骤即可。

![AIDA64](https://picture.wzmmmmj.com/overclocking-5.png)

由于Ryzen Master重启后需要手动启动才能生效，在确定CPU核心频率及电压后，只需要进一次BIOS，在对应的位置上设置好参数即可。

## 显卡超频

显卡超频与CPU超频类似，也是通过调整核心频率，再执行对应的压力测试即可。

通过MSI Afterburner来提高核心频率，如下图所示，先将功耗限制以及温度限制拉满。

![Afterburner](https://picture.wzmmmmj.com/overclocking-6.png)

按照每次提高50MHz核心频率，如果对显卡自信的话，也可以提高多一些。设置完成后点击右下角保存，接下来只需通过显卡压测软件进行压测即可。这里推荐 FurMark 或 3DMark的Time Spy压力测试，如果设置的频率未通过压测，就降低20MHz，如此反复，直到通过压测。

![3DMark](https://picture.wzmmmmj.com/overclocking-7.png)

通过压测后，如需开机启动超频配置，只需要对相应的配置点击startup即可

![startup](https://picture.wzmmmmj.com/overclocking-8.png)

另外，在许多评测视频中，经常能看到在游戏画面中显示硬件信息，就是通过MSI Afterburner的硬件实时监控功能，如果想在游戏中显示各类硬件信息，可在MSI Afterburner设置中，开启OSD显示。

![Afterburner OSD](https://picture.wzmmmmj.com/overclocking-9.png)

开启OSD并启动RTSS后，即可在游戏中显示数据，如下图左上角。

![OSD](https://picture.wzmmmmj.com/overclocking-10.png)

## 内存超频

由于CPU具有默认支持的内存频率，例如DDR4可能是2133MHz，如果购买了DDR4 3200MHz或更高，在未修改BIOS的前提下，开机内存频率只会是2133MHz。而达到3200MHz，则需要我们手动进入BIOS，开启XMP，也就是内存预设超频。

![XMP设置](https://picture.wzmmmmj.com/overclocking-11.png)

![开启XMP后的内存频率](https://picture.wzmmmmj.com/overclocking-12.png)

当然内存也支持手动超频，具体的超频能力实际跟购买内存颗粒有关。但在体验了一次扣主板电池后，我放弃了这个选项，不然不知道又要重置多少次BIOS，只能自我安慰，预设超频挺好的：）。具体的手动超频可参考[这篇文章](https://post.smzdm.com/p/a25rz47n/)。

## 总结

相对于内存超频，会导致电脑无法开机，需要重置BIOS，CPU和显卡则安心的多，大不了就是黑屏重启。可能本身性能过剩的原因，也可能是操作不到位，超频并没有给我太多直观上的提升。当然喜欢折腾的人可以来试试看
