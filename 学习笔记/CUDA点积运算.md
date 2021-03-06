### CUDA点积运算

1. **申请 共享内存cache**

   ```c
   __shared__ float cache[threadsPerBlock]
   ```

   这里要注意一个概念：**一旦这样声明共享内存，就会创建与线程块的数量相同的数组cache**，即每个**线程块**都会对应一个这样的数组cache且不同线程块中的共享内存是无法交流的

   在这个例子中，共享内存的大小与每个线程块中的线程个数相同；

   ==我个人的理解==：

   * 一个线程块对应一个cache
   * 共享内存用于同一个线程块之间交流，不同线程块之间没法通过共享内存交流

2. **每个线程单独工作**

   ```C
   	//tid等于之前写过的程序的i，即当前线程在所有线程中的位置标志
   	int tid = threadIdx.x + blockIdx.x * blockDim.x;
   	//cacheIndex是该线程在其所在的线程块中的下标
       int cacheIndex = threadIdx.x;   
   	
   
       float temp = 0;
   
   	//当tid(线程总数) < N(向量的大小)时
   	//疑问：如果线程总数大于向量大小的时候的代码咋没有
   
       while (tid < N)
       {
           //处理当前能够使用的线程数的数据，也就是所有线程总数
           temp += a[tid] * b[tid];
           //tid加上一个总线程数
           //因为我们的线程总数小于向量大小，
           //所以我们第一次求取的是当前线程总数所能求取的最大的内容
           //所以我们下一次运算的时候，需要把待处理数据整体向后移动总线程数
           //然后再求取
           //循环往复
           tid += blockDim.x * gridDim.x;
           //blockDim.x = 每个线程块x方向的线程总数
           //gridDim.x = 每个网格x方向线程块的总数
       }
   ```

   ==难点：tid += blockDim.x * gridDim.x==

   **注释中已经详细给出解答**

3. **多个线程协同工作**

   通过共享内存，线程之间完成协作。每个线程将temp的值保存到每个线程块的共享内存(shared memory)中，即数组cache中，相应的代码：

   ```c
   	//将当前结果保存在该线程对应的cache数组的对应位置
   	cache[cacheIndex] = temp;
   	//__syncthreads() is you garden variety thread barrier. Any thread reaching the barrier waits until all of the other threads in that block also reach it. It is designed for avoiding race conditions when loading shared memory, and the compiler will not move memory reads/writes around a __syncthreads(). 
   	//是否可以理解为操作系统中的互斥？
       __syncthreads();
   ```

   ==我个人的理解==：

   * 在不用共享内存之前，我需要定义一个数组来存放每个线程运算的结果

     即：

     ```c
     c[i] = a[i] + b[i]；
     ```

   * 在使用了共享内存之后，在一个内存块中有这样一块区域，这个区域是一块用于存放此线程块中所有线程运算结果的空间，此线程块中的每个线程都可以自由访问这个空间，

   这样每个线程块中对应的数组cache保存的就是每个线程的计算结果。为了节省带宽，这里又采用了并行计算中常用的==归约算法[^1]==，来计算数组中所有值之和，并保存在第一个元素(cache[0])内。这样每个线程就通过共享内存(shared memory)进行数据交流了。具体代码如下所示：

   ```c
   //约算法将每个线程块上的cache数组归约为一个值cache[0]，最终保存在数组c里
   int i = blockDim.x / 2;
   while (i != 0)
   {
       //计算cacheIndex小于i的这些内容
       //因为cache[cacheIndex]+cache[cacheIndex+i]的内容会保存到cache[cacheIndex]中
       //类似于二分
       if (cacheIndex < i)
       {
           cache[cacheIndex] += cache[cacheIndex + i];
       }
       __syncthreads();//确保每个线程已经执行完前面的语句
       
       i /= 2;
   }
   ```

   **NOTE**：不要遗漏\_\_syncthreads()函数

4. **保存归约结果**

   现在每个线程块的计算结果已经被保存到每个共享数组cache的第一个元素cache[0]中，这样可以大大节省带宽。下面就需要将这些归约结果保存到全局内存(global memory)中。

   观察核函数你会发现有一个传入参数——数组c。这个数组是位于全局内存中，每次使用线程块中线程ID为0的数组来将每个线程块的归约结果保存到该数组中，注意这里每个线程块中的结果保存到数组c中与之相对应的位置，即c[blockIdx.x]。

   ```c
   	//选择每个线程块中线程索引为0的线程将最终结果传递到全局内存中
   	if (cacheIndex == 0)
       {
           c[blockIdx.x] = cache[0];
       }
   ```


[^1]: parallel reduction 可以理解为将数组中所有数求和的过程并行化



### 如果不使用线程共享内存

* 如果不使用线程共享内存，在\_\_global\_\_函数里面开辟一个内存空间，这样就相当于每个线程都开辟了一个内存空间，并且他们之间是没办法进行通信的，也就是说这个办法行不通。
* 如果使用CPU中的内存进行处理，在\_\_global\_\_函数传入一个数组，用来存放每个线程的运算结果，再把结果带回CPU，然后在CPU中把所有结果相加。这个方法倒是可行，但是就不完全是GPU计算了。。。