```C
#include<sys/time.h>  
#include<sys/types.h>  
#include<unistd.h>  
#include<string.h>  
#include<stdlib.h>  
#include<stdio.h>  
int main()  
{  
    char buf[10]="";  
    fd_set rdfds;  
    struct timeval tv;  
    int ret;  
    FD_ZERO(&rdfds);  
    FD_SET(0,&rdfds);   //文件描述符0表示stdin键盘输入  
    tv.tv_sec = 3;  
    tv.tv_usec = 500;  
    ret = select(1,&rdfds,NULL,NULL,&tv);      //第一个参数是监控句柄号+1  
    if(ret<0)  
        printf("selcet error\r\n");  
    else if(ret == 0)  
        printf("timeout \r\n");  
    else  
        printf("ret = %d \r\n",ret);  
    if(FD_ISSET(0,&rdfds)){         //监控输入的确是已经发生了改变  
        printf(" reading");  
        read(0,buf,9);                 //从键盘读取输入  
    }  
    write(1,buf,strlen(buf));          //在终端中回显  
    printf(" %d \r\n",strlen(buf));  
    return 0;  
}
```


socket server是
socket
bind
listen
accept
while recv 和send

socket client是
socket
connnect
send和recv

select
http://www.cnblogs.com/gentleming/archive/2010/11/15/1877976.html

