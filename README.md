# Creating an mbed-os compatible communication API

The Network-Socket-API (NSAPI)  provides a TCP/UDP API on top of any IP based network interface. The NSAPI makes it easy to write applications and libraries that use TCP/UDP Sockets without regard to the type of IP connectivity. In addition to providing the TCP/UDP API, the NSAPI also includes virtual base classes for the different IP interface types.

## Class heirarchy 

All network-socket API implementations will inherit from two classes: a [NetworkStack](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.2/api/classNetworkStack.html) and a communication specific subclass of [NetworkInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classNetworkInterface.html). 

### NetworkInterface Class

The current NetworkInterface subclasses are [CellularInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classCellularInterface.html), [EthernetInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classEthernetInterface.html), [MeshInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classMeshInterface.html), and [WiFiInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classWiFiInterface.html). Your communication interface will be a subclass of one of these, as well as the NetworkStack. For example, the [ESP8266Interface](https://github.com/ARMmbed/esp8266-driver) inheritance structure looks like this: 

![Class](/img/esp-class.png)

There are three [pure virtual methods](https://en.wikipedia.org/wiki/Virtual_function#Abstract_classes_and_pure_virtual_functions) in the NetworkInterface class. 
* [`connect()`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L99) - to connect the interface to the network
* [`disconnect()`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L105) - to disconnect the interface from the network
* [`get_stack()`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L144) - to return the underlying NetworkStack object 

Each subclass has distinct pure virtual methods. Visit their class references (linked above) to determine those that must be implemented.


### NetworkStack class

`NetworkStack` provides a common interface that is shared between hardware that can connect to a network over IP. By implementing the NetworkStack, a class can be used as a target for instantiating network sockets.

NetworkStack requires that you implement the following functionalities:
* [getting an IP address from the network](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L45)
* [opening a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L120)
* [closing a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L130)
* [accepting connections on a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L184) 
* [attaching a callback to a state change of a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L270)
* [binding an address to a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L141)
* [connecting a socket to a remote host](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L164)
* [listening for incoming connections](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L153)
* [receving data on a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L218)
* [sending data on a socket](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h#L201)

### The `connect()` method

High level API calls to an implementation of a network-socket API are intended to be the **identical** across networking protocols. The only intended difference is the method used to connect to the network. For example, a Wifi connection requires and SSID and password, a Cellular connection requires an APN, while Ethernet doesn't require any credentials. These differences are reflected only in the `connect` method syntax of the derived classes. The intended design allows the user to change out the connectivity of their app simply by adding a new library and changing the API call for connecting to the network. 

Let's demonstrate with the code used to send an HTTP request over ethernet: 

```C++
    EthernetInterface net;
    net.connect();

    // Open a socket on the network interface, and create a TCP connection to api.ipify.org
    TCPSocket socket;
    socket.open(&net);
    socket.connect("api.ipify.org", 80);
    char *buffer = new char[256];

    // Send an HTTP request
    strcpy(buffer, "GET / HTTP/1.1\r\nHost: api.ipify.org\r\n\r\n");
    int scount = socket.send(buffer, strlen(buffer));

    // Recieve an HTTP response and print out the response line
    int rcount = socket.recv(buffer, 256);

    // Close the socket to return its memory and bring down the network interface
    socket.close();
    delete[] buffer;

    // Bring down the ethernet interface
    net.disconnect();
```

To change the connectivity to ESP8266 WiFi:

Change these lines: 
```C++
    EthernetInterface net;
    net.connect();
```

To:
```C++
    ESP8266Interface net;
    net.connect("my_ssid", "my_password");
```


## Case Study: ESP8266 Wifi component

Let's look at how we ported a driver for the ESP8266 WiFi module to the NSAPI.

### Required methods

We know that ESP8266 is a WiFi component, so we choose [`WiFiInterface`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/WiFiInterface.h) as our `NetworkworkInterface` parent class. 

`WiFiInterface` defines the following pure virtual functions: 
1. `set_credentials(const char *ssid, const char *pass, nsapi_security_t security)`
2. `set_channel(uint8_t channel)`
3. `get_rssi()`
4. `connect(const char *ssid, const char *pass, nsapi_security_t security, uint8_t channel)`
5. `connect()`
6. `disconnect()`
7. `scan(WiFiAccessPoint *res, nsapi_size_t count)`

Additionally, `WiFiInterface` parent class `NetworkInterface` introduces `NetworkStack *get_stack()` as a pure virtual function. 

We must also use [`NetworkStack`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkStack.h) as a parent class of our interface. We've already explored the pure virtual methods [here](#NetworkStack-class).

### Implementing `connect()`

We explained earlier that a WiFi connection requires an SSID and password. So how can we implement a connect function that doesn't have these as a parameter?

You might have noticed that one of the WiFiInterface pure virtual functions is `set_credentials(const char *ssid, const char *pass, nsapi_security_t security)`. So, we implemented `set_credentials` to store the SSID and password in private class variables. So, when we call `connect()` with no SSID and password, it is assumed that `set_credentials` has been called. 

Let's think about how we might implement this of the `connect()` method. 

This is the first method that will need to interact with the WiFi chip. We will need to do some configuration to get the chip in a state where we can open sockets, etc. We will need to send some [AT commands](https://www.espressif.com/sites/default/files/documentation/4a-esp8266_at_instruction_set_en.pdf) to the chip to accomplish this.

The AT commands we want to send are:

1. `AT+CWMODE=3` - This sets the WiFi mode of the chip to 'station mode' and 'SoftAP mode', where it acts as a client connection to a WiFi network, as well as a WiFi access point.
2. `AT+CIPMUX=1` - This allows the chip to have multiple socket connections open at once
3. `AT+CWDHCP=1,1` - To enable DHCP
4. `AT+CWJAP=[ssid,password]` - To connect to the network
5. `AT+CIFSR` - To query our IP address, and ensure that the network assigned us one through DHCP

#### Sending AT Commands 

We've created an [AT command parser](https://github.com/ARMmbed/ATParser) to easily send AT commands and parse their responses. The AT command parser operates with a `BufferedSerial` object that provides software buffers and interrupt driven TX and RX for Serial.

`ESP8266Interface` utilizes an underlying interface called [`ESP8266`](https://github.com/ARMmbed/esp8266-driver/tree/master/ESP8266) to handle the communication with the WiFi modem. `ESP8266` maintains an instance of AT command parser to handle communcation with the module. We have stored an instance of `ESP8266` in a private `ESP8266Interface` class variable `_esp`. In turn, `ESP8266` maintains an instance of AT command parser called `_parser`.

To send AT commands 1-2, we've made an `ESP8266` method called [`startup(int mode)`](https://github.com/ARMmbed/esp8266-driver/blob/master/ESP8266/ESP8266.cpp#L27). We will use the AT command parser's [`send`](https://github.com/ARMmbed/ATParser/blob/master/ATParser.h#L132) and [`recv`](https://github.com/ARMmbed/ATParser/blob/master/ATParser.h#L149) functions to accomplish this.

The necessary code is:
```C++

bool ESP8266::startup(int mode)
{
    ...

    bool success =
        && _parser.send("AT+CWMODE=%d", mode)
        && _parser.recv("OK")
        && _parser.send("AT+CIPMUX=1")
        && _parser.recv("OK");

    ...

```

The parser's `send` function returns true if the command was succesully sent to the WiFi chip. The `recv` function returns true if we receive the specified text. In the code example above, our success is determined by sending two commands and receiving the expected `OK` responses. 

#### Return values

So far, our connect method looks something like:

```C++
int ESP8266Interface::connect()
{
    if (!_esp.startup(3)) {
        return X;

```

What should we do if this `!_esp.startup(3)` evaluates to true? If it does, something went wrong when configuring the chip. So, we should return an error code. 

The NSAPI provides a set of error code return values for network operations. They are documented [here](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/nsapi_types.h#L37-L54).

Looking through them, the most appropriate seems to be ` NSAPI_ERROR_DEVICE_ERROR  = -3012,     /*!< failure interfacing with the network processor */`. So let's replace `X` in our `return` statement with `NSAPI_ERROR_DEVICE_ERROR`.

#### Finishing up 

We implemented similar methods to `startup` in ESP8266 to send AT commands 3-5. Then we used them to determine the success of the `connect()` method. The completed implementation can be found [here](https://github.com/ARMmbed/esp8266-driver/blob/master/ESP8266Interface.cpp#L47-L68).  


### Implementing `socket_connect`

The `NetworkStack` parent class dictates that we implement the functionality of connecting a socket to a remote server. This is the method signature:

```C++
 nsapi_error_t socket_connect(nsapi_socket_t handle, const SocketAddress &address)
```

The `nsapi_socket_t` type is an opaque handle for network sockets. In this case, the handle will be one that has been assigned in the [`socket_open`](https://github.com/ARMmbed/esp8266-driver/blob/master/ESP8266Interface.cpp#L137-L164) method. 

We've created an [`esp8266_socket struct`](https://github.com/ARMmbed/esp8266-driver/blob/master/ESP8266Interface.cpp#L130-L135) to hold the necessary socket information. As `nsapi_socket_t` is an opaque `typedef`, we can cast it to `esp8266_socket`.  We do this in the body of `socket_connect`: 

```C++
int ESP8266Interface::socket_connect(void *handle, const SocketAddress &addr)
{
    struct esp8266_socket *socket = (struct esp8266_socket *)handle;
    _esp.setTimeout(ESP8266_MISC_TIMEOUT);

    const char *proto = (socket->proto == NSAPI_UDP) ? "UDP" : "TCP";
    if (!_esp.open(proto, socket->id, addr.get_ip_address(), addr.get_port())) {
        return NSAPI_ERROR_DEVICE_ERROR;
    }
    
    socket->connected = true;
    return 0;
}
```


Focusing on this line: 
`!_esp.open(proto, socket->id, addr.get_ip_address(), addr.get_port()`. 

We access the socket ID and socket protocol from the members of `esp8266_socket`. We access the IP address and port of the server with the `SocketAddress addr` parameter. 

This line will send the AT command for opening a socket to the WiFi module. 

```C++
bool ESP8266::open(const char *type, int id, const char* addr, int port)
{
    //IDs only 0-4
    if(id > 4) {
        return false;
    }

    return _parser.send("AT+CIPSTART=%d,\"%s\",\"%s\",%d", id, type, addr, port)
        && _parser.recv("OK");
}
```

In this instance, we use the AT command parser to send `AT+CIPSTART=[id],[TCP or UDP], [address]` to the module. We expect to receive a response of `OK`. We only return true if we succesfully send the command AND receive an `OK` response. 








