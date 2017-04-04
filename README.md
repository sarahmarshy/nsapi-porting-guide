# Creating an mbed-os compatible communication API

The network-socket API provides a common interface for using sockets on network devices. It’s a class-based interface, which should be familiar to users experienced with other socket APIs.

## Class heirarchy 

All network-socket API implementations will inherit from two classes: a [NetworkStack](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.2/api/classNetworkStack.html) and a communication specific subclass of [NetworkInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classNetworkInterface.html). 

### NetworkInterface Class

The current NetworkInterface subclasses are [CellularInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classCellularInterface.html), [EthernetInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classEthernetInterface.html), [MeshInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classMeshInterface.html), and [WiFiInterface](https://docs.mbed.com/docs/mbed-os-api/en/mbed-os-5.4/api/classWiFiInterface.html). Your communication interface will be a subclass of one of these, as well as the NetworkStack. For example, the [ESP8266Interface](https://github.com/ARMmbed/esp8266-driver) inheritance structure looks like this: 

![Class](/img/esp-class.png)

There are three [pure virtual methods](https://en.wikipedia.org/wiki/Virtual_function#Abstract_classes_and_pure_virtual_functions) in the NetworkInterface class. 
* [`connect`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L99) - to connect the interface to the network
* [`disconnect`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L105) - to disconnect the interface from the network
* [`get_stack`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/NetworkInterface.h#L144) - to return the underlying NetworkStack object 

Each subclass has distinct pure virtual methods. Visit their class references (linked above) to determine those that must be implemented.


### NetworkStack class

`NetworkStack` provides a common interface that is shared between hardware that can connect to a network over IP. By implementing the NetworkStack, a network stack can be used as a target for instantiating network sockets.

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

High level API calls to an implementation of a network-socket API are intended to be the **identical** across networking protocols. The only intended difference is the method used to connect to the network. For example, a Wifi connection requires and SSID and password, while an Ethernet connetion does not. This homogeneity allows the user to change out the connectivity of their app simply by changing the API call for connecting to the network. 

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



## Implementation Details of a WiFiInterface

We will use the `ESP8266Interface` as a case study for implementing a communication interface.

### The constructor

[The defintion](https://github.com/ARMmbed/esp8266-driver/blob/master/ESP8266Interface.h#L37): 
`ESP8266Interface(PinName tx, PinName rx, bool debug = false);`

ESP8266 WiFi module communicates with the device MCU using a serial connection. The MCU will send [AT commands](https://en.wikipedia.org/wiki/Hayes_command_set) over this connection. We pass in the pins the MCU will use to communicate with the WiFi chip. 

### Communicating with the WiFi module

We've created an [AT command parser](https://github.com/ARMmbed/ATParser) to easily send AT commands and parse their responses. The AT command parser operates with a `BufferedSerial` object that provides software buffers and interrupt driven TX and RX for Serial. 

`ESP8266Interface` utilizes an underlying interface called [`ESP8266`](https://github.com/ARMmbed/esp8266-driver/tree/master/ESP8266) to handle the communication with the WiFi modem. `ESP8266` maintains an instance of AT command parser to handle communcation with the module.

### Implementing `connect()`

The `WiFiInterface` parent class requires us to implement [two methods for connecting to a WiFi network](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/WiFiInterface.h#L63-L83).

To utilize `nsapi_error_t connect()` we must have stored the SSID and password credentials for the network we want to connect to. For this, `WiFiInterface` requires that we implement [`set_credentials`](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/WiFiInterface.h#L46-L47).

The implementation of the connect method with SSID and password parameters is as follows:

```C++
int ESP8266Interface::connect(const char *ssid, const char *pass, nsapi_security_t security,
                                        uint8_t channel)
{
    if (channel != 0) {
        return NSAPI_ERROR_UNSUPPORTED;
    }

    set_credentials(ssid, pass, security);
    return connect();
}
```

The method that actually performs some communication with the WiFi module is as follows:

```C++
int ESP8266Interface::connect()
{
    _esp.setTimeout(ESP8266_CONNECT_TIMEOUT);

    if (!_esp.startup(3)) {
        return NSAPI_ERROR_DEVICE_ERROR;
    }

    if (!_esp.dhcp(true, 1)) {
        return NSAPI_ERROR_DHCP_FAILURE;
    }

    if (!_esp.connect(ap_ssid, ap_pass)) {
        return NSAPI_ERROR_NO_CONNECTION;
    }

    if (!_esp.getIPAddress()) {
        return NSAPI_ERROR_DHCP_FAILURE;
    }

    return NSAPI_ERROR_OK;
}
```

Let's step into this line `!_esp.connect(ap_ssid, ap_pass)`:

```C++
bool ESP8266::connect(const char *ap, const char *passPhrase)
{
    return _parser.send("AT+CWJAP=\"%s\",\"%s\"", ap, passPhrase)
        && _parser.recv("OK");
}
```

Here, the interface uses the AT command parser to send the correct command for connecting to a network. In this case, `AT+CWJAP=[ssid],[password]`. The AT command spec for this module specifies an `OK` response if the module successfully connects to the network. If the command is unsuccesfull, the logic dictates that the `connect()` method returns  `NSAPI_ERROR_NO_CONNECTION`. View the [documentation of Network errors](https://github.com/ARMmbed/mbed-os/blob/master/features/netsocket/nsapi_types.h#L37-L54) to determine the correct network error to return.

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

In this instance, we use the AT command parser to send `AT+CIPSTART=[id],[TCP or UDP], [address]` to the module. We expect to receive a response of `OK` We only return true if we succesfully send the command AND receive an `OK` response. 








