# Ch05 Execution Stages of an OpenCL Application

## 5.1 About the execution stages

平台设置和运行时设置是OpenCL执行阶段的两个主要部分。您的OpenCL应用程序必须获取有关您的硬件的信息，然后设置运行时环境。

### 5.1.1 Platform setup

使用平台API获取有关您的硬件的信息，然后**设置OpenCL上下文**。

平台API可帮助您：
- **确定哪些OpenCL设备可用**。使用OpenCL平台层函数以查找系统上可用的OpenCL设备。
- **设置OpenCL上下文**。创建并设置一个OpenCL上下文和至少一个命令队列来调度内核的执行。

### 5.1.2 Runtime setup

您可以将运行时API用于许多不同的操作。

运行时API可帮助您：
- 创建命令队列。
- 编译并生成**程序对象**。发出命令以编译和构建源代码，并从已编译的代码中提取内核对象。
  
  您必须遵循以下命令序列：
  1. 通过调用`clCreateProgramWithSource()`或`clCreateProgramWithBinary()`创建程序对象：
     - `clCreateProgramWithSource()`: 从内核源代码创建程序对象。
     - `clCreateProgramWithBinary()`: 使用预编译的二进制文件创建程序。
  2. 调用`clBuildProgram()`函数来为系统上的特定设备**编译**程序对象。
- 生成**程序可执行文件**。
- 创建**内核和内存对象**：
  1. 为每个内核调用`clCreateKernel()`函数，或调用`clCreateKernelsInProgram()`函数为OpenCL应用程序中的所有内核创建内核对象。
  2. 使用OpenCL API分配内存缓冲区。 您可以使用`map()`和`unmap()`操作来使应用程序处理器访问数据。
- **排队并执行内核**。
  
  将命令排入队列，这些命令控制内核执行的顺序和同步，内存的映射和取消映射以及内存对象的操作。
  
  要执行内核函数，您必须执行以下步骤：
  1. 调用`clSetKernelArg()`为内核函数定义中的每个参数设置内核参数值。
  2. 确定工作组的大小和用于执行内核的索引空间。
  3. 将内核排队以便在命令队列中执行。
   
- 排队命令，该命令使工作项中的结果对主机可用的。
- 清理未使用的对象。

## 5.2 Finding the available compute devices

要设置OpenCL，必须选择计算设备。调用`clGetDeviceIDs()`来查询OpenCL驱动程序，以获取机器上支持OpenCL的设备列表。

您可以将搜索限制为特定类型的设备或设备类型的任意组合。您还必须指定要返回的设备ID的最大数量。

如果有两个或更多设备，则可以安排不同的`NDrange`以便在设备上进行处理。

如果您的Mali GPU有两个shader核心组，则OpenCL驱动程序会**将每个核心组视为一个单独的OpenCL设备**，每个设备都有自己的`cl_device_id`。 这意味着您可以形成单独的队列，并彼此独立地执行它们。或者，您可以为OpenCL选择一个核心组，而将另一个核心组留给其他GPU任务。

OpenCL驱动程序按核心组ID顺序返回设备阵列。

:star: 注意: 一些Midgard设备是不对称的。一个核心组可能包含四个shader核心，而另一个核心组可能包含两个shader核心。

## 5.3 Initializing and creating OpenCL contexts

当您知道计算机上可用的OpenCL设备并具有至少一个有效的设备ID时，就可以创建一个OpenCL上下文。**上下文将设备分组在一起**，以使内存对象可以在不同的计算设备之间**共享**。

要在设备之间共享工作或与提交到多个命令队列的操作具有相互依赖性，请创建一个包含要以这种方式使用的所有设备的上下文。

将设备信息传递给`clCreateContext()`函数。 例如： 
```c
// Create an OpenCL context
context = clCreateContext(NULL, 1, &device_id, notify_function, NULL, &err);
if (err != CL_SUCCESS)
{
       Cleanup();
       return 1;
}
```

创建OpenCL上下文时，可以选择指定错误通知回调函数。当您将此参数保留为NULL值时，系统不会注册错误通知。

要接收特定OpenCL上下文的运行时错误，请提供回调函数。 例如：
```c
//        Optionally user_data can contain contextual information
//        Implementation specific data of size cb, can be returned in private_info
void context_notify(const char *notify_message, const void *private_info,
                     size_t cb, void *user_data)
{
          printf("Notification:\n\t%s\n", notify_message);
}
```

## 5.4 Creating a command queue

