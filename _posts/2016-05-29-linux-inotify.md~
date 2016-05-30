---
layout: post
section-type: post
title: INOTIFY
category: Category
tags: [ '文件夹监控' ]
---

LINUX 提供了一个监控文件的接口 inotify--通过该接口可以监控文件的移动，读取，写入或删除操作。

初始化：
<pre><code data-trim class="c">
{% raw %}
#include<sys/inotify.h>
int inotify_init(void);
int inotify_init1(int flags);
{% endraw %} 
</code></pre>
flags 为 0 时, inotify_init1 和 inotify_init 的效果是一样的， 
inotify_init 由 Linux 2.6.13 首次添加， glibc 库 2.4 添加。 inotify_init1 由 Linux 2.6.27 首次添加， gligc 库 2.9 添加。

flags 可能是下列标志位的或运算值：
IN_CLOEXEC		对新文件描述符设置执行后关闭 (close-on-exec)。
IN_NONBLOCK		对新文件描述符设置 非阻塞读。

成功时，返回新的文件描述符，出错时，返回 -1 ,并设置相应 errno 值为下列值之一：
EINVAL(inotify_init1()) flags 为无效的参数值。
EMFILE		inotify 达到用户最大的实例数上限。
ENFILE		文件描述符达到系统最大上限。
ENOMEM		剩余内存不足，无法完成请求。
<pre><code data-trim class="c">
{% raw %}
int fd;
fd = inotify_init1(0);
if(fd == -1){
	perror("inotify_init1");
	exit(EXIT_FAULURE);
}
{% endraw %} 
</code></pre>
进程完成初始化之后，可以设置监视(watches)。监视是由监视描述符(watch descroptor) 表示，由一个标准 UNIX 路径及一组相关联的监视掩码(watch mask)组成。监视掩码会通知内核，该进程关心哪些事件(比如 读取，写入，移入，移除)。
inotify 可以监视文件和目录。当监视目录时，inotify 会报告 目录本身 和 该目录 下的所有文件事件(但不包括子目录下的文件--监视不是递归的)。

<pre><code data-trim class="c">
{% raw %}
#include <sys/inotify.h>
int inotify_add_watch(int fd,const char *pathname, uint32_t mask);
{% endraw %} 
</code></pre>
inotify_add_watch 添加一个新的监视，或者修改一个已经存在监视(通过pathname指定)，应用程序必须拥有对 pathname 的读权限， fd 是由 inotify_init1 初始化的返回的描述符，对于指定pathname的检测事件 由mask指定。


下面是在调用read 时可被读取到的事件,也可以用于mask的值
IN_ACCESS(*) 	文件被访问
IN_MODIFY(+)	文件被修改（例如 wirite，truncate）
IN_ATTRIB(+)	元数据被修改（元数据之访问时间，权限等数据）
IN_CLOSE_WRITE(+)		以写模式打开的文件被关闭
IN_CLOSE_NOWRITE(*)		以非写模式打开的文件被关闭
IN_CREATE(+)			在监视的目录新创建了文件或目录
IN_DELETE(+)			文件或目录从监视目录删除
IN_DELETE_SELF			被监视的文件或目录删除
IN_MOVE_SELF		
IN_MOVED_FROM(+)	文件被移动出的目录（在linux下，文件的移动和重命名都是通过rename 来操作，所以可以理解为 被 rename 之前的目录）
IN_MOVED_TO(+)	文件被移动到目录（同上理解为 被 rename 后的目录）
IN_OPEN(*)		文件或文件夹被打开

有（*）标记的事件被监视的文件夹和文件夹下的对象都可以触发
有（+）标记的事件只有被监视的文件夹下的对象可以触发

可用于 mask 的其他值
IN_ALL_EVENTS 	表示监视所有的事件
IN_MOVE			相当于 IN_MOVED_FROM | IN_MOVE_TO
IN_CLOSE		相当于 IN_CLOSE_WRITE IN_CLOSE_NOWRITE
IN_DONT_FOLLOW	如果 pathname 是链接文件（symbolic link）将不引用。
IN_EXCL_UNLINK	忽略unlink事件
IN_MASK_ADD		如果pathname指定的目录已经是添加过的实例，修改已添加的实例mask （add(or) the events in mask to the watch mask）
IN_ONESHOT		监测一个事件，然后删除该实例
IN_ONLYDIR		如果pathname是一个路径，则指定监测

下面是在调用read时可能被设置的值，
IN_IGNORED		监测被移除（文件被删除，或者文件系统 unmounted）
IN_ISDIR		事件生成在目录
IN_Q_OVERFLOW	事件队列溢出
IN_UNMOUNT		被监视的目录所在文件系统卸载（unmounted）

添加完要监测的目录后，可以使用read 从 inotiry 描述符中读取产生的事件
<pre><code data-trim class="c">
{% raw %}
struct inotify_event {
	int      wd;       /* Watch descriptor */
	uint32_t mask;     /* Mask describing event */
	uint32_t cookie;   /* Unique cookie associating related
                                     events (for rename(2)) */
	uint32_t len;      /* Size of name field */
	char     name[];   /* Optional null-terminated name */
};
{% endraw %} 
</code></pre>
wd 		标识产生事件的描述符， 和 inotify_add_watch 返回的描述符对应
mask	产生的事件
len		对应的事件的 name 长度
每个事件长度应该是 结构体 struct inotify_event 的大小加上 len 的总大小
如果初始化使用了 IN_NONBLOCK 标志位，read 将会是非阻塞读取

