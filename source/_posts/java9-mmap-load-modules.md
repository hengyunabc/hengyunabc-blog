---
title: 从java9共享内存加载modules说起
date: 2018-02-23 16:01:41
tags:
 - java
 - java9
 - linux
 - mmap

categories:
 - 技术
---





## jdk9后加载lib/modules的方式

从jdk的代码里可以看出来，默认的实现加载`lib/modules`是用mmap来加载的。

```
class NativeImageBuffer {
    static {
        java.security.AccessController.doPrivileged(
                new java.security.PrivilegedAction<Void>() {
                    public Void run() {
                        System.loadLibrary("jimage");
                        return null;
                    }
                });
    }

    native static ByteBuffer getNativeMap(String imagePath);
}
```

在jimage动态库里最终是一个cpp实现的`ImageFileReader`来读取的。它在64位os上使用的是mmap方式：

https://github.com/dmlloyd/openjdk/blob/jdk/jdk10/src/java.base/share/native/libjimage/imageFile.cpp#L44

通过共享内存启动多个jvm时会有好处：

* 减少内存占用
* 加快启动速度

**突然有个想法，怎么验证多个jvm的确共享了内存？** 

下面来验证一下，思路是：

1. 先获取进程的mmap信息
1. 获取jvm进程映射`modules`的虚拟地址
1. 从虚拟地址转换为物理地址
1. 启动两个jvm进程，计算它们映射`modules`是否物理地址是一样的



### linux下查看进程的mmap信息
1. 使用`pmap -x $pid`命令
2. 直接查看 `cat /proc/$pid/maps`文件的内容

启动一个jshell之后，用pmap查看mmap信息，其中RSS（resident set size）列表示真实占用的内存。：

```
$ pmap -x 24615
24615:   jdk9/jdk-9.0.4/bin/jshell
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000       4       4       0 r-x-- jshell
0000000000601000       4       4       4 rw--- jshell
000000000111b000     132     120     120 rw---   [ anon ]
...
00007f764192c000      88      64       0 r-x-- libnet.so
00007f7641942000    2048       0       0 ----- libnet.so
00007f7641b42000       4       4       4 rw--- libnet.so
00007f7641b43000    2496     588     588 rwx--   [ anon ]
...
00007f7650b43000  185076    9880       0 r--s- modules
00007f765c000000    5172    5124    5124 rw---   [ anon ]

---------------- ------- ------- -------
total kB         2554068  128756  106560
```

我们可以找到`modules`文件的信息：

```
00007f7650b43000  185076    9880       0 r--s- modules
```
它的文件映射大小是185076kb，实际使用内存大小是9880kb。

### linux kernel关于pagemap的说明

上面我们获取到了`modules`的虚拟地址，但是还需要转换为物理地址。

正常来说一个进程是没有办法知道它自己的虚拟地址对应的是什么物理地址。不过我们用linux kernel提供的信息可以读取，转换为物理地址。

linux每个进程都有个`/proc/$pid/pagemap`文件，里面记录了内存页的信息：

https://www.kernel.org/doc/Documentation/vm/pagemap.txt

简而言之，在pagemap里每一个virtual page都有一个对应的64 bit的信息：

```
    * Bits 0-54  page frame number (PFN) if present
    * Bits 0-4   swap type if swapped
    * Bits 5-54  swap offset if swapped
    * Bit  55    pte is soft-dirty (see Documentation/vm/soft-dirty.txt)
    * Bit  56    page exclusively mapped (since 4.2)
    * Bits 57-60 zero
    * Bit  61    page is file-page or shared-anon (since 3.5)
    * Bit  62    page swapped
    * Bit  63    page present
```

只要把虚拟地址转换为pagemap文件里的offset，就可以读取具体的virtual page信息。计算方法是：

```cpp
// getpagesize()是系统调用
// 64bit是8字节
long virtualPageIndex = virtualAddress / getpagesize()
offset = virtualPageIndex * 8
```

从offset里读取出来的64bit里，可以获取到page frame number，如果想要得到真正的物理地址，还需要再转换：

```cpp
// pageFrameNumber * getpagesize() 获取page的开始地址
// virtualAddress % getpagesize() 获取到page里的偏移地址
long pageFrameNumber = // read from pagemap file
physicalAddress = pageFrameNumber * getpagesize() + virtualAddress % getpagesize();
```

