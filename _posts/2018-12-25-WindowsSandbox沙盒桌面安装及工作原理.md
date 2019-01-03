# Windows Sandbox沙盒桌面安装及工作原理 #
　　WindowsSandbox是一种新型的轻量级桌面环境，专为安全运行应用程序而量身定制。你有多少次下载了一个程序，却害怕运行它？是否曾经遇到过这样的情况：需要一个干净地Windows，但又不想安装虚拟机？因此Microsoft开发了Windows Sandbox：一个独立的临时桌面环境，可以在其中运行不受信任的软件，而不必担心会对你的电脑产生致命的破坏。Windows Sandbox中安装的任何软件仅保留在沙箱中，不会影响主机。Windows Sandbox关闭后，将永久删除其中包含的所有文件、状态及软件。

## Windows Sandbox具有以下功能： ##
　　Windows的一部分 - 此功能所需的一切都随Windows 10 Pro和Enterprise一起提供。无需下载VHD！
- 原始:每次Windows Sandbox运行时，它都像Windows的全新安装一样干净。
- 一次性:设备上没有任何东西，关闭应用程序后，一切都将被丢弃。
- 安全:使用基于硬件的虚拟化进行内核隔离，后者依靠Microsoft的虚拟机管理程序运行单独的内核，将Windows Sandbox与主机隔离开来。
- 高效:使用集成的内核调度程序，智能内存管理和虚拟GPU。

 

## 使用Windows Sandbox的必要条件 ##

1. Windows 10 Pro或Enterprise Insider内部版本18305或更高版本<font color=red>查看资料发现win10 1809就行</font>
1. AMD64位架构
1. 在BIOS中启用虚拟化功能
1. 至少4GB的RAM（推荐8GB）
1. 至少1 GB的可用磁盘空间（建议使用SSD）
1. 至少2个CPU核心（建议使用4个超线程核心）


## 快速安装Windows Sandbox ##

　　1、安装Windows 10 Pro或Enterprise，Insider build 18305或更新版本。
　　2、启用虚拟化：
　　　　如果使用的是物理机，请确保在BIOS中启用了虚拟化功能。
　　　　如果使用的是虚拟机，请使用此PowerShell cmdlet启用嵌套虚拟化：  

			Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $ true
　　3、打开Windows功能，然后选择Windows Sandbox。选择“ 确定”以安装Windows Sandbox。系统可能会要求重新启动计算机。
　　4、使用“ 开始”菜单，找到Windows Sandbox，运行它。
　　5、从主机复制可执行文件（应用程序）。
　　6、将可执行文件（应用程序）粘贴到Windows Sandbox的窗口中（在Windows桌面上）。
　　7、在Windows Sandbox中运行可执行文件（应用程序），如果是安装程序，请进行安装。
　　8、运行应用程序并像平常一样使用它。
　　9、完成实验后，只需关闭Windows Sandbox应用程序即可，沙盒中所有内容都将被丢弃并永久删除。
　　10、确认Windows Sandbox没有对主机进行任何修改。

## Windows Sandbox内部原理说明 ##

### 动态生成的镜像 ###

　　Windows Sandbox的核心是一个轻量级虚拟机，因此需要一个操作系统镜像才能启动。Microsoft为Windows Sandbox提供的一项重要增强功能，能够使用安装在计算机上的Windows 10 副本，而不是像使用普通虚拟机那样下载新的VHD映像。
　　Microsoft希望始终呈现一个干净的环境，但挑战在于某些操作系统文件可能会发生变化。解决方案是构建一个称之为“动态基础镜像”的操作系统镜像：一个干净的操作系统镜像，文件可以变动，大多数文件是链接（不可变文件），镜像中完整的操作系统的大小为100MB左右。

　　如果未安装Windows Sandbox，动态基础镜像仅仅是25MB的压缩包。安装动态基础包时，它占用大约100MB的磁盘空间。

