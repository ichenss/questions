# 网上看到的一些面试题

## 字节跳动 飞书 C++

### 1. new 和 malloc 区别是什么？

* new 是操作符可以运算符重载，而 malloc 是库函数
* 返回值不同，new 操作成功返回对象类型的指针，无需强转，类型安全；而 malloc 返回 void* ，需要强转成我们所需要的类型
* new 分配内存失败时，会抛出 bac_alloc 异常，它不会返回 NULL，malloc 分配失败返回NULL
* new 申请内存无需指定内存大小，编译器会自行计算分配；而 malloc 则需要显式给出内存大小
* new/delete 分配释放对象会调用构造析构；malloc 不会调用
* new[] 和 delete[] 可管理数组，new 数组它会分别调用构造函数初始化每一个数组元素，释放对象时为每个对象调用析构函数；而 malloc 则不会
* malloc 分配内存后，如果内存不足，可使用 realloc 函数进行内存重载分配实现内存扩充；new 没有这样的配套设施来扩充内存

### 2. new 底层用什么实现的？

**`operator new` 和`operator delete`是系统提供的全局函数**

**new的原理**
调用 operator new 函数申请空间
在申请的空间上执行构造函数，完成对象的构造
**delete的原理**
在空间上执行析构函数，完成对象中资源的清理工作
调用 operator delete 函数释放对象的空间
**new T[N]的原理**
调用 operator new[] 函数，在 operator new[] 中实际调用 operator new 函数完成 N 个对象空间的申请
在申请的空间上执行 N 次构造函数
**delete[]的原理**
在释放的对象空间上执行 N 次析构函数，完成 N 个对象中资源的清理
调用operator delete[]释放空间，实际在operator delete[]中调用operator delete来释放空间

* new 调用底层 operator new ；operator new 实际是通过 malloc 函数来申请内存的；如果成功直接返回，如果失败会尝试空间不足应对措施，失败则抛异常
* delete 调用底层 operator delete；operator delete 实际是调用 free 来释放空间
* new[], delete[] 是调用 operator new[]/operator delete[], 然后在调用operator new/operator delete

~~~c++
/*
operator new：该函数实际通过malloc来申请空间，当malloc申请空间成功时直接返回；申请空间
失败，尝试执行空间不足应对措施，如果该应对措施用户设置了，则继续申请，否则抛异常。
*/
void* __CRTDECL operator new(size_t size) _THROW1(_STD bad_alloc)
{
	// try to allocate size bytes
	void* p;
	while ((p = malloc(size)) == 0)
		if (_callnewh(size) == 0)
		{
			// report no memory
			// 如果申请内存失败了，这里会抛出bad_alloc 类型异常
			static const std::bad_alloc nomem;
			_RAISE(nomem);
		}
	return (p);
}

/*
operator delete: 该函数最终是通过free来释放空间的
*/
void operator delete(void* pUserData)
{
	_CrtMemBlockHeader* pHead;

	RTCCALLBACK(_RTC_Free_hook, (pUserData, 0));

	if (pUserData == NULL)
		return;

	_mlock(_HEAP_LOCK);  /* block other threads */
	__TRY

		/* get a pointer to memory block header */
		pHead = pHdr(pUserData);

		/* verify block type */
		_ASSERTE(_BLOCK_TYPE_IS_VALID(pHead->nBlockUse));

		_free_dbg(pUserData, pHead->nBlockUse);

	__FINALLY
		_munlock(_HEAP_LOCK);  /* release other threads */
	__END_TRY_FINALLY

	return;
}

/*
free的实现
*/
#define   free(p)               _free_dbg(p, _NORMAL_BLOCK)

~~~

### 3. 进程和线程的同步方法及效率，哪一种效率最高，为什么?

#### 进程同步

* 管程
* 信号量
* 消息传递

#### 线程同步

* 互斥锁
* 读写锁
* 自旋锁
* atomic 原子类型
* 信号量
* 条件变量

### 4. 构造函数能被声明为虚函数吗，为什么?

* **构造函数不能为虚函数**；因为构造一个对象时，必须知道对象的实际类型，而虚函数是在运行期间确定类型的；并且虚函数的执行依赖于虚函数表，而虚函数表是在构造函数中进行初始化的；而遭构造对象期间，虚函数表还未初始化，将无法进行

### 5. 析构函数能被声明为虚函数吗，为什么?

* **析构函数可以为虚函数**；为了防止内存泄漏；在继承体系中，当基类的指针或引用指向派生类，用基类delete时，如果析构函数没有声明为虚函数，只能析构基类对象，派生类对象将无法析构