### 虚拟地址转换物理地址的代码

参考这里的代码：https://github.com/cirosantilli/linux-kernel-module-cheat/blob/master/kernel_module/user/common.h

得到的一个从虚拟地址转换为物理地址的代码：

```c
#define _POSIX_C_SOURCE 200809L
#include <fcntl.h> /* open */
#include <stdint.h> /* uint64_t  */
#include <stdlib.h> /* size_t */
#include <unistd.h> /* pread, sysconf */

int BUFSIZ = 1024;

typedef struct {
    uint64_t pfn : 54;
    unsigned int soft_dirty : 1;
    unsigned int file_page : 1;
    unsigned int swapped : 1;
    unsigned int present : 1;
} PagemapEntry;

/* Parse the pagemap entry for the given virtual address.
 *
 * @param[out] entry      the parsed entry
 * @param[in]  pagemap_fd file descriptor to an open /proc/pid/pagemap file
 * @param[in]  vaddr      virtual address to get entry for
 * @return 0 for success, 1 for failure
 */
int pagemap_get_entry(PagemapEntry *entry, int pagemap_fd, uintptr_t vaddr)
{
    size_t nread;
    ssize_t ret;
    uint64_t data;

    nread = 0;
    while (nread < sizeof(data)) {
        ret = pread(pagemap_fd, &data, sizeof(data),
                (vaddr / sysconf(_SC_PAGE_SIZE)) * sizeof(data) + nread);
        nread += ret;
        if (ret <= 0) {
            return 1;
        }
    }
    entry->pfn = data & (((uint64_t)1 << 54) - 1);
    entry->soft_dirty = (data >> 54) & 1;
    entry->file_page = (data >> 61) & 1;
    entry->swapped = (data >> 62) & 1;
    entry->present = (data >> 63) & 1;
    return 0;
}

/* Convert the given virtual address to physical using /proc/PID/pagemap.
 *
 * @param[out] paddr physical address
 * @param[in]  pid   process to convert for
 * @param[in] vaddr virtual address to get entry for
 * @return 0 for success, 1 for failure
 */
int virt_to_phys_user(uintptr_t *paddr, pid_t pid, uintptr_t vaddr)
{
    char pagemap_file[BUFSIZ];
    int pagemap_fd;

    snprintf(pagemap_file, sizeof(pagemap_file), "/proc/%ju/pagemap", (uintmax_t)pid);
    pagemap_fd = open(pagemap_file, O_RDONLY);
    if (pagemap_fd < 0) {
        return 1;
    }
    PagemapEntry entry;
    if (pagemap_get_entry(&entry, pagemap_fd, vaddr)) {
        return 1;
    }
    close(pagemap_fd);
    *paddr = (entry.pfn * sysconf(_SC_PAGE_SIZE)) + (vaddr % sysconf(_SC_PAGE_SIZE));
    return 0;
}

int main(int argc, char ** argv){
    char *end;

    int pid;
    uintptr_t virt_addr;
    uintptr_t paddr;
    int return_code;

    pid = strtol(argv[1],&end, 10);
    virt_addr = strtol(argv[2], NULL, 16);

    return_code = virt_to_phys_user(&paddr, pid, virt_addr);

    if(return_code == 0)
        printf("Vaddr: 0x%lx, paddr: 0x%lx \n", virt_addr, paddr);
    else
        printf("error\n");
}
```


另外，收集到一些可以读取pagemap信息的工具：

* https://github.com/dwks/pagemap

### 检查两个jvm进程是否映射`modules`的物理地址一致

1. 先启动两个jshell
    ```
    $ jps
    25105 jdk.internal.jshell.tool.JShellToolProvider
    25142 jdk.internal.jshell.tool.JShellToolProvider
    ```
1. 把上面转换地址的代码保存为`mymap.c`，再编绎
    ```
    gcc mymap.c -o mymap
    ```


1. 获取两个jvm的modules的虚拟地址，并转换为物理地址
    ```
    $ pmap -x 25105 | grep modules
    00007f82b4b43000  185076    9880       0 r--s- modules
    $ sudo ./mymap 25105 00007f82b4b43000
    Vaddr: 0x7f82b4b43000, paddr: 0x33598000
    
    $ pmap -x 25142 | grep modules
    00007ff220504000  185076   10064       0 r--s- modules
    $ sudo ./mymap 25142 00007ff220504000
    Vaddr: 0x7ff220504000, paddr: 0x33598000
    ```

