>本文来自Linux官方文档英文版，由于需要使用Linux的GPIO进行实验，我翻译了这篇文档。

本文档描述了GPIO框架的使用者接口。注意它描述了新的基于描述符的接口。
不推荐使用的基于整数的GPIO接口请参考gpio-legacy.txt。

获取和使用GPIO的函数可以通过include以下文件来获得：

```
#include <linux/gpio/consumer.h>
```


基于描述符的GPIO接口的所有函数都是以gpiod\_为前缀。 gpio\_前缀用于传统接口。内核中的其他函数不应该使用这些前缀。强烈建议不要使用legacy的接口函数，新的代码应该使用<linux/gpio/consumer.h>中的基于描述符的接口函数。

## 获取和释放GPIO

使用基于描述符的接口时，GPIO被作为一个描述符来使用，
必须通过调用gpiod\_get()函数来获取该描述符。像许多其他内核子系统一样，gpiod\_get()接收将使用GPIO的设备和所请求的GPIO将要使用的功能:
```
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id,
		enum gpiod_flags flags)
```

如果通过一起使用多个GPIO来实现功能（例如，简单的LED显示数字的设备，可以指定一个额外的索引参数：

```
struct gpio_desc *gpiod_get_index(struct device *dev,
		const char *con_id, unsigned int idx,
		enum gpiod_flags flags)
```

有关DeviceTree情况中con\_id参数的更详细说明请参阅Documentation/gpio/board.txt

flags参数用于可选地指定GPIO的方向和初始值，它的值可以是：


*  GPIOD\_ASIS或0表示根本不初始化GPIO。需要随后使用专门的函数设置方向
*  GPIOD\_IN初始化GPIO作为输入。
*  GPIOD\_OUT\_LOW将GPIO初始化为输出，值为0。
*  GPIOD\_OUT\_HIGH将GPIO初始化为输出，值为1。
*  GPIOD\_OUT\_LOW\_OPEN\_DRAIN:与GPIOD\_OUT\_LOW相同，但强制以开漏的方式使用
*  GPIOD\_OUT\_HIGH\_OPEN\_DRAIN:与GPIOD\_OUT\_HIGH相同，但强制以开漏的方式使用
\end{itemize}


最后两个标志用于必须开漏方式的情况，比如GPIO被用作I2C时，如果该GPIO尚未在映射（参见board.txt）中被配置为开漏方式，将被强制配置为开漏方式并给出WARNING。

这两个函数都返回有效的GPIO描述符或可被IS\_ERR()检查的错误代码（它们永远不会返回NULL指针）。 返回-ENOENT只会发生在当且仅当没有为设备/功能/索引三元组成功分配GPIO的时候。其他错误代码用于已成功分配GPIO，但在试图获得它的时候发生了错误的情况：这可以用于区分错误原因是可选GPIO参数错误还是GPIO缺失这两种情况。

在允许GPIO不存在的情况下，可以使用gpiod\_get\_optional()和gpiod\_get\_index\_optional()函数。这两个函数在没有成功分配到GPIO的时候返回NULL而不是-ENOENT。

```
struct gpio_desc *gpiod_get_optional(struct device *dev,
		const char *con_id,
		enum gpiod_flags flags)
```

```
struct gpio_desc *gpiod_get_index_optional(struct device *dev,
		const char *con_id,
		unsigned int index,
		enum gpiod_flags flags)
```

请注意，gpio\_get*\_optional()函数（及其变体）不同于其余gpiolib的API，当禁用gpiolib支持时它们也会返回NULL。这对驱动程序作者很有帮助，因为这代表他们不需要考虑-ENOSYS返回码这种特殊情况。但是，系统集成商应该注意在需要它的系统上启用gpiolib。

对于使用多个GPIO的函数，可以通过一次调用获得所有需要的GPIO：

```
struct gpio_descs * gpiod_get_array(struct device * dev,
		const char * con_id,enum gpiod_flags flags)
```


此函数返回一个struct gpio\_descs，其中包含一个描述符数组：

```
struct gpio_descs {
		unsigned int ndescs;
		struct gpio_desc *desc[];
	}
```

如果没有GPIO被分配，则以下函数返回NULL而不是-ENOENT：