<div align=center>![](https://ws1.sinaimg.cn/large/68138b3egy1fyipotfal0j20rr08adgd.jpg)</div>


### 智能内存管理 ###
　　Microsoft的虚拟机管理程序，允许将单个物理机器划分为多个，共享相同物理硬件的虚拟机。虽然这种方法适用于传统的服务器，但它不适合在资源有限的设备上运行。Microsoft设计了Windows Sandbox，以便主机可以根据需要从沙盒中回收内存。
　　此外，由于Windows Sandbox所使用的镜像，基本上与主机的操作系统镜像保持一致，因此允许Windows沙箱通过“直接映射”的技术，使用与主机相同的物理内存页面。换句话说，ntdll的相同可执行页面，将映射到主机的沙箱中。

<div align=center>![](https://ws1.sinaimg.cn/large/68138b3egy1fyipotbiycj20rr079glx.jpg)</div>
 
### 集成内核调度程序 ###
　　对于普通虚拟机，Microsoft的虚拟机管理程序，只能控制VM中虚拟处理器的调度。但是，对于Windows Sandbox，Microsoft使用称为“集成调度程序”的新技术，该技术允许主机决定何时运行沙箱。
　　对于Windows Sandbox，采用了一种独特的调度策略，该策略允许以为进程调度线程的方式（类似的原理），调度沙箱的虚拟处理器。主机上的高优先级任务，可以抢占沙箱中不太重要任务的执行时间。使用集成调度程序的好处是，主机将Windows Sandbox作为一个进程，而不是虚拟机来管理，从而产生响应速度更快的主机，类似于Linux KVM。
　　Windows Sandbox的目标是将Sandbox视为应用程序，但仍具有虚拟机的安全保障。
### 快照和克隆 ###
　　如上所述，Windows Sandbox使用Microsoft的虚拟机管理程序。因此另启一个Windows副本进行运行时，是需要一些时间的。所以，最终的方案并不是每次启动Windows Sandbox时，都重塑一个沙盒操作系统，而是使用另外两种技术; “快照”和“克隆”。
　　快照允许我们在启动沙盒环境时并将内存、CPU和设备状态保存到磁盘。然后，当我们需要一个新的Windows Sandbox实例时，可以从磁盘恢复沙箱环境并将其放入内存而不是启动它。这显着改善了Windows Sandbox的启动时间。 
### 图形虚拟化 ###
　　对于图形密集型或多媒体密集型实例，硬件加速渲染是平稳和响应式用户体验的关键。但是，虚拟机与其主机隔离，无法访问GPU等高级设备。因此，图形虚拟化技术的作用是弥合这一差距，并在虚拟化环境中提供硬件加速，例如Microsoft RemoteFX。
　　<font color=red>**最近，Microsoft与图形生态系统伙伴合作，将现代图形虚拟化功能，直接集成到DirectX和WDDM中**</font>，以下是Windows上显示驱动程序使用的驱动模型。
　　从较高层面来看，这种形式的图形虚拟化的工作原理如下：
- 　　在Hyper-V VM中运行的应用程序正常使用图形API。
- 　　VM中的图形组件（已支持虚拟化）在VM边界与主机协调以执行图形工作负载。
- 　　主机在VM所运行的应用程序之间，分配和调度图形资源以及本机运行的应用程序。从概念上讲，它们表现为一个图形客户端池。
　　此过程如下所示

<div align=center>![](https://ws1.sinaimg.cn/large/68138b3egy1fyipot8ucqj20810b4q3e.jpg)</div>

　　这使得Windows Sandbox VM能够受益于硬件加速渲染，Windows可以在主机和来宾之间动态分配图形资源。其结果是提高了Windows Sandbox中，应用程序的性能和响应能力，并提高了图形处理较多的电脑的电池寿命。
　　要利用这些优势，需要一个具有兼容GPU和图形驱动程序（<font color=red>**WDDM2.5或更高版本**</font>）的系统。不兼容的系统，将导致Windows Sandbox中的应用程序，依然使用Microsoft基于CPU的渲染技术。

### 电池感应 ###
　　Windows Sandbox还能感受到主机的电池状态，从而可以优化功耗。这对于笔记本电脑的至关重要，不过高的消耗电池对用户来说非常重要。