---
tags: [ios, security, network]
date: "2021-06-26T10:54:00Z"
title: Writing SSL-Pinning by yourself 

---

SSL Certificate Pinning is a mechanism that can be used to improve the security of a mobile app network connection. Pinning allows you to protect your users from [man-in-the-middle attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM).
Yes, iOS/macOS/... and HTTPS have a high-security level, and you can't read a whole HTTPS traffic of apps of someone's iPhone. But it's recommended to implement SSL-Pinning for financial or medical services because users' data must be protected from such attacks. So let's dive into how easily you can protect your users and what disadvantages it has.

<!--more-->

## Basic explanation

Let's say you have a mobile app and Server API. On the server, there is a certificate that encrypts all HTTPS traffic. You can open your API address in Safari and check the details about the certificate.

![Ceritificate with disabled proxy](/assets/2021-06-26/certificate-1.png)

If MITM attacks you, the certificate will change. You can check it by enabling Proxyman or Charles proxy.

![Ceritificate with enabled proxy](/assets/2021-06-26/certificate-2.png)

So the idea of certificate pinning is to verify the identity of a server by comparing certificates. Consider certificates have a tree structure with one certificate authority (CA) as root.

![Certificate tree](/assets/2021-06-26/cert-tree.png)


## Different ways to do pinning

In terms of what certificate to pin, there are several ways:

1. Pin root certificate of CA.
2. Pin your server certificate.

And you can verify the identity of the certificate in different ways too:

1. Check fingerprint – hash of whole certificate (public key + private key).
2. Check the public key only.

Checking the public key is recommended because you don't need to release a new version of your app when your certificate expires. If you renew the certificate properly, your public key stays the same, and only the private key will change.

## How to find out a public key

A public key is public, so it should be easy to find this. But, unfortunately, you can't just copy and paste it from Safari or elsewhere. To do that, you should use `openssl` in a terminal:

```bash
echo | openssl s_client -servername mobile-api.moi-service.ru -connect mobile-api.moi-service.ru:443 |\
sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > certificate.crt
```

This command saves `*.crt` file with a public key. Open it by a text editor, and you can see base64 encoded public key.

```
-----BEGIN CERTIFICATE-----
MIIFOTCCBCGgAwIBAgISA+SSfOmrY2CBVa4K7uzcDRLQMA0GCSqGSIb3DQEBCwUA
MDIxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQswCQYDVQQD
EwJSMzAeFw0yMTA2MTExMDE1MzVaFw0yMTA5MDkxMDE1MzRaMCQxIjAgBgNVBAMT
GW1vYmlsZS1hcGkubW9pLXNlcnZpY2UucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IB
DwAwggEKAoIBAQD3iZOIDcpdau4twemTBYk2KD23hvtfw3aFjULhYxEh7GFq5vGK
rex2NUp4o6/Yu8oid9gLeAK1P9ANlcIbXW3wvTeBV03Lyywj/2/3iBAfWO8daDN2
z7Kq3wu5yJkmrnB2C2C+gn2AaeSmdtlV5eDyB3pDUjuVlTq28lyiLS4zdJHBn0up
4h4+EYFXyHaG34/i17SLe34VoWiqN5k3UK3ZmO83PiU3oDV3Nx6NbTqkDlyuUXX8
n6H061zbK60tqDiU7psMizhdzewLZbSjTmZFboBWmhlX9TG41K8JPNCBB5CT3M9W
vDL6wMIKsQRqH2cRFmvBgrpkyjU6fW1sku8fAgMBAAGjggJVMIICUTAOBgNVHQ8B
Af8EBAMCBaAwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB
/wQCMAAwHQYDVR0OBBYEFM5tX/NrrNRV2w75IdiOdgTyen3BMB8GA1UdIwQYMBaA
FBQusxe3WFbLrlAJQOYfr52LFMLGMFUGCCsGAQUFBwEBBEkwRzAhBggrBgEFBQcw
AYYVaHR0cDovL3IzLm8ubGVuY3Iub3JnMCIGCCsGAQUFBzAChhZodHRwOi8vcjMu
aS5sZW5jci5vcmcvMCQGA1UdEQQdMBuCGW1vYmlsZS1hcGkubW9pLXNlcnZpY2Uu
cnUwTAYDVR0gBEUwQzAIBgZngQwBAgEwNwYLKwYBBAGC3xMBAQEwKDAmBggrBgEF
BQcCARYaaHR0cDovL2Nwcy5sZXRzZW5jcnlwdC5vcmcwggEFBgorBgEEAdZ5AgQC
BIH2BIHzAPEAdgBc3EOS/uarRUSxXprUVuYQN/vV+kfcoXOUsl7m9scOygAAAXn6
yNOVAAAEAwBHMEUCIHlbBaCGOHw9N/rQDUIg3l17A1H35Pb5kyFBdB9DsTPvAiEA
x80CxqcP7L8e/k2+THsR5z2wiOBeFfYWO4MnoAOs/oUAdwB9PvL4j/+IVWgkwsDK
nlKJeSvFDngJfy5ql2iZfiLw1wAAAXn6yNPTAAAEAwBIMEYCIQCFCwGpCP4CCJZq
zEGniluDY+93WsboPITv/yK0ugo9zQIhAOh/HQ/KomLe7tc5mT0GybdtpY8/zTUo
FyjZWjWOur3+MA0GCSqGSIb3DQEBCwUAA4IBAQAjzq/+SAbkDx7sR/ZUc4sJZNau
128VL2+YbmC/iTSMoGnNy1YguqD/qe8H9+E9f04kTYPIMCNEb+8UHveZcXKGENE8
MlAqkMEBWybEsml9B/3Ws1b7w1ZKg3hI5SNbEOxUIgamUD1a0xOYRCQBrZeY00PH
i3CJ4Pnec64tWYvuaclXzPcp0yzTxRH+z+AcpULDgWTP/lxmZkevdK5PSqnrE1md
94orfQagUVR4Sy3zW11llx1wBZZEsfps/tpn4aEQ7XrwGpCEao7kNu0FBVyJAkdG
9ktiztlj7OLDW8+3YOZPJTz4m+ltpBFyyxLi5H4TBofkKCL6kQbgzNDHccED
-----END CERTIFICATE-----
```
But we need `*.cer` binary format. The easiest way I found is to save `*.crt` to Keychain and export to `*.cer` after. Remember that `*.cer` format never has a private key. If you want you can ask your backend engineers to give you this format and don't do it by yourself.

