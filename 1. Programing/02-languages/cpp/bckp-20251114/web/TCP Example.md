## 大致流程
![[20230716_21h57m59s_grim.png]]

## Server
```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main(int argc, char* argv[]) {
    if (argc < 3) {
        std::cerr << "Need More Argument" << std::endl;
        return -1;
    }
    // listen to socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);

    // server ip configuration
    sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(atoi(argv[2]));
    inet_pton(AF_INET, argv[1], &addr.sin_addr);

    // Binding
    if (bind(listenfd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) != 0) {
        perror("Binding error");
        close(listenfd);
        return -1;
    }

    // Listening
    if (listen(listenfd, 50) != 0) {
        perror("Listening error");
        close(listenfd);
        return -1;
    }

    while (true) {
        sockaddr_in client_addr;
        socklen_t len = sizeof(addr);
        memset(&addr, 0, sizeof(addr));

        // accept
        int sockfd =
            accept(listenfd, reinterpret_cast<sockaddr*>(&client_addr), &len);
        if (sockfd < 0) {
            perror("Accepting Error");
            close(sockfd);
        }
        std::cout << "Connect from client: "
                  << ntohs(client_addr.sin_addr.s_addr) << std::endl;

        // receive
        char buffer[2000] = {0};
        if (recv(sockfd, buffer, 1999, 0) < 0) {
            perror("Receiving Error");
        }
        std::cout << "Client: " << buffer << std::endl;

        // send
        std::string message = "Hi";
        if (send(sockfd, message.c_str(), message.size(), 0) < 0) {
            perror("Sending Fail");
            close(sockfd);
        }
    }
	close(listenfd);
    return 0;
}
```


## Client
```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cstring>
#include <iostream>

int main(int argc, char* argv[]) {
    if (argc < 3) {
        std::cerr << "Need More Argument" << std::endl;
        return -1;
    }

    // create socket
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("Can not create socket");
        close(sockfd);
        return -1;
    }

    // server ip
    std::string server_ip = argv[1];
    short port = atoi(argv[2]);
    sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, server_ip.c_str(), &addr.sin_addr);

    // connect
    if (connect(sockfd, reinterpret_cast<sockaddr*>(&addr), sizeof(sockaddr)) !=
        0) {
        perror("Connection Error");
        close(sockfd);
        return -1;
    }
    std::cout << "Connected" << std::endl;

    // send
    std::string message = "Hi";
    if (send(sockfd, message.c_str(), message.size(), 0) < 0) {
        perror("Sending Fail");
        close(sockfd);
        return -1;
    }

    // receive
    char buffer[2000] = {0};
    if (recv(sockfd, buffer, 1999, 0) < 0) {
        perror("Sending Fail");
        close(sockfd);
        return -1;
    }
    std::cout << "Server: " << buffer << std::endl;
	close(sockfd);
    return 0;
}
```