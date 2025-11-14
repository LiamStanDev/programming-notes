#### 頭文件
```cpp
#ifndef SERVER_H
#define SERVER_H
#include <array>
#include <fstream>
#include <netinet/in.h>
#include <string>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

class TCPServer
{
private:
  const static size_t BUFFER_SIZE = 4096;
  int m_listen_fd;
  int m_connected_fd;

  bool send(const void *buffer, size_t size);
  bool recv(void *buffer, size_t size);
  bool check_space(long require);

public:
  TCPServer() : m_listen_fd(-1), m_connected_fd(-1) {}

  ~TCPServer()
  {
    close_connted();
    close_listen();
  }

  bool listen(unsigned short port, size_t backlog = SOMAXCONN); // SOMAXCONN 默認為 4096
  bool accept();

  bool send(const std::string &message);
  bool recv_file(const std::string &filename);
  bool recv(std::string &buffer, size_t recv_size);

  bool close_listen();
  bool close_connted();
};

#endif
```

#### 源文件
```cpp
#include "server.hpp"

bool TCPServer::send(const void *buffer, size_t size)
{
  if (m_connected_fd == -1)
  {
    return false;
  }

  if (::send(m_connected_fd, buffer, size, 0) <= 0)
  {
    return false;
  }

  return true;
}

bool TCPServer::recv(void *buffer, size_t size)
{
  if (m_connected_fd == -1)
  {
    return false;
  }

  if (::recv(m_connected_fd, buffer, size, 0) <= 0)
  {
    return false;
  }

  return true;
}

bool TCPServer::check_space(long require) { return true; }

bool TCPServer::listen(unsigned short port, size_t backlog)
{
  if (m_listen_fd != -1)
  {
    return false;
  }

  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd == -1)
  {
    return false;
  }

  m_listen_fd = fd;

  sockaddr_in sockaddr{};
  sockaddr.sin_family = AF_INET;
  sockaddr.sin_addr.s_addr = htons(INADDR_ANY);
  sockaddr.sin_port = htons(port);

  if (::bind(m_listen_fd, reinterpret_cast<struct sockaddr *>(&sockaddr),
             sizeof(sockaddr)) == -1)
  {
    return false;
  }

  if (::listen(m_listen_fd, backlog) == -1)
  {
    return false;
  }

  return true;
}

bool TCPServer::accept()
{
  if (m_listen_fd == -1)
  {
    return false;
  }
  int fd = ::accept(m_listen_fd, nullptr, nullptr);
  if (fd == -1)
  {
    return false;
  }

  m_connected_fd = fd;
  return true;
}

bool TCPServer::send(const std::string &message)
{
  if (m_connected_fd == -1)
  {
    return false;
  }

  if (::send(m_connected_fd, message.data(), message.size(), 0) <= 0)
  {
    return false;
  }
  return true;
}

bool TCPServer::recv_file(const std::string &filename)
{
  // get file size
  long filesize = 0;
  if (!recv(&filesize, sizeof(filesize)))
  {
    return false;
  }

  // ckeck sapce
  if (!check_space(filesize))
  {
    return false;
  }
  std::ofstream fout(filename);
  if (!fout.is_open())
  {
    return false;
  }
  send("OK");

  std::array<char, BUFFER_SIZE> buffer;
  long onread = 0;
  long remain = filesize;

  while (remain > 0)
  {
    onread = remain >= BUFFER_SIZE ? BUFFER_SIZE : remain;
    if (!recv(buffer.data(), onread))
    {
      return false;
    }

    fout.write(buffer.cbegin(), onread);

    remain -= onread;
  }

  send("OK");

  return true;
}

bool TCPServer::recv(std::string &buffer, size_t recv_size)
{
  if (m_connected_fd == -1)
  {
    return false;
  }
  buffer.clear();
  buffer.resize(recv_size);

  if (::recv(m_connected_fd, &buffer[0], recv_size, 0) <= 0)
  {
    buffer.clear();
    return false;
  }

  return true;
}

bool TCPServer::close_listen()
{
  if (m_listen_fd == -1)
  {
    return false;
  }
  ::close(m_listen_fd);
  m_listen_fd = -1;
  return true;
}

bool TCPServer::close_connted()
{
  if (m_connected_fd == -1)
  {
    return false;
  }
  ::close(m_connected_fd);
  m_connected_fd = -1;
  return true;
}
```