```
struct gpio_descs *gpiod_get_array_optional(struct device *dev,
		const char *con_id,
		enum gpiod_flags flags)
```

这些函数的变体： 

```
struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id,
		enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_index(struct device *dev,
		const char *con_id,
		unsigned int idx,
		enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_optional(struct device *dev,
		const char *con_id,
		enum gpiod_flags flags)

struct gpio_desc *devm_gpiod_get_index_optional(struct device *dev,
		const char *con_id,
		unsigned int index,
		enum gpiod_flags flags)

struct gpio_descs *devm_gpiod_get_array(struct device *dev,
		const char *con_id,
		enum gpiod_flags flags)

struct gpio_descs *devm_gpiod_get_array_optional(struct device *dev,
		const char *con_id,
		enum gpiod_flags flags)
```

可以使用gpiod\_put()函数释放GPIO描述符：

```
void gpiod_put(struct gpio_desc *desc)
```

对于GPIO数组，可以使用此函数：

```
void gpiod_put_array(struct gpio_descs *descs)
```


在调用这些函数之后，严格禁止使用被释放的描述符。也不允许在使用gpiod\_get\_array()获取的数组中单独使用gpiod\_put()释放描述符。

相关变体：

```
void devm_gpiod_put(struct device *dev, struct gpio_desc *desc)

void devm_gpiod_put_array(struct device *dev, struct gpio_descs *descs)
```

## 使用GPIO
 
### 设置方向

设备驱动必须首先确定GPIO的方向。如果已经为gpiod\_get * ()提供了nodirection设置标志，则可以通过调用gpiod\_direction \_*()函数之一来完成：

```
int gpiod_direction_input(struct gpio_desc *desc)
int gpiod_direction_output(struct gpio_desc *desc, int value)
```
成功返回值为零，否则返回值为负的错误代码。该返回值应该被检查，因为get/set调用不会返回错误，所以错误的配置是有可能的。您通常应该在任务上下文进行这些调用。但是，对于自旋锁安全（Spinlock-Safe）的GPIO，可以作为板级设置初期的一部分，在启用任务之前使用它们。

对于输出GPIO，提供的值将成为初始输出值。这有助于避免系统启动期间的信号故障。驱动程序还可以查询GPIO的当前方向：

```
int gpiod_get_direction(const struct gpio_desc *desc)
```


此函数返回0表示输出，1表示输入，或错误代码（如果出错）

要意识到GPIO没有默认的方向。因此，\emph{使用GPIO而不首先设置其方向是非法的，并且将导致未定义的行为！}
### Spinlock-Safe的GPIO访问

大多数GPIO控制器可通过存储器读/写指令访问。对于不能睡眠，并且可以安全地从内部hard（非线程的）IRQ handler和类似的上下文中完成的操作（即原子操作中），使用以下调用来访问GPIO：

```
int gpiod_get_value(const struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);
```

value为布尔值，零为低，非零为高。读取输出引脚的值时，返回的值应该是引脚上的值。由于包括开漏信号和输出延迟在内的问题，它并不总是匹配指定的输出值。

get/set调用不会返回错误，因为“无效的GPIO”应该在这之前就从gpiod\_direction\_*()中得知。但请注意，并非所有平台都可以读取输出引脚的值;对于那些不能读取的平台，函数永远返回零。另外，使用这些函数访问需要睡眠才能安全访问的GPIO（见下文）是错误的操作。

### 允许睡眠的GPIO访问

有些GPIO控制器必须使用基于消息的总线（如I2C或SPI）访问。读取或写入这些GPIO值的命令需要等待到达队列的头部以传输命令并获得其响应。

这样就需要允许睡眠，导致这类GPIO的访问不能在内部IRQ处理程序内（原子上下文）完成。

支持这种类型的GPIO的平台通过从以下调用返回非零来区别于其他GPIO：

```
int gpiod_cansleep(const struct gpio_desc *desc)
```

要访问此类GPIO，请使用另一组GPIO访问函数：

```
int gpiod_get_value_cansleep(const struct gpio_desc * desc)
void gpiod_set_value_cansleep(struct gpio_desc * desc，int value)
```

