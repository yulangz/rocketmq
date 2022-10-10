# How to Configure TLS

## Prerequisite

TLS itself is a complex mechanism, involving several security issues including Certificate Issuing of CA, Handshaking, Choosing of encryption algorithm, etc. It is strongly advised to read [TLS Wikipedia Page](https://en.wikipedia.org/wiki/Transport_Layer_Security) and [JCA](https://docs.oracle.com/javase/8/docs/technotes/guides/security/crypto/CryptoSpec.html).

Though you do not have to be a security expert before getting started, it's advisable to familiarize yourself with the following Handshake diagram, quoted from Wikipedia.
![TLS Handshake](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/Full_TLS_1.2_Handshake.svg.png) 

## RocketMQ Security Policy

RocketMQ uses Java options to manage and configure TLS.  Through selectively combining these options and choices, RocketMQ servers(including name server and broker) can provide plaintext transport, single-side certificate verification TLS and mTLS.

### tls.server.mode
Valid choices of this Java Option are `disabled`, `permissive` and `enforcing`.

If `-Dtls.server.mode=disabled` is set, servers will only accept and server plaintext transports. Namely, no TLS is available.

If `-Dtls.server.mode=permissive` is set, servers will accept both plaintext and TLS traffics at the same time. 

If `-Dtls.server.mode=enforcing` is set, servers will accept TLS traffics only. Plaintext connections will be rejected since they would fail required handshake steps.




## Examples

### Development Environment with Self Signed Certificate


### Simple Production Example


### Production Example with Certificate Rotation

