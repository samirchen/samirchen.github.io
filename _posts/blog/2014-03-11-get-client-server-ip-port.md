---
layout: post
title: getsockname和getpeername示例
description: Linux下socket编程中常有Client和Server需要各自得到对方IP和Port的需求，这里就简单的写了一个小示例。
category: blog
tag: socket, linux, server, client, getsockname, getpeername, ip, port
---

##引言
[socket][2]编程中常有Client和Server需要各自得到对方IP和Port的需求，这里就简单的写了一个小示例，主要介绍了 `getsockname` 和 `getpeername` 的使用。

	#include <sys/socket.h>
	int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen);
	int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen);
	// 返回值：成功返回0，出错返回-1

使用场合：

对于服务器来说，在`bind`以后就可以调用`getsockname`来获取IP和Port，虽然这没有什么太多的意义。`getpeername`只有在链接建立以后才能使用，否则不能正确获得对方IP和Port，所以它的`sockfd参数`一般应该是 建立连接得到的套接字 而不是 监听套接字。

对于客户端来说，在调用socket时候内核还不会分配IP和Port，此时调用`getsockname`不会获得正确的IP和Port（当然链接没建立更不能调用`getpeername`）。不过如果调用了`bind`以后可以使用`getsockname`得到本地的IP和Port。想要正确的到对方IP和Port（一般客户端不需要这个功能），则必须在连接建立以后，同样连接建立以后，此时客户端IP和Port就已经被指定，此时也是调用`getsockname`的时机。


##Server端代码

	#include "stdio.h"
	#include "string.h" // for memset(), bzero()...
	#include <sys/socket.h> // for socket(), bind(), and connect()...
	#include <arpa/inet.h> // for sockaddr_in and inet_ntoa...
	
	#define MAXPENDING 10
	
	int main(int argc, char** argv) {
	    int listenSock, connectSock;
	    struct sockaddr_in serverAddress;
	
	    unsigned short serverPort = 1234;
	
	    // Create socket to listen incoming connections.
	    listenSock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
	
	    // Set local address info.
	    memset(&serverAddress, 0, sizeof(serverAddress)); // Init.
	    serverAddress.sin_family = AF_INET; // Internet address family.
	    serverAddress.sin_addr.s_addr = htonl(INADDR_ANY); // Any available interface. 0.0.0.0
	    serverAddress.sin_port = htons(serverPort); // Port.
	
	    // Bind to the local address.
	    bind(listenSock, (struct sockaddr*) &serverAddress, sizeof(serverAddress));
	
	    // Listen.
	    listen(listenSock, MAXPENDING);
	
	    struct sockaddr_in clientAddress;
	    socklen_t clientLen = sizeof(clientAddress);
	    char buffer[1000];
	    bzero(buffer, sizeof(buffer));
	
	    // Accept.
	    printf("Waiting connection...\n");
	    connectSock = accept(listenSock, (struct sockaddr*) &clientAddress, &clientLen);
	
	    // Get client IP:Port and Server IP:Port.
	    struct sockaddr_in c, s;
	    socklen_t cLen = sizeof(c);
	    socklen_t sLen = sizeof(s);
	    getsockname(connectSock, (struct sockaddr*) &s, &sLen); // ! use connectSock here.
	    getpeername(connectSock, (struct sockaddr*) &c, &cLen); // ! use connectSock here.
	    printf("Client: %s:%d\nServer: %s:%d\n", inet_ntoa(c.sin_addr), ntohs(c.sin_port), inet_ntoa(s.sin_addr), ntohs(s.sin_port));
	
	    // Receive message. 
	    recv(connectSock, buffer, sizeof(buffer), 0);
	    printf("Receive Message: %s\n", buffer);
	
	
	    return 0;
	}


##Client端代码

	#include <stdio.h>
	#include <string.h> // for memset(), bzero...
	#include <sys/socket.h> // for socket(), bind(), and connect()...
	#include <arpa/inet.h> // for sockaddr_in and inet_ntoa()...
	
	int main(int argc, char** argv) {
	    int sock; // Socket descriptor.
	    struct sockaddr_in serverAddress;
	    const char* serverIP = "192.168.37.229";
	    unsigned short serverPort = 1234;
	
	    // Create a reliable, stream socket using TCP.
	    sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
	
	    // Set server address info.
	    memset(&serverAddress, 0, sizeof(serverAddress)); // Init.
	    serverAddress.sin_family = AF_INET; // Internet address family.
	    serverAddress.sin_addr.s_addr = inet_addr(serverIP); // IP.
	    serverAddress.sin_port = htons(serverPort); // Port.
	
	    // Connect to server.
	    connect(sock, (struct sockaddr*) &serverAddress, sizeof(serverAddress));
	
	    // Get client IP:Port and Server IP:Port.
	    struct sockaddr_in c, s;
	    socklen_t cLen = sizeof(c);
	    socklen_t sLen = sizeof(s);
	    getsockname(sock, (struct sockaddr*) &c, &cLen);
	    getpeername(sock, (struct sockaddr*) &s, &sLen);
	    printf("Client: %s:%d\nServer: %s:%d\n", inet_ntoa(c.sin_addr), ntohs(c.sin_port), inet_ntoa(s.sin_addr), ntohs(s.sin_port));
	
	    // Send data to server.
	    char buffer[100];
	    int i = 0;
	    for (i = 0; i < 99; i++) {
	        buffer[i] = 'a';
	    }
	    buffer[99] = '\0';
	    send(sock, buffer, sizeof(buffer), 0);
	
	    // Close.
	    close(sock);
	
	
	    return 0;
	}

##其他
这里主要介绍了使用 `getsockname` 和 `getpeername` 来获取本地和连接端的IP和Port，其实通常Client端都会指定Server端的IP和Port，然后再去连接，也不用去获取了，其他信息也可以根据具体情况去灵活处理。


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://en.wikipedia.org/wiki/Network_socket