---
title: pthread_rwlock_t
categories: iOS原理
---

## 一、读写锁

读写锁实际是一种特殊的<font color=#cc0000>自旋锁</font>，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。

<font color=#cc0000>读操作可以共享，写操作是排他的</font>，可以有多个在读（与 CPU 数相关），只能有唯一个在写，但不能同时既有读者又有写者。

如果读写锁当前没有读者，也没有写者，那么写者可以立刻获得读写锁，否则它必须自旋在那里，直到没有任何写者或读者。如果读写锁没有写者，那么读者可以立即获得该读写锁，否则读者必须自旋在那里，直到写者释放该读写锁。

具有强读者同步和强写者同步两种形式：

- 强读者同步：当写者没有进行写操作，读者就可以访问；

- 强写者同步：当所有写者都写完之后，才能进行读操作。

在强写者情况，读者需要最新的信息，一些事实性较高的系统可能会用到该锁，比如定票之类的。


## 二、特性

<font color=#cc0000>一次只有一个线程可以占有写模式的读写锁，但是可以有多个线程同时占有读模式的读写锁</font>。

正因为这个特性，当读写锁是写加锁状态时，在这个锁被解锁之前, 所有试图对这个锁加锁的线程都会被<font color=#cc0000>阻塞</font>。

当读写锁在读加锁状态时, 所有试图以读模式对它进行加锁的线程都可以得到访问权，但是如果线程希望以写模式对此锁进行加锁，它必须直到所有的线程释放锁。

通常, 当读写锁处于读模式锁住状态时，如果有另外线程试图以写模式加锁，<font color=#cc0000>读写锁通常会阻塞随后的读模式锁请求</font>，这样可以<font color=#cc0000>避免读模式锁长期占用</font>，而等待的写模式锁请求长期阻塞.

读写锁适合于对数据结构的读次数比写次数多得多的情况。因为读模式锁定时可以共享，以写模式锁住时意味着独占，所以读写锁又叫共享-独占锁。

## 三、小结

1. 互斥锁与读写锁的区别

	当访问临界区资源时（访问的含义包括所有的操作：读和写），需要上互斥锁；
	
	当对数据（互斥锁中的临界区资源）进行读取时，需要上读取锁，当对数据进行写入时，需要上写入锁。

2. 读写锁的优点

	对于读数据比修改数据频繁的应用，用读写锁代替互斥锁可以提高效率。因为使用互斥锁时，即使是读出数据（相当于操作临界区资源）都要上互斥锁，而采用读写锁，则可以在任一时刻允许多个读者存在，提供了更高的并发度，同时在某个写入者修改数据期间保护该数据，以免任何其它读出者或写入者的干扰。

3. 读写锁描述：

	获取一个读写锁用于读称为共享锁，获取一个读写锁用于写称为独占锁，因此这种对于某个给定资源的共享访问也称为共享-独占上锁。

## 四、使用读写锁

配置读写锁的属性之后，即可初始化读写锁。以下函数用于初始化或销毁读写锁、锁定或解除锁定读写锁或尝试锁定读写锁。

1. 初始化读写锁

	使用 pthread\_rwlock\_init(3C) 通过 attr 所引用的属性初始化 rwlock 所引用的读写锁。

	```
	/**
	 * @param  attr  如果为 NULL，则使用缺省的读写锁属性，其作用与传递缺省读写锁属性对象的地址相同
	 *
	 * @return  如果成功，返回 0，否则将返回用于指明错误的错误号。 EINVAL : attr 或者 rwlock 指定的值无效 
	 */
	int pthread_rwlock_init(pthread_rwlock_t *rwlock, const pthread_rwlockattr_t *attr);
	
	
	
	pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
	```

	如果 attr 为 NULL，则使用缺省的读写锁属性，其作用与传递缺省读写锁属性对象的地址相同。

	初始化读写锁之后，该锁可以<font color=#cc0000>使用任意次数</font>，而无需重新初始化。成功初始化之后，读写锁的状态会变为<font color=#cc0000>已初始化和未锁定</font>。如果调用 pthread\_rwlock\_init() 来指定已初始化的读写锁，则结果是不确定的。如果读写锁在使用之前未初始化，则结果是不确定的。

	如果缺省的读写锁属性适用，则 PTHREAD\_RWLOCK\_INITIALIZER 宏可初始化<font color=#cc0000>以静态方式分配的读写锁</font>，其作用与通过调用pthread\_rwlock\_init() 并将参数 attr 指定为 NULL 进行动态初始化等效，区别在于<font color=#cc0000>不会执行错误检查</font>。

	如果 pthread\_rwlock\_init() 失败，将不会初始化 rwlock，并且 rwlock 的内容是不确定的。

