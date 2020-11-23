### LIT1 User Space安装

### 编译部分

进入`ulocks/src/litl/`目录。

使用`make -C`命令编译，报错：找不到-lpapi：

```shell
/usr/bin/ld: cannot find -lpapi
collect2: error: ld returned 1 exit status
Makefile:22: recipe for target '../lib/libmcs_spinlock.so' failed
make[1]: *** [../lib/libmcs_spinlock.so] Error 1
make[1]: Leaving directory '/home/yang/shflok/shfllock-master/ulocks/src/litl/src'
Makefile:25: recipe for target 'libmcs_spinlock.so' failed
make: *** [libmcs_spinlock.so] Error 2
make: Leaving directory '/home/yang/shflok/shfllock-master/ulocks/src/litl'

```

![image-20201121111648050](./image-20201121111648050.png)

查阅资料得知，papi库是一个用于测试性能的工具，我们需要先安装papi工具，使用sudo apt安装命令如下：

```shell
sudo apt-get update
sudo apt-get install libpapi-dev
```

安装成功后再次cmake，可以看到上一次的错误果然不见了，换了另外的**error**:

![image-20201121121418636](./image-20201121121418636.png)

```shell
../include/htlockepfl.h:72:25: error: flexible array member not at end of struct
     ticket_lock_local_t local[NUMA_NODES];
                         ^
Makefile:12: recipe for target '../obj/htlockepfl_original/interpose.o' failed
make[1]: *** [../obj/htlockepfl_original/interpose.o] Error 1
make[1]: Leaving directory '/home/yang/shflok/shfllock-master/ulocks/src/litl/src'
Makefile:25: recipe for target 'libhtlockepfl_original.so' failed
make: *** [libhtlockepfl_original.so] Error 2
```

这个柔性数组不再结构体的最后面？一个直接的想法就是进去把这个数组移到最后面不就行了？说干就干，在当前目录下的/include找到那个文件，把数组改到最后一行：

![image-20201121122530860](./image-20201121122530860.png)

重新make一下，发现原来的错误解决了，又来了新的错误：

![image-20201121122609409](./image-20201121122609409.png)

出现`NUMA_NODES`的地方都出错了，说明这个变量可能没有定义。emmmm，那就我帮他定义一下吧，我又进去代码里面，帮作者定义了一个`NUMA_NODES`

![image-20201121124038188](./image-20201121124038188.png)

这下再来make一次，果然又报错了，报错的地方又是`NUMA_NODES`,这个变量redefined了。好家伙，这个变量到底定义了没有呀，前面又说找不到，现在又说重复定义。

![image-20201121124153172](./image-20201121124153172.png)

仔细看报错信息，发现在/include/topologh.y的第27行已经定义过了，于是，我就进去看看，到底是怎么回事。

![image-20201121124317698](./image-20201121124317698.png)

一看，原来如此，NUMA_NODES被定义为空串，这怎么能行呢，怪不得一直报错，我这里就自作主张，给他定义为2。当然，之前在htlockepfl.h中自己添加的NUMA_NODES就要删掉了。

![image-20201121124523496](./image-20201121124523496.png)

![image-20201121124415134](./image-20201121124415134.png)

再来make一次，嘻嘻，终于成功啦。

![image-20201121124559086](./image-20201121124559086.png)

这些新生成的sh文件，应该就是编译的结果吧。

![image-20201121124703487](./image-20201121124703487.png)

### 测试部分

根据readme中的信息，我们可以用`./xxxxxxx.sh myprogram`的方式运行生成的锁，

![image-20201121125002738](./image-20201121125002738.png)

解下来，就可以测试锁啦，我从网上随便复制了一段锁测试代码。

```c++
#include <stdio.h>

#include <pthread.h>

pthread_mutex_t thread_mutex;

int global_val = 0;

void *thread1(void *arg){

while(1){

pthread_mutex_lock(&thread_mutex);

global_val = global_val + 1;

printf("thread1 global_val=%d\n", global_val);

global_val = global_val + 1;

usleep(100);

                printf("thread1 global_val=%d\n", global_val);

usleep(100);

pthread_mutex_unlock(&thread_mutex);

}

return NULL;

}

void *thread2(void *arg){

while(1){

pthread_mutex_lock(&thread_mutex);

        global_val = global_val + 1;

        printf("thread2 global_val=%d\n", global_val);

                usleep(100);

global_val = global_val + 1;

                printf("thread2 global_val=%d\n", global_val);

        usleep(100);

pthread_mutex_unlock(&thread_mutex);

}

return NULL;

}

int main(void){

pthread_t thread_id1 = 0, thread_id2 = 0;

pthread_mutex_init(&thread_mutex, NULL);

pthread_create(&thread_id1, NULL, thread1, NULL);

pthread_create(&thread_id2, NULL, thread2, NULL);

pthread_join(thread_id1, NULL);

pthread_join(thread_id2, NULL);

return 0;

}
```

在命令行中编译成可执行文件，然后再使用`./libxxxxx.sh myprogram`，我把上述代码保存在myprogram0.c中，命令如下:

