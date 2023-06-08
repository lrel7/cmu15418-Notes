
#  Program 1

### 使用*Spatial Decomposition*
* `workArgs`新增字段，方便给每个thread分配任务区域
*  在函数`mandelbrotThread`中，为每个thread分配任务：将整个区域按行等分，注意最后一个thread需要将所有剩下的行都完成（考虑到行数不一定是thread数的整数倍）
*  命令行输入`./main --t <# of thread>`运行
```c++
// workArgs新增字段
int startRow;
int numRows;
int startCol;
int numCols;

// mandelbrotThread函数中分配任务
int rows_per_thread = height / numThreads;
args[i].startRow = i * rows_per_thread;
if(i < numThreads-1){
    args[i].numRows = rows_per_thread;
}
else{
    // the last thread has to complete all the rest of the work
    args[i].numRows = height - rows_per_thread * (numThreads-1);
}
```

|线程数|理论加速比|实际加速比|
|---|---|---|
|2|2|1.97|
|3|3|1.65|
|4|4|2.35|
|8|8|3.66|
|16|16|4.86|
实际加速比与理论加速比相距甚远，原因在于每个线程分配的任务其实是不均匀的，观察图片可知，需要计算的区域（白色部分）集中在图片中部，因此靠近中间的线程会分配较多的任务量
![[Pasted image 20230607153008.png|300]]
为验证这一猜想，在`workerThreadStart`函数中加入对每个线程工作时间的计时，结果如下：
![[Pasted image 20230607153823.png|150]]
可以看到，位于中间的线程3、4的运行时间最长，上面的猜想正确

### 改进
* 让第i个线程计算所有$k*N+i$行（$k=0,1,\cdots$）
* 具体而言是修改`mandelbrotSerial`函数

view 1的运行结果：

|线程数|理论加速比|实际加速比|
|---|---|---|
|4|4|3.47|
![[Pasted image 20230607155934.png|150]]

view 2的运行结果：

|线程数|理论加速比|实际加速比|
|---|---|---|
|4|4|3.32|
![[Pasted image 20230607160117.png|150]]
可见这样的任务分配确实使得线程之间的工作量更均匀，并且适用于所有类型的图案

---
# Program 2
### 示例函数`absVector`的学习
![[13207103255bd4022abc17bb5bb5187.jpg|300]]
* 每次从连续的内存中取出`VECTOR_WIDTH`个数组成**向量**，进行整体操作

### 基本思路
* 若$N\%VECTOR\_WIDTH\neq 0$，最后一次的向量的后几位是无效的，需要用mask遮掉
* 作乘法应将`result`初始化为全1向量
* 观察serial程序可知，0元素的幂次永远为1，因此要避免0元素的计算；方法是在一开始用全0向量与原向量比对，将0元素用mask遮掉
* 最困难之处在于如何批次处理不同的幂次数：
	* 每作一次乘法，将y向量（幂次向量）批次减1（即减去一个全1向量）
	* 然后将y向量中小于等于0的部分（这说明幂次已经计算完了，不要再参与剩下的计算）都用mask遮掉
	* 接着计算y向量中大于0而又有效的元素个数`count`，如果`count == 0`则终止

### VECTOR-WIDTH对向量利用率的影响

|VECTOR-WIDTH|Utilization|
|---|---|
|2|90.523%|
|4|86.403%|
|8|84.281%|
|16|83.237%|
可以看出，随着`VECTOR-WIDTH`增大，vector的利用率下降，这是因为越长的vector越可能导致更多的位置被mask掉

### arraySumVector
* 先按向量相加
* 最后巧用`hadd`和`interleave`两个函数对结果向量作二分reduce