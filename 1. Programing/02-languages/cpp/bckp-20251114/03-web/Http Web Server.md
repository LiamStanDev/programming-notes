### What is addrinfo?
`addrinfo` is a struct to store information of adresses.
```c++
struct addrinfo
{
  int ai_flags;			
  int ai_family;		    // AF_INET = 2 or AF_INET6 = 10. equal to ai_addr->sa_family
  int ai_socktype;		    // SOCK_STREAM = 1 or SOCK_DGRAM = 2
  int ai_protocol;		    // IPROTO_TCP = 6 or IPROTO_UDP = 17
  socklen_t ai_addrlen;		// length of sockaddr. (size depends on protocal)
  struct sockaddr *ai_addr;
  char *ai_canonname;
  struct addrinfo *ai_next;
};
```
> `ai` means addrinfo

##### sockaddr
The old version of socket endpoint.
```cpp
struct socketaddr {
	unsigned short sa_family; // socket address family
	char sa_data[14]; // store port and address
};
```
> `sa` means sockaddr

###### Problem: `sa_data` can't store IPV6 with 128 bits. 
This is not important because `struct sockaddr*` is like `void*`. For this reason, function like `bind()` needs the `socklen_t`.

##### Why addrinfo is a linked list?
reference: [stackoverflow](https://stackoverflow.com/questions/21797913/purpose-of-linked-list-in-struct-addrinfo-in-socket-programming)
A host can have more that one IP address.
```c++
struct addrinfo *addrs, *addr;

getaddrinfo("www.google.com", NULL, &hints, &addrs);
```

##### freeaddrinfo
Because addrinfo is a linked list. We need to release it by `freeaddrinfo`


### Create a socket
```c++
// Step 1: get addrinfo
struct addrinfo* addrinfo;
// parse and create addrinfo
int err = getaddrinfo("127.0.0.1", "8080", nullptr, &addrinfo); // second order pointer because it need to form a linked list.
// If the getaddrinfo return value < 0 means error happen.
if (err != 0) {
	fmt::println("getaddrinfo: {} {}", gai_strerror(err))
	return 1;
}

// Step 2: get sockfd
int sockfd = socket(addrinfo->ai_family, addrinfo->ai_socktype, addrinfo->ai_protocol);
if (sockfd == -1) {
  fmt::println("socket: {}", strerror(errno)); // use global errno
  return 1;
}
```

##### Why getaddrinfo don't use errno?
Because of size the errno increase so fast, they choose to seperate specific error into they own. The `getaddrinfo` use `gai_error` as error code, and needs to use `gai_strerror`. see: `man getaddrinfo`

### Encapsulte socket creation
```c++
```