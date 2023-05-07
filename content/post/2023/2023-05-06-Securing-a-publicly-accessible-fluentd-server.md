---
date: 2023-05-06
image: 'post/2023/zero-trust.png'
title: 'Securing a publicly accessible Fluentd server'
slug: 'securing-a-publicly-accessible-fluentd-server'
toc: true
tags:
  - DevOps
  - Go
---

# Securing a publicly accessible Fluentd server: mTLS & Zero-Trust

In this post, I'll walk you through the process of enhancing the security of a publicly accessible Fluentd server by implementing a Go TCP proxy that leverages mutual TLS (mTLS) authentication. By utilizing client-side self-signed certificates and their fingerprints, the proxy will ensure that only authorized clients can connect to the Fluentd server. Additionally, I'll explore alternative zero-trust solutions like Cloudflare and OpenZiti.


## Why Securing Fluentd Matters

Fluentd may need to be publicly accessible from the internet for a variety of reasons. The specific use case I recently dealt with involved an application deployed across multiple cloud providers. The application needed to send logs to a centralized Fluentd instance for aggregation and analysis. The cloud providers were not peered and did not have a VPN connection between them. As a result, the only way to send the logs was to make Fluentd publicly accessible.

Altough it was convenient, it also left my logging infrastructure vulnerable to potential security threats. As a result, it's absolutely essential to employ strong security measures like mTLS and at a bare minimum an IP allow-list and Firewall. 

Newer zero-trust solutions like Cloudflare and OpenZiti can also be used. I'll discuss these solutions in more detail later in this post.


## A Closer Look at the mTLS-based Go TCP Proxy

I wrote a Go proxy package that takes care of the core functionality of a TCP proxy, including mTLS authentication. Acting as an intermediary between clients and the Fluentd server. This proxy allows connections exclusively from authorized clients with valid self-signed certificates. The proxy relies on the fingerprints of these certificates for validation.

```go
type FingerprintValidator interface {
	CheckFingerprint(fingerprint string) bool
}

type Proxy struct {
	listener  net.Listener
	group     *errgroup.Group
	validator FingerprintValidator
	stop      context.CancelFunc
	ctx       context.Context
	cfg       Opts
}

type Opts struct {
	Validator      FingerprintValidator
	ListenAddr     string
	FluentdAddress string
	CertFilePath   string
	KeyFilePath    string
}

func New(in Opts) *Proxy {
	p := Proxy{
		validator: in.Validator,
		group:     &errgroup.Group{},
		cfg:       in,
	}
	p.listener = p.mustListener()

	return &p
}
```

The Proxy struct contains the `FingerprintValidator` interface, which can be implemented depending on where the fingerprints are stored. For example, we can implement the `CheckFingerprint()` method to check the fingerprints against a database, a file, or any other data source (I recommend redis).

Another thing to note is that the `Opts` struct contains the `CertFilePath` and `KeyFilePath` fields. These are needed because the proxy acts as a TLS server. Meaning the traffic between the client and the proxy is encrypted. I used letsencrypt to generate the certificates for the proxy. But you can use any certificate authority or self-signed certificates.


```go
func (p *Proxy) mustListener() net.Listener {
	cer, err := tls.LoadX509KeyPair(p.cfg.CertFilePath, p.cfg.KeyFilePath)
	if err != nil {
		log.Panic().Err(err).Msgf("Error loading cert/key pair")
	}

	config := &tls.Config{
		Certificates:          []tls.Certificate{cer},
		ClientAuth:            tls.RequireAnyClientCert,
		VerifyPeerCertificate: p.verifyPeer,
		MinVersion:            tls.VersionTLS12,
	}

	l, err := tls.Listen("tcp", p.cfg.ListenAddr, config)
	if err != nil {
		log.Panic().Msg("Could not start proxy")
	}

	return l
}
```