2. 获取读写锁中的读锁

	pthread\_rwlock\_rdlock(3C) 可用来向 rwlock 所引用的读写锁应用读锁。

	```
	#include <pthread.h>
	
	/**
	 * @return  如果成功，返回 0。否则，将返回用于指明错误的错误号。 EINVAL : attr 或 rwlock 指定的值无效
	 *
	 */
	int  pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
	```

	如果写入器未持有读锁，并且没有任何写入器基于该锁阻塞，则调用线程会获取读锁。如果写入器未持有读锁，但有多个写入器正在等待该锁时，<font color=#cc0000>调用线程是否能获取该锁是不确定的</font>。如果某个写入器持有读锁，则调用线程无法获取该锁。如果调用线程未获取读锁，则它将阻塞。调用线程必须获取该锁之后，才能从 pthread\_rwlock\_rdlock() 返回。如果在进行调用时，调用线程持有 rwlock 中的写锁，则结果是不确定的。

	为避免写入器资源匮乏，允许在多个实现中使<font color=#cc0000>写入器的优先级高于读取器</font>。

	一个线程可以在 rwlock 中持有多个并发的读锁，该线程可以成功调用 pthread\_rwlock\_rdlock() n 次。该线程必须调用 pthread\_rwlock\_unlock() n 次才能执行匹配的解除锁定操作。

	如果针对未初始化的读写锁调用 pthread\_rwlock\_rdlock()，则结果是不确定的。

	线程信号处理程序可以处理传送给等待读写锁的线程的信号。从信号处理程序返回后，线程将继续等待读写锁以执行读取，就好像线程未中断一样。

3. 读取非阻塞读写锁中的锁

	pthread\_rwlock\_tryrdlock(3C)应用读锁的方式与 pthread\_rwlock\_rdlock() 类似，区别在于如果任何线程持有 rwlock 中的写锁或者写入器基于 rwlock 阻塞，则 pthread\_rwlock\_tryrdlock() 函数会失败。

	```
	#include <pthread.h>
	
	
	/**
	 * @return 如果获取了用于在 rwlock 所引用的读写锁对象中执行读取的锁，返回 0。否则，返回用于指明错误的错误号
	 *
	 *         EBUSY : 无法获取读写锁以执行读取，因为写入器持有该锁或者基于该锁已阻塞
	 */
	int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
	```

4. 写入读写锁中的锁

	pthread\_rwlock\_wrlock(3C) 可用来向 rwlock 所引用的读写锁应用写锁。

	```
	#include <pthread.h>
	
	
	/**
	 * @return  如果获取了用于在 rwlock 所引用的读写锁对象中执行写入的锁，返回 0。否则，返回指明错误的错误号
	 */
	int  pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
	```

	如果没有其他读取器线程或写入器线程持有读写锁 rwlock，则调用线程将获取写锁，否则，调用线程将阻塞。调用线程必须获取该锁之后，才能从 pthread\_rwlock\_wrlock() 调用返回。如果在进行调用时，调用线程持有读写锁（读锁或写锁），则结果是不确定的。

	为避免写入器资源匮乏，允许在多个实现中使写入器的优先级高于读取器。

	如果针对未初始化的读写锁调用 pthread\_rwlock\_wrlock()，则结果是不确定的。

	线程信号处理程序可以处理传送给等待读写锁以执行写入的线程的信号。从信号处理程序返回后，线程将继续等待读写锁以执行写入，就好像线程未中断一样。

