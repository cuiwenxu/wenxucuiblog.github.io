Facebook 技术副总裁 Jeff Rothschild 描述了一个他们称之为 Multiget Hole 的问题。

当 memcached 服务器受 CPU 限制时，您会陷入 multiget 漏洞，添加更多 memcached 服务器却无法处理更多的请求，怎么去解决这个问题？

What happens when you add more servers is that the number of requests is not reduced, only the number of keys in each request is reduced. The number keys returned in a request only matters if you are bandwidth limited. The server is still on the hook for processing the same number of requests. Adding more machines doesn't change the number of request a server has to process and since these servers are already CPU bound they simply can't handle more load. So adding more servers doesn't help you handle more requests. Not what we usually expect. This is another example of why architecture matters.
简单分析后我们可以发现其背后隐藏的问题，影响缓存服务器性能的因素有三个：缓存服务器的内存、调用者和缓存服务器的带宽和调用者的cpu。加了更多的缓存服务器，增加的只是缓存服务器的内存，如果带宽或者cpu已达到瓶颈的，加再多缓存服务器也只能是浪费。
# 理解mget
mget允许我们在一个请求中携带多个key，举例：比如一个用户有100个好友，这些好友信息的改变可以在一个请求中获取到，这比100个独立的请求效率高很多。
meget允许开发者透明地使用两种经典的可扩展性策略：批处理和并行化。
