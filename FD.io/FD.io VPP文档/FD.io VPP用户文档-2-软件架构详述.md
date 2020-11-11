<div align=center>
	<img src="_v_images/20200904171558212_22234.png" width="300"> 
</div>

<br/>
<br/>
<br/>

<center><font size='6'>FD.io VPP：用户文档 软件架构详述</font></center>
<br/>
<br/>
<center><font size='5'>荣涛</font></center>
<center><font size='5'>2020年9月 ~ 2020年10月</font></center>
<br/>
<br/>
<br/>
<br/>

[VPP /软件架构](https://wiki.fd.io/view/VPP/Software_Architecture)
[Software Architecture](https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/softwarearchitecture.html)



`fd.io vpp`实现是第三代矢量数据包处理实现。请注意，`Apache-2许可证`专门授予非专有的专利许可证。

为了提高性能，vpp数据平面由转发节点的有向图组成，该转发图每次调用处理多个数据包。这种模式可实现多种微处理器优化：流水线和预取以覆盖相关的读取延迟，固有的I缓存阶段行为，矢量指令。除了硬件输入和硬件输出节点之外，整个转发图都是可移植的代码。

根据当前的情况，我们经常启动多个工作线程，这些工作线程使用相同的转发图副本处理来自`多个队列`的进入哈希数据包

![](_v_images/20200908130112264_11402.png =800x)


<br/>

VPP软件分层如下图：
![VPP层-实施分类法](_v_images/20200907152226718_3254.png =1139x)

vpp数据平面包括四个不同的层：

* VPP Infra-`VPP基础结构层`，其中包含核心库源代码。该层执行存储功能，与向量和环配合使用，在哈希表中执行键查找，并与用于调度图形节点的计时器一起使用。
* VLIB-`向量处理库`。vlib层还处理各种应用程序管理功能：缓冲区，内存和图形节点管理，维护和导出计数器，线程管理，数据包跟踪。Vlib实现了调试CLI（命令行界面）。
* VNET-`VPP的网络接口`（第2层，第3层和第4层）配合使用，执行会话和流量管理，并与设备和数据控制平面配合使用。
* Plugins-包含越来越丰富的数据平面插件集，如上图所示。
* VPP-与以上所有内容链接的容器应用程序。
重要的是要对每一层都有一定的了解。最好在API级别处理大多数实现，否则就不去管它了。

<br/>

# 1. [VPPINFRA（基础设施）](https://fd.io/docs/vpp/master/gettingstarted/developers/infrastructure.html#vppinfra-infrastructure)

与VPP基础结构层关联的文件位于`./src/vppinfra`文件夹中。

`VPPinfra`是基本c库服务的集合，足以建立直接在裸机上运行的独立程序。它还提供高性能的动态数组，哈希，位图，高精度的实时时钟支持，细粒度的事件记录和数据结构序列化。

关于vppinfra的一个合理的评论/合理的警告：您不能总是仅通过名称来从普通函数中的内联函数中告诉宏。宏通常用于避免函数调用，并引起（有意的）副作用。

详细介绍请参见：[VPPINFRA](https://fd.io/docs/vpp/master/gettingstarted/developers/infrastructure.html)  ,   [wiki VLIB](https://wiki.fd.io/view/VPP/Software_Architecture)

## 1.1. 向量 vectors
Vppinfra向量是用户定义的“标头”随处可见的动态调整大小的数组。许多`vpppinfra`数据结构（例如哈希，堆，池）是具有各种不同标头的向量。
```
The memory layout looks like this:

                   User header (optional, uword aligned)
                   Alignment padding (if needed)
                   Vector length in elements
 User's pointer -> Vector element 0
                   Vector element 1
                   ...
                   Vector element N-1
```
如上所示，向量API处理指向向量第0个元素的指针。空指针是长度为零的有效向量。

为避免破坏**内存分配器**，通常在保留内存分配的同时将向量的长度重置为零。通过`vec_reset_length（v）`宏将向量长度字段设置为零。[使用宏！NULL指针很聪明。]

通常，用户头不存在。用户标头允许将其他数据结构构建在vppinfra向量之上。用户可以通过`vec * _aligned`宏指定数据元素的对齐方式。

向量元素可以是任何C类型，例如`（int，double，struct bar）`。对于在矢量之上构建的数据类型（例如，堆，池等）也是如此。许多宏具有支持向量数据对齐的`_a`变体和支持非零长度向量头的`_h`变体。`_ha`变体都支持。

标头和/或与对齐相关的宏变体用法不一致会导致延迟，混乱的故障。

标准编程错误：存储指向向量ith元素的指针，然后展开向量。向量扩展了3/2，因此此类代码似乎可以工作一段时间。正确的代码几乎总是记住向量索引，向量索引在重新分配中是不变的。

在典型的应用程序映像中，一个提供了一组全局函数，这些函数旨在从gdb调用。这里有一些例子：

* vl(v) - prints vec_len(v)
* pe(p) - prints pool_elts(p)
* pifi(p, index) - prints pool_is_free_index(p, index)
* debug_hex_bytes (p, nbytes) - hex memory dump nbytes starting at p

## 1.2. 位图 bitmap
Vppinfra位图是动态的，是使用vppinfra矢量API构建的。非常适合各种工作。

## 1.3. 池 pool
Vppinfra池结合了矢量和位图，可以快速分配和释放具有独立生存期的固定大小的数据结构。池非常适合分配每个会话的结构。

## 1.4. 散列 `hash`
Vppinfra提供了几种哈希样式。涉及数据包分类/会话查找的数据平面问题经常使用`... / src / vppinfra / bihash_template`。[ch]有界索引可扩展哈希。这些模板被实例化多次，以有效地服务于不同的固定键大小。

`Biashhes`是线程安全的。不需要读锁定。简单的自旋锁可确保一次仅一个线程写入一个条目。

`... / src / vppinfra / hash`。[ch]中的原始vppinfra哈希实现易于使用，并且经常用于需要精确字符串匹配的控制平面代码中。

无论哪种情况，几乎总是在哈希表中查找键，以获得相关向量或池中的索引。这些API非常简单，但是在使用不受管理的任意大小的密钥变体时，一定要小心。`Hash_set_mem`（哈希表，key_pointer，值）存储`key_pointer`。将向量元素的地址作为第二个参数传递给`hash_set_mem`通常是一个严重的错误。最好在文本段中存储常量字符串地址。

```c
#define hash_set_mem(h,key,value) hash_set3 (h, pointer_to_uword (key), (value), 0)
/* Public macro to set a (key, value) pair, return the old value */
#define hash_set3(h,key,value,old_value)				\
({									\
  uword _v = (uword) (value);						\
  (h) = _hash_set3 ((h), (uword) (key), (void *) &_v, (old_value));	\
})
```

## 1.5. 格式 `format`
Vppinfra格式大致等效于`printf`。

格式具有一些值得一提的属性。Format的第一个参数是一个（`u8 *`）向量，在该向量后附加了当前format操作的结果。链接调用非常简单：
```c
u8 * result;

result = format (0, "junk = %d, ", junk);
result = format (result, "more junk = %d\n", more_junk);
```
如前所述，NULL指针是完全正确的0长度向量。格式返回（`u8 *`）向量，而不是C字符串。如果要打印（`u8 *`）矢量，请使用“`％v`”格式字符串。如果您需要一个（u8 *）向量，它也是一个适当的C字符串，则可以使用以下两种方案之一：
```c
vec_add1 (result, 0)
or 
result = format (result, "<whatever>%c", 0); 
```
请记住，如果合适，请对`vec_free（）`结果进行处理。注意不要通过格式化未初始化的`u8 *`。

格式通过“`％U`”格式规范实现了一种特别方便的用户格式方案。例如：
```c
u8 * format_junk (u8 * s, va_list *va)
{
  junk = va_arg (va, u32);
  s = format (s, "%s", junk);
  return s;
}

result = format (0, "junk = %U, format_junk, "This is some junk");
```
如果需要，`format_junk（）`可以调用其他用户格式的函数。程序员负责参数类型检查。如果`va_arg（va，<type>）`宏与调用者的现实想法不符，通常会导致用户格式函数崩溃。

## 1.6. 取消格式 `unformat`
Vppinfra `unformat`与`scanf`隐约相关，但更为笼统。

典型的用例涉及从C字符串或（u8 *）向量初始化`unformat_input_t`，然后通过`unformat（）`进行解析，如下所示：
```c
unformat_input_t input;

unformat_init_string (&input, "<some-C-string>");
/* or */
unformat_init_vector (&input, <u8-vector>);
```
然后循环解析单个元素：
```c
while (unformat_check_input (&input) != UNFORMAT_END_OF_INPUT) 
{
  if (unformat (&input, "value1 %d", &value1))
    ;/* unformat sets value1 */
  else if (unformat (&input, "value2 %d", &value2)
    ;/* unformat sets value2 */
  else
    return clib_error_return (0, "unknown input '%U'", format_unformat_error, 
                              input);
} 
```
与格式化一样，取消格式化通过“`％U`”用户取消格式化功能方案实现了用户取消格式化功能。


## 1.7. Vppinfra错误和警告
vpp数据平面中的许多函数的返回值类型为`clib_error_t *`。`Clib_error_t`是任意字符串，带有一些元数据[致命，警告]，并且易于宣布。返回`NULL clib_error_t *`表示“没问题，没有错误”。

```c
typedef struct
{
  /* Error message. */
  u8 *what;
  /* Where error occurred (e.g. __FUNCTION__ __LINE__) */
  const u8 *where;
  uword flags;
  /* Error code (e.g. errno for Unix errors). */
  any code;
} clib_error_t;
```

`clib_warning（<format-args>）`是添加调试输出的便捷方法。clib警告优先于function：line info以明确定位消息源。`Clib_unix_warning（）`添加了`perror（）`样式的Linux系统调用信息。在生产映像中，clib_warnings会生成`syslog`条目。
```c
#define clib_warning(format,args...) \
  _clib_error (CLIB_ERROR_WARNING, clib_error_function, __LINE__, format, ## args)
```

## 1.8. 序列化
Vppinfra序列化支持使程序员可以轻松地序列化和反序列化复杂的数据结构。

基本的原始`序列化/反序列化`函数使用`网络字节顺序`，因此在小端字节序的主机上序列化和在大端字节序的主机上反序列化没有结构性问题。

## 1.9. 事件记录器，图形事件日志查看器
vppinfra事件记录器提供了非常轻量级（不到100ns），带有时间戳的精确事件记录服务。参见`... / src / vppinfra / {elog.c，elog.h}`

序列化支持使其易于保存，并最终组合了一组事件日志。在通过本地LAN运行NTP的分布式系统中，我们发现从多个系统元素收集的事件日志可以与不小于50us的时间不确定性结合使用。

典型的事件定义和日志记录调用如下所示：
```c
ELOG_TYPE_DECLARE (e) = 
{
  .format = "tx-msg: stream %d local seq %d attempt %d",
  .format_args = "i4i4i4",
};
struct { u32 stream_id, local_sequence, retry_count; } * ed;
ed = ELOG_DATA (m->elog_main, e);
ed->stream_id = stream_id;
ed->local_sequence = local_sequence;
ed->retry_count = retry_count;
```
`ELOG_DATA`宏返回一个指向20个字节的任意事件数据的指针，该数据将按照`format_args`的说明进行格式化（脱机，而不是在运行时）。除了明显的整数格式外，CLIB事件记录器还提供了一些有趣的功能。“ t4”格式漂亮地打印枚举值：
```c
ELOG_TYPE_DECLARE (e) = 
{
  .format = "get_or_create: %s",
  .format_args = "t4",
  .n_enum_strings = 2,
  .enum_strings = { "old", "new", },
};
```
“ `t`”格式说明符表示相应的数据是事件的枚举字符串集中的索引，如先前的事件类型定义中所示。

“ `T`”格式说明符表示相应的数据是事件日志的字符串堆中的索引。这使程序员可以发出任意格式的字符串。人们通常将此功能与哈希表结合使用，以防止事件日志字符串堆任意增大。

注意每个日志条目数据字段限制为20个八位字节，事件日志格式化程序支持这些数据类型的任意组合。如：“format”字段可能包含以下的一个或多个实例：

* i1-8位无符号整数
* i2-16位无符号整数
* i4-32位无符号整数
* i8-64位无符号整数
* F4-浮动
* f8-双
* s-以NULL结尾的字符串-注意
* sN-N字节字符数组
* t1,2,4-每个事件的枚举ID
* T4-事件日志字符串表偏移量

**vpp引擎事件日志是线程安全的**，并且由所有线程共享。注意不要序列化计算。尽管事件记录器的速度尽可能快，但它不适用于硬数据平面代码中的每个数据包。它最适合捕获罕见事件-上下链接事件，特定的控制面板事件等。

vpp引擎具有几个调试`CLI`命令，用于处理其事件日志：
```
vpp# event-logger clear
vpp# event-logger save <filename> # for security, writes into /tmp/<filename>.
                                  # <filename> must not contain '.' or '/' characters
vpp# show event-logger [all] [<nnn>] # display the event log
                                   # by default, the last 250 entries
```
事件日志默认为`128K`条目。命令行参数“ ... `vlib {elog-events <nnn>}`”配置事件日志的大小。

如上所述，vpp引擎事件日志是线程安全的并且是共享的。为避免混淆工作线程记录的事件的不出现，请确保编码`＆vlib_global_main.elog_main`-而不是`＆vm-> elog_main`。后一种形式在主线程中是正确的，但几乎可以肯定会在辅助线程中产生不好的结果。

## 1.10. G2图形事件查看器
g2图形事件查看器可以直接或通过`c2cpel`工具显示序列化的vppinfra事件日志。请参阅[g2 Wiki页面](https://wiki.fd.io/view/VPP/g2)。
G2图形事件查看器可以直接或通过c2cpel工具显示序列化的vppinfra事件日志。G2是一个细粒度的事件日志查看器。它具有高度的可扩展性，支持O（1e7事件）和O（1e3离散显示“轨道”）。G2显示由vppinfra“ elog。[ch]”记录器组件生成的二进制数据，并且还支持CPEL文件格式，如本节所述。

G2馆：此链接描述了[如何构建G2](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/cmakeandninja.html#building-g2)

### 1.10.1. 设置显示首选项
文件$ < HOMEDIR > /。g2包含显示首选项，可以覆盖它们。只需取消注释以下所示的节之一，或根据需要进行实验。
```
/*
 * Property / parameter settings for G2
 *
 * Setting for a 1024x768 display:
 * event_selector_lines=20
 * drawbox_height=800
 * drawbox_width=600
 *
 * new mac w/ no monitor:
 * event_selector_lines=20
 * drawbox_height=1200
 * drawbox_width=700
 *
 * 1600x1200:
 * drawbox_width=1200
 * drawbox_height=1000
 * event_selector_lines=25
 *
 * for making screenshots on a Macbook Pro
 * drawbox_width=1200
 * drawbox_height=600
 * event_selector_lines=20
 */
```
### 1.10.2. 屏幕分类法
这是带注释的G2查看器屏幕快照，对应于BGP前缀下载期间的活动。此数据在Cisco IOS-XR系统捕获了：
![](_v_images/20201009161702705_8796.png)

查看器具有两个主滚动条：水平轴滚动条可在时间上移动主绘图区域；水平滚动条可在时间上移动主绘图区域。垂直轴更改可见的过程迹线集。放大/缩小运算符更改时间刻度。

事件选择器PolyCheckMenu更改显示的事件集。使用这些工具以及一些耐心，您可以了解给定的事件日志。

### 1.10.3. 鼠标手势
G2具有三个相当复杂的鼠标手势界面，值得详细说明。首先，在显示事件上单击鼠标左键会弹出每个事件的详细信息框。
![](_v_images/20201009161726713_7986.png)

鼠标左键单击事件详细信息框将其关闭。要缩放到显示区域，请按住鼠标左键，然后向右或向左拖动直到出现缩放栅栏对：
![](_v_images/20201009161739441_1947.png)

缩放操作完成后，显示如下：
![](_v_images/20201009161751632_14285.png)

点击任意图将以全分辨率显示它们，右键点击将在新标签页中打开图，

### 1.10.4. 时间尺
要使用时间标尺，请按住鼠标右键；向右或向左拖动，直到标尺测量感兴趣的区域。如果时间轴刻度很粗，则事件框的时间宽度可能很大，因此在使用时间标尺时，请在每个事件框中使用“参考点”。

![](_v_images/20201009161812017_6241.png)


### 1.10.5. 活动选择
更改事件选择器设置可明显控制显示的点集。在这里，我们抑制所有事件，除了“此线程现在正在CPU上运行”：
![](_v_images/20201009161826487_23255.png)

设置相同，显示所有事件：
![](_v_images/20201009161839556_32722.png)
请注意，先前显示的事件详细信息框由于重新选择事件代码而被抑制，当重新选择事件代码时，该事件详细信息框将重新出现。在上面的示例中，“ THREAD / THREADY pid：491720 tid：12”详细信息框以这种方式出现。

### 1.10.6. 快照
g2主窗口左下角的三个按钮控制快照环。快照只是保存的视图：将查看器调整为“有趣的”配置，然后按“快照”按钮将快照添加到环中。

单击“下一步”还原下一个可用快照。本德尔键删除当前快照。

请参阅下面的热键部分，以访问快速简便的方法来保存和恢复快照环。最终，我们可能会添加安全/便携式/受支持的机制，以从CPEL和vppinfra事件日志文件中保存/恢复快照环。

### 1.10.7. 追逐事件
事件跟踪按发生的最后一个选定事件对跟踪轴进行排序。例如，如果选择一个事件，表示“ CPU上正在运行线程”，则显示的前N个跟踪将是要运行的前M个线程（N <= M；一个线程可能运行不止一次。此功能解决了导致分析的问题通过绘图区域的有限大小。

在标准（NoChaseEvent）模式下，看起来只有BGP线程5和9是活动的：
![](_v_images/20201009161910870_16638.png)

按下ChaseEvent按钮后，我们看到了另一幅图片：
![](_v_images/20201009161923768_19859.png)

### 1.10.8. 埋葬无聊的足迹
序列`<ctrl> <left-mouse-click>`将鼠标下方的轨道移动到轨道集的末尾，从而有效地将其掩埋。序列`<shift> <left-mouse-click>`将鼠标下的轨道移到轨道集的开头。后者的功能可能不完全正确–我认为我们最终可能会提供“撤消”堆栈来提供精确的线程挖掘。

### 1.10.9. 摘要模式
摘要模式通过将事件呈现为短的垂直线段而不是编号的框来使屏幕混乱。事件详细信息显示不受影响。G2以摘要模式启动，并充分缩小以显示轨迹中的所有事件。给定大量事件，摘要模式可将初始屏幕绘制时间减少到可容忍的值。充分放大后，键入“` e`”-进入事件模式，以启用装箱的数字事件显示。

### 1.10.10. 热键
G2支持以下热键操作，根据该功能的原始作者，据说（大约在1996年）类似Quake：

略，[点击查看](https://fd.io/docs/vpp/master/gettingstarted/developers/eventviewer.html)


<br/>

# 2. [VLIB（矢量处理库）](https://fd.io/docs/vpp/master/gettingstarted/developers/vlib.html#vlib-vector-processing-library)

与vlib关联的文件位于`./src/{vlib，vlibapi，vlibmemory}`文件夹中。这些库提供矢量处理支持，包括`图形节点调度`，`可靠的多播支持`，`超轻量协作多任务线程`，`CLI`，`插件.DLL支持`，`物理内存`和`Linux epoll`支持。

详细介绍请参见：[VLIB](https://fd.io/docs/vpp/master/gettingstarted/developers/vlib.html)    ,    [wiki VLIB](https://wiki.fd.io/view/VPP/Software_Architecture)


## 2.1. 初始化函数发现
vlib应用程序通过将结构和`__attribute __（（constructor））`函数放置到映像中来注册各种`[initialization]`事件。在适当的时候，vlib框架会遍历构造函数生成的单链接结构列表，从而调用指示的函数。vlib应用程序使用此机制创建图节点，添加CLI功能，启动协作多任务线程等。

>`__attribute__((constructor))`应该是在main函数之前,执行一个函数,便于我们做一些准备工作.
>The constructor attribute causes the function to be called automatically before execution enters main (). 
>Similarly, the destructor attribute causes the function to be called automatically after main () completes or exit () is called. 
>Functions with these attributes are useful for initializing data that is used implicitly during the execution of the program.

vlib应用程序始终包含许多`VLIB_INIT_FUNCTION（my_init_function）`宏。

每个`init/configure/etc`。函数的返回类型均为`clib_error_t *`。如果一切正常，请确保函数返回0，否则框架将宣布错误并退出。

vlib应用程序必须链接到vppinfra，并且经常链接到其他库，例如`VNET`。在后一种情况下，可能有必要明确引用一个或多个符号，否则库的大部分在运行时可能是AWOL。

## 2.2. 初始化函数构造和约束规范
添加一个init函数很容易：
```c
static clib_error_t *my_init_function (vlib_main_t *vm)
{
  /* ... initialize things ... */

  return 0; // or return clib_error_return (0, "BROKEN!");
}
VLIB_INIT_FUNCTION(my_init_function);
```
如给定的那样，my_init_function将“在某个时候”执行，但是没有顺序保证。

指定排序约束很容易：
```c
VLIB_INIT_FUNCTION(my_init_function) =
{
  .runs_before = VLIB_INITS("we_run_before_function_1",
                            "we_run_before_function_2"),
  .runs_after = VLIB_INITS("we_run_after_function_1",
                           "we_run_after_function_2),
};
```
以“ a然后b然后c然后d”的形式指定批量排序约束也很容易：
```c
VLIB_INIT_FUNCTION(my_init_function) =
{
  .init_order = VLIB_INITS("a", "b", "c", "d"),
};
```

可以为单个init函数指定所有三种排序约束，尽管很难想象为什么会有必要。

## 2.3. Graph Node初始化
vlib数据包处理应用程序总是定义一组图节点来处理数据包。

一个通常通过`VLIB_REGISTER_NODE`宏来构造`vlib_node_registration_t`。在运行时，框架将此类注册集处理为有向图。在运行时将节点添加到图很容易。该框架不支持删除节点。
```c
#define VLIB_REGISTER_NODE(x,...)                                       \
    __VA_ARGS__ vlib_node_registration_t x;                             \
static void __vlib_add_node_registration_##x (void)                     \
    __attribute__((__constructor__)) ;                                  \
static void __vlib_add_node_registration_##x (void)                     \
{                                                                       \
    vlib_main_t * vm = vlib_get_main();                                 \
    x.next_registration = vm->node_main.node_registrations;             \
    vm->node_main.node_registrations = &x;                              \
}                                                                       \
static void __vlib_rm_node_registration_##x (void)                      \
    __attribute__((__destructor__)) ;                                   \
static void __vlib_rm_node_registration_##x (void)                      \
{                                                                       \
    vlib_main_t * vm = vlib_get_main();                                 \
    VLIB_REMOVE_FROM_LINKED_LIST (vm->node_main.node_registrations,     \
                                  &x, next_registration);               \
}                                                                       \
__VA_ARGS__ vlib_node_registration_t x
```
vlib提供了几种类型的矢量处理图节点，主要用于控制框架的调度行为。vlib_node_registration_t的类型成员的功能如下：

* `VLIB_NODE_TYPE_PRE_INPUT`-在所有其他节点类型之前运行
* `VLIB_NODE_TYPE_INPUT`-在pre_input节点之后尽可能频繁地运行
* `VLIB_NODE_TYPE_INTERNAL`-仅在通过添加待处理帧显式地使其可运行时
* `VLIB_NODE_TYPE_PROCESS`-仅在明确使其可运行时。“进程”节点实际上是协作的多任务线程。他们**必须**在相当短的时间内明确暂停。
为了精确了解图节点调度程序，请阅读`... / src / vlib / main.c：vlib_main_loop`。

## 2.4. Graph Node调度程序
`Vlib_main_loop（）`调度图节点。基本的矢量处理算法非常简单，但是即使长时间盯着代码，也可能不太明显。它是这样工作的：
```c
static void
vlib_main_loop (vlib_main_t * vm)
{
  vlib_main_or_worker_loop (vm, /* is_main */ 1);
}
void
vlib_worker_loop (vlib_main_t * vm)
{
  vlib_main_or_worker_loop (vm, /* is_main */ 0);
}
```
* 某些输入节点或一组输入节点产生要处理的工作向量。
* 图节点调度程序将工作向量推过有向图，并根据需要对其进行细分，直到原始工作向量已被完全处理为止。
* 以上过程重复进行。

通过构造，该方案在框架尺寸上产生稳定的平衡。原因如下：随着帧大小的增加，每帧元素的处理时间减少。有几种相关的力量在起作用；最简单的描述是向量处理对`CPU L1 I-cache`的影响。**给定节点处理的第一个帧元素[packet]预热了L1 I缓存中的节点分发功能。**所有后续的框架元素都会获利。随着我们增加框架元素的数量，每个元素的成本下降。

在轻负载下，运行图节点分派器完全是CPU周期的疯狂浪费。因此，如果主要帧大小较小，则图节点调度程序将通过安排在定时epoll等待中来等待工作。该方案具有一定的滞后，以避免在中断和轮询模式之间不断地来回切换。尽管图调度程序支持中断和轮询模式，但我们当前的默认设备驱动程序不支持。

图节点调度程序使用分层计时器轮在计时器到期时重新调度过程节点。

## 2.5. Graph调度器内部
可以安全地跳过此部分。无需了解图调度程序内部即可创建图节点。

## 2.6. 矢量数据结构
在`vpp / vlib`中，我们将**向量**表示为`vlib_frame_t`类型的实例：
```c
typedef struct vlib_frame_t
{
  /* Frame flags. */
  u16 flags;
  /* Number of scalar bytes in arguments. */
  u8 scalar_size;
  /* Number of bytes per vector argument. */
  u8 vector_size;
  /* Number of vector elements currently in frame. */
  u16 n_vectors;
  /* Scalar and vector arguments to next node. */
  u8 arguments[0];
} vlib_frame_t;
```
请注意，可以使用这种结构构造所有类型的向量-包括具有一些相关标量数据的向量。在vpp应用程序中，向量通常使用**4字节**的向量元素大小，并使用零字节的关联每帧标量数据。

帧始终在`CLIB_CACHE_LINE_BYTES`边界上分配。帧具有利用对齐属性的u32索引，因此，帧的最大可行主堆偏移量为`CLIB_CACHE_LINE_BYTES * 0xFFFFFFFF：64 * 4 = 256 GB`。

## 2.7. 调度向量
如您所见，向量并不直接与图节点关联。我们以两种方式表示该关联。最简单的是`vlib_pending_frame_t`：
```c
/* A frame pending dispatch by main loop. */
typedef struct
{
  /* Node and runtime for this frame. */
  u32 node_runtime_index;
  /* Frame index (in the heap). */
  u32 frame_index;
  /* Start of next frames for this node. */
  u32 next_frame_index;
  /* Special value for next_frame_index when there is no next frame. */
#define VLIB_PENDING_FRAME_NO_NEXT_FRAME ((u32) ~0)
} vlib_pending_frame_t;
```
这是…/ src / vlib / main.c：`vlib_main_or_worker_loop（）`中处理帧的代码：
```c
  /*
   * Input nodes may have added work to the pending vector.
   * Process pending vector until there is nothing left.
   * All pending vectors will be processed from input -> output.
   */
  for (i = 0; i < _vec_len (nm->pending_frames); i++)
    cpu_time_now = dispatch_pending_node (vm, i, cpu_time_now);
  /* Reset pending vector for next iteration. */
```
待处理的帧node_runtime_index将帧与将处理该帧的节点相关联。

## 2.8. 并发
**系好安全带。这就是故事-以及数据结构-变得非常复杂的地方...**准备开车了.......

在100,000英尺处：vpp使用**有向图**，而**~~不是有向无环图~~**。
数据包多次访问ip [46]查找真的很正常。最坏的情况：一个图节点将数据包排入其自身。为了解决此问题，如果当前图节点的分配功能碰巧将数据包排回到自身，则图分配器必须强制分配新帧。

无法保证即将处理待处理的帧，这意味着在将其附加到`vlib_pending_frame_t`之后，可能会将更多数据包添加到基础`vlib_frame_t`。如果填充了（pending_frame，frame）"pending:待定"对，则必须小心分配新帧和未决帧。

## 2.9. 下一帧，下一帧所有权
`vlib_next_frame_t`是最后一个关键图调度程序数据结构：
```c
typedef struct
{
  /* Frame index. */
  u32 frame_index;
  /* Node runtime for this next. */
  u32 node_runtime_index;
  /* Next frame flags. */
  u32 flags;
  /* Reflects node frame-used flag for this next. */
#define VLIB_FRAME_NO_FREE_AFTER_DISPATCH \
  VLIB_NODE_FLAG_FRAME_NO_FREE_AFTER_DISPATCH
  /* This next frame owns enqueue to node
     corresponding to node_runtime_index. */
#define VLIB_FRAME_OWNER (1 << 15)
  /* Set when frame has been allocated for this next. */
#define VLIB_FRAME_IS_ALLOCATED	VLIB_NODE_FLAG_IS_OUTPUT
  /* Set when frame has been added to pending vector. */
#define VLIB_FRAME_PENDING VLIB_NODE_FLAG_IS_DROP
  /* Set when frame is to be freed after dispatch. */
#define VLIB_FRAME_FREE_AFTER_DISPATCH VLIB_NODE_FLAG_IS_PUNT
  /* Set when frame has traced packets. */
#define VLIB_FRAME_TRACE VLIB_NODE_FLAG_TRACE
  /* Number of vectors enqueue to this next since last overflow. */
  u32 vectors_since_last_overflow;
} vlib_next_frame_t;
```
图形节点分派函数调用`vlib_get_next_frame（…）`，将“`（u32 *）to_next`”设置到`vlib_frame_t`中与从当前节点到指示的下一个节点的第i个弧（aka next0）相对应的正确位置。

经过一番摸索-达到了两个宏级别-处理到达`vlib_get_next_frame_internal（…）`。Get-next-frame-internal挖掘与所需图形弧相对应的`vlib_next_frame_t`。

下一个帧数据结构相当于一个以图形弧为中心的帧缓存。一旦节点完成了向框架中添加元素，它将获取`vlib_pending_frame_t`并最终到达图调度程序的运行队列。但是，不能保证不会将更多的矢量元素从相同的（source_node，next_index）弧或不同的（source_node，next_index）弧添加到基础框架。

保持弧到帧缓存的一致性是必要的。保持一致性的第一步是确保一次只有一个图节点认为它“拥有”目标`vlib_frame_t`。

返回图节点调度功能。在通常情况下，一定数量的数据包将添加到通过调用`vlib_get_next_frame（…）`获得的`vlib_frame_t`中。

在分派函数返回之前，需要为其实际使用的所有图形弧调用`vlib_put_next_frame（…）`。此操作将`vlib_pending_frame_t`添加到图调度程序的暂挂帧向量。

`Vlib_put_next_frame`在帧索引以及`vlib_next_frame_t`索引的挂起帧中做笔记。

## 2.10. `dispatch_pending_node`调度待定的node操作
主图调度循环调用调度未决节点，如上所示。

Dispatch_pending_node恢复挂起的帧，以及图节点运行时/调度功能。此外，它恢复当前与`vlib_frame_t`关联的`next_frame`，并将vlib_frame_t与next_frame分离。

在…/ src / vlib / main.c：`dispatch_pending_node`（…）中，请注意以下节：
```c
/* Force allocation of new frame while current frame is being
 dispatched. */
restore_frame_index = ~0;
if (nf->frame_index == p->frame_index)
{
  nf->frame_index = ~0;
  nf->flags &= ~VLIB_FRAME_IS_ALLOCATED;
  if (!(n->flags & VLIB_NODE_FLAG_FRAME_NO_FREE_AFTER_DISPATCH))
restore_frame_index = p->frame_index;
}
```
由于它实现了一些二阶优化，因此值得一试。几乎在事后，它会调用dispatch_node，而后者实际上调用了图节点派发函数。


## 2.11. 进程/线程模型
vlib提供了**超轻量级**协作式多任务线程模型。图节点调度程序以与传统的矢量处理运行到完成图节点几乎相同的方式调用这些过程。加上或减去切换堆栈所需的`setjmp / longjmp`对。只需将`vlib_node_registration_t`类型字段设置为`VLIB_NODE_TYPE_PROCESS`。是的，过程是用词不当。这些是协作式多任务线程。
```c

typedef struct _vlib_node_registration
{
  /* Vector processing function for this node. */
  vlib_node_function_t *function;

  /* Node function candidate registration with priority */
  vlib_node_fn_registration_t *node_fn_registrations;

  /* Node name. */
  char *name;

  /* Name of sibling (if applicable). */
  char *sibling_of;

  /* Node index filled in by registration. */
  u32 index;

  /* Type of this node. */
  vlib_node_type_t type;

  /* Error strings indexed by error code for this node. */
  char **error_strings;

  /* Buffer format/unformat for this node. */
  format_function_t *format_buffer;
  unformat_function_t *unformat_buffer;

  /* Trace format/unformat for this node. */
  format_function_t *format_trace;
  unformat_function_t *unformat_trace;

  /* Function to validate incoming frames. */
  u8 *(*validate_frame) (struct vlib_main_t * vm,
			 struct vlib_node_runtime_t *,
			 struct vlib_frame_t * f);

  /* Per-node runtime data. */
  void *runtime_data;

  /* Process stack size. */
  u16 process_log2_n_stack_bytes;

  /* Number of bytes of per-node run time data. */
  u8 runtime_data_bytes;

  /* State for input nodes. */
  u8 state;

  /* Node flags. */
  u16 flags;

  /* protocol at b->data[b->current_data] upon entry to the dispatch fn */
  u8 protocol_hint;

  /* Size of scalar and vector arguments in bytes. */
  u16 scalar_size, vector_size;

  /* Number of error codes used by this node. */
  u16 n_errors;

  /* Number of next node names that follow. */
  u16 n_next_nodes;

  /* Constructor link-list, don't ask... */
  struct _vlib_node_registration *next_registration;

  /* Names of next nodes which this node feeds into. */
  char *next_nodes[];

} vlib_node_registration_t;
```
在撰写本文时，默认的堆栈大小为`2 << 15；32kb`。根据需要初始化节点注册的`process_log2_n_stack_bytes`成员。图节点调度程序会尽力检测堆栈溢出，例如通过在每个线程堆栈下方映射一个无访问页面。

进程节点分派函数应该是“ `while（1）{}`”循环，该循环在不占用其他空间时将挂起，并且不得在不合理的长时间内运行。

“不合理地长”是一个依赖于应用程序的概念。多年来，我们已经构建了帧大小敏感的控制平面节点，当帧大小较小时，它将占用可用CPU带宽的很大一部分。经典示例：修改转发表。只要表构建器使转发表处于有效状态，就可以暂停表构建器以避免由于控制平面活动而丢弃数据包。

流程节点可以挂起固定的时间，或者直到另一个实体发出事件信号为止，或者两者都挂起。有关vlib进程事件机制的说明，请参见下一节。

在vlib进程上下文中运行时，必须严格注意循环不变性问题。如果有人走一个数据结构并调用了一个可能挂起的函数，那么人们最好通过构造知道它不能更改。通常，最好仅制作数据结构的快照副本，悠闲地浏览副本，然后释放副本。

## 2.12. 流程事件
vlib流程事件机制API非常轻巧且易于使用。这是一个典型的例子：

```c
vlib_main_t *vm = &vlib_global_main;
uword event_type, * event_data = 0;

while (1) 
{
   vlib_process_wait_for_event_or_clock (vm, 5.0 /* seconds */);

   event_type = vlib_process_get_events (vm, &event_data);

   switch (event_type) {
   case EVENT1:
       handle_event1s (event_data);
       break;

   case EVENT2:
       handle_event2s (event_data);
       break; 

   case ~0: /* 5-second idle/periodic */
       handle_idle ();
       break;

   default: /* bug! */
       ASSERT (0);
   }

   vec_reset_length(event_data);
} 
```
在此示例中，VLIB进程节点等待事件发生或等待5秒钟。代码对事件类型进行多路分解，并调用适当的处理函数。每次对`vlib_process_get_events`的调用都会返回传递给后续`vlib_process_signal_event`调用的每个事件类型数据的向量；`vec_len（event_data）> = 1`。
```c
/** Return the first event type which has occurred and a vector of per-event
    data of that type, or a timeout indication

    @param vm - vlib_main_t pointer
    @param data_vector - pointer to a (uword *) vector to receive event data
    @returns either an event type and a vector of per-event instance data,
    or ~0 to indicate a timeout.
*/
always_inline uword
vlib_process_get_events (vlib_main_t * vm, uword ** data_vector)
{
  vlib_node_main_t *nm = &vm->node_main;
  vlib_process_t *p;
  vlib_process_event_type_t *et;
  uword r, t, l;

  p = vec_elt (nm->processes, nm->current_process_index);

  /* Find first type with events ready.
     Return invalid type when there's nothing there. */
  t = clib_bitmap_first_set (p->non_empty_event_type_bitmap);
  if (t == ~0)
    return t;

  p->non_empty_event_type_bitmap =
    clib_bitmap_andnoti (p->non_empty_event_type_bitmap, t);

  l = _vec_len (p->pending_event_data_by_type_index[t]);
  if (data_vector)
    vec_add (*data_vector, p->pending_event_data_by_type_index[t], l);
  _vec_len (p->pending_event_data_by_type_index[t]) = 0;

  et = pool_elt_at_index (p->event_type_pool, t);

  /* Return user's opaque value. */
  r = et->opaque;

  vlib_process_maybe_free_event_type (p, t);

  return r;
}
```
仅处理`event_data [0]`是错误的。

将event_data向量的长度重置为0（而不是调用vec_free）意味着事件方案不会燃烧连续分配和释放事件数据向量的周期。这是常见的`vppinfra / vlib`编码模式，值得在适当时使用。

发出事件信号很容易，例如：
```c
vlib_process_signal_event (vm, process_node_index, EVENT1,
    (uword)arbitrary_event1_data); /* and so forth 依此类推*/
```
可以通过构造了解过程节点索引-从适当的`vlib_node_registration_t`中挖掘出来-或通过使用`vlib_get_node_by_name`（...）查找`vlib_node_t`。

## 2.13. Buffers

尽管具有高性能，但**vlib缓冲**解决了常见的数据包处理问题。性能方面的关键：通常一次分配/释放N个缓冲区，而不是一次分配/释放N个缓冲区。除了直接在特定缓冲区上操作时，缓冲区是按索引而不是指针处理缓冲区的。

数据包处理帧实际上是`u32 []`，而不是`vlib_buffer_t []`。

数据包包含一个或多个vlib缓冲区，根据需要链接在一起。支持多种粒径；硬件输入节点仅要求提供所需的大小即可。提供合并支持。出于明显的原因，我们不鼓励编写自己的狂野古怪的缓冲区链遍历代码。

vlib缓冲区标头在缓冲区数据区域之前立即分配。在典型的数据包处理中，这样可以节省相关的读取等待时间：给定缓冲区的地址，就可以在缓冲区数据的第一条高速缓存行同时预取缓冲区头[metadata]。

缓冲区标头元数据（`vlib_buffer_t`）包括通常的重写扩展空间，current_data偏移量，RX和TX接口索引，数据包跟踪信息以及不透明区域。

不透明数据旨在以依赖于子图的任意方式控制数据包处理。程序员负责数据寿命分析，类型检查等。

缓冲区具有参考计数以支持例如多播复制。

## 2.14. 共享内存消息API
本地控制平面和应用程序进程通过单向队列中共享内存中的异步消息传递与vpp数据平面交互。可以通过套接字使用相同的应用程序API。

捕获API跟踪并在模拟环境中重播它们需要一种严格的方法来解决问题。这似乎是一项工作任务，但并非如此。在300,000或3,000,000次操作后，如果控制面板中出现故障，则高速重放导致事故的事件将是一个巨大的胜利。

共享内存消息API消息分配器`vl_api_msg_alloc`使用了一个特别可爱的技巧。由于消息是按顺序处理的，因此我们尝试从一组固定大小的预分配的环中分配消息缓冲。每个环项目都有一个“忙”位。释放预分配的消息缓冲区之一仅需要消息使用者清除忙位。无需锁定。

## 2.15. Plug-ins插件
vlib实现了一种简单的插件DLL机制。VLIB客户端应用程序指定目录以搜索插件.DLL，并指定要应用的名称过滤器（如果需要）。VLIB需要非常早地加载插件。

加载后，插件DLL机制使用dlsym在新加载的插件中查找和验证vlib_plugin_registration数据结构。

## 2.16. 调试命令行
向VLIB应用程序添加调试CLI命令非常简单。

这是一个完整的示例：
```c
static clib_error_t *
show_ip_tuple_match (vlib_main_t * vm,
                     unformat_input_t * input,
                     vlib_cli_command_t * cmd)
{
    vlib_cli_output (vm, "%U\n", format_ip_tuple_match_tables, &routing_main);
    return 0;
}

static VLIB_CLI_COMMAND (show_ip_tuple_command) = {
    .path = "show ip tuple match",
    .short_help = "Show ip 5-tuple match-and-broadcast tables",
    .function = show_ip_tuple_match,
};
```
本示例实现了“ `show ip tuple match`” debug cli命令。通常，可通过“ `vppctl`”应用程序访问vlib cli，该应用程序将流量发送到命名管道。可以在可配置的端口上配置调试CLI telnet访问。

cli实现具有输出重定向功能，可轻松通过共享内存API消息传递cli输出，

特别是对于调试或“显示技术支持”类型的命令，编写vlib应用程序代码以打包二进制数据，在其他地方编写更多代码以解压缩数据并最终打印答案将非常浪费。如果某个cli命令可能由于运行时间过长而有可能损害数据包处理性能，请在流程节点上逐步进行工作。客户可以等待。

## 2.17. 在线程之间传递缓冲区
Vlib包括一种易于使用的机制，用于在工作线程之间切换缓冲区。典型的用例：**软件入口流哈希**。在较高的级别上，创建了一个每个工作线程的队列，该队列将数据包发送到指定工作线程中的特定图形节点。有了队列，将数据包排队到您选择的工作线程中。

## 2.18. 初始化切换队列
足够简单，调用`vlib_frame_queue_main_init`：
```c
main_ptr->frame_queue_index
   = vlib_frame_queue_main_init (dest_node.index, frame_queue_size);
```
Frame_queue_size表示其含义：可以排队的帧数。由于帧包含1…256个数据包，因此frame_queue_size应该是一个相当小的数字（32…64）。如果帧队列生产者的速度比帧队列消费者的速度快，则会发生拥塞。建议让排队操作员处理队列拥塞，如下面的排队示例所示。

在地板下，`vlib_frame_queue_main_init`为每个辅助线程创建一个输入队列。

在明确使用它们之前，请不要创建帧队列。尽管主调度循环在轮询（整组）帧队列的频率方面相当聪明，但是轮询未使用的帧队列却浪费了时钟周期。

## 2.19. 交递数据包
实际的切换机制很简单，并且可以与典型的图节点调度功能很好地集成在一起：
```c
always_inline uword
do_handoff_inline (vlib_main_t * vm,
     	       vlib_node_runtime_t * node, vlib_frame_t * frame,
		       int is_ip4, int is_trace)
{
  u32 n_left_from, *from;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b;
  u16 thread_indices [VLIB_FRAME_SIZE];
  u16 nexts[VLIB_FRAME_SIZE], *next;
  u32 n_enq;
  htest_main_t *hmp = &htest_main;
  int i;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;

  vlib_get_buffers (vm, from, bufs, n_left_from);
  next = nexts;
  b = bufs;

  /*
   * Typical frame traversal loop, details vary with
   * use case. Make sure to set thread_indices[i] with
   * the desired destination thread index. You may
   * or may not bother to set next[i].
   */

  for (i = 0; i < frame->n_vectors; i++)
    {
      <snip>
      /* Pick a thread to handle this packet */
      thread_indices[i] = f (packet_data_or_whatever);
      <snip>

      b += 1;
      next += 1;
      n_left_from -= 1;
    }

   /* Enqueue buffers to threads */
   n_enq =
    vlib_buffer_enqueue_to_thread (vm, hmp->frame_queue_index,
                                   from, thread_indices, frame->n_vectors,
                                   1 /* drop on congestion */);
   /* Typical counters,
  if (n_enq < frame->n_vectors)
    vlib_node_increment_counter (vm, node->node_index,
				 XXX_ERROR_CONGESTION_DROP,
				 frame->n_vectors - n_enq);
  vlib_node_increment_counter (vm, node->node_index,
			         XXX_ERROR_HANDED_OFF, n_enq);
  return frame->n_vectors;
}
```
有关调用`vlib_buffer_enqueue_to_thread`（…）的说明：

* 如果传递的“丢弃时阻塞”为非零值，则入站帧中的所有数据包将以一种或另一种方式消耗。这是推荐设置。
* 在拥塞情况下，请不要尝试通过释放丢弃的数据包或将其推入“错误丢弃”的方式在入队节点中“提供帮助”。这些动作中的任何一个都是严重的错误。
* 将数据包排队到当前线程是完全可以的。

<br/>

# 3. [Plugins插件](https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/plugins.html)
vlib实现了一种简单的插件DLL机制。VLIB客户端应用程序指定目录以搜索插件.DLL，并指定要应用的名称过滤器（如果需要）。VLIB需要非常早地加载插件。

加载后，插件DLL机制使用dlsym在新加载的插件中查找和验证`vlib_plugin_registration`数据结构。
```c
/* *INDENT-OFF* */
typedef CLIB_PACKED(struct {
  u8 default_disabled;
  const char version[32];
  const char version_required[32];
  const char overrides[256];
  const char *early_init;
  const char *description;
}) vlib_plugin_registration_t;
```
有关插件的更多信息，请参阅[添加插件](https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/add_plugin.html#add-plugin)。

**`TODO`**

<br/>


# 4. [VNET（VPP网络堆栈）](https://fd.io/docs/vpp/master/gettingstarted/developers/vnet.html#vnet-vpp-network-stack)

与VPP网络堆栈层关联的文件位于 `./src/vnet`文件夹中。网络堆栈层基本上是其他层中代码的实例化。该层具有vnet库，该库提供矢量化的第2层和3个网络图节点，数据包生成器和数据包跟踪器。

在构建数据包处理应用程序方面，vnet提供了独立于平台的子图，一个子图连接了两个设备驱动程序节点。

典型的RX连接包括“以太网输入”（完整的软件分类，提供`ipv4-input`，`ipv6-input`，`arp-input`等）和“ `ipv4-input-no-checksum`” [如果硬件可以分类，请执行ipv4标头校验和] 。

详细介绍请参见：[VNET](https://fd.io/docs/vpp/master/gettingstarted/developers/vnet.html)   ,   [wiki VLIB](https://wiki.fd.io/view/VPP/Software_Architecture)

## 4.1. 有效的图调度功能编码
在过去的15年中，出现了两种不同的样式：单/双/四循环编码模型和全流水线编码模型。我们很少使用完全流水线的编码模型，因此我们不会对其进行详细描述

## 4.2. 单/双循环
单/双/四循环模型是方便解决事先不知道要处理的项目数量的问题的唯一方法：典型的硬件RX环处理。当给定节点不需要覆盖一组复杂的相关读取时，这种编码样式也非常有效。

这是一个四/单循环，可以利用多达avx512 SIMD向量单元将缓冲区索引转换为缓冲区指针：
```c
static uword
simulated_ethernet_interface_tx (vlib_main_t * vm,
				 vlib_node_runtime_t *
				 node, vlib_frame_t * frame)
{
 u32 n_left_from, *from;
 u32 next_index = 0;
 u32 n_bytes;
 u32 thread_index = vm->thread_index;
 vnet_main_t *vnm = vnet_get_main ();
 vnet_interface_main_t *im = &vnm->interface_main;
 vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b;
 u16 nexts[VLIB_FRAME_SIZE], *next;

 n_left_from = frame->n_vectors;
 from = vlib_frame_vector_args (frame);

 /*
  * Convert up to VLIB_FRAME_SIZE indices in "from" to
  * buffer pointers in bufs[]
  */
 vlib_get_buffers (vm, from, bufs, n_left_from);
 b = bufs;
 next = nexts;

 /*
  * While we have at least 4 vector elements (pkts) to process..
  */
 while (n_left_from >= 4)
   {
     /* Prefetch next quad-loop iteration. */
     if (PREDICT_TRUE (n_left_from >= 8))
	   {
	     vlib_prefetch_buffer_header (b[4], STORE);
	     vlib_prefetch_buffer_header (b[5], STORE);
	     vlib_prefetch_buffer_header (b[6], STORE);
	     vlib_prefetch_buffer_header (b[7], STORE);
       }

     /*
      * $$$ Process 4x packets right here...
      * set next[0..3] to send the packets where they need to go
      */

      do_something_to (b[0]);
      do_something_to (b[1]);
      do_something_to (b[2]);
      do_something_to (b[3]);

     /* Process the next 0..4 packets */
	 b += 4;
	 next += 4;
	 n_left_from -= 4;
	}
 /*
  * Clean up 0...3 remaining packets at the end of the incoming frame
  */
 while (n_left_from > 0)
   {
     /*
      * $$$ Process one packet right here...
      * set next[0..3] to send the packets where they need to go
      */
      do_something_to (b[0]);

     /* Process the next packet */
     b += 1;
     next += 1;
     n_left_from -= 1;
   }

 /*
  * Send the packets along their respective next-node graph arcs
  * Considerable locality of reference is expected, most if not all
  * packets in the inbound vector will traverse the same next-node
  * arc
  */
 vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);

 return frame->n_vectors;
}
```
给定要执行的数据包处理任务，需要寻找周围的类似任务，并考虑使用相同的编码模式。在性能优化过程中，多次重新编码给定的图节点分配函数并不罕见。

## 4.3. 从头开始创建数据包
有时，有必要从头创建数据包并将其发送。诸如发送`keepalive`或主动打开连接之类的任务浮现在脑海。这并不困难，但是需要准确的缓冲区元数据设置。

## 4.4. 分配缓冲区
使用`vlib_buffer_alloc`，它分配一组缓冲区索引。对于性能低下的应用程序，可以一次分配一个缓冲区。请注意，vlib_buffer_alloc（…）不会初始化缓冲区元数据。见下文。
```c
/** \brief Allocate buffers into supplied array

    @param vm - (vlib_main_t *) vlib main data structure pointer
    @param buffers - (u32 * ) buffer index array
    @param n_buffers - (u32) number of buffers requested
    @return - (u32) number of buffers actually allocated, may be
    less than the number requested or zero
*/
always_inline u32
vlib_buffer_alloc (vlib_main_t * vm, u32 * buffers, u32 n_buffers)
{
  return vlib_buffer_alloc_on_numa (vm, buffers, n_buffers, vm->numa_node);
}
```
在NUMA架构上申请，这个有意思了
```c
/** \brief Allocate buffers from specific numa node into supplied array

    @param vm - (vlib_main_t *) vlib main data structure pointer
    @param buffers - (u32 * ) buffer index array
    @param n_buffers - (u32) number of buffers requested
    @param numa_node - (u32) numa node
    @return - (u32) number of buffers actually allocated, may be
    less than the number requested or zero
*/
always_inline u32
vlib_buffer_alloc_on_numa (vlib_main_t * vm, u32 * buffers, u32 n_buffers,
			   u32 numa_node)
{
  u8 index = vlib_buffer_pool_get_default_for_numa (vm, numa_node);
  return vlib_buffer_alloc_from_pool (vm, buffers, n_buffers, index);
}
```
```c
always_inline u8
vlib_buffer_pool_get_default_for_numa (vlib_main_t * vm, u32 numa_node)
{
  ASSERT (numa_node < VLIB_BUFFER_MAX_NUMA_NODES);
  return vm->buffer_main->default_buffer_pool_index_for_numa[numa_node];
}
```
这个NUMA变量如下：
```c
u8 default_buffer_pool_index_for_numa[VLIB_BUFFER_MAX_NUMA_NODES];
```
然后是：
```c
/** \brief Allocate buffers from specific pool into supplied array

    @param vm - (vlib_main_t *) vlib main data structure pointer
    @param buffers - (u32 * ) buffer index array
    @param n_buffers - (u32) number of buffers requested
    @return - (u32) number of buffers actually allocated, may be
    less than the number requested or zero
*/

always_inline u32
vlib_buffer_alloc_from_pool (vlib_main_t * vm, u32 * buffers, u32 n_buffers,
			     u8 buffer_pool_index);
```
在高性能情况下，分配一个缓冲区索引向量，并将其从向量末尾分发出去；分配缓冲区索引时，递减_vec_len（..）。有关示例，请参见`tcp_alloc_tx_buffers（…）`和`tcp_get_free_buffer_index（…）`。

## 4.5. 缓冲区初始化示例
以下示例显示了要点，但不要盲目粘贴。
```c
u32 bi0;
vlib_buffer_t *b0;
ip4_header_t *ip;
udp_header_t *udp;

/* Allocate a buffer */
if (vlib_buffer_alloc (vm, &bi0, 1) != 1)
return -1;

b0 = vlib_get_buffer (vm, bi0);

/* Initialize the buffer */
VLIB_BUFFER_TRACE_TRAJECTORY_INIT (b0);

/* At this point b0->current_data = 0, b0->current_length = 0 */

/*
* Copy data into the buffer. This example ASSUMES that data will fit
* in a single buffer, and is e.g. an ip4 packet.
*/
if (have_packet_rewrite)
 {
   clib_memcpy (b0->data, data, vec_len (data));
   b0->current_length = vec_len (data);
 }
else
 {
   /* OR, build a udp-ip packet (for example) */
   ip = vlib_buffer_get_current (b0);
   udp = (udp_header_t *) (ip + 1);
   data_dst = (u8 *) (udp + 1);

   ip->ip_version_and_header_length = 0x45;
   ip->ttl = 254;
   ip->protocol = IP_PROTOCOL_UDP;
   ip->length = clib_host_to_net_u16 (sizeof (*ip) + sizeof (*udp) +
              vec_len(udp_data));
   ip->src_address.as_u32 = src_address->as_u32;
   ip->dst_address.as_u32 = dst_address->as_u32;
   udp->src_port = clib_host_to_net_u16 (src_port);
   udp->dst_port = clib_host_to_net_u16 (dst_port);
   udp->length = clib_host_to_net_u16 (vec_len (udp_data));
   clib_memcpy (data_dst, udp_data, vec_len(udp_data));

   if (compute_udp_checksum)
     {
       /* RFC 7011 section 10.3.2. */
       udp->checksum = ip4_tcp_udp_compute_checksum (vm, b0, ip);
       if (udp->checksum == 0)
         udp->checksum = 0xffff;
  }
  b0->current_length = vec_len (sizeof (*ip) + sizeof (*udp) +
                               vec_len (udp_data));

}
b0->flags |= (VLIB_BUFFER_TOTAL_LENGTH_VALID;

/* sw_if_index 0 is the "local" interface, which always exists */
vnet_buffer (b0)->sw_if_index[VLIB_RX] = 0;

/* Use the default FIB index for tx lookup. Set non-zero to use another fib */
vnet_buffer (b0)->sw_if_index[VLIB_TX] = 0;
```
如果您的用例需要大数据包传输，请使用`vlib_buffer_chain_append_data_with_alloc（…）`创建必需的缓冲区链。
```c
u16
vlib_buffer_chain_append_data_with_alloc (vlib_main_t * vm,
					  vlib_buffer_t * first,
					  vlib_buffer_t ** last, void *data,
					  u16 data_len)
{
  vlib_buffer_t *l = *last;
  u32 n_buffer_bytes = vlib_buffer_get_default_data_size (vm);
  u16 copied = 0;
  ASSERT (n_buffer_bytes >= l->current_length + l->current_data);
  while (data_len)
    {
      u16 max = n_buffer_bytes - l->current_length - l->current_data;
      if (max == 0)
	{
	  if (1 != vlib_buffer_alloc_from_pool (vm, &l->next_buffer, 1,
						first->buffer_pool_index))
	    return copied;
	  *last = l = vlib_buffer_chain_buffer (vm, l, l->next_buffer);
	  max = n_buffer_bytes - l->current_length - l->current_data;
	}

      u16 len = (data_len > max) ? max : data_len;
      clib_memcpy_fast (vlib_buffer_get_current (l) + l->current_length,
			data + copied, len);
      vlib_buffer_chain_increase_length (first, l, len);
      data_len -= len;
      copied += len;
    }
  return copied;
}
```
## 4.6. 使数据包入队以进行查找和传输
发送一组数据包的最简单方法是使用`vlib_get_frame_to_node（…）`将新帧分配给`ip4_lookup_node`或`ip6_lookup_node`，添加构造的缓冲区索引，然后使用`vlib_put_frame_to_node（…）`分发帧。
```c
vlib_frame_t *f;
f = vlib_get_frame_to_node (vm, ip4_lookup_node.index);
f->n_vectors = vec_len(buffer_indices_to_send);
to_next = vlib_frame_vector_args (f);

for (i = 0; i < vec_len (buffer_indices_to_send); i++)
  to_next[i] = buffer_indices_to_send[i];

vlib_put_frame_to_node (vm, ip4_lookup_node_index, f);
```
分配和调度单个分组帧效率低下。这是在典型情况下，你需要每秒发送一个数据包，但应该不是在一个for循环发生！



## 4.7. 封包追踪器
Vlib包括一个带有简单vlib cli接口的框架元素[packet]跟踪工具。cli很简单：“ `trace add <input-node-name> <count>`”。

要在运行dpdk插件的典型x86_64系统上跟踪100个数据包，请执行以下操作：“ trace add dpdk-input 100”。使用数据包生成器时：“ trace add pg-input 100”

每个图节点都有机会捕获自己的跟踪数据。这样做总是一个好主意。跟踪捕获API很简单。

数据包捕获API对二进制数据进行快照，以最大程度地减少捕获时的处理。每个参与的图形节点初始化都提供了vppinfra格式样式的用户功能，以在VLIB“ show trace”命令要求时漂亮地打印数据。

将VLIB节点注册“ .format_trace”成员设置为每图节点格式功能的名称。

这是一个简单的例子：
```c
u8 * my_node_format_trace (u8 * s, va_list * args)
{
    vlib_main_t * vm = va_arg (*args, vlib_main_t *);
    vlib_node_t * node = va_arg (*args, vlib_node_t *);
    my_node_trace_t * t = va_arg (*args, my_trace_t *);
    s = format (s, "My trace data was: %d", t-><whatever>);
    return s;
}
```
跟踪框架将每个节点的格式功能传递给数据包旋转时捕获的数据。格式化功能可根据需要漂亮地打印数据。

## 4.8. 图调度程序Pcap跟踪
vpp图形分派器知道如何在分派时捕获pcap格式的数据包矢量。pcap捕获如下：
```
    VPP graph dispatch trace record description:

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Major Version | Minor Version | NStrings      | ProtoHint     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer index (big endian)                                     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       + VPP graph node name ...     ...               | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Metadata ... ...                       | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Opaque ... ...                         | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Opaque 2 ... ...                       | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | VPP ASCII packet trace (if NStrings > 4)      | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Packet data (up to 16K)                                       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
图调度记录包括版本标记，指示将在记录头和数据包数据之前跟随多少以NULL结尾的字符串的指示以及协议提示。

缓冲区索引是一个不透明的`32位cookie`，它使这些数据的使用者在遍历转发图时可以轻松地过滤/跟踪单个数据包。

每个数据包有多个记录是正常现象，这是可以预期的。数据包遍历vpp转发图时，将显示倍增时间。这样，vpp图调度跟踪与来自终端站的常规网络数据包捕获显着不同。此属性使状态数据包分析复杂化。

将状态分析限制为来自单个vpp图节点（例如“以太网输入”）的记录似乎可以改善这种情况。

在撰写本文时：主要版本= 1，次要版本=0。Nstrings应该为4或5。消费者应保持警惕的值小于4或大于5。他们可以尝试显示声明的字符串数，或者可以对待条件为错误。

这是当前的协议提示集：
```c
typedef enum
{
    VLIB_NODE_PROTO_HINT_NONE = 0,
    VLIB_NODE_PROTO_HINT_ETHERNET,
    VLIB_NODE_PROTO_HINT_IP4,
    VLIB_NODE_PROTO_HINT_IP6,
    VLIB_NODE_PROTO_HINT_TCP,
    VLIB_NODE_PROTO_HINT_UDP,
    VLIB_NODE_N_PROTO_HINTS,
} vlib_node_proto_hint_t;
```
示例：`VLIB_NODE_PROTO_HINT_IP6`表示数据包数据的第一个八位字节应为0x60，并应以ipv6数据包头开头。

这些数据的下游使用者应注意协议提示。他们必须容忍不正确的提示，这些提示可能会不时出现。


## 4.9. 调度Pcap跟踪调试CLI
要开始对多达10,000条跟踪记录进行调度跟踪捕获：
```
pcap dispatch trace on max 10000 file dispatch.pcap
```
要启动调度跟踪，该跟踪还将包括对源自dpdk-input的数据包的标准vpp数据包跟踪：
```
pcap dispatch trace on max 10000 file dispatch.pcap buffer-trace dpdk-input 1000
```
要保存pcap跟踪，例如在/tmp/dispatch.pcap中：
```
pcap dispatch trace off
```


## 4.10. Wireshark剖析调度pcap痕迹

不用说，我们建立了一个伴生的`Wireshk解剖器`来显示这些痕迹。在撰写本文时，我们已将Wireshk解剖器上游。

由于`Wireshark / master / latest`进入所有流行的Linux发行版还需要一段时间，因此请参阅**“如何构建可识别vpp调度跟踪的Wireshark”**页面以获取构建信息。（见下章节）

这是数据包解剖示例，为清楚起见，省略了一些字段。关键是，wireshark剖析器可以准确显示所有vpp缓冲区元数据，以及相关图形节点的名称。
```
Frame 1: 2216 bytes on wire (17728 bits), 2216 bytes captured (17728 bits)
    Encapsulation type: USER 13 (58)
    [Protocols in frame: vpp:vpp-metadata:vpp-opaque:vpp-opaque2:eth:ethertype:ip:tcp:data]
VPP Dispatch Trace
    BufferIndex: 0x00036663
NodeName: ethernet-input
VPP Buffer Metadata
    Metadata: flags:
    Metadata: current_data: 0, current_length: 102
    Metadata: current_config_index: 0, flow_id: 0, next_buffer: 0
    Metadata: error: 0, n_add_refs: 0, buffer_pool_index: 0
    Metadata: trace_index: 0, recycle_count: 0, len_not_first_buf: 0
    Metadata: free_list_index: 0
    Metadata:
VPP Buffer Opaque
    Opaque: raw: 00000007 ffffffff 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
    Opaque: sw_if_index[VLIB_RX]: 7, sw_if_index[VLIB_TX]: -1
    Opaque: L2 offset 0, L3 offset 0, L4 offset 0, feature arc index 0
    Opaque: ip.adj_index[VLIB_RX]: 0, ip.adj_index[VLIB_TX]: 0
    Opaque: ip.flow_hash: 0x0, ip.save_protocol: 0x0, ip.fib_index: 0
    Opaque: ip.save_rewrite_length: 0, ip.rpf_id: 0
    Opaque: ip.icmp.type: 0 ip.icmp.code: 0, ip.icmp.data: 0x0
    Opaque: ip.reass.next_index: 0, ip.reass.estimated_mtu: 0
    Opaque: ip.reass.fragment_first: 0 ip.reass.fragment_last: 0
    Opaque: ip.reass.range_first: 0 ip.reass.range_last: 0
    Opaque: ip.reass.next_range_bi: 0x0, ip.reass.ip6_frag_hdr_offset: 0
    Opaque: mpls.ttl: 0, mpls.exp: 0, mpls.first: 0, mpls.save_rewrite_length: 0, mpls.bier.n_bytes: 0
    Opaque: l2.feature_bitmap: 00000000, l2.bd_index: 0, l2.l2_len: 0, l2.shg: 0, l2.l2fib_sn: 0, l2.bd_age: 0
    Opaque: l2.feature_bitmap_input:   none configured, L2.feature_bitmap_output:   none configured
    Opaque: l2t.next_index: 0, l2t.session_index: 0
    Opaque: l2_classify.table_index: 0, l2_classify.opaque_index: 0, l2_classify.hash: 0x0
    Opaque: policer.index: 0
    Opaque: ipsec.flags: 0x0, ipsec.sad_index: 0
    Opaque: map.mtu: 0
    Opaque: map_t.v6.saddr: 0x0, map_t.v6.daddr: 0x0, map_t.v6.frag_offset: 0, map_t.v6.l4_offset: 0
    Opaque: map_t.v6.l4_protocol: 0, map_t.checksum_offset: 0, map_t.mtu: 0
    Opaque: ip_frag.mtu: 0, ip_frag.next_index: 0, ip_frag.flags: 0x0
    Opaque: cop.current_config_index: 0
    Opaque: lisp.overlay_afi: 0
    Opaque: tcp.connection_index: 0, tcp.seq_number: 0, tcp.seq_end: 0, tcp.ack_number: 0, tcp.hdr_offset: 0, tcp.data_offset: 0
    Opaque: tcp.data_len: 0, tcp.flags: 0x0
    Opaque: sctp.connection_index: 0, sctp.sid: 0, sctp.ssn: 0, sctp.tsn: 0, sctp.hdr_offset: 0
    Opaque: sctp.data_offset: 0, sctp.data_len: 0, sctp.subconn_idx: 0, sctp.flags: 0x0
    Opaque: snat.flags: 0x0
    Opaque:
VPP Buffer Opaque2
    Opaque2: raw: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
    Opaque2: qos.bits: 0, qos.source: 0
    Opaque2: loop_counter: 0
    Opaque2: gbp.flags: 0, gbp.src_epg: 0
    Opaque2: pg_replay_timestamp: 0
    Opaque2:
Ethernet II, Src: 06:d6:01:41:3b:92 (06:d6:01:41:3b:92), Dst: IntelCor_3d:f6    Transmission Control Protocol, Src Port: 22432, Dst Port: 54084, Seq: 1, Ack: 1, Len: 36
    Source Port: 22432
    Destination Port: 54084
    TCP payload (36 bytes)
Data (36 bytes)

0000  cf aa 8b f5 53 14 d4 c7 29 75 3e 56 63 93 9d 11   ....S...)u>Vc...
0010  e5 f2 92 27 86 56 4c 21 ce c5 23 46 d7 eb ec 0d   ...'.VL!..#F....
0020  a8 98 36 5a                                       ..6Z
    Data: cfaa8bf55314d4c729753e5663939d11e5f2922786564c21…
    [Length: 36]
```
只需在Wireshark中单击几次鼠标即可将跟踪过滤到特定的缓冲区索引。通过这种特定的过滤方式，人们可以观察到数据包在转发图中的走动。注意任何/所有元数据更改，标头校验和更改等。

在开发新的vpp图节点时，这将具有重要价值。如果新的代码将`b-> current_data`放错了位置，那么通过查看wirehark中的调度跟踪将完全显而易见。


## 4.11. 如何构建可识别vpp调度跟踪的Wireshark
[如何构建可识别vpp调度跟踪的Wireshark](https://fd.io/docs/vpp/master/gettingstarted/developers/buildwireshark.html)

vpp pcap调度跟踪解剖器已合并到wireshark主分支中，因此过程很简单。下载wireshark，进行编译并安装。

**下载wireshark源代码**
`Wireshark git repo`很大，因此需要一段时间才能克隆。
```bash
git clone https://code.wireshark.org/review/wireshark
```
**安装必备软件包**
这是在Ubuntu 18.04系统上通常安装的必备软件包的列表，它们必须存在才能编译Wireshark：
```
libgcrypt11-dev flex bison qtbase5-dev qttools5-dev-tools qttools5-dev
    qtmultimedia5-dev libqt5svg5-dev libpcap-dev qt5-default
```
**编译Wireshark**
幸运的是，Wireshark使用cmake，因此至少在Ubuntu 18.04上相对容易构建。
```
$ cd wireshark
$ mkdir build
$ cd build
$ cmake -G Ninja ../
$ ninja -j 8
$ sudo ninja install
```
**进行pcap调度跟踪**
配置vpp以某种方式传递流量，然后：
```
vpp# pcap dispatch trace on max 10000 file vppcapture buffer-trace dpdk-input 1000
```
或类似。运行流量足够长的时间以捕获一些数据。像这样保存调度跟踪捕获：
```
vpp# pcap dispatch trace off
```
**在Wireshark中显示**
在wireshark的启用vpp的版本中显示`/ tmp / vppcapture`。运气好的话，wireshark的普通版本将拒绝处理vpp调度跟踪pcap文件，因为它们不了解encap类型。

将wireshark设置为对vpp.bufferindex进行过滤，以观察单个数据包遍历转发图。否则，您将在ip4-lookup中看到数据包向量，然后在ip4-rewrite中看到数据包向量，等等。





# 5. [Feature Arcs](https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/featurearcs.html)
可以在每个接口或每个系统的基础上配置大量的vpp功能。我们没有要求特征编码器手动构造所需的`图弧`，而是建立了一种通用的机制来管理这些机制。

具体来说，特征弧包括图节点的有序集合。弧中的每个特征节点都是独立控制的。要素弧节点通常不会相互意识到。将数据包交给“下一个功能节点”非常便宜。

特征弧实现解决了创建用于转向的图形弧的问题。

在特征弧的开头，需要进行一些设置工作，但前提是必须在弧上启用至少一个特征。

在每个弧度的基础上，各个要素定义会创建一组排序依存关系。功能基础结构执行排序依存关系的拓扑排序，以确定实际的功能顺序。缺少依赖项将导致运行时混乱。有关示例，请参见[https://gerrit.fd.io/r/#/c/12753](https://gerrit.fd.io/r/#/c/12753)。

如果不存在部分订单，则vpp将拒绝运行。“ a然后b，b然后c，c然后a”形式的循环依赖循环是无法满足的。

## 5.1. 向现有要素弧添加要素
没人惊讶的是，我们使用典型的“宏->构造函数->声明列表”模式设置特征弧：
```c
VNET_FEATURE_INIT (mactime, static) =
{
  .arc_name = "device-input",
  .node_name = "mactime",
  .runs_before = VNET_FEATURES ("ethernet-input"),
}; 
```
这在“设备输入”弧上创建了“ mactime”功能。

每帧一次，挖掘对应于“设备输入”功能弧的vnet_feature_config_main_t：
```c
vnet_main_t *vnm = vnet_get_main ();
vnet_interface_main_t *im = &vnm->interface_main;
u8 arc = im->output_feature_arc_index;
vnet_feature_config_main_t *fcm;

fcm = vnet_feature_get_config_main (arc);
```
请注意，在这种情况下，我们已将所需的弧索引（由要素基础结构分配）存储在vnet_interface_main_t中。创建特征弧时，程序员将决定放置弧索引的位置。

对于每个数据包，设置next0以将数据包引导到他们应访问的下一个节点：
```c
vnet_get_config_data (&fcm->config_main,
                      &b0->current_config_index /* value-result */, 
                      &next0, 0 /* # bytes of config data */);
```
配置数据是按功能的弧，通常不使用。注意，重置next0将数据包转移到其他地方是正常的。通常，将其丢弃是有原因的：
```c
ext0 = MACTIME_NEXT_DROP;
b0->error = node->errors[DROP_CAUSE];
```
## 5.2. 创建特征弧
再一次，我们使用构造函数宏创建特征弧：
```c
VNET_FEATURE_ARC_INIT (ip4_unicast, static) =
{
  .arc_name = "ip4-unicast",
  .start_nodes = VNET_FEATURES ("ip4-input", "ip4-input-no-checksum"),
  .arc_index_ptr = &ip4_main.lookup_main.ucast_feature_arc_index,
}; 
```
在这种情况下，我们配置两个弧起始节点来处理“是否经过硬件验证的ip校验和”情况。在初始化期间，要素基础设施将存储弧索引，如下所示。

在弧头节点中，执行以下操作以沿特征弧发送数据包：
```c
ip_lookup_main_t *lm = &im->lookup_main;
arc = lm->ucast_feature_arc_index;
```
个数据包一次，初始化数据包元数据以沿特征弧移动：
```c
vnet_feature_arc_start (arc, sw_if_index0, &next, b0);
```

## 5.3. 启用/禁用功能
只需调用vnet_feature_enable_disable即可启用或禁用特定功能：
```c
vnet_feature_enable_disable ("device-input", /* arc name */
                             "mactime",      /* feature name */
        		             sw_if_index,    /* Interface sw_if_index */
                             enable_disable, /* 1 => enable */
                             0 /* (void *) feature_configuration */, 
                             0 /* feature_configuration_nbytes */);
```
很少使用feature_configuration不透明。

如果您希望将功能设为事实上的系统级概念，请始终传递sw_if_index = 0。Sw_if_index 0始终有效，并且对应于“本地”接口。

## 5.4. 相关的“显示”命令
要显示整个功能集，请使用“显示功能[详细]”。详细形式显示弧索引，并在弧内显示特征标记

```
$ vppctl show features verbose
Available feature paths
<snip>
[14] ip4-unicast:
  [ 0]: nat64-out2in-handoff
  [ 1]: nat64-out2in
  [ 2]: nat44-ed-hairpin-dst
  [ 3]: nat44-hairpin-dst
  [ 4]: ip4-dhcp-client-detect
  [ 5]: nat44-out2in-fast
  [ 6]: nat44-in2out-fast
  [ 7]: nat44-handoff-classify
  [ 8]: nat44-out2in-worker-handoff
  [ 9]: nat44-in2out-worker-handoff
  [10]: nat44-ed-classify
  [11]: nat44-ed-out2in
  [12]: nat44-ed-in2out
  [13]: nat44-det-classify
  [14]: nat44-det-out2in
  [15]: nat44-det-in2out
  [16]: nat44-classify
  [17]: nat44-out2in
  [18]: nat44-in2out
  [19]: ip4-qos-record
  [20]: ip4-vxlan-gpe-bypass
  [21]: ip4-reassembly-feature
  [22]: ip4-not-enabled
  [23]: ip4-source-and-port-range-check-rx
  [24]: ip4-flow-classify
  [25]: ip4-inacl
  [26]: ip4-source-check-via-rx
  [27]: ip4-source-check-via-any
  [28]: ip4-policer-classify
  [29]: ipsec-input-ip4
  [30]: vpath-input-ip4
  [31]: ip4-vxlan-bypass
  [32]: ip4-lookup
<snip>
```
在这里，我们了解到ip4单播特征弧的索引为14，例如ip4-inacl是生成的部分顺序中的第25个特征。

要显示特定界面上当前激活的功能，请使用“显示界面” 特征”：
```
$ vppctl show interface GigabitEthernet3/0/0 features
Feature paths configured on GigabitEthernet3/0/0...
<snip>
ip4-unicast:
  nat44-out2in
<snip>
```

## 5.5. 特征弧表
只需搜索名称字符串即可跟踪弧的定义，弧索引的位置等。
```
|    Arc Name      |
|------------------|
| device-input     |
| ethernet-output  |
| interface-output |
| ip4-drop         |
| ip4-local        |
| ip4-multicast    |
| ip4-output       |
| ip4-punt         |
| ip4-unicast      |
| ip6-drop         |
| ip6-local        |
| ip6-multicast    |
| ip6-output       |
| ip6-punt         |
| ip6-unicast      |
| mpls-input       |
| mpls-output      |
| nsh-output       |
```


**`TODO`**

<br/>

# 6. [Buffer Metadata缓冲元数据](https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/developers/metadata.html)
每个vlib_buffer_t（数据包缓冲区）携带描述当前数据包处理状态的缓冲区元数据。基础技术已经在多种数据包处理环境中使用了数十年。

我们将详细研究vpp缓冲区元数据，但是需要操纵和/或扩展该方案的人们应该期望进行一定级别的代码检查。


## 6.1. Vlib（向量库）主缓冲区元数据
每个`vlib_buffer_t`的前64个八位位组携带主要缓冲区元数据。有关完整的详细信息，请参见…/ src / vlib / buffer.h。
```c

/** VLIB buffer representation. */
typedef union
{
  struct
  {
    CLIB_CACHE_LINE_ALIGN_MARK (cacheline0);

    /** signed offset in data[], pre_data[] that we are currently
      * processing. If negative current header points into predata area.  */
    i16 current_data;

    /** Nbytes between current data and the end of this buffer.  */
    u16 current_length;

    /** buffer flags:
	<br> VLIB_BUFFER_FREE_LIST_INDEX_MASK: bits used to store free list index,
	<br> VLIB_BUFFER_IS_TRACED: trace this buffer.
	<br> VLIB_BUFFER_NEXT_PRESENT: this is a multi-chunk buffer.
	<br> VLIB_BUFFER_TOTAL_LENGTH_VALID: as it says
	<br> VLIB_BUFFER_EXT_HDR_VALID: buffer contains valid external buffer manager header,
	set to avoid adding it to a flow report
	<br> VLIB_BUFFER_FLAG_USER(n): user-defined bit N
     */
    u32 flags;

    /** Generic flow identifier */
    u32 flow_id;

    /** Reference count for this buffer. */
    volatile u8 ref_count;

    /** index of buffer pool this buffer belongs. */
    u8 buffer_pool_index;

    /** Error code for buffers to be enqueued to error handler.  */
    vlib_error_t error;

    /** Next buffer for this linked-list of buffers. Only valid if
      * VLIB_BUFFER_NEXT_PRESENT flag is set. */
    u32 next_buffer;

    /** The following fields can be in a union because once a packet enters
     * the punt path, it is no longer on a feature arc */
    union
    {
      /** Used by feature subgraph arcs to visit enabled feature nodes */
      u32 current_config_index;
      /* the reason the packet once punted */
      u32 punt_reason;
    };

    /** Opaque data used by sub-graphs for their own purposes. */
    u32 opaque[10];

    /** part of buffer metadata which is initialized on alloc ends here. */
      STRUCT_MARK (template_end);

    /** start of 2nd cache line */
      CLIB_CACHE_LINE_ALIGN_MARK (cacheline1);

    /** Specifies trace buffer handle if VLIB_PACKET_IS_TRACED flag is
      * set. */
    u32 trace_handle;

    /** Only valid for first buffer in chain. Current length plus total length
      * given here give total number of bytes in buffer chain. */
    u32 total_length_not_including_first_buffer;

    /**< More opaque data, see ../vnet/vnet/buffer.h */
    u32 opaque2[14];

    /** start of third cache line */
      CLIB_CACHE_LINE_ALIGN_MARK (cacheline2);

    /** Space for inserting data before buffer start.  Packet rewrite string
      * will be rewritten backwards and may extend back before
      * buffer->data[0].  Must come directly before packet data.  */
    u8 pre_data[VLIB_BUFFER_PRE_DATA_SIZE];

    /** Packet data */
    u8 data[];
  };
#ifdef CLIB_HAVE_VEC128
  u8x16 as_u8x16[4];
#endif
#ifdef CLIB_HAVE_VEC256
  u8x32 as_u8x32[2];
#endif
#ifdef CLIB_HAVE_VEC512
  u8x64 as_u8x64[1];
#endif
} vlib_buffer_t;
```
重要领域：

* i16 current_data：我们当前正在处理的data []，pre_data []中的有符号偏移量。如果当前负标头指向预数据（重写空间）区域。
* u16 current_length：current_data和此缓冲区末尾之间的nBytes。
* u32 flags：缓冲区标志位。大量使用，剩下的位数不多
    src / vlib / buffer.h标志位
        VLIB_BUFFER_IS_TRACED：跟踪缓冲区
        VLIB_BUFFER_NEXT_PRESENT：缓冲区具有多个块
        VLIB_BUFFER_TOTAL_LENGTH_VALID：total_length_not_includes_first_buffer有效（请参见下文）
    src / vnet / buffer.h标志位
        VNET_BUFFER_F_L4_CHECKSUM_COMPUTED：已计算tcp / udp校验和
        VNET_BUFFER_F_L4_CHECKSUM_CORRECT：tcp / udp校验和正确
        VNET_BUFFER_F_VLAN_2_DEEP：存在两个vlan标签
        VNET_BUFFER_F_VLAN_1_DEEP：存在一个vlag标签
        VNET_BUFFER_F_SPAN_CLONE：数据包已被克隆（跨度功能）
        VNET_BUFFER_F_LOOP_COUNTER_VALID：数据包查找循环计数有效
        VNET_BUFFER_F_LOCALLY_ORIGINATED：由vpp构建的数据包
        VNET_BUFFER_F_IS_IP4：数据包为ipv4，用于校验和卸载
        VNET_BUFFER_F_IS_IP6：数据包为ipv6，用于校验和卸载
        VNET_BUFFER_F_OFFLOAD_IP_CKSUM：请求硬件IP校验和卸载
        VNET_BUFFER_F_OFFLOAD_TCP_CKSUM：请求硬件tcp校验和卸载
        VNET_BUFFER_F_OFFLOAD_UDP_CKSUM：请求硬件udp校验和卸载
        VNET_BUFFER_F_IS_NATED：整理的数据包，跳过输入检查
        VNET_BUFFER_F_L2_HDR_OFFSET_VALID：L2标头偏移有效
        VNET_BUFFER_F_L3_HDR_OFFSET_VALID：L3标头偏移有效
        VNET_BUFFER_F_L4_HDR_OFFSET_VALID：L4标头偏移有效
        VNET_BUFFER_F_FLOW_REPORT：数据包是一个ipfix数据包
        VNET_BUFFER_F_IS_DVR：数据包将被重新注入到l2输出路径中
        VNET_BUFFER_F_QOS_DATA_VALID：QoS数据在vnet_buffer_opaque2中有效
        VNET_BUFFER_F_GSO：请求通用分段卸载
        VNET_BUFFER_F_AVAIL1：可用位
        VNET_BUFFER_F_AVAIL2：可用位
        VNET_BUFFER_F_AVAIL3：可用位
        VNET_BUFFER_F_AVAIL4：可用位
        VNET_BUFFER_F_AVAIL5：可用位
        VNET_BUFFER_F_AVAIL6：可用位
        VNET_BUFFER_F_AVAIL7：可用位
* u32 flow_id：通用流标识符
* u8 ref_count：缓冲区引用/克隆计数（例如，跨度复制）
* u8 buffer_pool_index：拥有此缓冲区的缓冲池索引
* vlib_error_t（u16）错误：排队到错误处理程序的缓冲区的错误代码
* u32 next_buffer：链中下一个缓冲区的缓冲区索引。仅在设置了VLIB_BUFFER_NEXT_PRESENT时有效
* union
    u32 current_config_index：特征弧上的当前索引
    u32 punt_reason：原因数据包一旦被删除。与current_config_index互斥
* u32 opaque [10]：主vnet层不透明数据（请参见下文）
* 缓冲区分配器初始化的第一个缓存行/数据的END
* u32 trace_index：数据包跟踪子系统中缓冲区的索引
* u32 total_length_not_includes_first_buffer：请参见上面的VLIB_BUFFER_TOTAL_LENGTH_VALID
* u32 opaque2 [14]：次要vnet层不透明数据（请参见下文）
* u8 pre_data [VLIB_BUFFER_PRE_DATA_SIZE]：重写空间，通常用于前置隧道封装
* u8 data [0]：缓冲从线路接收的数据。通常，硬件设备使用b-> data [0]作为DMA目标，但也有例外。不要编写盲目地假设数据包数据以b-> data [0]开始的代码。使用vlib_buffer_get_current（…）。

## 6.2. Vnet（网络堆栈）主缓冲区元数据
Vnet主缓冲区元数据占用上面显示的vlib opaque字段中保留的空间，其类型名称为`vnet_buffer_opaque_t`。通常使用`vnet_buffer（b）`宏进行访问。有关完整的详细信息，请参见../src/vnet/buffer.h。
```c
#define vnet_buffer(b) ((vnet_buffer_opaque_t *) (b)->opaque)
```
`vnet_buffer_opaque_t`结构：
```c
/*
 * vnet stack buffer opaque array overlay structure.
 * The vnet_buffer_opaque_t *must* be the same size as the
 * vlib_buffer_t "opaque" structure member, 32 bytes.
 *
 * When adding a union type, please add a stanza to
 * foreach_buffer_opaque_union_subtype (directly above).
 * Code in vnet_interface_init(...) verifies the size
 * of the union, and will announce any deviations in an
 * impossible-to-miss manner.
 */
typedef struct
{
  u32 sw_if_index[VLIB_N_RX_TX];
  i16 l2_hdr_offset;
  i16 l3_hdr_offset;
  i16 l4_hdr_offset;
  u8 feature_arc_index;
  u8 dont_waste_me;

  union
  {
    /* IP4/6 buffer opaque. */
    struct
    {
      /* Adjacency from destination IP address lookup [VLIB_TX].
         Adjacency from source IP address lookup [VLIB_RX].
         This gets set to ~0 until source lookup is performed. */
      u32 adj_index[VLIB_N_RX_TX];

      union
      {
	struct
	{
	  /* Flow hash value for this packet computed from IP src/dst address
	     protocol and ports. */
	  u32 flow_hash;

	  union
	  {
	    /* next protocol */
	    u32 save_protocol;

	    /* Hint for transport protocols */
	    u32 fib_index;
	  };

	  /* Rewrite length */
	  u8 save_rewrite_length;

	  /* MFIB RPF ID */
	  u32 rpf_id;
	};

	/* ICMP */
	struct
	{
	  u8 type;
	  u8 code;
	  u32 data;
	} icmp;

	/* reassembly */
	union
	{
	  /* group input/output to simplify the code, this way
	   * we can handoff while keeping input variables intact */
	  struct
	  {
	    /* input variables */
	    struct
	    {
	      u32 next_index;	/* index of next node - used by custom apps */
	      u32 error_next_index;	/* index of next node if error - used by custom apps */
	    };
	    /* handoff variables */
	    struct
	    {
	      u16 owner_thread_index;
	    };
	  };
	  /* output variables */
	  struct
	  {
	    union
	    {
	      /* shallow virtual reassembly output variables */
	      struct
	      {
		u16 l4_src_port;	/* tcp/udp/icmp src port */
		u16 l4_dst_port;	/* tcp/udp/icmp dst port */
		u32 tcp_ack_number;
		u8 save_rewrite_length;
		u8 ip_proto;	/* protocol in ip header */
		u8 icmp_type_or_tcp_flags;
		u8 is_non_first_fragment;
		u32 tcp_seq_number;
	      };
	      /* full reassembly output variables */
	      struct
	      {
		u16 estimated_mtu;	/* estimated MTU calculated during reassembly */
	      };
	    };
	  };
	  /* internal variables used during reassembly */
	  struct
	  {
	    u16 fragment_first;
	    u16 fragment_last;
	    u16 range_first;
	    u16 range_last;
	    u32 next_range_bi;
	    u16 ip6_frag_hdr_offset;
	  };
	} reass;
      };
    } ip;

    /*
     * MPLS:
     * data copied from the MPLS header that was popped from the packet
     * during the look-up.
     */
    struct
    {
      /* do not overlay w/ ip.adj_index[0,1] nor flow hash */
      u32 pad[VLIB_N_RX_TX + 1];
      u8 ttl;
      u8 exp;
      u8 first;
      u8 pyld_proto:3;		/* dpo_proto_t */
      u8 rsvd:5;
      /* Rewrite length */
      u8 save_rewrite_length;
      /* Save the mpls header length including all label stack */
      u8 mpls_hdr_length;
      /*
       * BIER - the number of bytes in the header.
       *  the len field in the header is not authoritative. It's the
       * value in the table that counts.
       */
      struct
      {
	u8 n_bytes;
      } bier;
    } mpls;

    /* l2 bridging path, only valid there */
    struct opaque_l2
    {
      u32 feature_bitmap;
      u16 bd_index;		/* bridge-domain index */
      u16 l2fib_sn;		/* l2fib bd/int seq_num */
      u8 l2_len;		/* ethernet header length */
      u8 shg;			/* split-horizon group */
      u8 bd_age;		/* aging enabled */
    } l2;

    /* l2tpv3 softwire encap, only valid there */
    struct
    {
      u32 pad[4];		/* do not overlay w/ ip.adj_index[0,1] */
      u8 next_index;
      u32 session_index;
    } l2t;

    /* L2 classify */
    struct
    {
      struct opaque_l2 pad;
      union
      {
	u32 table_index;
	u32 opaque_index;
      };
      u64 hash;
    } l2_classify;

    /* vnet policer */
    struct
    {
      u32 pad[8 - VLIB_N_RX_TX - 1];	/* to end of opaque */
      u32 index;
    } policer;

    /* interface output features */
    struct
    {
      u32 sad_index;
      u32 protect_index;
    } ipsec;

    /* MAP */
    struct
    {
      u16 mtu;
    } map;

    /* MAP-T */
    struct
    {
      u32 map_domain_index;
      struct
      {
	u32 saddr, daddr;
	u16 frag_offset;	//Fragmentation header offset
	u16 l4_offset;		//L4 header overall offset
	u8 l4_protocol;		//The final protocol number
      } v6;			//Used by ip6_map_t only
      u16 checksum_offset;	//L4 checksum overall offset
      u16 mtu;			//Exit MTU
    } map_t;

    /* IP Fragmentation */
    struct
    {
      u32 pad[2];		/* do not overlay w/ ip.adj_index[0,1] */
      u16 mtu;
      u8 next_index;
      u8 flags;			//See ip_frag.h
    } ip_frag;

    /* COP - configurable junk filter(s) */
    struct
    {
      /* Current configuration index. */
      u32 current_config_index;
    } cop;

    /* LISP */
    struct
    {
      /* overlay address family */
      u16 overlay_afi;
    } lisp;

    /* TCP */
    struct
    {
      u32 connection_index;
      union
      {
	u32 seq_number;
	u32 next_node_opaque;
      };
      u32 seq_end;
      u32 ack_number;
      u16 hdr_offset;		/**< offset relative to ip hdr */
      u16 data_offset;		/**< offset relative to ip hdr */
      u16 data_len;		/**< data len */
      u8 flags;
    } tcp;

    /* SNAT */
    struct
    {
      u32 flags;
    } snat;

    u32 unused[6];
  };
} vnet_buffer_opaque_t;
```
重要领域：

* u32 sw_if_index [2]：RX和TX接口句柄。在ip查找阶段，vnet_buffer（b）-> sw_if_index [VLIB_TX]被解释为FIB索引。
* i16 l2_hdr_offset：与数据包L2头的b-> data [0]的偏移量。仅当设置了b-> flags和VNET_BUFFER_F_L2_HDR_OFFSET_VALID时有效
* i16 l3_hdr_offset：与数据包L3标头的b-> data [0]的偏移量。仅当设置了b-> flags和VNET_BUFFER_F_L_L3_HDR_OFFSET_VALID时才有效
* i16 l4_hdr_offset：与数据包L4标头的b-> data [0]的偏移量。仅当设置了b-> flags和VNET_BUFFER_F_L_L4_HDR_OFFSET_VALID时有效
* u8 feature_arc_index：数据包当前正在穿越的特征弧
* union
    ip
        u32 adj_index [2]：[VLIB_TX]中的dest IP查找的邻接，[VLIB_RX]中的源ip查找的邻接，设置为〜0，直到完成源查找
    union
        通用字段
        ICMP字段
        重组领域
* mpls字段
* l2桥接字段，仅在L2路径中有效
* l2tpv3字段
* l2分类字段
* vnet警察字段
* MAP字段
* MAP-T字段
* ip分片字段
* COP（白名单/黑名单过滤器）字段
* LISP字段
* TCP字段
    连接指数
    序号
    标头和数据偏移
    数据长度
    标志
* SCTP字段
* NAT字段
* u32未使用[6]

## 6.3. Vnet（网络堆栈）辅助缓冲区元数据
Vnet主缓冲区元数据占用上面显示的vlib opaque2字段中保留的空间，并且类型名称为`vnet_buffer_opaque2_t`。通常使用`vnet_buffer2（b）`宏进行访问。有关完整的详细信息，请参见../src/vnet/buffer.h。
```c
#define vnet_buffer2(b) ((vnet_buffer_opaque2_t *) (b)->opaque2)
```
`vnet_buffer_opaque2_t`结构：
```c
/* Full cache line (64 bytes) of additional space */
typedef struct
{
  /**
   * QoS marking data that needs to persist from the recording nodes
   * (nominally in the ingress path) to the marking node (in the
   * egress path)
   */
  struct
  {
    u8 bits;
    u8 source;
  } qos;

  u8 loop_counter;
  u8 __unused[1];

  /* Group Based Policy */
  struct
  {
    u8 __unused;
    u8 flags;
    u16 sclass;
  } gbp;

  /**
   * The L4 payload size set on input on GSO enabled interfaces
   * when we receive a GSO packet (a chain of buffers with the first one
   * having GSO bit set), and needs to persist all the way to the interface-output,
   * in case the egress interface is not GSO-enabled - then we need to perform
   * the segmentation, and use this value to cut the payload appropriately.
   */
  u16 gso_size;
  /* size of L4 prototol header */
  u16 gso_l4_hdr_sz;

  /* The union below has a u64 alignment, so this space is unused */
  u32 __unused2[1];

  struct
  {
    u32 arc_next;
    u32 unused;
  } nat;

  union
  {
    struct
    {
#if VLIB_BUFFER_TRACE_TRAJECTORY > 0
      /* buffer trajectory tracing */
      u16 *trajectory_trace;
#endif
    };
    struct
    {
      u64 pad[1];
      u64 pg_replay_timestamp;
    };
    u32 unused[8];
  };
} vnet_buffer_opaque2_t;
```
重要领域：

* QOS领域
    u8位
    u8源
* u8 loop_counter：用于检测和报告内部转发循环
* 基于组的策略字段
    u8标志
    u16 sclass：数据包的源类
* u16 gso_size：L4有效负载大小，在未启用GSO的情况下一直保留到接口输出
* u16 gso_l4_hdr_sz：L4协议标头的大小
* union
    数据包轨迹跟踪器（不推荐使用）
    u16 * trajectory_trace; 仅#if VLIB_BUFFER_TRACE_TRAJECTORY> 0
    包生成器
    u64 pg_replay_timestamp：重播的pcap跟踪数据包的时间戳
    u32未使用[8]

## 6.4. 缓冲区元数据扩展
插件开发人员可能希望扩展主要或辅助vnet缓冲区的不透明联合。请执行手动活动变量分析，否则使用共享缓冲区元数据空间的节点可能会破坏事情。

不能将插件或专有元数据添加到上面命名的核心vpp引擎头文件中。而是，请按照以下步骤进行。该示例涉及vnet主缓冲区不透明联合`vlib_buffer_opaque_t`。使用vnet辅助缓冲区不透明联合`vlib_buffer_opaque2_t`是一个非常简单的变体。
在插件头文件中：
```c
/* Add arbitrary buffer metadata */
#include <vnet/buffer.h>

typedef struct
{
  u32 my_stuff[6];
} my_buffer_opaque_t;

STATIC_ASSERT (sizeof (my_buffer_opaque_t) <=
               STRUCT_SIZE_OF (vnet_buffer_opaque_t, unused),
               "Custom meta-data too large for vnet_buffer_opaque_t");

#define my_buffer_opaque(b)  \
  ((my_buffer_opaque_t *)((u8 *)((b)->opaque) + STRUCT_OFFSET_OF (vnet_buffer_opaque_t, unused)))
```
要在给定`vlib_buffer_t * b`的自定义缓冲区不透明类型中设置数据：
```c
my_buffer_opaque (b)->my_stuff[2] = 123;
```
要从自定义缓冲区不透明类型读取数据：
```c
tuff0 = my_buffer_opaque (b)->my_stuff[2];
```


**`TODO`**

<br/>

# 7. 多架构支持
本参考指南介绍了如何使用vpp多体系结构支持方案

[多架构图节点谱](https://fd.io/docs/vpp/master/gettingstarted/developers/multiarch/nodefns.html)
[多架构任意函数谱](https://fd.io/docs/vpp/master/gettingstarted/developers/multiarch/arbfns.html)

## 7.1. 多架构图节点食谱
在图节点分发功能的上下文中，使用vpp多体系结构支持设置非常容易。该方案的重点很简单：对于性能至关重要的节点，生成多个与CPU硬件相关的节点分发功能版本，然后在运行时选择最佳版本。

vpp方案使用起来很简单，但是细节很重要。

100,000英尺：我们多次编译整个图节点调度功能实现文件。这些编译产生了图节点分配函数的多个版本。每个节点的构造函数会询问CPU硬件，选择要使用的节点分发功能变量，并将`vlib_node_registration_t`“ .function”成员设置为所选变量的地址。

如图所示，使用`VLIB_NODE_FN`宏声明节点分发功能。节点函数的名称必须与图节点的名称匹配。
```c
VLIB_NODE_FN (ip4_sdp_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
                             vlib_frame_t * frame)
{
  if (PREDICT_FALSE (node->flags & VLIB_NODE_FLAG_TRACE))
    return ip46_sdp_inline (vm, node, frame, 1 /* is_ip4 */ ,
                        1 /* is_trace */ );
  else
    return ip46_sdp_inline (vm, node, frame, 1 /* is_ip4 */ ,
                        0 /* is_trace */ );
}
```
我们需要精确地生成`vlib_node_registration_t`，错误字符串和数据包跟踪解码功能的一个副本。

只需用“ `#ifndef CLIB_MARCH_VARIANT…＃endif`”将这些项目括起来：
```c
#ifndef CLIB_MARCH_VARIANT
static u8 *
format_sdp_trace (u8 * s, va_list * args)
{
   <snip>
}
#endif

...

#ifndef CLIB_MARCH_VARIANT
static char *sdp_error_strings[] = {
#define _(sym,string) string,
  foreach_sdp_error
#undef _
};
#endif

...

#ifndef CLIB_MARCH_VARIANT
VLIB_REGISTER_NODE (ip4_sdp_node) =
{
  // DO NOT set the .function structure member.
  // The multiarch selection __attribute__((constructor)) function
  // takes care of it at runtime
  .name = "ip4-sdp",
  .vector_size = sizeof (u32),
  .format_trace = format_sdp_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,

  .n_errors = ARRAY_LEN(sdp_error_strings),
  .error_strings = sdp_error_strings,

  .n_next_nodes = SDP_N_NEXT,

  /* edit / add dispositions here */
  .next_nodes =
  {
    [SDP_NEXT_DROP] = "ip4-drop",
  },
};
#endif
```
令人困惑的是：不要设置“ .function”成员！这就是多体系结构选择__attribute __（（constructor））函数（在main之前将被执行的函数）的工作

### 7.1.1. 始终内联节点调度功能
图分派函数通常包含一个或多个对内联函数的调用。往上看。如果您的节点调度功能是按照这种方式构造的，请绝对使用“ always_inline”宏：
```c
always_inline uword
ip46_sdp_inline (vlib_main_t * vm, vlib_node_runtime_t * node,
             vlib_frame_t * frame,
             int is_ip4, int is_trace)
{ ... }
```
否则，编译器很可能不会构建调度功能的多个版本。

在“ perf top”中发现此错误相当容易。例如，如果您在配置文件中看到一堆名称形式为“ xxx_node_fn_avx2”的函数，但您的全新节点函数的名称形式为“ xxx_inline.isra.1”，则很可能是将该内联声明为“静态内联”，而不是“ always_inline”。

### 7.1.2. 修改CMakeLists.txt
如果相关组件已列出“ MULTIARCH_SOURCES”，只需将指示的.c文件添加到列表中。否则，如下所示添加。请注意，添加文件“new_multiarch_node.c”应该出现在 这两个来源和MULTIARCH_SOURCES：
```makefile
add_vpp_plugin(myplugin
  SOURCES
  new_multiarch_node.c
  ...

  MULTIARCH_SOURCES
  new_ multiarch_node.c
  ...
 )
```

## 7.2. 多架构任意函数谱
为多种架构优化任意函数非常简单，并且与用于生成多架构图节点分派函数的过程非常相似。

与多体系结构图节点一样，我们多次编译源文件，生成原始功能和公共选择器功能的多个实现。

用CLIB_MARCH_FN宏装饰函数定义。例如：

更改原始功能原型…
```c
u32 vlib_frame_alloc_to_node (vlib_main_t * vm, u32 to_node_index,
                              u32 frame_flags)
```
…通过将函数名称和返回类型重铸为CLIB_MARCH_FN宏的前两个参数：
```c
CLIB_MARCH_FN (vlib_frame_alloc_to_node, u32, vlib_main_t * vm,
               u32 to_node_index, u32 frame_flags)
```
在实际的vpp映像中，将出现vlib_frame_alloc_to_node的多个版本：vlib_frame_alloc_to_node_avx2，vlib_frame_alloc_to_node_avx512，依此类推。

对于每个多体系结构函数，请使用CLIB_MARCH_FN_SELECT宏来帮助生成一个唯一的多体系结构选择器函数：
```c
#ifndef CLIB_MARCH_VARIANT
u32
vlib_frame_alloc_to_node (vlib_main_t * vm, u32 to_node_index,
                      u32 frame_flags)
{
  return CLIB_MARCH_FN_SELECT (vlib_frame_alloc_to_node)
    (vm, to_node_index, frame_flags);
}
#endif /* CLIB_MARCH_VARIANT */
```
一旦绑定，多体系结构选择器函数与间接函数调用一样昂贵。这就是说：不是很贵。

### 7.2.1. 修改CMakeLists.txt
如果相关组件已列出“ MULTIARCH_SOURCES”，只需将指示的.c文件添加到列表中。否则，如下所示添加。请注意，添加文件“new_multiarch_node.c”应该出现在 这两个来源和MULTIARCH_SOURCES：

```makefile
add_vpp_plugin(myplugin
  SOURCES
  multiarch_code.c
  ...

  MULTIARCH_SOURCES
  multiarch_code.c
  ...
 )
```
### 7.2.2. 明智的话
一个文件随意混合了值得为多个体系结构编译的功能和不值得编译的功能，这些文件最终将充满#ifndef CLIB_MARCH_VARIANT条件。这不会使代码看起来更好。

根据要求，将函数移至（新）文件以降低复杂性和/或提高结果代码的可读性可能是有意义的。

**`TODO`**

<br/>


# 8. [有界索引可扩展哈希（bihash）](https://fd.io/docs/vpp/master/gettingstarted/developers/bihash.html)


Vpp使用有界索引可扩展哈希来解决各种精确匹配（键，值）查找问题。当前实施的好处：

* 很高的记录数量扩展，已测试到100,000,000条记录。
* 随着记录数量的增加，查询性能会正常降低
* 无需锁定阅读器
* 模板实现，很容易支持任意（键，值）类型

有界索引可扩展哈希已经在数据库中广泛使用了数十年。

`Bihash`使用两级数据结构：
```
+-----------------+                                              
| bucket-0        |                                             
|  log2_size      |                                             
|  backing store  |                                             
+-----------------+                                             
| bucket-1        |                                             
|  log2_size      |           +--------------------------------+
|  backing store  | --------> | KVP_PER_PAGE * key-value-pairs |
+-----------------+           | page 0                         |
     ...                      +--------------------------------+
+-----------------+           | KVP_PER_PAGE * key-value-pairs |
| bucket-2**N-1   |           | page 1                         |
|  log2_size      |           +--------------------------------+
|  backing store  |                       ---                   
+-----------------+           +--------------------------------+
                              | KVP_PER_PAGE * key-value-pairs |
                              | page 2**(log2(size)) - 1       |
                              +--------------------------------+
```

## 8.1. 算法讨论
这种结构具有两个主要优点。实际上，每个存储桶条目都适合一个64位整数。巧合的是，vpp的目标CPU体系结构支持**64位原子操作**。修改特定存储桶的内容时，我们执行以下操作：

* 制作存储桶后备存储的工作副本
* 以原子方式将指向工作副本的指针交换到存储桶数组中
* 更改原始后备存储数据
* 原子交换回原始

因此，**搜索bihash表无需锁定读者。**
在查找时，实现会计算密钥哈希码。我们使用哈希的最低有效N位来选择存储桶。
有了存储桶，我们将为所选存储桶学习log2（nBackingPages）。此时，我们使用哈希码中的下一个log2_size位来选择将在其中找到（key，value）页面的特定后备页面。

最终结果：我们只搜索一个底页，而不是2 ** log2_size页。这是算法的关键特性。

当发生足够的冲突以填充给定存储桶的底页时，我们将存储桶的大小加倍，重新哈希，然后将存储桶中的内容处理为一组双倍的底页。将来，我们可以将大小表示为两个2的幂的线性组合，以提高空间效率。

为了解决“记录头案例”，即一组记录在哈希下以不良方式发生冲突，该实现将回落到每个存储桶上跨2 ** log2_size支持页面的线性搜索。

为了保持空间效率，我们应该配置存储桶阵列，以便有效地利用底页。查找性能趋于改变很少，如果斗阵过小或过大。

Bihash取决于选择有效的哈希函数。如果要使用真正损坏的哈希函数，例如“ return 1ULL”。bihash仍然可以工作，但是它等同于编程不良的线性搜索。

我们经常使用cpu内在函数（例如crc32）来快速计算具有良好统计数据的哈希码。


## 8.2. Bihash Cookbook使用当前（键，值）模板实例类型
使用其中一种模板实例类型非常容易。在撰写本文时，`…/ src / vppinfra`提供了针对**8、16、20、24、40和48字节密钥**，u8 *向量密钥和8字节值的预构建模板。

参见`…/ src / vppinfra / {bihash__8} .h`

要定义数据类型，＃include一个特定的模板实例，通常在子系统头文件中：
```c
#include <vppinfra/bihash_8_8.h>
```
如果要构建独立的应用程序，则需要通过＃在C源文件中包括方法实现文件来定义各种功能。

核心vpp引擎当前使用大多数（如果不是全部）已知的bihash类型，因此您可能不需要#include方法实现文件。
```c
#include <vppinfra/bihash_template.c>
```
将所选bihash数据结构的实例添加到例如“ main_t”结构中：
```c
typedef struct
{
  ...
  BVT (clib_bihash) hash;
  or
  clib_bihash_8_8_t hash;
  ...
} my_main_t;
```
BV宏将其参数与预处理器符号BIHASH_TYPE的值连接在一起。BVT宏将其参数与BIHASH_TYPE的值和固定字符串“ _t”连接起来。因此，在上面的示例中，BVT（clib_bihash）生成“ clib_bihash_8_8_t”。

如果您确定以后不会决定更改模板/类型名称，则完全可以编写代码“ `clib_bihash_8_8_t`”，依此类推。

实际上，如果您在单个源文件中#include多个模板实例，则必须使用完全枚举的类型名称。宏没有机会工作。

## 8.3. 初始化bihash表
如图所示，调用init函数。作为一个粗略的指导，从相关的模板实例头文件中选择一些数量大约为`number_of_expected_records / BIHASH_KVP_PER_PAGE`的存储桶。参见前面的讨论。

所选的内存量应轻松包含所有记录，并为哈希冲突提供足够的余地。Bihash内存是与主堆分开分配的，除了被内核PTE占用，它不会花费任何东西，因此可以合理地慷慨。

例如：
```c
my_main_t *mm = &my_main;
clib_bihash_8_8_t *h;
    
h = &mm->hash_table;

clib_bihash_init_8_8 (h, "test", (u32) number_of_buckets, 
                       (uword) memory_size);
```
## 8.4. 添加或删除键/值对
使用BV（`clib_bihash_add_del`）或显式类型变量：
```c
clib_bihash_kv_8_8_t kv;
clib_bihash_8_8_t * h;
my_main_t *mm = &my_main;
clib_bihash_8_8_t *h;
    
h = &mm->hash_table;
kv.key = key_to_add_or_delete;
kv.value = value_to_add_or_delete;

clib_bihash_add_del_8_8 (h, &kv, is_add /* 1=add, 0=delete */);
```
在删除的情况下，kv.value无关紧要。要更改与现有（键，值）对关联的值，只需重新添加[new]对。

## 8.5. 简单搜索
最简单的（键，值）搜索如下所示：
```c
clib_bihash_kv_8_8_t search_kv, return_kv;
clib_bihash_8_8_t * h;
my_main_t *mm = &my_main;
clib_bihash_8_8_t *h;
    
h = &mm->hash_table;
search_kv.key = key_to_add_or_delete;

if (clib_bihash_search_8_8 (h, &search_kv, &return_kv) < 0)
 key_not_found()
else
 key_not_found();
```
请注意，收集查询结果非常好
```c
if (clib_bihash_search_8_8 (h, &search_kv, &search_kv))
     key_not_found();
etc.
```

## 8.6. Bihash向量处理
在处理需要执行特定查找的数据包向量时，计算密钥哈希值并提前预取正确的存储区是很麻烦的。

这是编写所需代码的一种方法的示意图：

双回路：

* 提前6​​个数据包，预取2个vlib_buffer_t和2个数据包数据以形成记录密钥
* 提前4个数据包，形成2个记录密钥并调用BV（clib_bihash_hash）或显式哈希函数来计算记录哈希。调用2x BV（clib_bihash_prefetch_bucket）预取存储桶
* 提前2个数据包，调用2x BV（clib_bihash_prefetch_data）以预取2x（键，值）数据页。
* 在处理部分中，调用2x BV（clib_bihash_search_inline_with_hash）执行搜索

程序员的选择是将哈希码存储在vnet_buffer（b）元数据中的某个位置，还是使用局部变量。

单回路：

* 使用上面显示的简单搜索。

## 8.7. 走在一张桌子上
构建“ show”命令的一个相当常见的场景涉及遍历bihash表。很简单：
```c
my_main_t *mm = &my_main;
clib_bihash_8_8_t *h;
void callback_fn (clib_bihash_kv_8_8_t *, void *);

h = &mm->hash_table;

BV(clib_bihash_foreach_key_value_pair) (h, callback_fn, (void *) arg);
```
没人感到惊讶的是：clib_bihash_foreach_key_value_pair遍历整个表，并使用活动条目调用callback_fn。


## 8.8. Bihash表迭代安全性
必须谨慎使用迭代器模板“ clib_bihash_foreach_key_value_pair”。一方面，迭代器模板并没有采取bihash哈希表作家锁。如果您的用例需要它，请锁定表。

另外，迭代器模板并非在所有情况下都是安全的：

* 它的确定删除时表行走bihash表项。迭代器检查每次调用callback_fn（...）后是否已释放当前存储桶 。
* 在表遍历期间添加条目是不正确的。

在旅途中添加案件涉及一个大奖：在处理特定存储桶中的键值对时，添加一定数量的条目。幸运的是，假设添加的一个或多个条目导致当前存储桶拆分并重新混合。

由于我们根据构成不同哈希函数的内容将KVP重新哈希到不同的页面，因此以下两种情况都可能出错：

* 我们可能会重新访问以前访问的条目。根据一个用例的编码方式，我们可能会遇到递归添加的情况。
* 我们可能会跳过尚未访问的条目

一个人可以构建一个附加安全的迭代器，而这会花费很多性能：复制整个存储桶，然后遍历副本。

首先，很难想象有一个值得添加的步行用例。更不用说通过修改表然后添加一组记录而无法实现的方法了。

## 8.9. 创建一个新的模板实例
创建新模板很容易。使用现有模板之一作为模型，并进行明显的更改。hash和key_compare方法在多种意义上对性能至关重要。

如果键比较方法很慢，则每次查找都会很慢。如果散列函数很慢，则说明相同。如果哈希函数的统计属性较差，则空间效率将受到影响。在限制内，足够糟糕的哈希函数将导致表的大部分恢复为线性搜索。

在hash和key_compare函数中使用最佳可用向量单位是很值得的。


**`TODO`**

<br/>

# 9. [VPP API模块](https://fd.io/docs/vpp/master/gettingstarted/developers/vpp_api_module.html)

VPP API模块允许通过共享内存接口与VPP通信。该API包含3部分：

* **通用代码-低级API**
* **生成的代码-高级API**
* **代码生成器-生成您自己的高级API，例如用于自定义插件**

## 9.1. 通用代码

C通用代码表示基本的低级API，提供连接/断开连接，执行消息发现以及发送/接收消息的功能。C变体位于`vapi.h`中。
C ++由vapi.hpp提供，并包含高级API模板，这些模板由生成的代码专用。

## 9.2. 生成的代码
源代码树中存在的每个API文件都会自动转换为JSON文件，代码生成器会解析并生成C（`vapi_c_gen.py`）或C ++（vapi_cpp_gen.py）代码。

然后可以将其包含在客户端应用程序中，并提供与VPP进行交互的便捷方法。这包括：

* 自动字节交换
* 基于上下文的自动请求-响应匹配
* 调用回调时自动强制转换为适当的类型（类型安全）
* 自动发送转储邮件的控制命令

该API支持两种操作模式：

* 阻塞
* 非阻塞

>在阻塞模式下，每当启动操作时，代码就会等待，直到操作完成为止。这意味着在发送消息时，调用会阻塞，直到消息可以写入共享内存为止。同样，接收消息会一直阻塞，直到有消息可用为止。在更高级别上，这还意味着在执行请求（例如show_version）时，调用将阻塞直到响应返回（例如show_version_reply）。
>在非阻塞模式下，它们是解耦的，每当无法执行操作时，API都会返回VAPI_EAGAIN，并且在发送请求后，由客户端等待和处理响应。

## 9.3. 代码生成器
Python代码生成器有两种形式-`C`和C ++，并生成高级API标头。所有代码都存储在标题中。


## 9.4. C用法
低级API
有关功能的描述，请参阅vapi.h标头中doxygen格式的内联API文档。建议使用专用标头（例如vpe.api.vapi.h或vpe.api.vapi.hpp）提供的更安全的高级API。

C高级API
回调
C高级API严格基于回调，以实现最大效率。每当启动操作时，带有回调上下文的回调就是该操作的一部分。然后，当响应（或多个响应）与请求绑定在一起时，将调用回调。另外，如果注册了回调，则每当事件到达时都会调用回调。所有指向响应/事件的指针都指向共享内存，并在回调完成后立即释放，因此客户端需要提取/复制其感兴趣的任何数据。

封锁模式

在简单阻塞模式下，整个操作（作为简单请求或转储）已完成，并且在函数调用期间调用了它的回调（可能对转储多次）。

在这种模式下，一个简单请求的伪代码示例：

vapi_show_version（消息，回调，回调上下文）

* 生成唯一的内部上下文并将其分配给message.header.context
* 将消息交换为网络字节顺序
* 向vpp发送消息（消息已被消耗，vpp将释放它）
* 创建内部“未完成的请求上下文”，该上下文存储回调，回调上下文和内部上下文值
* 呼叫分派，在这种模式下，它接收并处理响应，直到内部“未解决的请求”队列为空。在阻止模式下，此队列始终最多包含一个项目。

>注意: 在事件存储在共享内存队列中的情况下，可以在调用响应回调之前调用不同的不相关的回调。

非阻塞模式 在非阻塞模式下，所有请求仅被字节交换，并且上下文信息和回调一起存储在本地（因此，在以上示例中，仅执行步骤1-4，而跳过了步骤5）。调用调度取决于客户端应用程序。这允许在发送/接收消息之间交替，或具有调用分派的专用线程。

## 9.5. C ++高级API
回调
在C ++ API中，响应自动绑定到相应的Request，Dump或Event_registration对象。可以选择指定一个回调，然后在收到响应时调用该回调。

>注意:响应占用共享的内存空间，应在不再需要时手动释放（如果是结果集）或自动释放（通过销毁拥有它们的对象）。一旦执行了Request或Dump对象，就无法重新发送该对象，因为该请求本身（存储在共享内存中）已被vpp占用，并且无法访问（设置为nullptr）。

## 9.6. C ++用法
请求和转储

创建一个Connection类型的对象，并调用connect（）以连接到vpp。

* 使用其typedef创建请求或转储类型的对象（例如Show_version）
* 如果需要，使用get_request（）获取和处理基础请求。
* 发出execute（）发送请求。
* 使用wait_for_response（）或dispatch（）等待响应。
* 使用get_response_state（）获取状态，并使用get_response（）读取响应。

大事记

创建一个连接并执行适当的请求以订阅事件（例如，Want_stats）

* 使用模板参数作为您感兴趣的事件类型来创建Event_registration。
* 调用dispatch（）或wait_for_response（）等待事件。事件发生时（如果传递给Event_registration（）构造函数），将调用回调。或者，读取结果集。

>注意:结果集中存储的事件会占用共享内存中的空间，应定期释放（例如，一旦处理了事件，则在回调中）。


**`TODO`**

<br/>

# 10. [二进制API支持](https://fd.io/docs/vpp/master/gettingstarted/developers/binary_api_support.html)
VPP提供了一种二进制API方案，以允许各种各样的客户端代码对数据平面表进行编程。在撰写本文时，有数百种二进制API。

消息在* .api文件中定义。如今，大约有80个api文件，随着人们添加可编程功能，还有更多文件。API文件编译器源位于`src / tools / vppapigen`中。

在`src / vnet / interface.api`中，这是典型的请求/响应消息定义：

```c
autoreply define sw_interface_set_flags
{
  u32 client_index;
  u32 context;
  u32 sw_if_index;
  /* 1 = up, 0 = down */
  u8 admin_up_down;
};
```
首先，API编译器将此定义呈现到 `vpp / build-root / install-vpp_debug-native / vpp / include / vnet / interface.api.h`中 ，如下所示：
```c
/****** Message ID / handler enum ******/

#ifdef vl_msg_id
vl_msg_id(VL_API_SW_INTERFACE_SET_FLAGS, vl_api_sw_interface_set_flags_t_handler)
vl_msg_id(VL_API_SW_INTERFACE_SET_FLAGS_REPLY, vl_api_sw_interface_set_flags_reply_t_handler)
#endif
/****** Message names ******/

#ifdef vl_msg_name
vl_msg_name(vl_api_sw_interface_set_flags_t, 1)
vl_msg_name(vl_api_sw_interface_set_flags_reply_t, 1)
#endif
/****** Message name, crc list ******/

#ifdef vl_msg_name_crc_list
#define foreach_vl_msg_name_crc_interface \
_(VL_API_SW_INTERFACE_SET_FLAGS, sw_interface_set_flags, f890584a) \
_(VL_API_SW_INTERFACE_SET_FLAGS_REPLY, sw_interface_set_flags_reply, dfbf3afa) \
#endif
/****** Typedefs *****/

#ifdef vl_typedefs
#ifndef defined_sw_interface_set_flags
#define defined_sw_interface_set_flags
typedef VL_API_PACKED(struct _vl_api_sw_interface_set_flags {
    u16 _vl_msg_id;
    u32 client_index;
    u32 context;
    u32 sw_if_index;
    u8 admin_up_down;
}) vl_api_sw_interface_set_flags_t;
#endif

#ifndef defined_sw_interface_set_flags_reply
#define defined_sw_interface_set_flags_reply
typedef VL_API_PACKED(struct _vl_api_sw_interface_set_flags_reply {
    u16 _vl_msg_id;
    u32 context;
    i32 retval;
}) vl_api_sw_interface_set_flags_reply_t;
#endif
...
#endif /* vl_typedefs */
```

要更改接口的管理状态，二进制api客户端会将`vl_api_sw_interface_set_flags_t`发送 到VPP，VPP会以`vl_api_sw_interface_set_flags_reply_t`消息作为响应。

多层软件，传输类型和共享库实现了多种功能：

* API消息分配，跟踪，精美打印和重播。
* 通过全局共享内存，成对/专用共享内存和套接字的消息传输。
* 跨线程不安全消息处理程序的工作线程的屏障同步。

正确编码的消息处理程序对用于向VPP传递消息或从VPP传递消息的传输一无所知。同时使用多种API消息传输类型相当简单。

>由于历史原因，二进制api消息（可能）以网络字节顺序发送。在撰写本文时，我们正在认真考虑该选择是否有意义。

## 10.1. 邮件分配
由于二进制API消息始终按顺序处理，因此我们尽可能**使用环形分配器分配消息**。与传统的内存分配器相比，该方案非常快，并且不会引起堆碎片。请参阅`src / vlibmemory / memory_shared.c` `vl_msg_api_alloc_internal（）`。

无论采用哪种传输方式，二进制api消息始终遵循msgbuf_t标头：
```c
/** Message header structure */
typedef struct msgbuf_
{
  svm_queue_t *q; /**< message allocated in this shmem ring  */
  u32 data_len;                  /**< message length not including header  */
  u32 gc_mark_timestamp;         /**< message garbage collector mark TS  */
  u8 data[0];                    /**< actual message begins here  */
} msgbuf_t;
```

这种结构使跟踪消息变得很容易，而无需对其进行解码-只需保存data_len字节-并允许 `vl_msg_api_free（）` 快速处理消息缓冲区：
```c
void
vl_msg_api_free (void *a)
{
  msgbuf_t *rv;
  void *oldheap;
  api_main_t *am = &api_main;

  rv = (msgbuf_t *) (((u8 *) a) - offsetof (msgbuf_t, data));

  /*
   * Here's the beauty of the scheme.  Only one proc/thread has
   * control of a given message buffer. To free a buffer, we just clear the
   * queue field, and leave. No locks, no hits, no errors...
   */
  if (rv->q)
    {
      rv->q = 0;
      rv->gc_mark_timestamp = 0;
      <more code...>
      return;
    }
  <more code...>
}
```

## 10.2. 邮件跟踪和重播
VPP可以捕获并重放相当大的二进制API跟踪，这一点非常重要。涉及数十万个API事务的系统级问题可以在一秒钟或更短的时间内重新运行。部分重播允许对车轮掉落的点进行二进制搜索。可以在数据平面上添加脚手架，以在遇到复杂情况时触发。

通过二进制API跟踪，打印和重放，系统级错误报告的格式为“在300,000次API事务之后，VPP数据平面停止转发流量，FIX IT！”。可以离线解决。

人们经常发现，控制平面客户端经过很长时间或在复杂环境下对数据平面进行了错误编程。没有直接的证据，“这是一个数据平面问题！”

参见src / vlibmemory / memory_vlib.c vl_msg_api_process_file（）和src / vlibapi / api_shared.c。另请参见调试CLI命令“ api跟踪”

## 10.3. API跟踪重播警告
重放二进制API跟踪的vpp实例必须与捕获跟踪的vpp实例具有相同的消息ID编号空间。重播实例必须加载与捕获实例相同的插件集。否则，错误的API消息处理程序将处理 API消息！

始终使用命令行参数（包括“ api-trace on”节）启动vpp，因此vpp将从头开始跟踪二进制API消息：
```
api-trace {
  on
}
```
给定/ tmp / api_trace中的二进制api跟踪，请执行以下操作得出一组插件：
```
DBGvpp# api trace custom-dump /tmp/api_trace
vl_api_trace_plugin_msg_ids: abf_54307ba2 first 846 last 855
vl_api_trace_plugin_msg_ids: acl_0d7265b0 first 856 last 893
vl_api_trace_plugin_msg_ids: cdp_8f707b96 first 894 last 895
vl_api_trace_plugin_msg_ids: flowprobe_f2f0286c first 898 last 901
<etc>
```
在这里，我们看到了“ `abf`”，“ `acl`”，“ `cdp`”和“ `flowprobe`”插件。使用插件列表来构造匹配的“插件”命令行参数节：
```
plugins {
    ## Disable all plugins, selectively enable specific plugins
    plugin default { disable }
    plugin abf_plugin.so { enable }
    plugin acl_plugin.so { enable }
    plugin cdp_plugin.so { enable }
    plugin flowprobe_plugin.so { enable }
}
```
首先，使用捕获了跟踪的同一vpp映像重播它。重建vpp重播实例，添加脚手架以方便在复杂条件或类似条件下设置gdb断点是完全公平的。

## 10.4. API跟踪接口问题
同样，可能需要制造[模拟]物理接口，以便API跟踪能够正确重放。跟踪源系统上的“显示界面”可以提供帮助。如上所示，API跟踪“自定义转储”可以很明显地创建多少个环回接口。如果您看到正在创建和配置虚拟主机接口，则跟踪中的第一条此类配置消息将告诉您涉及了多少个物理接口。
```
CRIPT: create_vhost_user_if socket /tmp/foosock server
SCRIPT: sw_interface_set_flags sw_if_index 3 admin-up
```
在这种情况下，可以公平地猜测一个人需要创建两个环回接口以“帮助”正确地回放跟踪。

通过在创建跟踪的系统上重播跟踪，可以在一定程度上缓解这些问题，但是在现场调试的情况下这是不现实的。

## 10.5. 客户端连接详细信息
从C语言客户端到VPP建立二进制API连接很容易：
```c
int
connect_to_vpe (char *client_name, int client_message_queue_length)
{
  vat_main_t *vam = &vat_main;
  api_main_t *am = &api_main;
  if (vl_client_connect_to_vlib ("/vpe-api", client_name,
                                client_message_queue_length) < 0)
    return -1;
  /* Memorize vpp's binary API message input queue address */
  vam->vl_input_queue = am->shmem_hdr->vl_input_queue;
  /* And our client index */
  vam->my_client_index = am->my_client_index;
  return 0;
}
```
32是`client_message_queue_length`的典型值。VPP 在需要将API消息发送到二进制API客户端时无法阻止。VPP端的二进制API消息处理程序非常快。因此，在发送异步消息时，请务必快速地轮询二进制API rx环！

**二进制API消息RX pthread**

调用`vl_client_connect_to_vlib`会 启动二进制API消息RX pthread：
```c
int
vl_client_connect_to_vlib (const char *svm_name,
			   const char *client_name, int rx_queue_size)
{
  return connect_to_vlib_internal (svm_name, client_name, rx_queue_size,
				   rx_thread_fn, 0 /* thread fn arg */ ,
				   1 /* do map */ );
}
```
```c
static void *
rx_thread_fn (void *arg)
{
  svm_queue_t *q;
  memory_client_main_t *mm = &memory_client_main;
  api_main_t *am = &api_main;
  int i;

  q = am->vl_input_queue;

  /* So we can make the rx thread terminate cleanly */
  if (setjmp (mm->rx_thread_jmpbuf) == 0)
    {
      mm->rx_thread_jmpbuf_valid = 1;
      /*
       * Find an unused slot in the per-cpu-mheaps array,
       * and grab it for this thread. We need to be able to
       * push/pop the thread heap without affecting other thread(s).
       */
      if (__os_thread_index == 0)
        {
          for (i = 0; i < ARRAY_LEN (clib_per_cpu_mheaps); i++)
            {
              if (clib_per_cpu_mheaps[i] == 0)
                {
                  /* Copy the main thread mheap pointer */
                  clib_per_cpu_mheaps[i] = clib_per_cpu_mheaps[0];
                  __os_thread_index = i;
                  break;
                }
            }
          ASSERT (__os_thread_index > 0);
        }
      while (1)
        vl_msg_api_queue_handler (q);
    }
  pthread_exit (0);
}
```
在20.05版本中代码是这样的：
```c
static void *
rx_thread_fn (void *arg)
{
  rx_thread_fn_arg_t *a = (rx_thread_fn_arg_t *) arg;
  memory_client_main_t *mm;
  svm_queue_t *q;

  vlibapi_set_main (a->am);
  vlibapi_set_memory_client_main (a->mm);
  free (a);

  mm = vlibapi_get_memory_client_main ();
  q = vlibapi_get_main ()->vl_input_queue;

  /* So we can make the rx thread terminate cleanly */
  if (setjmp (mm->rx_thread_jmpbuf) == 0)
    {
      mm->rx_thread_jmpbuf_valid = 1;
      clib_mem_set_thread_index ();
      while (1)
	vl_msg_api_queue_handler (q);
    }
  pthread_exit (0);
}
```
要自己处理二进制API消息队列，请使用 `vl_client_connect_to_vlib_no_rx_pthread`。
```c
int
vl_client_connect_to_vlib_no_rx_pthread (const char *svm_name,
					 const char *client_name,
					 int rx_queue_size)
{
  return connect_to_vlib_internal (svm_name, client_name, rx_queue_size,
				   0 /* no rx_thread_fn */ ,
				   0 /* no thread fn arg */ ,
				   1 /* do map */ );
}
```

**排队非空信令**

`vl_msg_api_queue_handler`（…）使用互斥/ condvar信号唤醒，处理VPP->客户端流量，然后进入休眠状态。当VPP->客户端API消息队列从空变为非空时，VPP提供condvar广播。

VPP以很高的速度检查自己的二进制API输入队列。VPP根据数据平面数据包处理要求，以可变速率在“进程”上下文（也称为协作多任务线程上下文）中调用消息处理程序。

## 10.6. 客户端断开连接详细信息
要与VPP断开连接，请调用[vl_client_disconnect_from_vlib](https://docs.fd.io/vpp/18.11/da/d25/memory__client_8h.html#a82c9ba6e7ead8362ae2175eefcf2fd12)。如果客户端应用程序异常终止，请安排调用此函数。VPP竭尽全力为死客户举行一场体面的葬礼，但VPP无法保证释放共享二进制API段中泄漏的内存。

## 10.7. 将二进制API消息发送到VPP
练习的重点是将二进制API消息发送到VPP，并接收来自VPP的答复。许多VPP二进制API包含一个客户端请求消息和一个简单的状态回复。
例如，要设置接口的管理员状态：
```c
vl_api_sw_interface_set_flags_t *mp;
mp = vl_msg_api_alloc (sizeof (*mp));
memset (mp, 0, sizeof (*mp));
mp->_vl_msg_id = clib_host_to_net_u16 (VL_API_SW_INTERFACE_SET_FLAGS);
mp->client_index = api_main.my_client_index;
mp->sw_if_index = clib_host_to_net_u32 (<interface-sw-if-index>);
vl_msg_api_send (api_main.shmem_hdr->vl_input_queue, (u8 *)mp);
```
关键点：

* 使用`vl_msg_api_alloc`分配消息缓冲区
* 分配的消息缓冲区未初始化，必须假定包含垃圾箱。
* 不要忘记设置`_vl_msg_id`字段！
* 在撰写本文时，二进制API消息ID和数据以网络字节顺序发送
* 客户端库全局数据结构api_main跟踪用于与VPP通信的足够的指针和句柄

## 10.8. 从VPP接收二进制API消息
除非您进行了其他安排（请参见`vl_client_connect_to_vlib_no_rx_pthread`）， 否则 将在单独的rx pthread上接收消息。与客户端应用程序主线程同步是应用程序的责任！

如下设置消息处理程序：
```c
#define vl_typedefs         /* define message structures */
#include <vpp/api/vpe_all_api_h.h>
#undef vl_typedefs
/* declare message handlers for each api */
#define vl_endianfun                /* define message structures */
#include <vpp/api/vpe_all_api_h.h>
#undef vl_endianfun
/* instantiate all the print functions we know about */
#define vl_print(handle, ...)
#define vl_printfun
#include <vpp/api/vpe_all_api_h.h>
#undef vl_printfun
/* Define a list of all message that the client handles */
#define foreach_vpe_api_reply_msg                            \
   _(SW_INTERFACE_SET_FLAGS_REPLY, sw_interface_set_flags_reply)
   static clib_error_t *
   my_api_hookup (vlib_main_t * vm)
   {
     api_main_t *am = &api_main;
   #define _(N,n)                                                  \
       vl_msg_api_set_handlers(VL_API_##N, #n,                     \
                              vl_api_##n##_t_handler,              \
                              vl_noop_handler,                     \
                              vl_api_##n##_t_endian,               \
                              vl_api_##n##_t_print,                \
                              sizeof(vl_api_##n##_t), 1);
     foreach_vpe_api_msg;
   #undef _
     return 0;
    }
```
用于建立消息处理程序的关键API是 [vl_msg_api_set_handlers](https://docs.fd.io/vpp/18.11/d6/dd1/api__shared_8c.html#aa8a8e1f3876ec1a02f283c1862ecdb7a) ，它在[api_main_t](https://docs.fd.io/vpp/18.11/dd/db2/structapi__main__t.html)结构中的多个并行向量中设置值。撰写本文时：并非所有矢量元素值都可以通过API设置。您会看到零星的API消息注册，然后对该表单进行一些细微调整：
```c
/*
 * Thread-safe API messages
 */
am->is_mp_safe[VL_API_IP_ADD_DEL_ROUTE] = 1;
am->is_mp_safe[VL_API_GET_NODE_GRAPH] = 1;
```


## 10.9. 插件中的API消息编号
插件中的二进制API消息编号依靠vpp发出消息ID块供插件使用：
```c
static clib_error_t *
my_init (vlib_main_t * vm)
{
  my_main_t *mm = &my_main;

  name = format (0, "myplugin_%08x%c", api_version, 0);

  /* Ask for a correctly-sized block of API message decode slots */
  mm->msg_id_base = vl_msg_api_get_msg_ids
    ((char *) name, VL_MSG_FIRST_AVAILABLE);

  }
```
控制平面代码使用`vl_client_get_first_plugin_msg_id`（…）api恢复消息ID块库：
```c
/* Ask the vpp engine for the first assigned message-id */
name = format (0, "myplugin_%08x%c", api_version, 0);
sm->msg_id_base = vl_client_get_first_plugin_msg_id ((char *) name);
```
注册消息处理程序或发送消息时忘记添加msg_id_base是一个相当常见的错误。使用…/ src / vlibapi / api_helper_macros.h中的宏可以自动执行该过程，但请记住在＃包括文件之前#define REPLY_MSG_ID_BASE：
```c
#define REPLY_MSG_ID_BASE mm->msg_id_base
#include <vlibapi/api_helper_macros.h>
```


**`TODO`**

<br/>


# 11. [建立系统](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/index.html)
本指南详细介绍了vpp构建系统。在撰写本文时，构建系统使用了make / Makefiles，cmake和ninja的混合以实现出色的构建性能。

* [顶级Makefile简介](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/mainmakefile.html)
* [cmake和ninja简介](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/cmakeandninja.html)
    [vpp cmake配置文件](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/cmakeandninja.html#vpp-cmake-configuration-files)
    [如何编写插件CMakeLists.txt文件](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/cmakeandninja.html#how-to-write-a-plugin-cmakelists-txt-file)
    在源代码树的其他位置添加目标
    修改构建选项：ccmake
* [build-root / Makefile简介](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/buildrootmakefile.html)
    储存库组和源路径
    单遍构建系统，依赖项和组件
    …/ build-root
    …/ build-root / build-config.mk
    …/ vpp / build-root / Makefile
    PLATFORM变量
    特定于平台的.mk片段
    TAG变量
    [重要目标build-root / Makefile](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/buildrootmakefile.html#important-targets-build-root-makefile)
    其他build-root / Makefile环境变量设置
    …/ build-root / config.site
    […/ build-data / platforms.mk](https://fd.io/docs/vpp/master/gettingstarted/developers/buildsystem/buildrootmakefile.html#build-data-platforms-mk)
    …/ build-data / packages / *。mk


## 11.1. cmake和ninja简介
Cmake加忍者分别大约等于GNU自动工具加GNU make。cmake和GNU自动工具都支持自编译和交叉编译，以检查所需的组件和版本。

对于像vpp这样的大型项目，使用（cmake，ninja）可以大大提高构建性能。

cmake输入语言看起来像是一种实际的语言，而不是类固醇上的shell脚本方案。

Ninja不假装支持手动生成的输入文件。可以将其视为快速，笨拙的机器人，它会吃一些清晰易读的字节码。

有关其他信息，请参见[cmake网站](http://cmake.org/)和[ninja网站](https://ninja-build.org/)。

## 11.2. vpp cmake配置文件
vpp项目cmake层次结构的顶部位于`…/ src / CMakeLists.txt`中。该文件定义了vpp项目，并且（递归地）包括两种文件：规则/函数定义和目标列表。

规则/函数定义位于`…/ src / cmake / {*.cmake}`中。尽管这些文件的内容很容易阅读，但不必经常修改它们

构建目标列表来自子目录中的CMakeLists.txt文件，这些文件在`…/ src / CMakeLists.txt`的SUBDIRS列表中命名。
```cmake
##############################################################################
# 12. subdirs - order matters
##############################################################################
if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  find_package(OpenSSL REQUIRED)
  set(SUBDIRS
    vppinfra svm vlib vlibmemory vlibapi vnet vpp vat vcl plugins
    vpp-api tools/vppapigen tools/g2 tools/perftool)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  set(SUBDIRS vppinfra)
else()
  message(FATAL_ERROR "Unsupported system: ${CMAKE_SYSTEM_NAME}")
endif()

foreach(DIR ${SUBDIRS})
  add_subdirectory(${DIR})
endforeach()
```
vpp cmake配置层次结构通过在`…/ src / plugins`中搜索包含CMakeLists.txt文件的子目录来发现要构建的插件列表。
```cmake
##############################################################################
# find and add all plugin subdirs
##############################################################################
FILE(GLOB files RELATIVE
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/*/CMakeLists.txt
)
foreach (f ${files})
  get_filename_component(dir ${f} DIRECTORY)
  add_subdirectory(${dir})
endforeach()
```
## 11.3. 如何编写插件CMakeLists.txt文件
这真的很简单。根据规律：
```cmake
dd_vpp_plugin(mactime
  SOURCES
  mactime.c
  node.c

  API_FILES
  mactime.api

  INSTALL_HEADERS
  mactime_all_api_h.h
  mactime_msg_enum.h

  API_TEST_SOURCES
  mactime_test.c
)
```
## 11.4. 在源代码树的其他位置添加目标
在合理的范围内，将子目录添加到`…/ src / CMakeLists.txt`中的SUBDIRS列表中完全可以。指示的目录将需要CMakeLists.txt文件。

这是我们构建g2事件数据可视化工具的方式：
```cmake
option(VPP_BUILD_G2 "Build g2 tool." OFF)
if(VPP_BUILD_G2)
  find_package(GTK2 COMPONENTS gtk)
  if(GTK2_FOUND)
    include_directories(${GTK2_INCLUDE_DIRS})
    add_vpp_executable(g2
      SOURCES
      clib.c
      cpel.c
      events.c
      main.c
      menu1.c
      pointsel.c
      props.c
      g2version.c
      view1.c

      LINK_LIBRARIES vppinfra Threads::Threads m ${GTK2_LIBRARIES}
      NO_INSTALL
    )
  endif()
endif()
```

g2组件是可选的，默认情况下未构建。有几种方法可以告诉cmake将其包括在build.ninja [或Makefile中。

手动调用cmake时（很少完成且不太容易），请指定-DVPP_BUILD_G2 = ON：
```bash
$ cmake ... -DVPP_BUILD_G2=ON
```
仔细查看`…/ build-data / packages / vpp.mk`，以查看顶级Makefile和`…/ build-root / Makefile`在哪里以及如何设置所有cmake参数。启用可选组件的一种策略非常明显。将`-DVPP_BUILD_G2 = ON`添加到`vpp_cmake_args`。

当然可以，但这不是一个特别优雅的解决方案。

## 11.5. 修改构建选项：ccmake
设置`VPP_BUILD_G2`或坦率地说任何cmake参数的简单方法是安装“ `cmake-curses-gui`”软件包并使用它。

使用顶级Makefile，“` make build`”或“ `make build-release`”进行简单的vpp构建

暂停到`…/ build-root / build-vpp-native / vpp`或`…/ build-root / build-vpp_debug-native / vpp`

调用“ ccmake”。根据需要重新配置项目

您大致会看到以下内容：
```
 CCACHE_FOUND                     /usr/bin/ccache
 CMAKE_BUILD_TYPE
 CMAKE_INSTALL_PREFIX             /scratch/vpp-gate/build-root/install-vpp-nati
 DPDK_INCLUDE_DIR                 /scratch/vpp-gate/build-root/install-vpp-nati
 DPDK_LIB                         /scratch/vpp-gate/build-root/install-vpp-nati
 MBEDTLS_INCLUDE_DIR              /usr/include
 MBEDTLS_LIB1                     /usr/lib/x86_64-linux-gnu/libmbedtls.so
 MBEDTLS_LIB2                     /usr/lib/x86_64-linux-gnu/libmbedx509.so
 MBEDTLS_LIB3                     /usr/lib/x86_64-linux-gnu/libmbedcrypto.so
 MUSDK_INCLUDE_DIR                MUSDK_INCLUDE_DIR-NOTFOUND
 MUSDK_LIB                        MUSDK_LIB-NOTFOUND
 PRE_DATA_SIZE                    128
 VPP_API_TEST_BUILTIN             ON
 VPP_BUILD_G2                     OFF
 VPP_BUILD_PERFTOOL               OFF
 VPP_BUILD_VCL_TESTS              ON
 VPP_BUILD_VPPINFRA_TESTS         OFF

CCACHE_FOUND: Path to a program.
Press [enter] to edit option Press [d] to delete an entry   CMake Version 3.10.2
Press [c] to configure
Press [h] for help           Press [q] to quit without generating
Press [t] to toggle advanced mode (Currently Off)
```
使用光标指向`VPP_BUILD_G2`行。按返回键将OFF更改为ON。按“ c”重新生成build.ninja等。

此时，“ `make build`”或“ `make build-release`”将构建g2。等等。

请注意，切换高级模式[“ t”]可以访问基本上所有的cmake选项，发现的目录和路径。

## 11.6. `build-root/Makefile`简介
vpp构建系统由一个顶级Makefile，一个数据驱动的`build-root / Makefile`和一组Makefile片段组成。一系列经过深思熟虑的约定将各个部分放在一起。

本节将详细介绍build-root / Makefile。

### 11.6.1. 储存库组和源路径
当前的vpp工作空间仅包含一个存储库组。文件`…/ build-root / build-config.mk`定义了一个称为SOURCE_PATH的键变量。SOURCE_PATH变量命名存储库组的集合。目前，只有一个存储库组。

### 11.6.2. 单遍构建系统，依赖项和组件
vpp构建系统迎合了使用`GNU autoconf / automake`构建的组件。添加此类组件是一个简单的过程。处理使用BSD风格的原始Makefile的组件更加困难。处理gcc，glibc和binutils之类的工具链组件可能要复杂得多。

vpp构建系统是单遍构建系统。任何一组组件都必须存在部分顺序：组（在b之前的a）元组必须解析为有序列表。如果创建表单的循环依赖关系；（a，b）（b，c）（c，a），gmake会尝试建立目标列表，但是有0.0％的机会使结果令人满意。`…/ build-data / packages / .mk`中的cut-n-paste错误可能导致令人困惑的失败。

在单遍构建系统中，最好将实例化的库和应用程序分开。例如，如果vpp依赖于libfoo.a，而myapp同时依赖于vpp和libfoo.a，则最好将libfoo.a和myapp放在单独的组件中。构建系统将构建libfoo.a，vpp，然后构建myapp（作为单独的组件）。如果尝试从同一组件构建libfoo.a和myapp，它将无法正常工作。

如果绝对肯定要在同一源代码树中包含myapp和libfoo.a，则可以在…/ build-data / packages /目录中的单独.mk文件中创建伪组件。定义代码phoneycomponent_source = realcomponent，并提供手动配置/构建/安装目标。

myapp，libfoo.a和vpp的单独组件是最好和最简单的解决方案。但是，存在“ mumble_source = realsource”自由度来解决棘手的循环依赖性，例如：构建gcc-bootstrap，然后构建glibc，再构建“真实” gcc / g ++（也依赖于glibc）。


### 11.6.3. …/build-root
`…/ build-root`目录包含存储库组规范`build-config.mk`，主Makefile和`config.site`中系统范围内的`autoconf / automake`变量替代集。我们将详细描述这些文件。要明确期望：主Makefile和config.site文件是微妙而复杂的。您不太可能需要或不想修改它们。在这两个地方计划不佳的更改通常会导致难以解决的错误。

### 11.6.4. …/build-root/build-config.mk
如上所述，`build-config.mk`文件很简单：它将make变量`SOURCE_PATH`设置为存储库组绝对路径的列表。

SOURCE_PATH变量如果选择移动工作区，请确保修改SOURCE_PATH变量定义的路径。这些路径需要匹配您在工作空间路径中所做的更改。例如，如果将vpp目录放置在名为jsmith的用户的工作区中，则可以将SOURCE_PATH更改为：
```
SOURCE_PATH = / home / jsmithuser / workspace / vpp
```

“开箱即用”设置应在99.5％的时间内起作用：
```
OURCE_PATH = $(CURDIR)/..
```

### 11.6.5. …/vpp/build-root/Makefile
主Makefile在许多方面都很复杂。如果您认为需要对其进行修改，那么最好进行一些研究或在更改之前寻求建议。

主要的Makefile的组织和设计具有以下特征：出色的性能，准确的依赖项处理，高速缓存启用，时间戳优化，git集成，可扩展性，使用交叉编译工具链进行构建以及使用嵌入式Linux发行版进行构建。

如果确实需要这样做，则可以使用它构建双叉工具，而不必大惊小怪。例如，您可以：在x86_64上编译gdb以在PowerPC上运行，以调试Xtensa指令集。

### 11.6.6. `PLATFORM`变量
PLATFORM的make / environment变量控制许多重要的特征，主要是：

* CPU架构
* 要构建的图像列表。

对于`…/ build-root / Makefile`，要构建的映像列表由目标指定。例如：
```bash
make PLATFORM=vpp TAG=vpp_debug install-deb
```
构建vpp调试Debian软件包。

主要的Makefile通过尝试“-包括”文件`/build-data/platforms.mk`来解释$ PLATFORM：
```cmake
(foreach d,$(FULL_SOURCE_PATH), \
  $(eval -include $(d)/platforms.mk))
```
按照惯例，我们不在`…// build-data / platforms.mk`文件中定义平台。

在vpp情况下，我们在`…/ vpp / build-data / platforms.mk`中搜索平台定义makefile片段，如下所示：
```cmake
(foreach d,$(SOURCE_PATH_BUILD_DATA_DIRS), \
     $(eval -include $(d)/platforms/*.mk))
```
使用如上所述使用“ vpp”平台的vpp，我们最终将“ -include”添加到... / vpp / build-data / platforms / vpp.mk。


### 11.6.7. 特定于平台的.mk片段
以下是…/ build-data / platforms / vpp.mk的内容：
```cmake
MACHINE=$(shell uname -m)

vpp_arch = native
ifeq ($(TARGET_PLATFORM),thunderx)
vpp_dpdk_target = arm64-thunderx-linuxapp-gcc
endif
vpp_native_tools = vppapigen

vpp_uses_dpdk = yes

# Uncomment to enable building unit tests
# vpp_enable_tests = yes

vpp_root_packages = vpp vom

# DPDK configuration parameters
# vpp_uses_dpdk_mlx4_pmd = yes
# vpp_uses_dpdk_mlx5_pmd = yes
# vpp_uses_external_dpdk = yes
# vpp_dpdk_inc_dir = /usr/include/dpdk
# vpp_dpdk_lib_dir = /usr/lib
# vpp_dpdk_shared_lib = yes

# Use '--without-libnuma' for non-numa aware architecture
# Use '--enable-dlmalloc' to use dlmalloc instead of mheap
vpp_configure_args_vpp = --enable-dlmalloc
sample-plugin_configure_args_vpp = --enable-dlmalloc

# load balancer plugin is not portable on 32 bit platform
ifeq ($(MACHINE),i686)
vpp_configure_args_vpp += --disable-lb-plugin
endif

vpp_debug_TAG_CFLAGS = -g -O0 -DCLIB_DEBUG -DFORTIFY_SOURCE=2 \
    -fstack-protector-all -fPIC -Werror
vpp_debug_TAG_CXXFLAGS = -g -O0 -DCLIB_DEBUG -DFORTIFY_SOURCE=2 \
    -fstack-protector-all -fPIC -Werror
vpp_debug_TAG_LDFLAGS = -g -O0 -DCLIB_DEBUG -DFORTIFY_SOURCE=2 \
    -fstack-protector-all -fPIC -Werror

vpp_TAG_CFLAGS = -g -O2 -DFORTIFY_SOURCE=2 -fstack-protector -fPIC -Werror
vpp_TAG_CXXFLAGS = -g -O2 -DFORTIFY_SOURCE=2 -fstack-protector -fPIC -Werror
vpp_TAG_LDFLAGS = -g -O2 -DFORTIFY_SOURCE=2 -fstack-protector -fPIC -Werror -pie -Wl,-z,now

vpp_clang_TAG_CFLAGS = -g -O2 -DFORTIFY_SOURCE=2 -fstack-protector -fPIC -Werror
vpp_clang_TAG_LDFLAGS = -g -O2 -DFORTIFY_SOURCE=2 -fstack-protector -fPIC -Werror

vpp_gcov_TAG_CFLAGS = -g -O0 -DCLIB_DEBUG -fPIC -Werror -fprofile-arcs -ftest-coverage
vpp_gcov_TAG_LDFLAGS = -g -O0 -DCLIB_DEBUG -fPIC -Werror -coverage

vpp_coverity_TAG_CFLAGS = -g -O2 -fPIC -Werror -D__COVERITY__
vpp_coverity_TAG_LDFLAGS = -g -O2 -fPIC -Werror -D__COVERITY__
```

请注意以下变量设置：

* 变量_arch设置用于构建每个平台的交叉编译工具链的CPU体系结构。除了在本示例中使用的“本机”体系结构之外，vpp构建系统还会生成交叉编译的二进制文件。
* 变量_native_tools列出了所需的自编译工具集。
* 变量_root_packages列出了在指定目标时要构建的映像集：make PLATFORM = TAG = [install-deb | install-rpm]。

## 11.7. TAG变量
TAG变量间接设置CFLAGS和LDFLAGS以及…/ vpp / build-root目录中的构建和安装目录名称。请参阅上面的定义。

## 11.8. 重要目标build-root / Makefile
主Makefile和各种makefile片段实现了以下用户可见的目标：

|  目标   | ENV变量设置    |  笔记   |
| --- | --- | --- |
|  foo   |  bar   |   mumble  |
|  bootstrap-tools   |  none   |   构建vpp构建系统构建图像所需的一组本机工具。示例：vppapigen。在完整的交叉编译情况下，可能包括“ make”，“ git”，“ find”和“ tar”  |
| install-tools    |  PLATFORM   |  为指示的\<platform\>构建工具链。未在vpp构建中使用   |
| distclean    |  none   |  Roto扎根于眼前的一切：工具链，图像等。   |
| install-deb    |  PLATFORM and TAG   |  使用TAG定义的编译/链接选项，构建包含在\<platform\> _root_packages中列出的组件的Debian软件包。   |
|  install-rpm   |  PLATFORM and TAG   |  使用TAG定义的编译/链接选项构建包含在\<platform\> _root_packages中列出的组件的RPM。   |


## 11.9. 其他build-root / Makefile环境变量设置
这些变量设置可能有用：


|     ENV变量      |                                                           笔记                                                           |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| BUILD_DEBUG = vx | 指导Makefile等。进行真诚的努力，以惊人的细节展示正在发生的事情。如下使用它：“ make…BUILD_DEBUG = vx”。在Makefile调试情况下相当有效 |
| V = 1            | 打印详细的cc / ld命令行。对于发现-DFOO = 11是否在命令行中很有用                                                               |
| CC = mygcc       | 覆盖已配置的C编译器                                                                                                        |

## 11.10. …/ build-root / config.site
`…/ build-root / config.site`的内容将覆盖单独的autoconf / automake默认变量设置。以下是一些与构建完整工具链有关的示例设置：
```
# glibc needs these setting for cross compiling
libc_cv_forced_unwind=yes
libc_cv_c_cleanup=yes
libc_cv_ssp=no
```
确定需要覆盖的变量集和覆盖值是一个反复试验的问题。无需修改此文件以与fd.io vpp一起使用。

## 11.11. …/ build-data / platforms.mk
每个存储库组都包含platform.mk文件，该文件包含在主Makefile中。vpp / build-data / platforms.mk文件并不十分复杂。在撰写本文时，…/ build-data / platforms.mk文件完成两项任务。

首先，它包括vpp / build-data / platforms / *。mk：
```cmake
# Pick up per-platform makefile fragments
$(foreach d,$(SOURCE_PATH_BUILD_DATA_DIRS), \
  $(eval -include $(d)/platforms/*.mk))
```
如上所述，这将收集平台定义makefile片段集。

其次，platforms.mk实现了用户可见的“ install-deb”目标。

## 11.12. …/ build-data / packages / *。mk
每个组件都需要一个makefile片段，以便构建系统能够识别它。每个组件的makefile片段的复杂性差异很大。对于使用GNU autoconf / automake构建的不依赖于其他组件的组件，make片段可以为空。请参阅…/ build-data / packages / vpp.mk，以获取简单但完全现实的示例。

以下是每个组件的makefile片段中的一些重要变量设置：

|         变量         |                                                                                                              笔记                                                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| xxx_configure_depend | 列出xxx组件的组件构建依赖项集。简而言之：在成功构建指示的目标之前，请勿尝试配置此组件。xxx_configure_depend几乎总是列出一组“ yyy-install”目标。注意以下模式：“变量名称包含下划线，使目标名称包含连字符”                                         |
| xxx_configure_args   | （可选）列出要传递给xxx组件“配置”脚本的所有其他参数。Makefile％-configure主规则为–libdir，–prefix和–host添加了必需的设置（交叉编译时）                                                                                                   |
| xxx_CPPFLAGS         | 将-I节添加到XXX所依赖的组件的CPPFLAGS中。几乎总是“ xxx_CPPFLAGS = $（调用installed_includes_fn，dep1 dep2 dep3）”，其中xxx_configure_depend中列出了dep1，dep2和dep3。在此处设置“ -g -O3”是不好的做法。这些设置属于TAG。                   |
| xxx_LDFLAGS          | 为xxx所依赖的组件将-Wl，-rpath -Wl，depN节添加到LDFLAGS。几乎总是“ xxx_LDFLAGS = $（调用installed_lib_fn，dep1 dep2 dep3）”，其中xxx_configure_depend中列出了dep1，dep2和dep3。在这里设置“自由或死亡”是不礼貌的。这些设置属于Makefile.am。 |

当处理由原始Makefile构建的“令人讨厌”的组件（仅在源树中构建时才起作用）时，我们在xxx.mk文件中使用特定策略。

这些组件的策略很简单：我们将源代码树复制到…/ vpp / build-root / build-xxx。这行得通，但是完全打败了依赖处理。这种策略仅适用于不需要大量（或最好是任何）修改的第三方软件。

看一下…/ vpp / build-data / packages / dpdk.mk。调用时，dpdk_configure变量将源代码复制到$（PACKAGE_BUILD_DIR）中，并执行等效于BSD的“ autoreconf -i -f”来配置构建区域。该文件的其余部分类似：一堆手动粘贴的代码，即使不是，它也可以使dpdk像一个很好的vpp构建公民那样工作。



**`TODO`**

<br/>


# 12. [FIB 2.0分层，协议，独立](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/index.html)

本节介绍了一些必备主题和术语，这些主题和术语是理解FIB体系结构的基础。

[图表](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/graphs.html)
[前缀](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/prefixes.html)

## 12.1. Graphs

[Graphs](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/graphs.html)

FIB本质上是相关Graphs的集合。**`图论`**中的术语通常在以下各节中使用。从维基百科：

Graphs是一组对象的表示，其中一些对象对通过链接连接。相互连接的对象由称为顶点（也称为节点或点）的数学抽象表示，连接一些顶点对的链接称为边（也称为圆弧或直线）……边缘可以是有向的或无向的。

在**有向图**中，边缘只能沿一个方向遍历-从子级到父级。选择名称以表示多对一关系。一个孩子有一个父母，但一个父母有多个孩子。在**无向图**中，边沿遍历可以在任一方向上进行，但在FIB中，父子命名法仍然代表多对一关系。同一父母的子女称为兄弟姐妹。当遍历是从孩子到父母的遍历时，则被视为向前遍历或步行，而从父母到许多孩子的遍历将被视为后向遍历。向前走很便宜，因为它们从许多地方开始，然后向少数地方走去。从少数几个人开始，然后步行许多人就很昂贵。

**子对象与父对象之间的多对一关系意味着父对象的生存期必须延长到其子对象的生存期**。如果控制平面在其子项之前删除了父对象，则父项必须保持 不完整状态，直到子项本身被删除为止。同样，如果在其父级之前创建了一个子级，则父级会以不完整状态完成。需要这些不完整的对象来维护图依赖性。没有他们，当添加父母时，将通过许多数据库搜索那些孩子，以寻找受影响的孩子。为了延长父母的寿命，所有子女必须上锁在父母身上。这是一个简单的参考计数。然后，子代遵循加或锁/解锁语义来查找父代，而不是malloc / free。

## 12.2. 前缀
一些用于描述前缀的术语：

`1.1.1.1`这是一个地址，因为它没有关联的掩码
`1.1.1.0/24`这是一个前缀。
`1.1.1.1/32`这是一个主机前缀（掩码长度是地址的大小）。

如果前缀A的掩码长度较长，则其前缀B的特异性更高；如果掩码较短，则前缀B的特异性较低。例如，`1.1.1.0/28`比`1.1.1.0/24`更具体。较不具体的前缀与较具体的前缀重叠是覆盖前缀。例如，`1.1.1.0/24`为`1.1.1.0/28`和`1.1.1.0/28`覆盖前缀被称为覆盖前缀。因此，覆盖前缀始终不如其覆盖前缀那么具体。

## 12.3. 数据模型
**FIB数据模型**包括两个部分：**控制平面（CP）**和**数据平面（DP）**。CP数据模型表示由上层编程到VPP中的数据。DP模型表示VPP如何在交换数据包时派生对数据包执行的操作。

## 12.4. 控制平面
[控制平面](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/controlplane.html)
控制平面遵循分层数据表示。本文档从最低层开始描述模型。该描述使用IPv4地址和协议，但所有概念均等同地适用于IPv6等效项。这些图都描绘了用于在VPP中安装信息的CLI命令以及用于表示该信息的数据结构的UML图1的近似图。

## 12.5. `ARP`条目

[ARP条目](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/arpentries.html)
![ARP数据模型](_v_images/20201009164043322_22074.png =800x)

图1显示了ARP条目的数据模型。ARP条目包含由IPv4地址标识的对等方与其给定接口上的MAC地址之间的映射。接口绑定的VRF，不是数据的一部分。VRF是入口函数，而不是出口。ARP条目描述了如何向对等方发送流量，这是一种出口功能。

该`arp_entry_t`代表控制平面除了ARP表项。该 `ip_adjacency_t`包含从得到的数据`arp_entry_t`是需要转发数据包到对端。邻接中的其他数据是rewrite 和link_type。的LINK_TYPE是将与此邻接被转发的分组的协议的描述; 这可以是IPv4或MPLS。该LINK_TYPE 直接映射到以太网头中的以太类型或GRE头中归档的协议。重写是报头的字节字符串表示形式，当发送到该对等方时，该报头将被附加在该数据包之前。对于以太网接口，这将是src，dst MAC和以太类型。对于LISP隧道，使用IP src，dst对和LISP标头。

该arp_entry_t将安装= IPv4的LINK_TYPE当接口启用MPLS时创建的条目和`LINK_TYPE = MPLS`。出于安全原因，必须为接口明确启用`MPLS`。

这样就可以在路由之间共享邻接关系，将邻接关系存储在单个数据库中，其关键字为{接口，下一跳，链接类型}。


## 12.6. routes

[routes](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/routes.html)

控制平面将通过路径列表在表中为前缀安装路由。FIB的主要功能是解决该路由。解决一条路线就是构造一个对象图，该图完全描述路线的所有元素。在图3中，随着图表从`fib_entry_t`到`ip_adjacency_t`的完成，路径被解析。
fib_entry_t
```c
/**
 * An entry in a FIB table.
 *
 * This entry represents a route added to the FIB that is stored
 * in one of the FIB tables.
 */
typedef struct fib_entry_t_ {
    /**
     * Base class. The entry's node representation in the graph.
     */
    fib_node_t fe_node;
    /**
     * The prefix of the route. this is const just to be sure.
     * It is the entry's key/identity and so should never change.
     */
    const fib_prefix_t fe_prefix;
    /**
     * The index of the FIB table this entry is in
     */
    u32 fe_fib_index;
    /**
     * The load-balance used for forwarding.
     *
     * We don't share the EOS and non-EOS even in case when they could be
     * because:
     *   - complexity & reliability v. memory
     *       determining the conditions where sharing is possible is non-trivial.
     *   - separate LBs means we can get the EOS bit right in the MPLS label DPO
     *     and so save a few clock cycles in the DP imposition node since we can
     *     paint the header straight on without the need to check the packet
     *     type to derive the EOS bit value.
     */
    dpo_id_t fe_lb;
    /**
     * Vector of source infos.
     * Most entries will only have 1 source. So we optimise for memory usage,
     * which is preferable since we have many entries.
     */
    fib_entry_src_t *fe_srcs;
    /**
     * the path-list for which this entry is a child. This is also the path-list
     * that is contributing forwarding for this entry.
     */
    fib_node_index_t fe_parent;
    /**
     * index of this entry in the parent's child list.
     * This is set when this entry is added as a child, but can also
     * be changed by the parent as it manages its list.
     */
    u32 fe_sibling;

    /**
     * A vector of delegate indices.
     */
    index_t *fe_delegates;
} fib_entry_t;
```

ip_adjacency_t
```c
/**
 * @brief IP unicast adjacency.
 *  @note cache aligned.
 *
 * An adjacency is a representation of a peer on a particular link.
 */
typedef struct ip_adjacency_t_
{
  CLIB_CACHE_LINE_ALIGN_MARK (cacheline0);

  /**
   * Linkage into the FIB node graph. First member since this type
   * has 8 byte alignment requirements.
   */
  fib_node_t ia_node;
  /**
   * feature [arc] config index
   */
  u32 ia_cfg_index;

  union
  {
    /**
     * IP_LOOKUP_NEXT_ARP/IP_LOOKUP_NEXT_REWRITE
     *
     * neighbour adjacency sub-type;
     */
    struct
    {
      ip46_address_t next_hop;
    } nbr;
      /**
       * IP_LOOKUP_NEXT_MIDCHAIN
       *
       * A nbr adj that is also recursive. Think tunnels.
       * A nbr adj can transition to be of type MIDCHAIN
       * so be sure to leave the two structs with the next_hop
       * fields aligned.
       */
    struct
    {
      /**
       * The recursive next-hop.
       *  This field MUST be at the same memory location as
       *   sub_type.nbr.next_hop
       */
      ip46_address_t next_hop;
      /**
       * The next DPO to use
       */
      dpo_id_t next_dpo;
      /**
       * A function to perform the post-rewrite fixup
       */
      adj_midchain_fixup_t fixup_func;
      /**
       * Fixup data passed back to the client in the fixup function
       */
      const void *fixup_data;
      /**
       * the FIB entry this midchain resolves through. required for recursive
       * loop detection.
       */
      fib_node_index_t fei;

      /** spare space */
      u8 __ia_midchain_pad[4];

    } midchain;
    /**
     * IP_LOOKUP_NEXT_GLEAN
     *
     * Glean the address to ARP for from the packet's destination.
     * Technically these aren't adjacencies, i.e. they are not a
     * representation of a peer. One day we might untangle this coupling
     * and use a new Glean DPO.
     */
    struct
    {
      ip46_address_t receive_addr;
    } glean;
  } sub_type;

  CLIB_CACHE_LINE_ALIGN_MARK (cacheline1);

  /** Rewrite in second and third cache lines */
  VNET_DECLARE_REWRITE;

  /**
   * more control plane members that do not fit on the first cacheline
   */
  CLIB_CACHE_LINE_ALIGN_MARK (cacheline3);

  /**
   * A sorted vector of delegates
   */
  struct adj_delegate_t_ *ia_delegates;

  /**
   * The VLIB node in which this adj is used to forward packets
   */
  u32 ia_node_index;

  /**
   * Next hop after ip4-lookup.
   *  This is not accessed in the rewrite nodes.
   * 1-bytes
   */
  ip_lookup_next_t lookup_next_index;

  /**
   * link/ether-type
   * 1 bytes
   */
  vnet_link_t ia_link;

  /**
   * The protocol of the neighbor/peer. i.e. the protocol with
   * which to interpret the 'next-hop' attributes of the sub-types.
   * 1-bytes
   */
  fib_protocol_t ia_nh_proto;

  /**
   * Flags on the adjacency
   * 1-bytes
   */
  adj_flags_t ia_flags;

  /**
   * Free space on the fourth cacheline (not used in the DP)
   */
  u8 __ia_pad[48];
} ip_adjacency_t;
```
在某些路由模型中，`VRF`将由一组IPv4和IPv6以及单播和多播的表组成。在VPP中，没有这样的分组。每个表彼此不同。表格由其数字ID标识。每个地址系列的ID范围都是独立的。

一个表由两个路由数据库组成。转发和非转发。转发数据库包含路由，数据包将针对这些路由执行数据平面中的**最长前缀匹配（LPM）**。非转发DB包含已使用VPP编程的所有路由，其中​​某些路由可能由于无法阻止其插入转发DB的原因而无法解析（请参阅：邻接源FIB条目）。

路线数据被分解为三个部分：**条目，路径列表和路径；**

* 包含路由前缀的`fib_entry_t`是FIB表中该前缀条目的表示。
* 该`fib_path_t`是在哪里发往到该路由的前缀报文中的描述。有几种类型的路径。
    附加的下一跳：路径通过接口和下一跳来描述。下一跳与该接口上路由器自身地址位于同一子网中，因此认为对等体已连接
    附加：仅通过接口描述路径。前缀覆盖的所有地址都在该路由器接口所连接的同一L2网段上。这意味着可以对前缀所覆盖的任何地址进行ARP，而通常情况并非如此（因此，IOS中的代理ARP崩溃）。连接的路径仅适用于不需要ARP的点对点（P2P）接口，即GRE隧道。
    递归：仅通过next-hop和table-id来描述路径。
    解聚合：仅通过特殊的全零地址和一个表ID来描述路径。这意味着应在表中进行后续查找。

* 该`fib_path_list_t`表示从路径列表中选择一个时转发。路径列表是一个共享库，即它是多个fib_entry_t子级的父级。为了共享任何对象类型，孩子必须搜索符合其要求的现有对象。为此，必须有一个数据库。路径列表数据库的关键是它包含的所有路径的组合描述1。每次添加路由时都需要搜索路径列表数据库，因此仅在路径列表中填充共享将带来融合优势的路径列表（请参见章节：[快速收敛](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/fastconvergence.html#fastconvergence)）。

```c
/**
 * FIB path-list
 * A representation of the list/set of path trough which a prefix is reachable
 */
typedef struct fib_path_list_t_ {
    /**
     * A path-list is a node in the FIB graph.
     */
    fib_node_t fpl_node;
    /**
     * Flags on the path-list
     */
    fib_path_list_flags_t fpl_flags;
    /**
     * Vector of paths indicies for all configured paths.
     * For shareable path-lists this list MUST not change.
     */
    fib_node_index_t *fpl_paths;
    /**
     * the RPF list calculated for this path list
     */
    fib_node_index_t fpl_urpf;
    /**
     * Hash table of paths. valid only with INDEXED flag
     */
    uword *fpl_db;
} fib_path_list_t;
```

![图2：路由数据模型类图](_v_images/20201009164220265_5372.png =800x)
图2：路由数据模型类图

图2显示了一条具有两条连接的下一跳路径的路由示例。这些路径中的每一个都将通过找到与路径属性匹配的邻接来解析，该邻接属性与邻接数据库2的键相同的转发信息（FI） 是一组可用于负载平衡的数据平面流量的邻接。路径为路由的转发信息提供了邻接关系，路径列表为IP数据包提供了完整的转发信息。

![图3：路由对象图](_v_images/20201009164256224_25282.png =800x)
图3：路由对象图
图3显示了为解析路由而创建的对象实例及其关系。这些关系的图形性质显而易见。在图的顶部显示孩子，在其下方显示父母。因此，向前走是从上到下，向后走是从下到上。该图显示了共享的对象，路径列表和邻接关系。共享对象对于快速收敛至关重要（请参阅[快速收敛](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/fastconvergence.html#fastconvergence)部分）。

## 12.7. FIB来源
系统中可以将各种路由添加到FIB表的实体。这些实体中的每个实体都称为源。当不同的源添加相同的前缀时，FIB必须在它们之间进行仲裁以确定哪个源将贡献转发信息。由于每个源使用不同的最佳路径和环路预防算法来确定转发信息，因此将多个源的转发信息进行组合是不正确的。相反，FIB必须选择仅使用来自一个来源的转发信息。该选择基于静态优先级分配3。FIB必须维护每个来源添加的信息，以便在该来源成为最佳来源时可以将其还原。VPP有两个 控制平面资料来源；API和CLI的优先级较高。每个源数据由一个表示`fib_entry_src_t`其中一个对象 fib_entry_t维持排序vector.n前缀被连接时，它被施加到一个路由器接口。


fib_entry_src_t
```c
/**
 * Information related to the source of a FIB entry
 */
typedef struct fib_entry_src_t_ {
    /**
     * A vector of path extensions
     */
    fib_path_ext_list_t fes_path_exts;
    /**
     * The path-list created by the source
     */
    fib_node_index_t fes_pl;
    /**
     * Flags the source contributes to the entry
     */
    fib_entry_flag_t fes_entry_flags;
    /**
     * Which source this info block is for
     */
    fib_source_t fes_src;
    /**
     * Flags on the source
     */
    fib_entry_src_flag_t fes_flags;
    /**
     * 1 bytes ref count. This is not the number of users of the Entry
     * (which is itself not large, due to path-list sharing), but the number
     * of times a given source has been added. Which is even fewer
     */
    u8 fes_ref_count;
    /**
     * Source specific info
     */
    union {
	struct {
	    /**
	     * the index of the FIB entry that is the covering entry
	     */
	    fib_node_index_t fesr_cover;
	    /**
	     * This source's index in the cover's list
	     */
	    u32 fesr_sibling;
	} rr;
	struct {
	    /**
	     * the index of the FIB entry that is the covering entry
	     */
	    fib_node_index_t fesi_cover;
	    /**
	     * This source's index in the cover's list
	     */
	    u32 fesi_sibling;
            /**
             * DPO type to interpose. The dpo type needs to have registered
             * it's 'contribute interpose' callback function.
             */
            dpo_id_t fesi_dpo;
	} interpose;
	struct {
	    /**
	     * the index of the FIB entry that is the covering entry
	     */
	    fib_node_index_t fesa_cover;
	    /**
	     * This source's index in the cover's list
	     */
	    u32 fesa_sibling;
	} adj;
	struct {
	    /**
	     * the index of the FIB entry that is the covering entry
	     */
	    fib_node_index_t fesi_cover;
	    /**
	     * This source's index in the cover's list
	     */
	    u32 fesi_sibling;
	} interface;
	struct {
	    /**
	     * This MPLS local label associated with the prefix.
	     */
	    mpls_label_t fesm_label;
	    /**
	     * the indicies of the LFIB entries created
	     */
	    fib_node_index_t fesm_lfes[2];
	} mpls;
	struct {
	    /**
	     * The source FIB index.
	     */
            fib_node_index_t fesl_fib_index;
	} lisp;
    } u;
} fib_entry_src_t;
```


配置如下：
```
$ set interface address 192.168.1.1/24 GigabitEthernet0/8/0
```
结果增加了两个FIB条目；已连接和连接的`192.168.1.0/24`，以及已连接和本地的`192.168.1.1/32`（又称接收或使用）。这两个前缀都是接口来源的。接口源具有较高的优先级，因此意外或恶意添加相同的前缀不会阻止路由器正确转发。匹配已连接前缀的数据包将为数据包目标地址生成ARP请求，此过程称为glean。

一个附加前缀也导致了搜集，但路由器没有在子网自己的地址。以下配置将产生一条附加路由，该路由通过一条附加路径进行解析；
```
$ ip route add table X 10.10.10.0/24 via gre0
```
如前所述，这些仅适用于点对点链接。附加的主机前缀由附加的前缀覆盖（请注意，附加的连接前缀也已附加）。如果表X不是gre0绑定到的表，那么附件导出就是这种情况（请参见“[附件导出](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/attachedexport.html#attachedexport)”部分）。

## 12.8. 邻接源FIB条目
每当创建ARP条目时，它都将发送fib_entry_t。在这种情况下，路线的形式为：
```
$ ip route add table X 10.0.0.1/32 via 10.0.0.1 GigabitEthernet0/8/0
```
它是主机前缀，其路径的下一跳地址相同。该路由突出显示了路由前缀（要匹配的流量的描述）和路径（匹配的流量的发送位置）之间的区别。表X是接口绑定到的同一表。由邻接来源产生的FIB条目称为adj-fibs。邻接源的优先级低于API源，因此以下配置：
```
$ set interface address 192.168.1.1/24 GigabitEthernet0/8/0
$ ip arp 192.168.1.2 GigabitEthernet0/8/0 dead.dead.dead
$ ip route add 192.168.1.2 via 10.10.10.10 GigabitEthernet1/8/0
```
将通过`GigabitEthernet1 / 8/0`转发`192.168.1.2`的流量。也就是说，控制平面添加的路由比ARP发现的邻接更受青睐。控制平面及其相关的身份验证被视为权威来源。为了通过恶意注入邻接来阻止有害添加的恶作剧，FIB还需要确保在转发中仅安装附有覆盖范围较不明确的前缀的fib。这需要使用掩体追踪，其中路由与该路由保持依赖性关系，这是其不太明确的覆盖范围。当此封面更改（即，有一条新的覆盖路线）或更新了封面的转发信息时，便会通知该覆盖路线。未通过此项掩盖检查的调整纤维未安装在fib_table_t的转发表中，仅存在于非转发表中。

不支持重叠子网，因此没有adj-fib具有多个路径。在接口更改RF之前，预计控制平面会删除为接口配置的前缀。

因此，虽然接受以下配置：
```
$ set interface address 192.168.1.1/32 GigabitEthernet0/8/0
$ ip arp 192.168.1.2 GigabitEthernet0/8/0 dead.dead.dead
$ set interface ip table GigabitEthernet0/8/0 2
```
它不会产生所需的行为，将adj-fib和连接的邻接关系移至表2。

## 12.9. 递归路由
图4显示了用于描述递归路由的数据结构。该表示几乎与连接的下一跳路径相同。区别在于，`fib_path_t`的父级是另一个`fib_entry_t`，称为过 孔条目

![图4：递归路由类图。](_v_images/20201009164612303_24258.png =800x)
图4：递归路由类图。

为了将流量转发到`64.10.128.0/20`，FIB必须首先确定如何将流量转发到`1.1.1.1/32`。这是递归解析。递归解析本质上是数据平面结果的缓存，它模拟* via-table表0 4中“ via-address” 1.1.1.1的最长前缀匹配 。

递归解析（RR）将在过孔表中为过孔地址提供主机前缀条目。RR源是低优先级源。在不太可能5事件的RR源是最好的来源，那么就必须得到来自其覆盖前缀转发信息。

有两种情况需要考虑：

* 盖子已连接。然后，via-address是连接的主机，并且RR源可以通过邻接关系直接使用键`{via-address，interface-of-connected-cover}`进行解析。
* 盖子未连接。RR源可以直接从其封面继承转发信息。

在盖前缀这种依赖性意味着RR源将跟踪其盖的覆盖前缀将改变时;

* 插入一个更特定的前缀。因此，无论何时将条目插入FIB表中，都必须找到其封面，以便可以通知其所覆盖的家属。
* 现有的护盖已卸下。覆盖的前缀必须与下一个较不具体的前缀形成新的关系。

修改覆盖前缀的路由时，将更新覆盖。封面跟踪机制将在更改或更新封面的情况下向RR源条目提供通知，并且源可以采取必要的措施。

源自RR的FIB条目成为`fib_path_t`的父级，并将其转发信息贡献给该路径，以便子级的FIB条目可以构造自己的转发信息。

图5显示了创建的对象实例，以表示递归路线及其解析路线。
![图5：递归路由对象图](_v_images/20201009164741160_28797.png =800x)
图5：递归路由对象图
如果添加递归路由的源代码本身未执行递归解析8， 则该源代码可能会无意中编写了递归循环。

递归循环的一个示例是以下配置：
```
$ ip route add 5.5.5.5/32 via 6.6.6.6
$ ip route add 6.6.6.6/32 via 7.7.7.7
$ ip route add 7.7.7.7/32 via 5.5.5.5
```
这显示了三个级别上的循环，但任何数量都可以。FIB将向前行走，当检测到递归循环图fib_entry_t形成了一个父子关系fib_path_list_t。漫游检查是否遇到相同的对象实例。当形成递归循环时，控制平面9的图变为循环的，从而允许形成子对父项的依存关系。这是必要的，以便在循环中断时，受影响的孩子可以被更新。

## 12.10. 输出标签
路由可能已关联了MPLS标签10。这些是在转发数据包时应加在数据包上的标签。重要的是要注意，MPLS标签是按路由和按路径的，因此，即使路由共享路径，对于该路径11也不一定具有相同的标签。因此，一个标签唯一地关联到一个fib_entry_t和与所述的一个相关联fib_path_t到将其转发。MPLS标签经由的一般概念建模路径延伸甲fib_entry_t 因此具有0至许多载体fib_path_ext_t对象来表示与所配置的标签。

## 12.11. Attached Export
外联网在VRF A中也可以从VRF B到达前缀。VRFA是导出VRF，B是导入。在导出VRF中考虑此路线；
```
# ip route add table 2 1.1.1.0/24 via 10.10.10.0 GigabitEthernet0/8/0
```

有两种方法可以考虑在导入VRF中表示此路由：

1. ip路由通过`10.10.10.0 GigabitEthernet0 / 8/0`添加表3 `1.1.1.0/24`
2. ip route通过表2添加表3 `1.1.1.0/24`

其中选项2）是解聚合路由的示例，在表2中执行了第二次查找，即导出VRF。选项2）显然效率较低，因为第二次查找的成本很高。因此，选项1）是首选。但是，连接和连接的前缀，尤其是它们覆盖的adj-fib，需要特别注意。控制平面知道需要导出的已连接和已连接的前缀，但它不知道adj-fib。因此，FIB有责任确保每当导出附加前缀时，覆盖的adj-fibs和本地前缀也是如此，只有adj-fibs和locals被覆盖，没有被覆盖的更具体（例如，通过API来源） 。导入的FIB条目来源为Attached-export 这是低优先级的来源，因此，如果这些前缀已存在于API来源的导入VRF中，则它们将继续使用该信息进行转发。

![图6：附加的导出类图。](_v_images/20201009164951843_18698.png =800x)
图6：附加的导出类图。

图6显示了用于执行附加导出的数据结构。

* `fib_import_t`。表示需要导入覆盖的前缀。实例与导入VRF中的FIB条目关联。当将附加路由添加到与其所附加的接口的表不同的表中时，便会认识到需要导入前缀。创建fib_import_t将触发创建fib_export_t。
* `fib_export_t`。表示需要导出前缀。实例与导出VRF中的附加条目关联。甲fib_export_t可以有许多相关的fib_import_t表示多个VRF到其中的前缀导出的对象。

![图7：附加的导出对象图](_v_images/20201009165016483_31062.png =800x)
图7：附加的导出对象图

图7显示了一个对象实例图，用于将表1的连接导出到另外两个表。导出VRF中的/ 32 adj-fib和本地前缀将导出到导入VRF中，在此将它们作为Attached-export来获取，并从导出的条目中继承转发信息。导入VRF中附加的前缀还使用导出VRF中连接的前缀执行覆盖跟踪，因此它可以响应对该前缀的更新，这将需要删除导入的覆盖前缀。

## 12.12. Graph walks
[Graph Walks](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/graphwalks.html)
所有FIB对象类型都是从VPP内存池1中分配的。因此，对象易于进行内存重新分配，因此无法使用裸露的“ C”指针来引用子代或父代。相反，有一个`fib_node_ptr_t`的概念， 它是index类型的元组。类型指示对象是什么类型（以及使用哪个对象），索引是该对象中的索引。这样可以安全地检索任何对象类型。
fib_node_ptr_t
```c
/**
 * A representation of one pointer to another node.
 * To fully qualify a node, one must know its type and its index so it
 * can be retrieved from the appropriate pool. Direct pointers to nodes
 * are forbidden, since all nodes are allocated from pools, which are vectors,
 * and thus subject to realloc at any time.
 */
typedef struct fib_node_ptr_t_ {
    /**
     * node type
     */
    fib_node_type_t fnp_type;
    /**
     * node's index
     */
    fib_node_index_t fnp_index;
} fib_node_ptr_t;
```

当孩子通过父母解决时，便知道该父母的类型。因此，孩子与父母之间的关系对于孩子来说是完全已知的，因此，图表的向前走（从孩子到父母）是微不足道的。但是，父母不会选择其子女，甚至不会选择类型。构成FIB控制平面图一部分的所有对象类型都继承自单个基类。`fib_node_t`。一个`fib_node_t `识别对象的指数及其相关的虚函数表提供了父机制步行期间访问该对象。退步的原因是告知所有孩子父母的状态已经发生某种改变，并且孩子本身可能需要更新。
fib_node_t
```c
/**
 * An node in the FIB graph
 *
 * Objects in the FIB form a graph.
 */
typedef struct fib_node_t_ {
    /**
     * The node's type. make sure we are dynamic/down casting correctly
     */
    fib_node_type_t fn_type;

    /**
     * Some pad space the concrete/derived type is free to use
     */
    u16 fn_pad;

    /**
     * Vector of nodes that depend upon/use/share this node
     */
    fib_node_list_t fn_children;

    /**
     * Number of dependents on this node. This number includes the number
     * of children
     */
    u32 fn_locks;
} fib_node_t;
```
为了支持多对一，子女与父母的关系，父母必须维护其子女的清单。此列表的要求是；

* O（1）插入和删除时间。在路由添加/删除过程中建立/中断了几个孩子-父母关系。
* 订购 高优先级的子项位于前面，低优先级的子项位于后面（请参阅快速收敛部分）
* 在任意位置插入。

为了实现这些要求，子列表是一个双向链接列表，其中每个元素都包含一个`fib_node_ptr_t`。VPP池内存模型适用于列表元素，因此它们也由索引标识。将子级添加到列表后，将返回该元素的索引。使用该索引可以在恒定时间内删除元素。该列表支持“前推”和“后推”语义进行排序。走父母的孩子就是要遍历这个清单。

图的回溯是深度优先搜索，其中访问了层次结构所有级别的所有子级。因此，这种行走可能会遇到FIB控制平面图中的所有对象实例，数量达数百万。FIB控制平面图在存在递归循环的情况下是循环的，因此，walk实现具有检测到这种情况并提早退出的机制。

后退可以是同步的也可以是异步的。同步步行将在控制权返回给调用者之前访问图的整个部分，异步步行将把步行排队到后台进程，以在稍后时间运行，然后立即返回给调用者。为了实现异步遍历，将fib_walk_t对象添加到父级的子级列表的前面。当孩子们被探访时，fib_walk_t对象在列表中前进。由于它已插入列表中，因此当步行暂停并继续时，它可以在正确的位置继续。从列表中删除孩子也是安全的。新的孩子被添加到列表的头部，因此不会遇到步行，但是由于它们是新的，因此它们已经具有父对象的最新状态。

VLIB进程“ fib-walk”运行以执行异步漫游。VLIB在各个进程之间没有优先级调度，因此fib-walk进程确实以很小的增量工作，因此它不会阻塞主路由下载进程。由于主下载过程实际上具有优先级，因此可以在运行fib-walk进程之前在同一父实例上启动许多异步向后遍历。FIB是“最终状态”应用程序。如果父母更改了n次，则孩子也不必更新n次，而是仅需要此孩子更新到最新或最终状态即可。因此，当对父母的多个步行（以及对孩子的潜在更新）排队时，这些步行可以合并为一个步行。

因此，在同步游走和异步游走之间进行选择是在将父项中的更改传播给其所有子级所花费的时间与对单个路由更新进行操作所花费的时间之间的权衡。例如，如果路由更新会影响数百万条子递归路由，那么处理此类更新的速度将取决于子递归路由的数量，而这将是不好的。在撰写本文时，FIB2.0在所有位置上使用同步步行，但在步行路径列表的子代时除外，它具有32个以上的3个子代。这避免了上述情况。

>[1]快速的内存分配对于快速的路由更新时间至关重要。
>[2]VPP可以用C而不是C ++编写，但是继承仍然可能。
>[3]该值是任意的，尚待调整。

## 12.13. 数据平面
数据平面数据模型是异构对象的有向无环1图。数据包将在切换时向前走图。每个对象都描述了要对数据包执行的操作。每种对象类型都有一个关联的VLIB图节点。因此，对于一个要向前走的数据包，该图是从一个VLIB节点移动到下一个VLIB节点，每个节点都执行所需的操作。这是VPP模型的核心。

数据平面图由通用数据路径对象（DPO）组成。父DPO由元组标识：`{type，index，next_node}`。该next_node参数是该数据包下一步应该发送的VLIB节点的索引，这是目前最大限度地提高性能-确保不需要家长要读取重要的是2，而处理这个孩子。DPO的专业3执行不同的操作。最常见的DPO及其简要表示为：

* 负载均衡：ECMP集中的一种选择。
* 相邻性：通过接口进行重写和转发
* MPLS标签：施加MPLS标签。
* 查找：在另一个表中执行另一个查找。

数据平面图是通过其中的对象将DPO“贡献”到数据平面图而从控制平面图得出的。数据平面中的对象仅包含交换数据包所需的信息，因此它们更简单，并且在内存方面更小，目的是将一个DPO放在单个高速缓存行上。从控制平面派生的意思是数据平面图仅包含其当前状态可以转发数据包的对象。例如，fib_path_list_t和load_balance_t之间的区别在于，前者表示控制平面的期望状态，后者表示数据平面的可用状态。如果路径列表中的某些路径未解析或关闭，则负载平衡将不将其包括在转发选项中。

![图8：非递归路线的DPO贡献](_v_images/20201009165327231_20744.png =800x)
图8：非递归路线的DPO贡献
图8显示了控制平面图的简化视图，指示了贡献DPO的那些对象。还显示了使用DPO的VLIB节点图。

每个`fib_entry_t`都贡献自己的`load_balance_t`，其原因有三个：

* 在IPv [46]表中查找的结果是一个32位无符号整数。这是内存池的索引。因此，每个结果的对象类型必须相同。有些路由需要负载均衡，有些则不需要，但是在图中插入另一个对象来表示此选择会浪费周期，因此负载均衡对象始终是结果。如果路由没有ECMP，则负载平衡只有一种选择。
* 为了收集按路由计数器，查找结果必须以某种方式唯一地标识fib_entry_t。共享的负载平衡（由路径列表提供）将不允许这样做。
* 在fib_entry_t具有MPLS输出标签的情况下，因此具有fib_path_ext_t，则负载平衡必须是按前缀的，因为作为其父级的MPLS标签本身就是per-fib_entry_t。
![](_v_images/20201009165409594_3292.png =800x)
图9：递归路由的DPO贡献。
图9显示了为递归路由贡献的负载平衡对象。

![](_v_images/20201009165420342_23187.png =800x)

图10：来自标记的递归路由的DPO贡献。

图10显示了标记的递归路由的导出数据平面图。MPLS标签DPO实例的数量可以与路由乘以每个路由的路径数一样多。因此，mpls-label DPO应该尽可能小4。

数据平面图是通过将DPO的一个实例“堆叠”在另一个实例上以形成子对父关系来构造的。发生这种堆叠时，将从相关DPO类型的已注册图形节点自动构建必要的VLIB图形弧。

上图显示，对于任何给定的路由，在任何数据包到达之前就已经知道完整的数据平面图。如果该图由n个对象组成，则该数据包将访问n个节点，从而导致转发成本约为图节点成本的n倍。如果将图形折叠 到单个DPO和关联的节点中，则可以减少这种情况。但是，折叠图形会删除提供快速收敛的间接对象（请参见快速收敛部分）。崩溃是快速转发和快速收敛之间的权衡；VPP支持后者。

这种DPO模型在今天有效存在，但已非正式定义。当前，数据平面中唯一的对象是`ip_adjacency_t`，但是功能（例如ILA，逐跳OAM，SR，MAP等）将邻接类型归为子类型。成员lookup_next_index等效于定义新的子类型。添加到现有的联合，或将子类型特定的数据强制转换为不透明成员，甚至覆盖重写字符串（例如，新的端口范围检查器），都等同于定义新的C结构类型。幸运的是，这时所有这些子类型的内存都小于ip_adjacency_t。现在可以使用ip_register_adjacency（）动态注册新的邻接子类型，并提供自定义格式功能。

在我看来，一个严格定义的对象模型将使贡献者更容易理解，并且实现起来更加健壮。

>[1]定向意味着它不能后退。即使存在递归循环，它也是非循环的。
>[2]加载到缓存中，因此可能导致d缓存未命中。
>[3]敬业的读者可以访问vnet / vnet / dpo / *
>[4]也就是说，我们不应该重复使用邻接结构。

## 12.14. 隧道`tunnels`
[隧道](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/tunnels.html)
隧道与递归路由具有相似的属性，因为在应用隧道封装后，必须转发新的数据包，即转发是递归的。但是，与递归路由一样，隧道的目的地是事先已知的，因此，如果数据包可以遵循已构建的隧道目的地的数据平面图，则可以避免使用递归交换。将DP图连接在一起的过程称为堆栈。
![图11：隧道控制平面对象图](_v_images/20201009165545531_20488.png =800x)
图11：隧道控制平面对象图

图11显示了通过隧道的路线的控制平面对象图。左右分别显示了通过隧道的路线和通往目的地的路线的两个子图。红线通过堆叠两个子图显示了关系形式。隧道接口上的邻接称为“中间链”，它现在位于图/链的中间，而不是其通常的终端位置。

中间链邻接由gre_tunnel_t贡献，它也成为FIB控制平面图的一部分。因此，当隧道目的地的转发信息发生更改时，它将通过一条小路进行访问。这将触发它在父级fib_entry_t贡献的新`load_balance_t`上重新堆叠中链邻接 。

如果后退路线表明没有通往隧道的路由，或者该路由不满足分辨率限制，则可以将该隧道标记为已关闭，并且可以采用与物理接口相同的方式触发快速收敛（请参见部分 …）。


## 12.15. `MPLS` FIB

MPLS FIB使用与IP FIB完全相同的数据结构实现。唯一的区别是表的实现。**对于IPv4**，这是一个mtrie；**对于IPv6**，这是一个哈希表；**对于MPLS**，这是一个由21位密钥（标签和EOS位）索引的平面数组。选择该实现以有利于分组转发速度。

>**多协议标签交换（英语：Multi-Protocol Label Switching，缩写为MPLS）**是一种在开放的通信网上利用标签引导数据高速、高效传输的新技术。多协议的含义是指MPLS不但可以支持多种网络层层面上的协议，还可以兼容第二层的多种数据链路层技术。


缺省情况下，没有使能MPLS。有两个步骤可以开始。首先，创建默认的MPLS FIB：
```
$ mpls table add 0
```
其中“ 0”是“默认”表的magic数字（就像对IPv [46]一样）。一个人可以创建其他MPLS表，但是与IP表不同，不能将非默认MPLS表“绑定”到接口，换句话说，在接口上收到的所有MPLS数据包将始终在默认表中进行查找。使用非默认表必须更具创造力...

其次，对于要在其上接收MPLS数据包的每个接口，该接口必须已启用MPLS。
```
$ set interface mpls GigEthernet0/0/0 enable
```
没有等效的传输使能，仅需要使用接口作为出口路径。

MPLS FIB中的条目可以显示为：
```
$ sh mpls fib [table X] [label]
```
IP和MPLS转发之间存在紧密的耦合。MPLS转发对等类（FEC）通常是IP前缀-也就是说，将匹配给定IP前缀的流量路由到MPLS标签交换路径（LSP）。
因此，必须能够将给定的前缀/路由与在转发数据包时将要施加的[传出] MPLS标签相关联。配置为：
```
$ ip route add 1.1.1.1/32 via 10.10.10.10 GigEthernet0/0/0 out-labels 33
```
**匹配1.1.1.1/32的数据包将转发到GigEthernet0 / 0/0，并加上MPLS标签33。**可以指定多个外发标签。传出的MPLS标签可以应用于递归和非递归路由，例如：
```
$ ip route add 2.2.2.0/24 via 1.1.1.1 out-labels 34
```
因此，**匹配2.2.2.0/24的数据包将具有两个MPLS标签**；参见图34和33。这是例如MPLS BGP VPNv4的实现。

为一个前缀关联/分配一个本地标签，从而使到达该本地标签的数据包等效地转发给该前缀；

```
$ mpls local-label 99 2.2.2.0/24
```

在API中，此操作称为“绑定”。接收MPLS封装数据包的路由器需要使用与每个标签值相关联的动作进行编程-这就是MPLS FIB的作用。MPLS FIB是一个表，其键是MPLS标签值和堆栈结束（EOS）位，该表存储要对具有匹配封装的数据包执行的操作。当前支持的操作是：

1. 弹出标签并在指定的表中执行IPv [46]查找
2. 弹出标签并通过指定的下一跳转发（这是倒数第二跳，PHP）
3. 交换标签并通过指定的下一跳转发。

这些可以分别通过以下方式编程：
```
$ mpls local-label 33 eos ip4-lookup-in-table X
$ mpls local-label 33 [eos] via 10.10.10.10 GigEthernet0/0/0
$ mpls local-label 33 [eos] via 10.10.10.10 GigEthernet0/0/0 out-labels 66
```
后者是MPLS交叉连接的示例。对IP前缀有效的对下一跳，递归，非递归，标记，非标记等的任何描述也对`MPLS`本地标记有效。请注意，使用了“ eos”关键字，该关键字表示标签在堆栈末尾时的编程情况。最后两个操作可同时应用于eos和非eos数据包，但是pop和IP查找仅适用于eos数据包。

## 12.16. MPLS VPN
要为PE配置`MPLS VPN`，可以使用以下示例。

第1步; 配置到iBGP对等体的路由-请注意，这些路由务必具有传出标签；
```
$ ip route add 10.0.0.1/32 via 192.168.1.2 Eth0 out-labels 33
$ ip route add 10.0.0.2/32 via 192.168.2.2 Eth0 out-labels 34
```
第2步; 配置客户“ VRF”
```
$ ip table add 2
```
步骤3; 通过具有该对等体发布的MPLS标签的iBGP对等体添加路由
```
$ ip route add table 2 10.10.10.0/24 via 10.0.0.2 next-hop-table 0 out-label 122
$ ip route add table 2 10.10.10.0/24 via 10.0.0.1 next-hop-table 0 out-label 121
```
第4步; 通过eBGP对等体添加路由
```
$ ip route add table 2 10.10.20.0/24 via 172.16.0.1 next-hop-table 2
```

步骤5; 根据使用的标签分配方案，将路由添加到MPLS FIB以接受传入的标记数据包：

每前缀标签方案-此命令将标签“绑定”到与IP路由相同的转发中
```
$ mpls local-label 99 10.10.20.0/24
```
每CE标签方案-弹出输入标签并通过提供的下一跳转发。如果需要，将配置附加到“ out-labels”。
```
$ mpls local-label 99 via 172.16.0.1 next-hop-table 2
```
每个VRF标签方案
```
$ mpls local-label 99 via ip4-lookup-in-table 2
```

## 12.17. MPLS隧道
MPLS隧道是单向的，可以强加一堆标签。它们是“普通”接口，因此可以用作例如IP路由和L2交叉连接的目标。构造隧道：
```
$ mpls tunnel add via 10.10.10.10 GigEthernet0/0/0 out-labels 33 44 55
```
然后使创建的隧道执行ECMP：
```
$ mpls tunnel add mpls-tunnel0 via 10.10.10.11 GigEthernet0/0/0 out-labels 66 77 88
```
用
```
$ sh mpls tunnel [X]
```
看看你所创造的怪物。

MPLS隧道接口是与其他接口一样的接口，现在可以与通常的一组接口命令一起使用，例如：
```
$ set interface state mpls-tunnel0 up
$ set interface ip address mpls-tunnel0 192.168.1.1/30
$ ip route 1.1.1.1/32 via mpls-tunnel0
```

## 12.18. IP组播FIB
[IP组播FIB](https://fd.io/docs/vpp/master/gettingstarted/developers/fib20/multicast.html)
IP组播FIB（mFIB）是一种数据结构，其中包含代表（S，G）或（*，G）组播组的条目。每个IP表有一个IPv4和一个IPv6 mFIB，即，每次用户调用“ ip [6]表添加X”时，都会创建一个mFIB。

路径描述了将数据包发送到何处或从何处接收到数据包。mFIB条目维护两组“路径”；转发集和接受集。转发集中的每个路径将输出接收到的数据包的副本。接收到的数据包仅在其进入与接受集中匹配的路径上进入时才接受转发。这是RPF检查。

要将条目复制到GigEthernet0 / 0/0和GigEthernet0 / 0/1的组（1.1.1.1、239.1.1.1）的默认mFIB中，请执行以下操作：
```
$ ip mroute add 1.1.1.1 239.1.1.1 via GigEthernet0/0/0 Forward
$ ip mroute add 1.1.1.1 239.1.1.1 via GigEthernet0/0/1 Forward
```
路径附带的标志“ Forward”指定此路径为复制集的一部分。要将路径从GigEthernet0 / 0/2添加到接受（RPF）集，请执行以下操作：
```
$ ip mroute add 1.1.1.1 239.1.1.1 via GigEthernet0/0/2 Accept
```
通过不指定源地址来添加（*，G）条目：
```
$ ip mroute add 232.2.2.2 via GigEthernet0/0/2 Forward
```
通过不指定源地址并为组地址提供掩码来添加（*，G / m）条目：
```
$ ip mroute add 232.2.2.0/24 via GigEthernet0/0/2 Forward
```
当所有路径都已删除并且所有条目标志（请参阅下文）也都删除时，条目将被删除。

有一组仅与条目关联的标志，请参阅：

```
$ show mfib route flags
```
其中只有一些与API / CLI有关：

* 信号-与该条目匹配的数据包将生成一个事件，该事件将发送到控制平面（可以通过信号转储API进行检索）
* 已连接-指示应向控制平面通知已连接的源（也可以通过信号转储API检索）
* accept-all-itf-条目应接受来自所有接口的数据包，从而消除了RPF检查
* 丢弃-丢弃所有与此条目匹配的数据包。

条目上的标志可以通过以下方式更改：
```
$ ip mroute <PREFIX> <FLAG>
```
RPF检查的另一种方法是检查条目和RPF-ID，它会检查接受路径集：
```
$ ip mroute <PREFIX> rpf-id X
```
PF-ID是接收到的数据包的元数据的属性，并且当它进入给定实体（例如MPLS隧道或BIER表配置条目）时，将添加到该数据包中。

## 12.19. 快速收敛

**`待写`**

FIB规模的唯一限制因素是分配给FIB使用的每个堆的内存量，共有4个：

* IP4堆
* IP6堆
* 主堆
* 统计资料堆


## 12.20. IP4堆
IPv4堆用于分配存储IPv4前缀的数据结构所需的内存。每个表，由用户创建，即带有；
```
$ ip table add 1
```
或默认表，包含2个ip4_fib_t对象。“非转发” ip4_fib_t包含表中的所有条目，“转发”包含在数据平面中匹配的条目。两组之间的差异是数据平面中不应匹配的条目。每个ip4_fib_t包含一个mtrie（用于在数据平面中快速查找）和一个哈希表每个前缀长度（用于在控制平面中查找）。

要查看IPv4表消耗的内存量，请使用：
```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:333056 hash:3523
ipv4-VRF:1 mtrie:333056 hash:3523
totals: mtrie:666112 hash:7046 all:673158

Mtrie Mheap Usage: total: 32.06M, used: 662.44K, free: 31.42M, trimmable: 31.09M
    free chunks 3 free fastbin blks 0
    max total allocated 32.06M
no traced allocations
```
此输出显示两个“空”（即，未添加路由）表。每个mtrie使用大约150k的内存，因此每个表大约300k。最后显示IP4堆的总堆使用情况统计信息。

在输出下方分别添加了1M，2M和4M路由：
```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:335744 hash:4695
totals: mtrie:335744 hash:4695 all:340439

Mtrie Mheap Usage: total: 1.00G, used: 335.20K, free: 1023.74M, trimmable: 1023.72M
    free chunks 3 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```
```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:5414720 hash:41177579
totals: mtrie:5414720 hash:41177579 all:46592299

Mtrie Mheap Usage: total: 1.00G, used: 46.87M, free: 977.19M, trimmable: 955.93M
    free chunks 61 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```
```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:22452608 hash:168544508
totals: mtrie:22452608 hash:168544508 all:190997116

Mtrie Mheap Usage: total: 1.00G, used: 198.37M, free: 825.69M, trimmable: 748.24M
    free chunks 219 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```
VPP从1G IP4堆开始。

## 12.21. IP6堆
IPv6堆用于分配存储IPv6前缀的数据结构所需的内存。IPv6也具有转发和非转发条目的概念，但是对于IPv6，所有转发条目都存储在单个哈希表中（非转发也是如此）。哈希表的关键字包括IPv6表ID。

要查看IPv4表消耗的内存量，请使用：
```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    7 active elements 7 active buckets
    1 free lists
    0 linear search buckets
    arena: base 7f2fe28bf000, next 803c0
           used 525248 b (0 Mbytes) of 33554432 b (32 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    7 active elements 7 active buckets
    1 free lists
    0 linear search buckets
    arena: base 7f2fe48bf000, next 803c0
           used 525248 b (0 Mbytes) of 33554432 b (32 Mbytes)
```
随着我们扩展到128k IPv6条目：
```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    131079 active elements 32773 active buckets
    2 free lists
       [len 1] 2 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 4805c0
           used 4720064 b (4 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    131079 active elements 32773 active buckets
    2 free lists
       [len 1] 2 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 4805c0
           used 4720064 b (4 Mbytes) of 1073741824 b (1024 Mbytes)
```
和256k：
```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    262151 active elements 65536 active buckets
    2 free lists
       [len 1] 6 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 880840
           used 8915008 b (8 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    262151 active elements 65536 active buckets
    2 free lists
       [len 1] 6 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 880840
           used 8915008 b (8 Mbytes) of 1073741824 b (1024 Mbytes)
```
和1M：
```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    1048583 active elements 65536 active buckets
    4 free lists
       [len 1] 65533 free elts
       [len 2] 65531 free elts
       [len 4] 9 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 3882740
           used 59254592 b (56 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    1048583 active elements 65536 active buckets
    4 free lists
       [len 1] 65533 free elts
       [len 2] 65531 free elts
       [len 4] 9 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 3882740
           used 59254592 b (56 Mbytes) of 1073741824 b (1024 Mbytes)
```
从输出可以看出，在这种情况下，IPv6堆已扩展为1GB，而100万个前缀已使用了56MB。

## 12.22. 主堆
主堆用于分配表示控制和数据平面中的FIB条目的对象（请参见控制平面和 数据平面），例如fib_entry_t和load_balance_t。这些来自主堆，因为它们不是特定于协议的（即，它们用于表示IPv4，IPv6或MPLS条目）。

分配了1M前缀后，内存使用情况为：
```
vpp# sh fib mem
FIB memory
 Tables:
            SAFI              Number     Bytes
        IPv4 unicast             1     33619968
        IPv6 unicast             2     118502784
            MPLS                 0         0
       IPv4 multicast            1       1175
       IPv6 multicast            1      525312
 Nodes:
            Name               Size  in-use /allocated   totals
            Entry               72   1048589/ 1048589    75498408/75498408
        Entry Source            40   1048589/ 1048589    41943560/41943560
    Entry Path-Extensions       76      0   /    0       0/0
       multicast-Entry         192      6   /    6       1152/1152
          Path-list             40     18   /    18      720/720
          uRPF-list             16     14   /    14      224/224
            Path                72     22   /    22      1584/1584
     Node-list elements         20   1048602/ 1048602    20972040/20972040
       Node-list heads          8      24   /    24      192/192
```
和2M
```
vpp# sh fib mem
FIB memory
 Tables:
            SAFI              Number     Bytes
        IPv4 unicast             1     33619968
        IPv6 unicast             2     252743040
            MPLS                 0         0
       IPv4 multicast            1       1175
       IPv6 multicast            1      525312
 Nodes:
            Name               Size  in-use /allocated   totals
            Entry               72   2097165/ 2097165    150995880/150995880
        Entry Source            40   2097165/ 2097165    83886600/83886600
    Entry Path-Extensions       76      0   /    0       0/0
       multicast-Entry         192      6   /    6       1152/1152
          Path-list             40     18   /    19      720/760
          uRPF-list             16     18   /    18      288/288
            Path                72     22   /    23      1584/1656
     Node-list elements         20   2097178/ 2097178    41943560/41943560
       Node-list heads          8      24   /    24      192/192
```
但是，情况并非如此简单。上面添加的所有1M前缀都可以通过同一下一跳访问，因此它们使用的路径列表（和路径）是共享的。由于添加了使用不同（下一跳）（多组）下一跳的前缀，因此所需的路径列表和路径的数量将增加。

## 12.23. 统计堆
VPP收集每个路由的统计信息。对于每个路由，VPP都会收集字节和数据包计数器，用于发送到该前缀的数据包（即该路由在数据平面中已匹配）以及通过该前缀发送的数据包（即，匹配的前缀可以通过它到达，就像BGP对等体一样）。这需要在统计信息段中为每个路由分配4个计数器。

下面显示了具有1M，2M和4M路由的统计信息段的大小。
```
total: 1023.99M, used: 127.89M, free: 896.10M, trimmable: 830.94M
total: 1023.99M, used: 234.14M, free: 789.85M, trimmable: 668.15M
total: 1023.99M, used: 456.83M, free: 567.17M, trimmable: 388.91M
```
VPP是从1G统计数据堆开始的。


**`TODO`**

<br/>


# 13. Punting Packets
[Punting Packets](https://fd.io/docs/vpp/master/gettingstarted/developers/punt.html)

“Punt”对不同的人可能意味着不同的事情。在VPP中，当数据包无法由其他任何节点处理时，数据平面将启动。Punt与drop有所不同，因为VPP为系统的其他元素提供了处理此数据包的机会。

punt 的流行含义是将数据包发送到用户/控制平面。这是上述更一般情况的特定选项，其中VPP将数据包移交给控制平面以进行进一步处理。

## 13.1. punt基础设施
异常包是给定节点无法通过常规机制处理的那些。通过VLIB'punt infra'处理异常数据包。有两种类型的节点；源和汇。来源从基础架构和加载时间分配了一个“理由”。当他们在切换期间遇到异常时，它将使用原因标记数据包，并发送平底船调度节点的数据包。接收器将在加载时向下入点注册，因此它可以接收由于该原因而被破坏的数据包。如果由于给定的原因未注册任何接收器，则丢弃数据包，如果多个接收器注册，则将复制数据包。

这种机制使我们能够扩展系统，以处理源节点否则会丢弃的数据包。

## 13.2. Punting控制平面

**Active Punt**
用户/控制平面指定这是我要接收的数据包类型，这是我要发送的数据包的位置。

当前有3种方法来描述如何对要打孔的数据包进行匹配/分类：

* 匹配的UDP端口
* 匹配的IP协议（即OSPF）
* 匹配的平底锅异常原因（请参见上文）

根据要打孔的数据包的类型/分类，该活动平底锅将自己注册到VLIB图中以接收那些数据包。例如，如果它是与UDP端口匹配的数据包，则它将挂接到UDP端口分派功能；udp_register_port（）。

对于被动平底船，unix域套接字只有一个接收器。但是，这方面的工作正在进行中。

请参阅以下网址中的API：vnet / ip / punt.api

**Passive Punt**
VPP输入数据包处理可以描述为一系列分类器。例如，输入分类的序列可以是IP吗？是为了我们吗？是UDP吗？它是已知的UDP端口吗？如果在此管道中的某个点上VPP没有其他分类，则可以对数据包进行打孔，这意味着将其发送到ipX-punt节点。这被描述为无源的，因为控制平面因此正在接收VPP本身不处理的每个数据包。对于被动平底锅，用户可以指定应将数据包发送到何处以及是否/如何管理/限制速率。

请参阅以下网址中的API：vnet / ip / ip.api



**`TODO`**

<br/>

# 14. QUIC HostStack

[QUIC HostStack](https://fd.io/docs/vpp/master/gettingstarted/developers/quic_plugin.html)

>**QUIC（Quick UDP Internet Connections，快速 UDP 网络连接）是一种实验性的网络传输协议。**从网络层级来看，QUIC 是类似于 TCP，UDP 和 SPDY 的数据传输协议，目前正在由 Internet 工程任务组（IETF）进行标准化。
>关于 QUIC 的研究始于 2010 年代初期，由 Google 率先进行的尝试。当时 Google 希望创建一个更快，更强调性能的数据传输协议来代替 HTTPS/HTTP。
>QUIC 借鉴了 TCP、UDP 和 TLS（用于加密）的原理和功能，在这个基础上优化了传输的速度。QUIC 的数据传输从第一个数据包传送（0-RTT）开始立即开始，从而减少了应用程序延迟时间。并且可以在数据量已满时调整管理流程（拥塞控制），从而更快更安全。QUIC 协议在登录成功、推拉流成功的耗时，大幅低于 TCP 协议，优化百分比在 30% 以上，极端场景甚至超过 90%。

quic插件提供了[IETF QUIC协议](https://tools.ietf.org/html/draft-ietf-quic-transport-22)实现。它基于[quicly](https://github.com/h2o/quicly)库。

该插件将QUIC协议添加到VPP的主机堆栈中。因此，QUIC可在内部VPP应用程序和外部应用程序中使用。

* 此插件正在开发中：它应该可以正常工作，但尚未经过全面测试，因此不能在生产中使用。
* 当前仅支持双向流。

>[H2O](https://h2o.examp1e.net/)是新一代HTTP服务器，与上一代的Web服务器相比，它以较低的CPU利用率为用户提供更快的响应。从头开始设计，该服务器充分利用HTTP / 2功能，包括优先的内容服务和服务器推送，可为您的网站访问者带来出色的体验。


* 常见的示例设置是将两个vpp实例互连在一起#twovppinstances
* 确保您的vpp配置文件包含 session { evt_qs_memfd_seg }
* 然后在debug cli（vppctl）中运行session enable

该插件可以在以下情况下进行测试。

**Internal client**
该应用程序是在调试cli上运行的简单命令，用于通过调试cli（vppctl）在QUIC上测试连接性和吞吐量。它不能反映实际情况，主要用于内部测试。

* 在您的第一个实例上运行test echo server uri quic://1.1.1.1/1234
* 然后在第二个test echo client uri quic://20.20.1.1/1

内部客户的资料来源 src/plugins/hs_apps/echo_client.c

**External client**
此设置反映了使用vpp创建快速客户端/服务器的应用程序开发人员的用例。该应用程序是一个外部二进制文件，通过其二进制API连接到VPP。

设置两个互连的vpp之后，可以将quic_echo二进制文件附加到每个vpp。

* 二进制文件可以在 ./build-root/build-vpp[_debug]-native/vpp/bin/quic_echo
* 要运行客户端和服务器使用 quic_echo socket-name /vpp.sock client|server uri quic://1.1.1.1/1234
* 有几个选项可用于自定义发送的数据量，线程数，日志记录和计时。

运行该应用程序时，其行为是首先与给定的对等方建立2个连接，一旦打开所有内容，便开始打开4个quic流并传输数据。流程如下。nclient 2/4

![](_v_images/20201009172346483_27986.png)
这允许在评估协议性能时安排整个设置和拆卸或特定阶段的时间

内部客户的资料来源 src/plugins/hs_apps/sapi/quic_echo.c

## 14.1. VCL客户端
主机栈公开了一个简化的API调用，称为**VCL（阻止posix之类的调用）**，该API由支持QUIC，TCP和UDP的示例客户端和服务器实现使用。

* 二进制文件可以在` ./build-root/build-vpp[_debug]-native/vpp/bin/`
* 创建VCL conf文件 `echo "vcl { api-socket-name /vpp.sock }" | tee /tmp/vcl.conf]`
* 对于服务器 `VCL_CONFIG=/tmp/vcl.conf ; vcl_test_server -p QUIC 1234"`
* 对于客户 `VCL_CONFIG=/tmp/vcl.conf ; vcl_test_client -p QUIC 1.1.1.1 1234"`

内部客户的资料来源 `src/plugins/hs_apps/vcl/vcl_test_client.c`

基本用法是以下客户端
```c
#include <vcl/vppcom.h>
int fd = vppcom_session_create (VPPCOM_PROTO_QUIC);
vppcom_session_tls_add_cert (/* args */);
vppcom_session_tls_add_key (/* args */);
vppcom_session_connect (fd, "quic://1.1.1.1/1234"); /* create a quic connection */
int sfd = vppcom_session_create (VPPCOM_PROTO_QUIC);
vppcom_session_stream_connect (sfd, fd); /* open a quic stream on the connection*/
vppcom_session_write (sfd, buf, n);
```
服务器端
```c
#include <vcl/vppcom.h>
int lfd = vppcom_session_create (VPPCOM_PROTO_QUIC);
vppcom_session_tls_add_cert (/* args */);
vppcom_session_tls_add_key (/* args */);
vppcom_session_bind (fd, "quic://1.1.1.1/1234");
vppcom_session_listen (fd);
int fd = vppcom_session_accept (lfd); /* accept quic connection*/
vppcom_session_is_connectable_listener (fd); /* is true */
int sfd = vppcom_session_accept (fd); /* accept quic stream */
vppcom_session_is_connectable_listener (sfd); /* is false */
vppcom_session_read (sfd, buf, n);
```

## 14.2. QUIC构造

>**QUIC（Quick UDP Internet Connections，快速 UDP 网络连接）是一种实验性的网络传输协议。**从网络层级来看，QUIC 是类似于 TCP，UDP 和 SPDY 的数据传输协议，目前正在由 Internet 工程任务组（IETF）进行标准化。

QUIC构造如下：

* QUIC连接和流都是常规的主机堆栈会话，通过具有其64位句柄的API公开。
* QUIC连接可以创建并定期销毁connect，并close用电话TRANSPORT_PROTO_QUIC。
* 通过connect再次调用并传递新流应属于的连接句柄，可以在连接中打开流。
* 可以通过常规close调用关闭流。
* 可以从与QUIC连接相对应的会话中接受对等方打开的流。
* 可以使用常规会话send和recv流会话上的调用来交换数据。

## 14.3. 数据结构
**Quic依赖于主机堆栈构造**，即**应用程序，会话，transport_connections和app_listeners**。当使用quic协议侦听端口时，外部应用程序：

* 附加到vpp并注册一个 application
* 它创建一个app_listener和一个quic_listen_session。
* 将quic_listen_session依赖于transport_connection（lctx）访问底层udp_listen_session将接收的数据包。
* 当连接请求，我们创建了相同的数据结构（quic_session，qctx，udp_session）和手柄传递给quic_session在接受回调承认建立一个QUIC连接。连接两端的对等方的所有其他UDP数据报将通过udp_session
* 收到流打开请求后，我们创建stream_session和的传输sctx，并将句柄传递stream_session回应用程序。这里没有任何UDP数据结构，因为所有数据报都绑定到该连接。

这些结构的链接如下：

![quic_plugin_datastructures](_v_images/20201009172612152_29979.png =1200x)


**`TODO`**

<br/>

# 15. 在MacOS上进行交叉编译
[在MacOS上进行交叉编译](https://fd.io/docs/vpp/master/gettingstarted/developers/cross_compile_macos.html)
**`略`**

# 16. 云NAT

[云NAT](https://fd.io/docs/vpp/master/gettingstarted/developers/cnat.html)

该插件涵盖了特定的NAT用例，这些用例主要来自容器网络世界。与用于家庭网关的NAT概念相反，没有“外部”和“内部”的概念。我们处理虚拟（或实际）IP以及发送给它们的数据包的转换

## 16.1. 术语和用法
设置NAT将包括创建translation 具有多个后端的。Atranslation是一个三元组，包含：完全限定的IP地址，端口和协议。然后，发往它的所有数据包（ip，端口）将选择一个后端，并遵循其重写规则。

backend由四个重写组件（源和目标地址，源和目标端口）组成，这些组件将在传入途中应用于数据包，并在返回途中还原。

后端通过流哈希同样实现负载均衡。backend为流选择a将触发NAT的创建，该NATsession将存储要重写的数据包和要撤消的数据包，直到流复位或达到超时为止。

session是一个完全解析的9元组， src_ip, src_port, dest_ip, dest_port, proto用于匹配传入的数据包及其新属性new_src_ip, new_src_port, new_dest_ip, new_dest_port。它允许backend粘性和建立连接的快速路径。

这些sessions在常规时间30秒后过期，sessions而在建立TCP连接后1小时后过期。这些可以在vpp的配置文件中更改
```
cnat {
    session-max-age 60
    tcp-max-age 3600
}
```

通过插入以表示的FIB条目来匹配流量client。这些保持引用的数量sessions 和/或translations取决于它们，并且在所有组件消失后将其清除。

## 16.2. 翻译地址
在此示例中，所有目的地为的数据包都30.0.0.2:80将被重写，以便它们的目标IP为20.0.0.1和目的端口8080。这里30.0.0.2必须是虚拟IP，不能将其分配给接口
```
cnat translation add proto TCP vip 30.0.0.2 80 to ->20.0.0.1 8080
```
如果30.0.0.2是接口的地址，我们可以使用以下命令进行相同的转换，并另外更改源。地址为1.2.3.4
```
cnat translation add proto TCP real 30.0.0.2 80 to 1.2.3.4->20.0.0.1 8080
```
要显示现有翻译和会话，您可以使用
```
nat show session verbose
cant show translation
```

## 16.3. SourceNAT传出流量
插件的独立部分允许在每个接口的基础上更改传出流量的源地址。

在以下示例中，所有来自tap0和不来自的流量20.0.0.0/24都将源NAT-ed 30.0.0.1。在返回的途中，翻译将被撤消。

注意：30.0.0.1FIB应该知道地址（例如，分配给接口的地址）
```
cnat snat with 30.0.0.1
cnat snat exclude 20.0.0.0/24
set interface feature tap0 ip4-cnat-snat arc ip4-unicast
```

## 16.4. 其他参数
在vpp的启动文件中，您还可以为

翻译比哈什语 (proto, port) -> translation

会话bihash src_ip, src_port, dest_ip, dest_port, proto -> new_src_ip, new_src_port, new_dest_ip, new_dest_port

snat bihash用于搜索前缀snat exclude
```
cnat {
    translation-db-memory 64K
    translation-db-buckets 1024
    session-db-memory 1M
    session-db-buckets 1024
    snat-db-memory 64M
    snat-db-buckets 1024
}
```

## 16.5. 扩展NAT
该插件构建为可扩展的。目前定义了两种NAT类型，cnat_node_vip.c和cnat_node_snat.c。它们都继承自cnat_node.h提供：

* 会话查找：rv将设置为0是否找到会话
* cnat_translation_ip4基于会话的翻译原语
* 会话创建原语 cnat_session_create

创建会话还将创建反向会话（用于匹配返回流量），并回调将执行转换的NAT节点。


## 16.6. 已知限制
该插件仍在开发中，它缺少以下功能：*负载平衡不支持参数概率*不支持VRF。所有规则仅适用于fib表0 *不支持编程会话处理（删除，生存期更新）*不支持ICMP *不支持仅基于`(proto, dst_addr, dst_port)`源匹配进行流量匹配*统计和会话跟踪仍是基本的。





**`TODO`**

<br/>

<br/>
<div align=right>	以上内容由荣涛翻译整理自网络。</div>