## Write a code

If you are using Alamofire, I recommend reading the documentation carefully and use built-in API. But in my case, I use [Apollo](https://www.apollographql.com) for GraphQL, and there is no mechanism for SSL-pinning. 

Step zero is to save `*.cer` certificate to the main bundle because you need to verify it with server one (for fingerprint check, you can save a hash string to constant or config file).

To verify server identity you should use [`urlSession(_:didReceive:completionHandler:)`](https://developer.apple.com/documentation/foundation/urlsessiondelegate/1409308-urlsession#) method from `URLSessionDelegate `. This method is called for each request from your app. 

```swift
    override func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)
```

1. That's important to pin several certificates if you have several hosts. Also, skip some if it's not yours (Analytics API or similar). In this example, I verify my base host (`baseURL` is a private property of my class) with the request host. If it's different, I skip all checks.

```swift
guard
    challenge.protectionSpace.host == baseURL.host,
    challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
    let trust = challenge.protectionSpace.serverTrust
else {
    completionHandler(.performDefaultHandling, nil)
    return
}
```

2.  Verify certificate validity and obtain your server certificate. It's always at the index zero. The root certificate is at the last index.

```swift
var error: CFError?
let success = SecTrustEvaluateWithError(trust, &error)
    
guard
    success,
    let serverCertificate = SecTrustGetCertificateAtIndex(trust, 0)
else {
    completionHandler(.cancelAuthenticationChallenge, nil)
    return
}
```

3. We convert the server certificate to `Data` and open the file of our saved certificate `*.cer`.

```swift
let serverCertificateCFData = SecCertificateCopyData(serverCertificate)
    
guard
    let serverCertData = CFDataGetBytePtr(serverCertificateCFData),
    let filePath = Bundle.main.path(forResource: "production-api", ofType: "cer")
else {
    completionHandler(.cancelAuthenticationChallenge, nil)
    return
}
 
let fileURL = URL(fileURLWithPath: filePath)
let serverCert = Data(bytes: serverCertData, count: CFDataGetLength(serverCertificateCFData))
```

4. Saved certificate is also converted to `Data` and compared with the server certificate.

```swift
guard
    let fileCert = try? Data(contentsOf: fileURL),
    serverCert == fileCert
else {
    completionHandler(.cancelAuthenticationChallenge, nil)
    return
}
    
completionHandler(.performDefaultHandling, nil)
```

The entire method you can find [here](https://gist.github.com/vani2/819a45d0951af6ac62f7c6a7f0d8c263).
That's it! You can test it by enabling proxy on your iPhone. Then, any request to your server from your app would be interrupted with an error.

## Final recommendations

1. Do not pin certificates from 3rd party server. Google or Facebook can change it without asking you.
2. It's convenient to turn on SSL-pinning only on release builds. It's more accessible to you and QA to analyze traffic to detect bugs.
3. Pin Public Key. It's a pain in the ass when a certificate will be renewed.
