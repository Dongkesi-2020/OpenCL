# Ch04 Developing an OpenCL Application

## 4.1 Software and hardware requirements for Mali GPU OpenCL development

OpenCL的实现可用于多个操作系统。 您可以在使用有OpenCL实现的其他硬件平台上进行开发。

要为Mali GPU开发OpenCL应用程序，您需要：
- 兼容的OS。
- Mali GPU OpenCL驱动程序。
- 带有Mali GPU的平台。

:star: 注意，Mali GPU必须是Mali Midgard或Bifrost GPU。

根据不同系统的结果估算Mali GPU性能将产生不准确的数据。

## 4.2 Development stages for OpenCL

开发OpenCL应用程序分几个阶段。首先，您必须确定要并行化的内容。然后，您必须编写内核。最后，为内核编写基础结构并执行它们。

您必须执行以下阶段来开发和使用OpenCL应用程序：

- 确定要并行化的对象

  决定使用OpenCL的第一步是查看应用程序的工作，并确定应用程序中可以并行运行的部分。这通常是开发OpenCL应用程序中最难的部分。
  
  :star: 注意，只有在可能会有好处的情况下，才需要将应用程序的各个部分转换为OpenCL。分析您的应用程序以找到最活跃的部分，并考虑将这些部分进行转换。

- 编写内核
  
  OpenCL应用程序由一组内核函数组成。您必须编写执行计算的内核。
   
  如果可能，请对内核进行分区，以便在它们之间传输最少的数据。**加载大量数据通常是操作中最昂贵的部分**。

- 为内核编写基础架构
  
  OpenCL应用程序需要基础架构代码来设置数据并为执行做好准备，

- 执行内核

  使内核排队执行并读回结果。