client.c
```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
int main(int argc, char **argv)
{
    int sockfd;
    struct sockaddr_in servaddr;
    sockfd = socket(PF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(50001);
    servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
    connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    char sendline[100];
    sprintf(sendline, "Hello, world!");
    write(sockfd, sendline, strlen(sendline));
    close(sockfd);
    return 1;
}
```
[root@mcompute616 hao]# cat backup/server.c 
```c

#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
int main(int argc, char **argv)
{
    int listenfd;
    int connfd;
    struct sockaddr_in servaddr;
    listenfd = socket(PF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(50001);
    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    listen(listenfd, 10);
    while(1){
        connfd = accept(listenfd, (struct sockaddr *)NULL, NULL);
        int n;
        char recvline[1024];
        while((n=read(connfd, recvline, 1024)) > 0)
        {
            recvline[n] = 0;
            printf("%s\n", recvline);
        }
        close(connfd);
    }
    close(listenfd);
    return 1;
}
```
server.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#define BACKLOG 5 
#define BUF_SIZE 200
int fd_A[BACKLOG];
int conn_amount;
int main(int argc, char **argv)
{
    int listenfd, sockMax ,new_fd;
    int connfd;
    int ret;
    int i;
    char buf[BUF_SIZE];
    struct sockaddr_in servaddr;
    struct sockaddr_in client_addr; // connector's address information
    socklen_t sin_size;
    sin_size = sizeof(client_addr);
    
    fd_set fdsr;
    struct timeval tv;
    conn_amount = 0;

    listenfd = socket(PF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(50001);
    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    listen(listenfd, BACKLOG);

    while(1){
        FD_ZERO(&fdsr);
        FD_SET(listenfd, &fdsr);
        tv.tv_sec = 30; 
        tv.tv_usec = 0;
        sockMax = listenfd;
        for (i = 0; i < BACKLOG; i++) {
            printf("BACKLOG [%d]\n",i);
            if (fd_A[i] != 0) {
                printf("BACKLOG---- [%d]\n",i);
                FD_SET(fd_A[i], &fdsr);
            }   
        }
        ret = select(sockMax + 1, &fdsr, NULL, NULL, &tv);
        if (ret < 0) {
            perror("select");
            break;
        } else if (ret == 0) {
            printf("timeout\n");
            continue;
        }   
         // check every fd in the set
        for (i = 0; i < conn_amount; i++) {
            printf("conn_amount [%d] \n",i);
            if (FD_ISSET(fd_A[i], &fdsr)) {
                printf("FD_ISSET fd_A[%d]\n",i);
                ret = recv(fd_A[i], buf, sizeof(buf), 0); 
                if (ret <= 0) {        // client close
                    printf("client[%d] close\n", i); 
                    close(fd_A[i]);
                    FD_CLR(fd_A[i], &fdsr);
                    fd_A[i] = 0;
                } else {        // receive data
                    if (ret < BUF_SIZE)
                        memset(&buf[ret], '\0', 1); 
                    printf("client[%d] send:%s\n", i, buf);
                }
            }
        }
        // check whether a new connection comes
        if (FD_ISSET(listenfd, &fdsr)) {
            new_fd = accept(listenfd, (struct sockaddr *)&client_addr, &sin_size);
            if (new_fd <= 0) {
                perror("accept");
                continue;
            }
            // add to fd queue
            if (conn_amount < BACKLOG) 
            {
                fd_A[conn_amount++] = new_fd;
                printf("new connection client[%d] %s:%d\n", conn_amount,inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                if (new_fd > sockMax)
                    sockMax = new_fd;
            }
            else 
            {
                printf("max connections arrive, exit\n");
                send(new_fd, "bye", 4, 0);
                close(new_fd);
                break;
            }
        }

    }
    for (i = 0; i < BACKLOG; i++) {
        if (fd_A[i] != 0) {
            close(fd_A[i]);
        }
    }
    return 1;
}
```

############

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#define MYPORT 59000   // the port users will be connecting to
#define BACKLOG 5     // how many pending connections queue will hold
#define BUF_SIZE 200
int fd_A[BACKLOG];    // accepted connection fd
int conn_amount;    // current connection amount
void showclient()
{
    int i;
    printf("client amount: %d\n", conn_amount);
    for (i = 0; i < BACKLOG; i++) {
        printf("[%d]:%d  ", i, fd_A[i]);
    }
    printf("\n\n");
}
int main(void)
{
    int sock_fd, new_fd;  // listen on sock_fd, new connection on new_fd
    struct sockaddr_in server_addr;    // server address information
    struct sockaddr_in client_addr; // connector's address information
    socklen_t sin_size;
    int yes = 1;
    char buf[BUF_SIZE];
    int ret;
    int i;
    if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("socket");
        exit(1);
    }
    if (setsockopt(sock_fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int)) == -1) {
        perror("setsockopt");
        exit(1);
    }
    server_addr.sin_family = AF_INET;         // host byte order
    server_addr.sin_port = htons(MYPORT);     // short, network byte order
    server_addr.sin_addr.s_addr = INADDR_ANY; // automatically fill with my IP
    memset(server_addr.sin_zero, '\0', sizeof(server_addr.sin_zero));
    if (bind(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        exit(1);
    }
    if (listen(sock_fd, BACKLOG) == -1) {
        perror("listen");
        exit(1);
    }
    printf("listen port %d\n", MYPORT);
    fd_set fdsr;
    int maxsock;
    struct timeval tv;
    conn_amount = 0;
    sin_size = sizeof(client_addr);
    maxsock = sock_fd;
    while (1) {
        // initialize file descriptor set
        FD_ZERO(&fdsr);
        FD_SET(sock_fd, &fdsr);
        // timeout setting
        tv.tv_sec = 30;
        tv.tv_usec = 0;
        // add active connection to fd set
        for (i = 0; i < BACKLOG; i++) {
            if (fd_A[i] != 0) {
                FD_SET(fd_A[i], &fdsr);
            }
        }
        ret = select(maxsock + 1, &fdsr, NULL, NULL, &tv);
        if (ret < 0) {
            perror("select");
            break;
        } else if (ret == 0) {
            printf("timeout\n");
            continue;
        }
        // check every fd in the set
        for (i = 0; i < conn_amount; i++) {
            if (FD_ISSET(fd_A[i], &fdsr)) {
                ret = recv(fd_A[i], buf, sizeof(buf), 0);
                if (ret <= 0) {        // client close
                    printf("client[%d] close\n", i);
                    close(fd_A[i]);
                    FD_CLR(fd_A[i], &fdsr);
                    fd_A[i] = 0;
                } else {        // receive data
                    if (ret < BUF_SIZE)
                        memset(&buf[ret], '\0', 1);
                    printf("client[%d] send:%s\n", i, buf);
                }
            }
        }
        // check whether a new connection comes
        if (FD_ISSET(sock_fd, &fdsr)) {
            new_fd = accept(sock_fd, (struct sockaddr *)&client_addr, &sin_size);
            if (new_fd <= 0) {
                perror("accept");
                continue;
            }
            // add to fd queue
            if (conn_amount < BACKLOG) {
                fd_A[conn_amount++] = new_fd;
                printf("new connection client[%d] %s:%d\n", conn_amount,
                        inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                if (new_fd > maxsock)
                    maxsock = new_fd;
            }
            else {
                printf("max connections arrive, exit\n");
                send(new_fd, "bye", 4, 0);
                close(new_fd);
                break;
            }
        }
        showclient();
    }
    // close other connections
    for (i = 0; i < BACKLOG; i++) {
        if (fd_A[i] != 0) {
            close(fd_A[i]);
        }
    }
    exit(0);
}
```



