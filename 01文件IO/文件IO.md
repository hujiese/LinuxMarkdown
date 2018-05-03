## Linux编程之文件操作 ##
### C库函数与系统工作流程 ###
![](https://i.imgur.com/kVH2m36.png)
### C库函数与系统函数的关系 ###
![](https://i.imgur.com/nFVMhC2.png)
### 虚拟地址空间 ###
![](https://i.imgur.com/vgW7Zqm.png)
### pcb和文件描述符 ###
![](https://i.imgur.com/rCXyrAH.png)
关于函数库的使用，可以查看文档，现在输入man man:
![](https://i.imgur.com/6902pMa.png)

可以看到系统调用在第2章节，所以输入：man 2 open
![](https://i.imgur.com/NoT9tVO.png)

通过这个可以查看open系统调用函数的内容。

### 文件IO ###
![](https://i.imgur.com/j3moBug.png)

#### 普通文件读写相关Demo: ####

	#include <stdio.h>
	#include <unistd.h>
	#include <sys/stat.h>
	#include <sys/types.h>
	#include <fcntl.h>
	#include <stdlib.h>
	
	int main(int argc, const char* argv[])
	{
	    int fd = open("english.txt", O_RDONLY);
	    if(fd == -1)
	    {
	        perror("open error");
	        exit(1);
	    }
	    // 读文件
	    char buf[1024];
	    int len = read(fd, buf, sizeof(buf));
	
	    // 写文件
	    int fd1 = open("temp", O_WRONLY | O_CREAT, 0664);
	    if(fd1 == -1)
	    {
	        perror("open error");
	        exit(1);
	    }
	
	    while( len > 0 )
	    {
	        // 写文件
	        int ret = write(fd1, buf, len);
	        len = read(fd, buf, sizeof(buf));
	    }
	    close(fd);
	    close(fd1);
	
	    return 0;
	}

程序从english.txt中读内容，并将内容写入到temp中，编译运行temp结果如下：

![](https://i.imgur.com/vQm1N7n.png)

#### 阻塞读案例--block_read.c ：####

	// 阻塞读终端
	#include <unistd.h>
	#include <stdlib.h>
	#include <stdio.h>
	
	// 阻塞读终端
	int main(void)
	{
		char buf[10];//字符缓存区，最大为10字节
		int n;//保存读出的值
		n = read(STDIN_FILENO, buf, 10);//从标准输入流中读最多十个字节字符串
		if (n < 0) //读取错误，退出
		{
			perror("read STDIN_FILENO");
			exit(1);
		}
		write(STDOUT_FILENO, buf, n);//将读入的字符串写到标准输出流中
		return 0;
	}

编译运行结果如下：

![](https://i.imgur.com/iLP3XEh.png)

可见，如果读入超过十个字节，那么就会出错！

#### 非阻塞读案例--unblock_read.c ： ####

	#include <unistd.h>
	#include <fcntl.h>
	#include <errno.h>
	#include <string.h>
	#include <stdlib.h>
	#include <stdio.h>
	#define MSG_TRY "try again\n"
	
	// 非阻塞读终端
	int main(void)
	{
		char buf[10];
		int fd, n;
	    // /dev/tty --> 当前打开的终端设备
		fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
		if(fd < 0) 
		{
			perror("open /dev/tty");
			exit(1);
		}
		tryagain:
		n = read(fd, buf, 10);
		if (n < 0)
		{
	        // 如果write为非阻塞，但是没有数据可读，此时全局变量 errno 被设置为 EAGAIN
			if (errno == EAGAIN) 
			{
				sleep(3);
				write(STDOUT_FILENO, MSG_TRY, strlen(MSG_TRY));
				goto tryagain;
			}
			perror("read /dev/tty");
			exit(1);
		}
		write(STDOUT_FILENO, buf, n);
		close(fd);
		return 0;
	}

编译运行结果如下：

![](https://i.imgur.com/fkDqPUQ.png)

在打开/dev/tty终端使用了非阻塞方式，当调用read函数时，如果时间超时，那么将循环tryagain这部分代码，当有输入并回车换行确认后将输出输入内容并关闭文件，退出程序。

#### 超时非阻塞读--timeout_unblock_read.c : ####

	#include <unistd.h>
	#include <fcntl.h>
	#include <errno.h>
	#include <string.h>
	#include <stdlib.h>
	#include <stdio.h>
	
	#define MSG_TRY "try again\n"
	#define MSG_TIMEOUT "timeout\n"
	
	// 非阻塞读终端和等待超时
	int main(void)
	{
		char buf[10];
		int fd, n, i;
	    // /dev/tty --> 当前终端设备
		fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
		if(fd<0) 
		{
			perror("open /dev/tty");
			exit(1);
		}
		for(i=0; i<5; i++) 
		{
			n = read(fd, buf, 10);
			if(n >= 0)
			break;
	        // 如果write为非阻塞，但是没有数据可读，此时全局变量 errno 被设置为 EAGAIN
			if(errno != EAGAIN) 
			{
				perror("read /dev/tty");
				exit(1);
			}
			sleep(3);
			write(STDOUT_FILENO, MSG_TRY, strlen(MSG_TRY));
		}
		if(i==5)
		{
			write(STDOUT_FILENO, MSG_TIMEOUT, strlen(MSG_TIMEOUT));
		}
		else
		{
			write(STDOUT_FILENO, buf, n);
		}
		close(fd);
		return 0;
	}

编译运行如下：

![](https://i.imgur.com/tlCcLrQ.png)

从上面可以看到，首先打开终端方式为非阻塞，然后进入了一个5次的for循环，每次等待用户3s钟，三秒后从新一次for循环，如此5次，如果五次都没有获取用户输入，那么结束循环，打印timeout超时信息，如果在此期间有用户输入，那么将打印输入信息并关闭文件退出程序！

### 获取文件属性 ###
![](https://i.imgur.com/NEpgZCq.png)
![](https://i.imgur.com/QISHznl.png)
![](https://i.imgur.com/W8xWllA.png)

#### 文件的权限： ####

![](https://i.imgur.com/JI1XupL.png)

可以查看下文件的信息：

![](https://i.imgur.com/jvSZRBr.png)

下面这个例子lstat查看文件的相关信息:

	#include <sys/types.h>
	#include <sys/stat.h>
	#include <unistd.h>
	#include <stdio.h>
	#include <stdlib.h>
	
	
	int main(int argc, char* argv[])
	{
	    if(argc < 2)
	    {
	        printf("a.out filename\n");
	        exit(1);
	    }
	    
	    struct stat buf_st;
	    int ret = lstat(argv[1], &buf_st);
	    if(ret == -1)
	    {
	        perror("stat");
	        exit(1);
	    }
	
	    printf("file size = %d\n", (int)buf_st.st_size);
	
	    return 0;
	}

编译运行结果如下：

![](https://i.imgur.com/HDlwxit.png)

程序查看文件的大小，结果与实际的一致。

### 文件属性函数 ###

![](https://i.imgur.com/l878awN.png)

实例如下：

	#include <stdio.h>
	#include <stdlib.h>
	#include <unistd.h>
	
	int main(int argc, char* argv[])
	{
	    if(argc < 2)
	    {
	        printf("a.out filename\n");
	        exit(1);
	    }
	
	    int ret = access(argv[1], W_OK);
	    if(ret == -1)
	    {
	        perror("access");
	        exit(1);
	    }
	    printf("you can write this file.\n");
	    return 0;
	}


编译运行结果如下：

![](https://i.imgur.com/FZe8zxi.png)

![](https://i.imgur.com/TwWlh8A.png)

![](https://i.imgur.com/xRy7ep6.png)

实例如下：

	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/stat.h>
	#include <unistd.h>
	
	
	int main(int argc, char* argv[])
	{
	    if(argc < 3)
	    {
	        printf("a.out filename mode\n");
	        exit(1);
	    }
	    int mode = strtol(argv[2], NULL, 8);
	    int ret = chmod(argv[1], mode);
	    if(ret == -1)
	    {
	        perror("chmod");
	        exit(1);
	    }
	
	    ret = chown(argv[1], 1001, 1002);
	    if(ret == -1)
	    {
	        perror("chown");
	        exit(1);
	    }
	    return 0;
	}


![](https://i.imgur.com/8bMWibG.png)

实例如下:

	#include <stdio.h>
	#include <stdlib.h>
	#include <unistd.h>
	#include <sys/types.h>
	
	int main(int argc, char* argv[])
	{
	    if(argc < 3)
	    {
	        printf("a.out filename size\n");
	        exit(1);
	    }
	
	    long int len = strtol(argv[2], NULL, 10); 
	    int  aa = truncate(argv[1], len);
	    if(aa == -1)
	    {
	        perror("truncate");
	        exit(1);
	    }
	    return 0;
	}


### 目录操作相关函数 ###
![](https://i.imgur.com/Jrs9cOP.png)

![](https://i.imgur.com/gqOm3G4.png)


	#include <stdio.h>
	#include <fcntl.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	#include <stdlib.h>
	#include <unistd.h>
	
	int main(int argc, char* argv[])
	{
	    if(argc < 2)
	    {
	        printf("a.out dir\n");
	        exit(1);
	    }
	
	    int ret = chdir(argv[1]);
	    if(ret == -1)
	    {
	        perror("chdir");
	        exit(1);
	    }
	
	    int fd = open("chdir.txt", O_CREAT | O_RDWR, 0777);
	    if(fd == -1)
	    {
	        perror("open");
	        exit(1);
	    }
	    close(fd);
	
	    char buf[128];
	    getcwd(buf, sizeof(buf));
	    printf("current dir: %s\n", buf);
	
	    return 0;
	}


编译运行结果如下:

![](https://i.imgur.com/Us896qb.png)

程序切换到上级目录，修改了"chdir.txt"文件的权限

![](https://i.imgur.com/33oiJsk.png)

![](https://i.imgur.com/T02sFcH.png)

	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	
	int main(int argc, char* argv[])
	{
	    if(argc < 3)
	    {
	        printf("a.out newDir mode\n");
	        exit(1);
	    }
	
	    int mode = strtol(argv[2], NULL, 8);
	    int ret = mkdir(argv[1], mode);
	    if(ret == -1)
	    {
	        perror("mkdir");
	        exit(1);
	    }
	    return 0;
	}

编译运行结果如下:

![](https://i.imgur.com/uGDxtGo.png)


![](https://i.imgur.com/pYDyMYV.png)

### 目录遍历相关函数 ###

![](https://i.imgur.com/I3oOgEL.png)


	#include <stdio.h>
	#include <string.h>
	#include <stdlib.h>
	#include <dirent.h>
	#include <sys/types.h>
	
	
	int get_file_num(char* root)
	{
		int total = 0;
		DIR* dir = NULL;
		// 打开目录
		dir = opendir(root);
		// 循环从目录中读文件
	
		char path[1024];
		// 定义记录xiang指针
		struct dirent* ptr = NULL;
		while( (ptr = readdir(dir)) != NULL)
		{
			// 跳过. he ..
			if(strcmp(ptr->d_name, ".") == 0 || strcmp(ptr->d_name, "..") == 0)
			{
				continue;
			}
			// 判断是不是目录
			if(ptr->d_type == DT_DIR)
			{
				sprintf(path, "%s/%s", root, ptr->d_name);
				// 递归读目录
				total += get_file_num(path);
			}
			// 如果是普通文件
			if(ptr->d_type == DT_REG)
			{
				total++;
			}
		}
		closedir(dir);
		return total;
	}
	
	int main(int argc, char* argv[])
	{
		if(argc < 2)
		{
			printf("./a.out path");
			exit(1);
		}
	
		int total = get_file_num(argv[1]);
		printf("%s has regfile number: %d\n", argv[1], total);
		return 0;
	}


编译运行结果如下:

![](https://i.imgur.com/dm4hmFN.png)

程序读取当前目录下的文件，跳过'.'和'..'，递归读取并统计当前路径下文件和文件夹的数量并打印.

### dup-dup2-fcntl ：###

![](https://i.imgur.com/Fq1VIgK.png)

案例一 -- dup函数使用:

	#include <string.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	#include <unistd.h>
	#include <fcntl.h>
	
	int main(void)
	{
		int fd = open("temp", O_RDWR | O_CREAT, 0664);
		if(fd == -1)
		{
			perror("open");
			exit(1);
		}
	
		// 复制文件描述符
		int fd2 = dup(fd);
		// 写文件
		char* p = "让编程改变世界。。。。。。";
		write(fd2, p, strlen(p));
		close(fd2);
	
		char buf[1024];
		lseek(fd, 0, SEEK_SET);
		read(fd, buf, sizeof(buf));
		printf(" buf = %s\n", buf);
		close(fd);
	
		return 0;
	}

编译运行结果如下:

![](https://i.imgur.com/IfPfWT9.png)

其中，程序执行后temp文件内容如下:"让编程改变世界。。。。。。"，程序首先打开"temp"文件，获取文件描述符fd然后通过调用dup函数将fd拷贝到fd2中，通过fd2将字符串p中内容写到文件temp中，然后关闭文件描述符fd2，然后通过fd移动光标至文件头，将读取内容拷贝到buf中并打印。由此可见，两个描述符指向同一个文件，可以操作同一个文件。

案例二 -- dup2函数使用：

	#include <string.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	#include <unistd.h>
	#include <fcntl.h>
	
	int main(void)
	{
		int fd = open("temp", O_RDWR | O_CREAT | O_APPEND, 0664);
		if(fd == -1)
		{
			perror("open");
			exit(1);
		}
	
		int fd2 = open("temp1", O_RDWR | O_CREAT | O_APPEND, 0664);
		if(fd2 == -1)
		{
			perror("open open");
			exit(1);
		}
		// 复制文件描述符
		dup2(fd, fd2);
		// 写文件
		char* p = "change the world by programing。。。。。。\n";
		write(fd2, p, strlen(p));
		close(fd2);
	
		char buf[1024];
		lseek(fd, 0, SEEK_SET);
		read(fd, buf, sizeof(buf));
		printf(" buf = %s\n", buf);
		close(fd);
	
		return 0;
	}

编译运行结果如下:

![](https://i.imgur.com/698bcwM.png)

程序一开始打开了两个文件，temp和temp1，分别对应文件描述符为fd和fd2，通过dup2函数将fd2断开与文件temp1的联系，然后让fd2拷贝fd内容并指向temp文件，通过fd2讲p内容写入temp文件中，然后关闭fd2，通过fd读取temp文件内容并打印出来。

![](https://i.imgur.com/cw37fXj.png)