The full code for the proxy can be found [here](https://github.com/donchev7/fluentd-mtls-proxy/blob/main/proxy/proxy.go)

The connection handling logic is based on `one connection one goroutine`. Meaning that for every connection, a new goroutine is spawned. Based on the benchmarks [here](https://github.com/smallnest/1m-go-tcp-server) the performance of this approach is sufficient for most use cases. If you are memory constrained, you can use a pool of goroutines to handle the connections. Just adjust the constructor (New function) and set a limit on the number of goroutines.

```go
func New(in Opts) *Proxy {
  maxGoroutines := 1000
	group := errgroup.Group{}
	group.SetLimit(maxGoroutines)

	p := Proxy{
		validator: in.Validator,
		group:     &group,
		cfg:       in,
	}
	p.listener = p.mustListener()

	return &p
}
```

## Setting Up the Proxy for Fluentd

To set up the proxy, follow these simple steps:

### Initialize the Proxy

Adjust the `ListenAddr` and `FluentdAddress` in the  main.go file and run `go run main.go`. You also need to have a running Fluentd instance. The easiest way to do this is to use the official [docker image](https://hub.docker.com/r/fluent/fluentd):

```bash
docker run -it -p 24224:24224 -e FLUENTD_CONF="<source>@type forwardport 24224bind 0.0.0.0</source><match **>@type stdout</match>" fluent/fluentd:latest
```

### Test the proxy

To test the proxy you will need to send a msgpack-formatted message to the proxy. We can send a message using the `fluent-cat` command line tool:

```bash
echo '{"message":"hello"}' | fluent-cat debug.log --host localhost --port 8080
```

If everything is set up correctly, you should see the message in the Fluentd logs:

```bash
2023-05-07 15:20:20.000000000 +0000 debug.log: {"message":"hello"}
```

## Alternative Solutions

While the mTLS-based Go TCP proxy with a Firewall and IP allow list effectively secures a publicly accessible Fluentd instance. It's not the only solution. 

Recently I have been experimenting with Cloudflare's Zero Trust service and OpenZiti. Both of these solutions allow me to create a zero-trust connection between clients and servers. 

### Zero Trust Networking

Zero-Trust networking is a security paradigm that shifts the focus from traditional `perimeter-based defenses` such as firewalls, VPNs, and IP allow lists. Under this model, the network is not assumed to be trusted. Instead, every access request, whether it originates from inside or outside the network, is treated with equal scrutiny and `must be verified` before granting access to resources. This model is built on the principle of "never trust, always verify," and requires continuous authentication and authorization of users, devices, and applications. By implementing granular access controls, Zero-Trust networking minimizes the attack surface and reduces the risk of unauthorized access, data breaches, and `lateral movement` of threats within the network.

In the case of Cloudflare, we can leverage their Zero Trust service to create a zero-trust connection between our clients and the Fluentd server. OpenZiti, on the other hand, is open source and offers a Go SDK that allows to create zero-trust connections from our clients to my backend infrastructure.


#### Cloudflare

Some of the benefits when using Cloudflares Zero-Trust service are:

1. It is a managed service - no need to manage the infrastructure and the certificates
2. WARP client - No custom code needed on the client. It's available for Windows, Mac, Linux, iOS, and Android.
3. Scalability and performance -  Cloudflare's global network ensures high performance and scalability. The WARP client connects to the nearest Cloudflare PoP and the traffic is routed to the Fluentd server. This ensures that the Fluentd server is not exposed to the internet and is protected by Cloudflare's network.

(disclaimer: I have no affiliation with Cloudflare but I do use their services and own stock in the company)

#### OpenZiti

OpenZiti is another option for implementing a zero-trust architecture. With its Go SDK, I can create secure connections between my clients and the backend without needing to install anything on my clients. The OpenZiti SDK handles authentication, authorization, and encryption.

Obviously, this solution requires more work than the Cloudflare one. But it's open source and can be customized to fit my needs. Here is an example [server](https://github.com/openziti/sdk-golang/blob/main/example/simple-server/simple-server.go) and [client](https://github.com/openziti/sdk-golang/blob/main/example/curlz/curlz.go) implementation. It's basically a custom Go HTTP Client Transport:

```go
zitiDialContext := ZitiDialContext{context: ctx}

zitiTransport := http.DefaultTransport.(*http.Transport).Clone() // copy default transport
zitiTransport.DialContext = zitiDialContext.Dial
zitiTransport.TLSClientConfig.InsecureSkipVerify = true
client := &http.Client{Transport: zitiTransport}
```

Not much code if you ask me.

## Conclusion

Securing a publicly accessible Fluentd instance is crucial. Implementing a Go TCP proxy with mTLS authentication is an effective way to ensure that only authorized clients can connect to the Fluentd server. Additionally, considering alternative zero-trust solutions like Cloudflare Zero Trust and OpenZiti can further enhance the security.

By combining these security measures, we can build a robust and secure public Fluentd deployment. Without having to worry about unauthorized access.

If you have any questions or comments, feel free to reach out to me on [Twitter](https://twitter.com/bobby_donchev)