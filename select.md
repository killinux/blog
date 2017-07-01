client.c
```c
#include <stdio.h>  
#include <string.h>  
#include <netinet/in.h>  
#include <sys/types.h>  
#include <sys/socket.h>  
  
int main()  
{  
    int socketfd;  
    struct sockaddr_in servaddr;  
  
    if((socketfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)  
    {  
        printf("socket error\n");  
        return -1;  
    }  
  
    bzero(&servaddr, sizeof(servaddr));  
  
    servaddr.sin_addr.s_addr = inet_addr("192.168.139.218");  
    servaddr.sin_family = AF_INET;  
    servaddr.sin_port = htons(59000);  
  
    if(connect(socketfd, (struct sockaddr*) &servaddr, sizeof(servaddr)) < 0)  
    {  
        printf("connect error\n");  
    }  
    char * buffer;
    buffer = "haoning\n";
    if (send(socketfd, buffer, 8, 0) != 8) {  
        printf("send error\n");  
          
    }   
    close(socketfd);
    return 0;  
}  

```

server.c
```c
#include <stdio.h>  
#include <string.h>  
#include <netinet/in.h>  
#include <sys/types.h>  
#include <sys/socket.h>  
  
int main()  
{  
    int count = 0;  
    int listenfd, socketfd;  
    int nread;  
    struct sockaddr_in servaddr;  
    struct timeval timeoutval;  
    char readbuf[256];  
  
    printf("accept started\n");  
  
    //socket      
    if((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)  
    {  
        printf("socket error\n");  
        return -1;  
    }  
  
    bzero(&servaddr, sizeof(servaddr));  
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);  
    servaddr.sin_family = AF_INET;  
    servaddr.sin_port = htons(59000);  
  
    //bind  
    if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)  
    {  
        printf("bind error\n");  
        //return -1;  
    }  
  
    //listen  
    listen(listenfd, 5);  
  
    //accept  
  
  
  
    while(1)  
    {  
        socketfd = accept(listenfd, NULL, NULL);  

        printf("start receive %d...\n", count++);  
        memset(readbuf, sizeof(readbuf), 0);  
  
        nread = recv(socketfd, readbuf, 10, 0);  
        printf("nread %d\n",nread);
        if(nread>0)  
        {  
            readbuf[10] = '\0';  
            printf("receiveed %s, nread = %d\n\n", readbuf, nread);  
        }  
        close(socketfd);
    }  
  
    return 0;  
}  
```


```select-server.c


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
int fd_A[BACKLOG];
int conn_amount;
int main()  
{  
    conn_amount = 0;
    int count = 0;  
    int i;
    int listenfd, socketfd,sockMax;  
    int nread;  
    struct sockaddr_in servaddr;  
    struct sockaddr_in client_addr; // connector's address information
    struct timeval timeoutval;  
    char readbuf[256];  
    socklen_t sin_size;
    sin_size = sizeof(client_addr);
    fd_set fdsr;
    printf("accept started\n");  
    //socket      
    if((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)  
    {  
        printf("socket error\n");  
        return -1;  
    }  
    bzero(&servaddr, sizeof(servaddr));  
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);  
    servaddr.sin_family = AF_INET;  
    servaddr.sin_port = htons(59000);  
    //bind  
    if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)  
    {  
        printf("bind error\n");  
        //return -1;  
    }  
    //listen  
    listen(listenfd, 5);  
    //accept  
    int ret;
    while(1)  
    {  
        FD_ZERO(&fdsr);
        FD_SET(listenfd, &fdsr);
        
        sockMax = listenfd;
        for (i = 0; i < BACKLOG; i++) {
            printf("BACKLOG [%d]\n",i);
            if (fd_A[i] != 0) {
                printf("BACKLOG--FD_SET--fd_A[%d]  %d\n",i,fd_A[i]);
                FD_SET(fd_A[i], &fdsr);
            //    int thisstat;
             //   thisstat=FD_ISSET(fd_A[i], &fdsr);
              //  printf("this is thisstat %d\n",thisstat);
            }
        }
        timeoutval.tv_sec = 30;
        timeoutval.tv_usec = 0;
        ret = select(sockMax + 1, &fdsr, NULL, NULL, &timeoutval);
        if (ret < 0) {
            perror("select");
            break;
        } else if (ret == 0) {
            printf("timeout\n");
            continue;
        }

        if (FD_ISSET(listenfd, &fdsr)) {
//            socketfd = accept(listenfd, NULL, NULL);  
            socketfd = accept(listenfd, (struct sockaddr *)&client_addr, &sin_size);
            if (socketfd <= 0) {
                perror("accept");
                continue;
            } 
            if (conn_amount < BACKLOG)
            {
                fd_A[conn_amount++] = socketfd;
                printf("new connection client[%d] \n", conn_amount);
                printf("new connection client[%d] %s:%d\n", conn_amount,inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                printf("new connection client[%d] %s:%d\n", conn_amount,inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                printf("new connection client[%d] %s:%d\n", conn_amount,inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
                if (socketfd > sockMax)
                    sockMax = socketfd;
            }
            else
            {
                printf("max connections arrive, exit\n");
                send(socketfd, "bye", 4, 0);
                close(socketfd);
                break;
            }
//            printf("start receive %d...\n", count++);  
//            memset(readbuf, sizeof(readbuf), 0);  
//            nread = recv(socketfd, readbuf, 10, 0);  
//            printf("nread %d\n",nread);
//            if(nread>0)  
//            {  
//                readbuf[10] = '\0';  
//                printf("receiveed %s, nread = %d\n\n", readbuf, nread);  
//            }  
            close(socketfd);
        }
    }  
    
//    for (i = 0; i < BACKLOG; i++) {
//        if (fd_A[i] != 0) {
//            close(fd_A[i]);
//        }
//    }
    return 0;  
} 
```














