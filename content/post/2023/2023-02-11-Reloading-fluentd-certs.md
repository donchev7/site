---
date: 2023-02-11
image: 'post/2023/reloading-fluentd-certs.png'
title: 'Reloading Fluentd certificates'
slug: 'reloading-fluentd-certificates'
toc: true
tags:
  - DevOps
  - Kubernetes
  - Go
---

## Introduction

Fluentd is a great tool for collecting logs. Usually it's deployed in Kubernetes as a Log Aggregator and / or a Log Forwarder. In this post I will focus on Fluentd as a Log Aggregator. In a usual log aggregation setup we would want to use TLS for encrypting our traffic and even though Fluentd supports TLS certificates, it does not support `automatic` cert reloads. 

Why do we need to reload the certificates? lets-encrypt certificates are valid for 90 days and they are renewed every 60 days. And since I am 99% of the time using lets-encrypt certificates, I will focus on that =)

In this post I will show you how to configure Fluentd to use lets-encrypt certificates. Primarily, I will show you how to reload the certificates when they are renewed.


## Configuring Fluentd to use lets-encrypt certificates

Configuring Fluentd to use lets-encrypt certificates is pretty straightforward. We just need to add the following lines to our Fluentd config:

```Fluentd
    <source>
      @type forward
      @log_level error
      <transport tls>
        version TLSv1_2
        ciphers ALL:!aNULL:!eNULL:!SSLv2
        insecure false

        cert_path /letsencrypt/tls.crt
        private_key_path /letsencrypt/tls.key
        client_cert_auth false
      </transport>

      bind 0.0.0.0
      port 24224
    </source>
```

The only thing we need to do for the above config to work is to mount the lets-encrypt certificates to the Fluentd container. In my case I am using  [cert-manager](https://cert-manager.io/) to manage my certificates. So I just need to add the following lines to my Fluentd deployment:

```yaml
      volumes:
        - name: letsencrypt
          secret:
            secretName: Fluentd-tls
```

I am omitting the rest of the Fluentd deployment config for brevity. But if you have any questions, feel free to contact me on twitter.

## Checking if the certificates have been renewed

Now that we have configured Fluentd to use lets-encrypt certificates, we need to monitor the certificates for renewal. I wrote a small go application for this purpose. The application is pretty simple. It checks if the certificate served by the Fluentd server has a different expiration date than the certificate created by cert-manager. 

I deployed the application as a cronJob in my Kubernetes cluster with a once a day schedule.

Full application code is available on [GitHub](https://github.com/donchev7/Fluentd-reloader)

An alternative to this approach might be to deploy the service as a kubernetes deployment and watch for changes to the certificate CRD. However, I didn't want to add another long running service to my cluster just for this purpose. IMO, a cronJob is a much better fit for this use case.

The application is pretty simple. It has a main function that is executed once a day. The main function will call the `checkCert` function to get the expiration date of the certificate served by the Fluentd server. Then it will call the `getCRD` function to get the expiration date of the certificate created by cert-manager. Finally, it will compare the two dates and if they are different it will call the `reloadFluentdConfig` function trigger Fluentd to do a graceful config reload.

Checking the expiration date of a server certificate using the go tls package is pretty straightforward:

```go
// get the expiration date of the certificate served by the Fluentd server
func checkCert(serviceURL string) (time.Time, error) {
	conn, err := tls.Dial("tcp", fmt.Sprintf("%s:443", serviceURL), nil)
	if err != nil {
		return time.Time{}, fmt.Errorf("Server doesn't support SSL certificate err: %w", err)
	}

	err = conn.VerifyHostname(serviceURL)
	if err != nil {
		return time.Time{}, fmt.Errorf("Hostname doesn't match with certificate: %w", err)
	}
	expiry := conn.ConnectionState().PeerCertificates[0].NotAfter
	log.Printf("Issuer: %s\nExpiry: %v\n", conn.ConnectionState().PeerCertificates[0].Issuer, expiry.Format(time.RFC850))

	return expiry, nil
}
```

The function checkCert will return the expiration date of the current certificate served by the Fluentd server or an error if there is a problem with the server / certificate.


The following code will fetch the certificate CRD (Custom Resource Definition) object created by cert-manager from the Kubernetes API. The CRD object contains the expiration date of the certificate.

```go
func (a app) getCRD() (cmapi.Certificate, error) {
	certificates := cmapi.CertificateList{}
	uri := fmt.Sprintf("/apis/cert-manager.io/v1/namespaces/%s/certificates", a.namespace)
	err := a.client.RESTClient().Get().RequestURI(uri).Do(context.Background()).Into(&certificates)
	if err != nil {
		return cmapi.Certificate{}, fmt.Errorf("failed to get certificates: %w", err)
	}

	for _, cert := range certificates.Items {
		if strings.EqualFold(cert.Name, a.certName) {
			return cert, nil
		}

		log.Printf("Certificate %s is not Fluentd cerificate", cert.Name)
	}

	return cmapi.Certificate{}, fmt.Errorf("failed to find Fluentd certificate")
}

```

Now that I have the expiration dates I can compare them. Having looked at the cert-manager code, I found that the certificate CRD object has a helper function for comparing expiration dates. So I am going to use that function for the comparison:

```go
t := metav1.NewTime(expiry)
if certificate.Status.NotAfter.Equal(&t) {
  log.Println("The certificates are the same. No need to reload Fluentd config")

  return
}
```

## Reloading the certificates (Fluentd config)

Now that I have the code for checking the expiration dates, I can reload the certificates if they are different. Fluentd has a built-in API for reloading the config. Instead of killing the running pod we can just call the API and Fluentd will reload the config.

First we need to enable the RPC API. This is done by adding the following lines to the Fluentd config:

```Fluentd
    <system>
      rpc_endpoint 0.0.0.0:24444
    </system>
```

Now we can call the API to reload the config:

```go
func reloadFluentdConfig(ips ...string) error {
	for _, ip := range ips {
		log.Println("Reloading Fluentd config on", ip)

		url := fmt.Sprintf("http://%s:24444/api/config.gracefulReload", ip)
		req, err := http.NewRequest("GET", url, nil)
		if err != nil {
			return fmt.Errorf("failed to create request: %w", err)
		}

		client := &http.Client{
			Timeout: 5 * time.Second,
		}
		resp, err := client.Do(req)
		if err != nil {
			return fmt.Errorf("failed to send request: %w", err)
		}
		defer resp.Body.Close()

		// check response
		if resp.StatusCode >= 400 {
			return fmt.Errorf("failed to reload Fluentd config: %s", resp.Status)
		}

		b, err := io.ReadAll(resp.Body)
		if err != nil {
			return fmt.Errorf("failed to read response body: %w", err)
		}

		log.Printf("Response: %s", string(b))
	}

	return nil
}
```

## Conclusion

You can find the full code for the application on [GitHub](https://github.com/donchev7/Fluentd-reloader). I hope you found this article useful. If you have any questions, feel free to contact me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev)

## References

  * [cert-manager](https://cert-manager.io/)
  * [Fluentd TLS](https://docs.fluentd.org/input/forward#tls-configuration)
  * [Fluentd RPC API](https://docs.fluentd.org/deployment/rpc)
  * [Fluentd graceful reload](https://docs.fluentd.org/deployment/signals#sigusr2)