访问这样的GPIO需要一个可以休眠的上下文，例如一个threaded IRQ处理程序，并且必须使用上述访问函数而不是Spinlock-Safe的没有cansleep()后缀的访问函数。

除了可以睡眠，无法在hardIRQ处理程序访问的特点以外，这些调用与Spinlock-Safe的调用相同。

### 低有效和开漏语义
由于使用者不必关心物理线路级别，所有的gpiod\_set\_value\_xxx()或 gpiod\_set\_array\_value\_xxx() 函数都以逻辑值操作。通过这种方式，它们会将低电平有效的性质考虑在内。这意味着它们会检查GPIO是否配置为低电平有效，如果是，它们会在物理线路电平被驱动之前调整传递的值。

这同样适用于开漏或开源输出：它们并不输出高电平（开漏）或低电平（开源），它们只是将输出切换到高阻抗值。使用者应该不需要关注。（有关的详细信息，请参阅driver.txt中关于开漏的细节。）

这样，所有gpiod\_set\_(array)\_value\_xxx()函数都将参数“value”解释为“asserted”（“1”）或“de-asserted”（“0” ）。相应地驱动物理线路级别。

例如，如果设置了GPIO的低电平有效属性，并且gpiod\_set\_(array)\_value\_xxx()传递了“asserted”（“1”），则物理线路电平将被驱动为低电平。

总结：
<table border="1">
  <tr>
    <th>函数（示例）</th>
    <th>线路属性</th>
    <th>物理线路</th>
  </tr>
  <tr>
    <td>gpiod_set_raw_value(desc, 0);</td>
    <td>-</td>
    <td>低电平</td>
  </tr>
  <tr>
    <td>gpiod_set_raw_value(desc, 0);</td>
    <td>-</td>
    <td>高电平</td>
  </tr>
  <tr>
    <td>gpiod_set_value(desc, 0);</td>
    <td>默认（高电平有效）</td>
    <td>低电平</td>
  </tr>
  <tr>
    <td>gpiod_set_value(desc, 1);</td>
    <td>默认（高电平有效）</td>
    <td>高电平</td>
  </tr>
 <tr>
    <td>gpiod_set_value(desc, 0);</td>
    <td>低电平有效</td>
    <td>高电平</td>
  </tr>
  <tr>
    <td>gpiod_set_value(desc, 1);</td>
    <td>低电平有效</td>
    <td>低电平</td>
  </tr>
<tr>
    <td>gpiod_set_value(desc, 0);</td>
    <td>开漏</td>
    <td>低电平</td>
  </tr>
  <tr>
    <td>gpiod_set_value(desc, 1);</td>
    <td>开漏</td>
    <td>高阻态</td>
  </tr>
<tr>
    <td>gpiod_set_value(desc, 0);</td>
    <td>开漏</td>
    <td>高阻态</td>
  </tr>
  <tr>
    <td>gpiod_set_value(desc, 1);</td>
    <td>开漏</td>
    <td>高电平</td>
  </tr>
</table>

可以使用*set\_raw /'get\_raw函数覆盖这些语义，但应尽可能避免，尤其是系统无关的驱动程序，它们不需要关心实际的物理线路级别而是关心逻辑值。
### 访问原始GPIO值
对于的确需要管理GPIO线路物理状态的使用者，他们关心设备将实际接收到的值。

下面的一组调用忽略GPIO的低有效或开漏属性，并在原始线路状态上工作：

```
int gpiod_get_raw_value(const struct gpio_desc *desc)
void gpiod_set_raw_value(struct gpio_desc *desc, int value)
int gpiod_get_raw_value_cansleep(const struct gpio_desc *desc)
void gpiod_set_raw_value_cansleep(struct gpio_desc *desc, int value)
int gpiod_direction_output_raw(struct gpio_desc *desc, int value)
```

还可以使用以下方法查询GPIO的低有效属性：

```
int gpiod_is_active_low(const struct gpio_desc *desc)
```

请注意，这些函数只能在使用者明白自己在做什么的情况下使用；驱动程序一般不应该关心线路物理状态或开漏语义。
### 使用单个函数调用访问多个GPIO

以下函数获取或设置GPIO数组的值：

