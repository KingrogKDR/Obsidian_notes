# How HTTP/1.0 worked?

- Each time the client sends a request to the server, it has to open a new TCP connection and once it receives the response, then the connection is destroyed. This was quite an expensive operation and was really slow.
# How HTTP/1.1 solved the issues?

It introduced persistent connections, also called keep-alive. So, a TCP connection is setup between the client and the server only once. Then, the client can send requests and receive response without creating a TCP connection each time. 

But, there was still one problem. Each request was blocking, i.e another request cannot be sent unless the response of the previous request was received.

Therefore, it also introduced pipelining. Now, multiple requests can be sent at the same time without one waiting for another. However, the order in which they were sent still mattered. 

For e.g two requests A and B are sent to the server and 

Request A → /large-file
Request B → /small-file

The server finishes `/small-file` first.  But it still can't simply send Response B first and then Response A. It has to respect the order. Therefore, Response B has to wait until Response A is processed. This is called head-of-line (HOL) blocking problem.

Also, pipelining turned out to be difficult to deploy reliably across the Internet. Many servers, proxies, network devices and other intermediaries also didn't always handle pipelined requests correctly or efficiently.

HTTP traffic often passes through things like load balancers, proxies, reverse proxies, web servers, etc. For pipelining to be useful, the entire path should behave correctly. Also, there were some application level complications.

Therefore, browsers opened multiple TCP connections instead of heavily relying on pipelining.
# How HTTP/2 solves it? Also, what is the frame layer?

- HTTP/2 introduces a **binary framing layer**. Instead of sending HTTP messages as plain-text structures, it breaks communication into typed frames such as `HEADERS`, `DATA`, `SETTINGS`, and `WINDOW_UPDATE`. Each frame contains metadata such as its length, type, flags, and stream ID, followed by its payload. A single HTTP request or response can be represented using multiple frames.
- HTTP/2 allows multiple logical **streams** to share a single persistent TCP connection. Each stream has its own stream ID, which identifies which logical conversation a frame belongs to. Frames from different streams can therefore be **interleaved** on the same connection, while frames within each individual stream remain ordered. This is called **multiplexing**