创建OpenCL上下文后，使用`clCreateCommandQueue()`创建命令队列。

OpenCL 1.2**不支持将工作自动分配到设备**。如果要在设备之间共享工作，或要在设备上排队的操作之间具有依赖关系，则必须在同一OpenCL上下文中创建命令队列。

示例命令队列：
```c
// Create a command-queue on the first device available
// on the created context
commandQueue = clCreateCommandQueue(context, device, properties, errcode_ref);
if (commandQueue == NULL)
{
       Cleanup();
       return 1;
}
```
如果有多个OpenCL设备，则必须：
1. 为每个设备创建命令队列。
2. 划分工作。
3. 分别向每个设备提交命令。

## 5.5 Creating OpenCL program objects

创建一个OpenCL程序对象。
   
OpenCL程序对象封装以下组件：
- OpenCL程序source。
- 最新成功构建的程序可执行文件。
- 构建选项。
- 构建日志。
- 程序所针对的设备的列表。

程序对象将加载内核源代码，然后为连接到上下文的设备编译代码。必须在应用程序源代码中使用`__kernel`限定符标识所有内核功能。OpenCL应用程序还可以包含可以从内核函数调用的函数。

加载OpenCL C内核源并从中创建一个OpenCL程序对象。

要创建程序对象，请使用`clCreateProgramWithSource()`函数。 例如：

```c
//        Create OpenCL program
program = clCreateProgramWithSource(context, device, “<kernel source>”);
if (program == NULL)
{
       Cleanup();
       return 1;
}
```

生成OpenCL程序有不同的选项：
- 您可以直接从OpenCL应用程序的源代码创建程序对象，然后在运行时对其进行编译。在应用程序启动时执行此操作，以在应用程序运行时节省计算资源。
  
  如果可以在应用程序调用之间缓存二进制文件，请在平台启动时编译程序对象。

- 为了避免运行时的编译开销，可以使用以前构建的二进制文件构建程序对象。
  
  :star: 注意：带有预构建程序对象的应用程序**不能跨平​​台和驱动程序版本移植**。

从二进制文件创建程序对象与从源代码创建程序对象相似，只是必须为要在其上执行内核的每个设备提供二进制文件。使用`clCreateProgramWithBinary()`函数来执行此操作。

生成二进制文件后，使用`clGetProgramInfo()`函数获取二进制文件。

## 5.6 Building a program executable

创建程序对象后，必须从程序对象的内容构建可执行程序。使用`clBuildProgram()`函数生成可执行文件。

编译程序对象中的所有内核： 
```c
err = clBuildProgram(program, 1, &device_id, "", NULL, NULL);
if (err != CL_SUCCESS)
{
       Cleanup();
       return 1;
}
```

## 5.7 Creating kernel and memory objects

创建内核对象和内存对象有单独的过程。您必须创建内核对象和内存对象。

### 5.7.1 Creating kernel objects

调用clCreateKernel()函数创建单个内核对象，或调用clCreateKernelsInProgram()函数为OpenCL应用程序中的所有内核创建内核对象。

例如：
```c
// Create OpenCL kernel
kernel = clCreateKernel(program, "<kernel_name>", NULL);
if (kernel == NULL)
{
       Cleanup();
       return 1;
}
```
### 5.7.2 Creating memory objects 

创建并注册内核后，将程序数据发送到内核。

过程
1. 将数据打包在一个内存对象中。
2. 将内存对象与内核关联。
   
这些是内存对象的类型：
- Buffer对象: 简单的内存块。
- Images对象: 这些是专门用于表示2D或3D图像的结构。这些是不透明的结构。这意味着您看不到这些结构的实现细节。

要创建buffer对象，请使用`clCreateBuffer()`函数。
要创建image对象，请使用`clCreateImage()`函数。

## 5.8 Executing the kernel 

内核执行分为几个阶段。初始阶段与确定工作组和工作项大小以及数据维度有关。完成初始阶段后，您可以排队并执行您的内核。

### 5.8.1 Determining the data dimensions

如果您的数据是`x`像素宽`y`像素高的图像，则它是二维数据集。如果要处理涉及节点的`x`，`y`和`z`位置的空间数据，则它是三维数据集。

原始数据集中的维数在OpenCL中不必相同。 例如，您可以在OpenCL中将三维数据集作为一维数据集进行处理。

### 5.8.2 Determining the optimal global work size

全局work size是所有维度组合在一起所需的**工作项总数**。

您可以通过在单个工作项中处理多个数据项来更改全局work size。然后，新的全局work size是原始的全局work size除以每个工作项处理的数据项的数量。

