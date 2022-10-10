# How to Configure TLS

RocketMQ uses Java options to manage and configure TLS. TLS itself is a complex mechanism, involving several security issues including Certificate Issuing of CA, Handshaking, Choosing of encryption algorithm, etc. It is strongly advised to read [TLS Wikipedia Page](https://en.wikipedia.org/wiki/Transport_Layer_Security) and [JCA](https://docs.oracle.com/javase/8/docs/technotes/guides/security/crypto/CryptoSpec.html).

Though you do not have to be a security expert before getting started, it's advisable to familiarize yourself with the following Handshake diagram, quoted from Wikipedia.
![TLS Handshake](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/Full_TLS_1.2_Handshake.svg.png) 

