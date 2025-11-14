# Server
```cpp
#include <arpa/inet.h>
#include <cstring>
#include <iostream>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
  int serverSocket = socket(AF_INET, SOCK_STREAM, 0);

  sockaddr_in serverAddress;
  serverAddress.sin_family = AF_INET;
  serverAddress.sin_addr.s_addr = inet_addr("127.0.0.1");
  serverAddress.sin_port = htons(8080);

  if (bind(serverSocket, (const struct sockaddr *)&serverAddress,
           sizeof(serverAddress)) == -1)
  {
    std::cerr << "Error on binding..." << std::endl;
    close(serverSocket);
    return -1;
  }

  if (listen(serverSocket, 10) == -1)
  {
    std::cerr << "Error on listing..." << std::endl;
    close(serverSocket);
    return -1;
  }

  std::cout << "Server is listening on port " << ntohs(serverAddress.sin_port)
            << std::endl;

  while (true)
  {
    sockaddr_in clientAddress;
    socklen_t clientAddressLenght = sizeof(clientAddress);
    int handle = accept(serverSocket, (struct sockaddr *)&clientAddress,
                        &clientAddressLenght);

    if (handle == -1)
    {
      close(serverSocket);
      return -1;
    }

    std::string buffer;
    buffer.resize(1024);
    while (recv(handle, &buffer[0], buffer.size(), 0) > 0)
    {
      std::cout << "Revceived: " << buffer << std::endl;

      std::string message;
      std::cin >> message;
      send(handle, &message[0], message.size(), 0);
    }
  }

  return 0;
}
```

# Client
```cpp
#include <arpa/inet.h>
#include <iostream>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

int main()
{
  int clientSocket = socket(AF_INET, SOCK_STREAM, 0);
  if (clientSocket == -1)
  {
    std::cerr << "Error creating socket" << std::endl;
    return -1;
  }

  sockaddr_in address;
  address.sin_family = AF_INET;
  address.sin_addr.s_addr = inet_addr("127.0.0.1");
  address.sin_port = htons(8080);

  if (connect(clientSocket, (struct sockaddr *)&address, sizeof(address)) == -1)
  {
    std::cerr << "Error connecting to server" << std::endl;
    close(clientSocket);
    return -1;
  }

  std::cout << "Connected to server" << std::endl;
  while (true)
  {
    std::string message;
    std::cin >> message;

    send(clientSocket, message.c_str(), message.size(), 0);

    std::string buffer;
    buffer.resize(1024);

    int bytesRead = recv(clientSocket, &buffer[0], buffer.size(), 0);
    std::cout << "Received from server: " << buffer << std::endl;
  }

  close(clientSocket);
  return 0;
}
```