```shell
gcc myprogram0.c -o myprogram0 -lpthread
./libcna_spinlock.sh ./myprogram0
```

然后就能看到满屏的输出了，代码测试成功！

![image-20201121130905476](./image-20201121130905476.png)

稍微修改一下代码，让他们不要无限输出。

```shell
#include <stdio.h>

#include <pthread.h>

pthread_mutex_t thread_mutex;

int global_val = 0;

void *thread1(void *arg){

while(global_val < 1000000){

pthread_mutex_lock(&thread_mutex);

global_val = global_val + 1;

printf("thread1 global_val=%d\n", global_val);

global_val = global_val + 1;



                printf("thread1 global_val=%d\n", global_val);



pthread_mutex_unlock(&thread_mutex);

}

return NULL;

}

void *thread2(void *arg){

while(global_val < 1000000){

pthread_mutex_lock(&thread_mutex);

        global_val = global_val + 1;

        printf("thread2 global_val=%d\n", global_val);


global_val = global_val + 1;

                printf("thread2 global_val=%d\n", global_val);


pthread_mutex_unlock(&thread_mutex);

}

return NULL;

}

int main(void){

pthread_t thread_id1 = 0, thread_id2 = 0;

pthread_mutex_init(&thread_mutex, NULL);

pthread_create(&thread_id1, NULL, thread1, NULL);

pthread_create(&thread_id2, NULL, thread2, NULL);

pthread_join(thread_id1, NULL);

pthread_join(thread_id2, NULL);

return 0;

}
```

重新编译一下

```shell
gcc myprogram0.c -o myprogram0 -lpthread
```

顺便记录一下运行时间,运行！

```shell
time ./libaqmwonode_spin_then_park.sh ./myprogram0
time ./libalockepfl_original.sh ./myprogram0
```



跑步比赛，现在开始!

![image-20201121132934327](./image-20201121132934327.png)

比赛结束！

![image-20201121133008876](./image-20201121133008876.png)

左边锁比右边锁快了2s多。再测试一下？

![image-20201121133236860](./image-20201121133236860.png)

同样也相差了2s多，在这个测试中，左边锁完胜!

### 使用论文中的程序测试锁

#### jemaloc

根据文档的信息，可以通过./autogen.sh调用，因此，我们直接尝试使用`./autogen.sh`

![image-20201123112619714](./image-20201123112619714.png)

发现报错了，找不到autoconf这个包

![image-20201123112715504](./image-20201123112715504.png)

autoconf是一个包，我们可以使用aptget安装

```shell
sudo apt-get install autoconf
```

![image-20201123112835093](./image-20201123112835093.png)

完成之后重复一下,就没有报错了，正常安装！

```shell
./autogen.sh
```

![image-20201123112915107](./image-20201123112915107.png)

成功之后会输出一些安装信息

![image-20201123112948023](./image-20201123112948023.png)

继续按照文档给出的安装方式.

```shell
make dist
make 
sudo make install
```



安装完成之后，应该就可以调用run_tests.sh了

```shell
./run_tests.sh
```

发现报错了：

![image-20201123111405490](./image-20201123111405490.png)

这里报错的是一个python脚本，而且是print方法使用错误，因为我的电脑默认环境是py3.7环境，print加冒号是py2.7的语法。这时候，一个合理的猜测就是应该要修改python环境。

这里我用anaconda管理环境，anaconda的安装可以看另外一篇文档。

下面这个命令创建了一个python2.7环境，并且命名为py27

```shell
conda create -n py27 python=2.7
```

![image-20201123111541982](./image-20201123111541982.png)

切换到py27环境，然后再次调用刚刚那个脚本run_tests.sh。

![image-20201123113122755](./image-20201123113122755.png)

还是报错了

![image-20201123124224280](./image-20201123124224280.png)

在run_test.out中，发现是一个expected变量声明了，没有使用，加上`cc1plus: warnings being treated as errors`,把warning当作error。因此报错了。

对此，我查找资料，发现和-Werror这个编译选项有关系，所以，我直接更改gcc的编译选项。

首先看一下run_tests.sh做了什么：

![image-20201123124539327](./image-20201123124539327.png)

原来它间接调用了另外的脚本，打开scripts下的gen_run_tests.py文件，找到config_line这一行。

![image-20201123124637684](./image-20201123124637684.png)

将原来的编译选项注释掉，改成空白字符。

```shell
                config_line = (
                    # 'EXTRA_CFLAGS=-Werror EXTRA_CXXFLAGS=-Werror '
			''
                    + 'CC="{} {}" '.format(cc, " ".join(compiler_opts))
                    + 'CXX="{} {}" '.format(cxx, " ".join(compiler_opts))
                    + '../../configure '
                    + " ".join(config_opts) + (' --with-malloc-conf=' +
                    ",".join(malloc_conf_opts) if len(malloc_conf_opts) > 0
                    else '')
```

再次运行，不报错啦。

![image-20201123124751573](./image-20201123124751573.png)

先慢慢等结果吧。