可以看到两个jvm进程映射`modules`的物理地址是一样的，证实了最开始的想法。

### kernel 里的 page-types 工具

其实在kernel里自带有一个工具`page-types`可以输出一个page信息，可以通过下面的方式来获取内核源码，然后自己编绎：

```bash
sudo apt-get source linux-image-$(uname -r)
sudo apt-get build-dep linux-image-$(uname -r)
```

到`tools/vm`目录下面，可以直接`sudo make`编绎。

```
sudo ./page-types -p 25105
             flags	page-count       MB  symbolic-flags			long-symbolic-flags
0x0000000000000000	         2        0  ____________________________________
0x0000000000400000	     14819       57  ______________________t_____________	thp
0x0000000000000800	         1        0  ___________M________________________	mmap
0x0000000000000828	        33        0  ___U_l_____M________________________	uptodate,lru,mmap
0x000000000000086c	       663        2  __RU_lA____M________________________	referenced,uptodate,lru,active,mmap
0x000000000000087c	         2        0  __RUDlA____M________________________	referenced,uptodate,dirty,lru,active,mmap
0x0000000000005868	     10415       40  ___U_lA____Ma_b_____________________	uptodate,lru,active,mmap,anonymous,swapbacked
0x0000000000405868	        29        0  ___U_lA____Ma_b_______t_____________	uptodate,lru,active,mmap,anonymous,swapbacked,thp
0x000000000000586c	         5        0  __RU_lA____Ma_b_____________________	referenced,uptodate,lru,active,mmap,anonymous,swapbacked
0x0000000000005878	       356        1  ___UDlA____Ma_b_____________________	uptodate,dirty,lru,active,mmap,anonymous,swapbacked
             total	     26325      102
```


### jdk8及之前加载jar也是使用mmap的方式

在验证了jdk9加载`lib/modules`之后，随便检查了下jdk8的进程，发现在加载jar包时，也是使用mmap的方式。

一个tomcat进程的map信息如下：

```
$ pmap -x 27226 | grep jar
...
00007f42c00d4000      16      16       0 r--s- tomcat-dbcp.jar
00007f42c09b7000    1892    1892       0 r--s- rt.jar
00007f42c45e5000      76      76       0 r--s- catalina.jar
00007f42c45f8000      12      12       0 r--s- tomcat-i18n-es.jar
00007f42c47da000       4       4       0 r--s- sunec.jar
00007f42c47db000       8       8       0 r--s- websocket-api.jar
00007f42c47dd000       4       4       0 r--s- tomcat-juli.jar
00007f42c47de000       4       4       0 r--s- commons-daemon.jar
00007f42c47df000       4       4       0 r--s- bootstrap.jar
```

可以发现一些有意思的点：

* 所有jar包的`Kbytes` 和 `RSS(resident set size)`是相等的，也就是说整个jar包都被加载到共享内存里了
* 从URLClassLoader的实现代码来看，它在加载资源时，需要扫描所有的jar包，所以会导致整个jar都要被加载到内存里
* 对比jdk9里的`modules`，它的`RSS`并不是很高，原因是JImage的格式设计合理。所以jdk9后，jvm占用真实内存会降低。

### jdk8及之前的 sun.zip.disableMemoryMapping 参数

* 在jdk6里引入一个 `sun.zip.disableMemoryMapping`参数，禁止掉利用mmap来加载zip包。http://www.oracle.com/technetwork/java/javase/documentation/overview-156328.html#6u21-rev-b09

* https://bugs.openjdk.java.net/browse/JDK-8175192 在jdk9里把这个参数去掉了。因为jdk9之后，jdk本身存在`lib/modules` 这个文件里了。


### 总结

* linux下可以用pmap来获取进程mmap信息
* 通过读取`/proc/$pid/pagemap`可以获取到内存页的信息，并可以把虚拟地址转换为物理地址
* jdk9把类都打包到`lib/modules`，也就是JImage格式，可以减少真实内存占用
* jdk9多个jvm可以共用`lib/modules`映射的内存
* 默认情况下jdk8及以前是用mmap来加载jar包