如果要确保高性能，全局work size必须很大。通常，数量是几千，但是理想的数量取决于设备中shader核心的数量。

### 5.8.3 Determining the local work-group size

您可以指定将队列加入设备中执行时OpenCL使用的**工作组的大小**。为此，您必须知道在其上执行工作项的OpenCL设备**所允许的最大工作组大小**。要查找特定内核的最大工作组大小，请使用`clGetKernelWorkGroupInfo()`函数并请求`CL_KERNEL_WORK_GROUP_SIZE`属性。

如果**不需要**您的应用程序在**工作项之间共享数据**，请在使内核排队时将`local_work_size`参数设置为`NULL`。这使OpenCL驱动程序可以确定内核的有效工作组大小，但这可能不是最佳工作组大小。

若要获取每个维度上的最大工作组大小，请使用`CL_DEVICE_MAX_WORK_ITEM_SIZES`调用`clGetDeviceInfo()`。 这是针对最简单的内核的，而对于更复杂的内核，尺寸可能会较小。工作组的尺寸的乘积可能会限制工作组的大小。

:star: 注意：要获取工作组的总大小，请使用`CL_KERNEL_WORK_GROUP_SIZE`调用`clGetKernelWorkGroupInfo()`。 如果内核的最大工作组大小小于`128`，则会降低性能。如果是这种情况，请尝试简化内核。

**每个维度的工作组大小必须平均划分为该维度的总数据大小**。这意味着工作组的`x`大小必须平均划分为总数据的`x`大小。如果此要求意味着用额外的工作项填充工作组，请确保其他工作项立即返回并且不执行任何工作。

### 5.8.4 Enqueuing kernel execution 

确定了表示数据所需的维度，每个维度的必要工作项以及适当的工作组大小后，请使用`clEnqueueNDRangeKernel()`将内核排队以便执行。

例如：

```c
size_t globalWorkSize[1] = { ARRAY_SIZE };
size_t localWorkSize[1] = { 4 };
//   Queue the kernel up for execution across the array
errNum = clEnqueueNDRangeKernel(commandQueue, kernel, 1, NULL, globalWorkSize, localWorkSize, 0, NULL, NULL);
if (errNum != CL_SUCCESS)
{
       printf("Error queuing kernel for execution.\n");
       Cleanup();
       return 1;
}
```

### 5.8.5 Executing kernels 

使内核排队执行并不意味着它立即执行。 内核执行将放入命令队列中，以便设备可以稍后对其进行处理。

对`clEnqueueNDRangeKernel()`的调用不是阻塞调用，并且在内核执行之前返回。 有时它可以在内核开始执行之前返回。

可以让内核等待执行，直到先前的事件完成为止。 您可以指定某些内核，直到其他特定内核完成后再执行。

除非创建命令队列时设置了属性`CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE`，否则将按排队顺序执行内核。

排入有序队列的内核会自动等待先前排入同一队列的内核。 您不需要编写任何代码来同步它们。

## 5.9 Reading the results

内核完成执行后，必须使结果对主机可访问。 要从内核访问结果，请使用`clEnqueueMapBuffer()`将缓冲区映射到主机内存。

例如： 
```c
local_buffer = clEnqueueMapBuffer(queue, buffer, CL_NON_BLOCKING, CL_MAP_READ, 0,
               (sizeof(buffer_size), num_deps, &deps[0], NULL, &err);
ASSERT(CL_SUCCESS == err);
```
注意
- 必须调用`clFinish()`才能使缓冲区可用。
- 在前面的示例中，`clEnqueueMapBuffer()`的第三个参数是`CL_NON_BLOCKING`。如果将`clEnqueueMapBuffer()`或`clFinish()`中的此参数更改为`CL_BLOCKING`，则该调用将成为阻塞调用，并且必须在`clEnqueueMapBuffer()`返回之前完成读取。
  
## 5.10 Cleaning up unused objects 

当应用程序不再需要与OpenCL运行时和上下文关联的对象时，必须释放这些资源。 您可以使用几个函数来释放您的OpenCL对象。

这些函数减少关联对象的引用计数： 
- clReleaseMemObject().
- clReleaseKernel().
- clReleaseProgram().
- clReleaseCommandQueue().
- clReleaseContext().

当您的应用程序不再需要它们时，请确保所有OpenCL对象的引用计数都达到零。 您可以通过查询对象来获得引用计数。例如，通过调用`clGetMemObjectInfo()`。