5. 写入非阻塞读写锁中的锁

	pthread\_rwlock\_trywrlock(3C)应用写锁的方式与 pthread\_rwlock\_wrlock() 类似，区别在于如果任何线程当前持有用于读取和写入的 rwlock，则pthread\_rwlock\_trywrlock() 函数会失败。

	```
	#include <pthread.h>
	
	/**
	 * @return  如果获取了用于在 rwlock 引用的读写锁对象中执行写入的锁，则返回 0，否则，返回用于指明错误的错误号
	 *          EBUSY : 无法为写入获取读写锁，因为已为读取或写入锁定该读写锁
	 */
	int pthread_rwlock_trywrlock(pthread_rwlock_t  *rwlock);
	```

	如果针对未初始化的读写锁调用 pthread\_rwlock\_trywrlock()，则结果是不确定的。

	线程信号处理程序可以处理传送给等待读写锁以执行写入的线程的信号。从信号处理程序返回后，线程将继续等待读写锁以执行写入，就好像线程未中断一样。
	
6. 解除锁定读写锁

	pthread\_rwlock\_unlock(3C) 可用来释放在 rwlock 引用的读写锁对象中持有的锁。

	```
	#include <pthread.h>
	
	/**
	 *  @return 如果成功返回 0，否则返回用于指明错误的错误号
	 */
	int pthread_rwlock_unlock (pthread_rwlock_t  *rwlock);
	```

	如果调用线程未持有读写锁 rwlock，则结果是不确定的。

	如果通过调用 pthread\_rwlock\_unlock() 来释放读写锁对象中的读锁，并且其他读锁当前由该锁对象持有，则该对象会保持读取锁定状态。如果 pthread\_rwlock\_unlock() 释放了调用线程在该读写锁对象中的最后一个读锁，则调用线程不再是该对象的属主。如果 pthread\_rwlock\_unlock() 释放了该读写锁对象的最后一个读锁，则该读写锁对象将处于<font color=#cc0000>无属主、解除锁定状态</font>。

	如果通过调用 pthread\_rwlock\_unlock() 释放了该读写锁对象的最后一个写锁，则该读写锁对象将处于无属主、解除锁定状态。

	如果 pthread\_rwlock\_unlock() 解除锁定该读写锁对象，并且多个线程正在等待获取该对象以执行写入，则通过调度策略可确定获取该对象以执行写入的线程。如果多个线程正在等待获取读写锁对象以执行读取，则通过调度策略可确定等待线程获取该对象以执行写入的顺序。<font color=#cc0000>如果多个线程基于 rwlock 中的读锁和写锁阻塞，则无法确定读取器和写入器谁先获得该锁</font>。

	如果针对未初始化的读写锁调用 pthread\_rwlock\_unlock()，则结果是不确定的。

7. 销毁读写锁

	pthread\_rwlock\_destroy(3C) 可用来销毁 rwlock 引用的读写锁对象并释放该锁使用的任何资源。

	```
	#include <pthread.h>
	
	/**
	 * @return  如果成功，返回 0。否则，返回用于指明错误的错误号。EINVAL : attr 或者 rwlock 指定的值无效 
	 */
	int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
	
	pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
	```

再次调用 pthread\_rwlock\_init() 重新初始化该锁之前，使用该锁所产生的影响是不确定的。实现可能会导致 pthread\_rwlock\_destroy() 将 rwlock 所引用的对象设置为无效值。如果在任意线程持有 rwlock 时调用 pthread\_rwlock\_destroy()，则结果是不确定的。尝试销毁未初始化的读写锁会产生不确定的行为。已销毁的读写锁对象可以使用 pthread\_rwlock\_init() 来重新初始化。销毁读写锁对象之后，如果以其他方式引用该对象，则结果是不确定的。


## 五、文章

[百度百科](https://baike.baidu.com/item/%E8%AF%BB%E5%86%99%E9%94%81/1756708)