下面是一个例子

<pre><code data-trim class="c">
{% raw %}
#include <errno.h>
#include <poll.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/inotify.h>
#include <unistd.h>

/* Read all available inotify events from the file descriptor 'fd'.
          wd is the table of watch descriptors for the directories in argv.
          argc is the length of wd and argv.
          argv is the list of watched directories.
          Entry 0 of wd and argv is unused. */

static void
handle_events(int fd, int *wd, int argc, char* argv[])
{
    /* Some systems cannot read integer variables if they are not
              properly aligned. On other systems, incorrect alignment may
              decrease performance. Hence, the buffer used for reading from
              the inotify file descriptor should have the same alignment as
              struct inotify_event. */

    char buf[4096]
            __attribute__ ((aligned(__alignof__(struct inotify_event))));
    const struct inotify_event *event;
    int i;
    ssize_t len;
    char *ptr;

    /* Loop while events can be read from inotify file descriptor. */
    for (;;) {

        /* Read some events. */

        len = read(fd, buf, sizeof buf);
        if (len == -1 && errno != EAGAIN) {
            perror("read");
            exit(EXIT_FAILURE);
        }

        /* If the nonblocking read() found no events to read, then
                  it returns -1 with errno set to EAGAIN. In that case,
                  we exit the loop. */

        if (len <= 0)
            break;

        /* Loop over all events in the buffer */

        for (ptr = buf; ptr < buf + len; ptr += sizeof(struct inotify_event) + event->len) {

            event = (const struct inotify_event *) ptr;

            /* Print event type */

            if (event->mask & IN_OPEN)
                printf("IN_OPEN: ");
            if (event->mask & IN_CLOSE_NOWRITE)
                printf("IN_CLOSE_NOWRITE: ");
            if (event->mask & IN_CLOSE_WRITE)
                printf("IN_CLOSE_WRITE: ");

			printf("event:=========%x\n",event->mask);
            /* Print the name of the watched directory */
            for (i = 1; i < argc; ++i) {
                if (wd[i] == event->wd) {
                    printf("%s/", argv[i]);
                    break;
                }
            }

            /* Print the name of the file */

            if (event->len)
                printf("%s", event->name);

            /* Print type of filesystem object */

            if (event->mask & IN_ISDIR)
                printf(" [directory]\n");
            else
                printf(" [file]\n");
        }
    }
}

int main(int argc, char* argv[])
{
    char buf;
    int fd, i, poll_num;
    int *wd;
    nfds_t nfds;
    struct pollfd fds[2];

    if (argc < 2) {
        printf("Usage: %s PATH [PATH ...]\n", argv[0]);
        exit(EXIT_FAILURE);
    }
    printf("Press ENTER key to terminate.\n");

    /* Create the file descriptor for accessing the inotify API */

    fd = inotify_init1(IN_NONBLOCK);
    if (fd == -1) {
        perror("inotify_init1");
        exit(EXIT_FAILURE);
    }

    /* Allocate memory for watch descriptors */

    wd = calloc(argc, sizeof(int));
    if (wd == NULL) {
        perror("calloc");
        exit(EXIT_FAILURE);
    }

    /* Mark directories for events
              - file was opened
              - file was closed */
    for (i = 1; i < argc; i++) {
        wd[i] = inotify_add_watch(fd, argv[i],
                                  IN_OPEN | IN_CLOSE);
        if (wd[i] == -1) {
            fprintf(stderr, "Cannot watch '%s'\n", argv[i]);
            perror("inotify_add_watch");
            exit(EXIT_FAILURE);
        }
    }

    /* Prepare for polling */

    nfds = 2;

    /* Console input */

    fds[0].fd = STDIN_FILENO;
    fds[0].events = POLLIN;

    /* Inotify input */

    fds[1].fd = fd;
    fds[1].events = POLLIN;

    /* Wait for events and/or terminal input */

    printf("Listening for events.\n");
    while (1) {
        poll_num = poll(fds, nfds, -1);
        if (poll_num == -1) {
            if (errno == EINTR)
                continue;
            perror("poll");
            exit(EXIT_FAILURE);
        }

        if (poll_num > 0) {
            if (fds[0].revents & POLLIN) {

                /* Console input is available. Empty stdin and quit */

                while (read(STDIN_FILENO, &buf, 1) > 0 && buf != '\n')
                    continue;
                break;
            }

            if (fds[1].revents & POLLIN) {

                /* Inotify events are available */

                handle_events(fd, wd, argc, argv);
            }
        }
    }

    printf("Listening for events stopped.\n");

    /* Close inotify file descriptor */

    close(fd);

    free(wd);
    exit(EXIT_SUCCESS);
}
{% endraw %} 
</code></pre>

注：以上内容都来自 man 手册