```
int gpiod_get_array_value(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array);
int gpiod_get_raw_array_value(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array);
int gpiod_get_array_value_cansleep(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array);
int gpiod_get_raw_array_value_cansleep(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array);

void gpiod_set_array_value(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array)
void gpiod_set_raw_array_value(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array)
void gpiod_set_array_value_cansleep(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array)
void gpiod_set_raw_array_value_cansleep(unsigned int array_size,
		struct gpio_desc **desc_array,
		int *value_array)
```

数组可以是任意一组GPIO。如果相应的芯片驱动器支持，这些函数将尝试同时访问属于同一存储体或芯片的GPIO。在这种情况下，可以预期显著改善的性能。如果无法同时访问，GPIO将按顺序访问。

这些函数有三个参数：

*  array\_size - 数组元素的数量
*  desc\_array - GPIO描述符数组
*  value\_array - 存储GPIO值（get）的数组或要分配给GPIO的值数组（set）

可以使用gpiod\_get\_array()函数或其变体之一来获取描述符数组。如果该函数返回的描述符组与所需的GPIO组匹配，则只需使用gpiod\_get\_array()返回的struct gpio\_descs即可访问这些GPIO：

```
struct gpio_descs *my_gpio_descs = gpiod_get_array(...);
gpiod_set_array_value(my_gpio_descs->ndescs, my_gpio_descs->desc,
		my_gpio_values);
```

也可以访问完全任意的描述符数组。可以使用gpiod\_get()和gpiod\_get\_array()的任意组合来获得描述符。之后，必须先手动设置描述符数组才能将其传递给上述函数之一。

注意，为了获得最佳性能，属于同一芯片的GPIO应该在描述符数组中是连续的。

gpiod\_get\_array\_value()及其变体成功时返回0，错误返回负数。请注意与gpiod\_get\_value()的区别，gpiod\_get\_value()在成功传递GPIO值时返回0或1。使用数组函数时，GPIO值存储在value\_array中，而不是作为返回值传回。
### GPIO映射到IRQ

GPIO行经常被使用作为IRQ。您可以使用以下调用获取与给定GPIO相对应的IRQ编号：

```
int gpiod_to_irq(const struct gpio_desc *desc)
```

如果映射不成功，它将返回IRQ编号或负的errno代码（很可能是因为该特定GPIO不能用作IRQ）。使用未使用gpiod\_direction\_input()设置为输入的GPIO，或者使用最初不是来自gpiod\_to\_irq()的IRQ编号，是错误的操作。 gpiod\_to\_irq()不允许休眠。

从gpiod\_to\_irq()返回的非错误值可以传递给request\_irq()或free\_irq()。它们通常通过特定于板的初始化代码存储到平台设备的IRQ资源中。注意，IRQ触发选项是IRQ接口的一部分，例如， IRQF\_TRIGGER\_FALLING，系统唤醒功能.
### GPIO和ACPI
在ACPI系统上，GPIO由设备的\_CRS配置对象列出的GpioIo()/ GpioInt()资源描述。这些资源不提供GPIO的连接ID（名称），因此有必要为此目的使用附加机制。

符合ACPI 5.1或更新版本的系统可能可以提供\_DSD配置对象，它可以用于提供\_CRS中的GpioIo()/ GpioInt()资源描述的特定GPIO的连接ID。如果是这种情况，它将由GPIO子系统自动处理。但是，如果不存在\_DSD，则GpioIo()/ GpioInt()资源与GPIOconnection ID之间的映射需要由设备驱动程序提供。

有关详细信息，请参阅Documentation/acpi /gpio-properties.txt
### 使用旧版GPIO子系统进行交互
许多内核子系统仍使用传统的基于整数的GPIO接口。虽然强烈建议将它们升级到更安全的基于描述符的API，但以下两个函数允许您将GPIO描述符转换为GPIO整数命名空间，及其反向转换：

```
int desc_to_gpio(const struct gpio_desc *desc)
struct gpio_desc *gpio_to_desc(unsigned gpio)
```

只要没有释放GPIO描述符，就可以安全地使用desc\_to\_gpio()返回的GPIO号。同样，必须正确获取传递给gpio\_to\_desc()的GPIO号，并且只有在获取GPIO号后才能使用返回的GPIO描述符。

禁止使用不同类的API分别获取和释放GPIO，这是一个未进行检查的错误。
