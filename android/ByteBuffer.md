## ByteBuffer(缓冲区)

- Capacity  : 容量，在缓冲区建立时已经不可改变
- Limit : 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。且极限是可以修改的
- Position ： 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备
- Mark： 标记，调用mark()来设置mark=position，再调用reset()可以让position恢复到标记的位置

## 实例化

- allocate(int capacity)  从堆空间中分配一个容量大小为capacity的byte数组作为缓冲区的byte数据存储器
- allocateDirect(int capacity)  是不使用JVM堆栈而是通过操作系统来创建内存块用作缓冲区，它与当前操作系统能够更好的耦合，因此能进一步提高I/O操作速度。但是分配直接缓冲区的系统开销很大，因此只有在缓冲区较大并长期存在，或者需要经常重用时，才使用这种缓冲区
- wrap(byte[] array)    这个缓冲区的数据会存放在byte数组中，bytes数组或buff缓冲区任何一方中数据的改动都会影响另一方。其实ByteBuffer底层本来就有一个bytes数组负责来保存buffer缓冲区中的数据，通过allocate方法系统会帮你构造一个byte数组
- wrap(byte[] array, int offset, int length)   在上一个方法的基础上可以指定偏移量和长度，这个offset也就是包装后byteBuffer的position，而length呢就是limit-position的大小，从而我们可以得到limit的位置为length+position(offset)

## 方法

- limit(), limit(10)    其中读取和设置这4个属性的方法的命名和jQuery中的val(),val(10)类似，一个负责get，一个负责set
- reset()    把position设置成mark的值，相当于之前做过一个标记，现在要退回到之前标记的地方
- clear()   position = 0;limit = capacity;mark = -1; 有点初始化的味道，但是并不影响底层byte数组的内容
- flip()  limit = position;position = 0;mark = -1; 翻转，也就是让flip之后的position到limit这块区域变成之前的0到position这块，翻转就是将一个处于存数据状态的缓冲区变为一个处于准备取数据的状态
- rewind()  把position设为0，mark设为-1，不改变limit的值
- remaining()  return limit - position; 返回limit和position之间相对位置差
- hasRemaining()   return position < limit返回是否还有未读内容
- compact()  把从position到limit中的内容移到0到limit-position的区域内，position和limit的取值也分别变成limit-position、capacity。如果先将positon设置到limit，再compact，那么相当于clear()
- get()   相对读，从position位置读取一个byte，并将position+1，为下次读写作准备
- get(int index)    绝对读，读取byteBuffer底层的bytes中下标为index的byte，不改变position
- get(byte[] dst, int offset, int length) 从position位置开始相对读，读length个byte，并写入dst下标从offset到offset+length的区域
- put(byte b)  相对写，向position的位置写入一个byte，并将postion+1，为下次读写作准备
- put(int index, byte b)  绝对写，向byteBuffer底层的bytes中下标为index的位置插入byte b，不改变position
- put(ByteBuffer src)   用相对写，把src中可读的部分（也就是position到limit）写入此byteBuffer
- put(byte[] src, int offset, int length) 从src数组中的offset到offset+length区域读取数据并使用相对写写入此byteBuffer

​	

