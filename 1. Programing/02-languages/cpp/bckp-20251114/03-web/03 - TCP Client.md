#### 頭文件
```cpp
#ifndef CLIENT_H
#define CLIENT_H
#include <arpa/inet.h>
#include <array>
#include <fstream>
#include <ios>
#include <netinet/in.h>
#include <stdexcept>
#include <sys/socket.h>
#include <unistd.h>

class TCPClient
{
private:
  const static size_t BUFFER_SIZE = 4096;
  int m_sockfd;
  bool send(const void *buffer, size_t size);
  bool recv(void *buffer, size_t size);

public:
  TCPClient() : m_sockfd(-1) {}
  ~TCPClient() { close(); }

  bool connect(const std::string &ip, unsigned short port);
  bool send(const std::string &message);
  bool send_file(const std::string &filename);
  bool recv(std::string &buffer, size_t recv_size);
  bool close();
};
#endif
```


#### 源文件
```cpp
#include "client.h"

bool TCPClient::connect(const std::string &ip, unsigned short port)
{
  if (m_sockfd != -1)
  {
    return false;
  }

  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd == -1)
  {
    throw new std::runtime_error("Can not get socket file descriptor");
    return false;
  }
  m_sockfd = fd;

  sockaddr_in target_addr{};

  target_addr.sin_family = AF_INET;
  target_addr.sin_addr.s_addr = inet_addr(ip.c_str());
  target_addr.sin_port = htons(port);

  if (::connect(m_sockfd, reinterpret_cast<struct sockaddr *>(&target_addr),
                sizeof(target_addr)) == -1)
  {
    return false;
  }

  return true;
}

bool TCPClient::send(const std::string &message)
{
  if (m_sockfd == -1)
  {
    return false;
  }

  if (::send(m_sockfd, message.data(), message.size(), 0) <= 0)
  {
    return false;
  }
  return true;
}

bool TCPClient::send_file(const std::string &filename)
{
  std::ifstream fin(filename, std::ios_base::binary);
  if (!fin.is_open())
  {
    return false;
  }

  // get filesize
  fin.seekg(0, std::ios_base::end);
  const long filesize = fin.tellg();
  fin.seekg(0, std::ios_base::beg);

  // send to server for checking space
  if (!send(&filesize, sizeof(filesize)))
  {
    return false;
  }

  std::string server_reply;
  if (!recv(server_reply, 2) || server_reply != "OK")
  {
    return false;
  }

  std::array<char, BUFFER_SIZE> buffer;
  long onread = 0;
  long remain = filesize;

  while (remain > 0)
  {
    onread = remain >= BUFFER_SIZE ? BUFFER_SIZE : remain;
    fin.read(buffer.begin(), onread);

    if (!send(buffer.cbegin(), buffer.size()))
    {
      return false;
    }

    remain -= onread;
  }

  if (!recv(server_reply, 2) || server_reply != "OK")
  {
    return false;
  }

  return true;
}

bool TCPClient::send(const void *buffer, size_t size)
{
  if (m_sockfd == -1)
  {
    return false;
  }

  if (::send(m_sockfd, buffer, size, 0) <= 0)
  {
    return false;
  }

  return true;
}

bool TCPClient::recv(std::string &buffer, size_t recv_size)
{
  if (m_sockfd == -1)
  {
    return false;
  }
  buffer.clear();
  buffer.resize(recv_size);

  if (::recv(m_sockfd, &buffer[0], recv_size, 0) <= 0)
  {
    buffer.clear();
    return false;
  }

  return true;
}

bool TCPClient::recv(void *buffer, size_t size)
{
  if (m_sockfd == -1)
  {
    return false;
  }

  if (::recv(m_sockfd, buffer, size, 0) <= 0)
  {
    return false;
  }

  return true;
}

bool TCPClient::close()
{
  if (m_sockfd == -1)
  {
    return false;
  }
  ::close(m_sockfd);
  return false;
}

```