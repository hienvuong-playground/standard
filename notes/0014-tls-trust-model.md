# 0014 – TLS trust model: why each piece exists

## Context

TLS has several parties, two key pairs, and two different kinds of proof. Reading about them one at a time gets confusing fast. While working on certs (0010, 0012, 0013), it helped to see TLS as a chain where each mechanism fills a gap left by the one before. This note writes that chain down, at a high level and without protocol details.

## The chain

### Starting point: keep the traffic private

A browser wants to talk to `example.com`. Plain text is readable by anyone on the path (café wifi, the ISP, a compromised router). So the server holds a key pair, and the browser uses the server's public key to set up an encrypted connection.

### Gap 1: whose key is this?

Encryption only helps if the key belongs to the right party. On the first connection the browser has never seen `example.com`'s key, so it accepts whatever key shows up. An attacker sitting on the path can answer first with their own key. The browser then has a perfectly encrypted connection to the attacker.

The browser can't settle this alone. It needs a third party it already trusts to say which key belongs to `example.com`. That third party is the CA. Browsers and operating systems ship with a list of trusted CAs and their public keys. The CA issues the server a certificate that says "`example.com` uses public key K" and signs it with the CA's own private key. The browser checks that signature against the CA public key it already has. If the signature is valid, the browser accepts that K is `example.com`'s key. This is a second key pair, and it's used only to sign certs, never to encrypt traffic.

### Gap 2: how does the CA know?

The CA is only useful if it doesn't sign for just anyone. Before issuing, it has to confirm that the requester controls `example.com`. Keys can't help here, because nothing trusted links the requester's key to the domain yet. That link is exactly what the CA is about to create.

So the CA tests control of the domain directly. It asks the requester to publish a token in the domain's DNS or to serve it over HTTP on the domain (DNS-01 / HTTP-01, see 0012). Only whoever controls `example.com` can do that. If the check passes, the CA signs a cert binding `example.com` to the public key that came with the request. An attacker can't pass the check, so they can't get a cert for the domain.

### Gap 3: certificates are public

Any client that connects gets a copy of the certificate, attacker included. So showing a valid cert proves nothing about who you are. The attacker can just replay the real one.

The fix is that the cert only names a key, and the server must prove during the handshake that it holds the matching private key. The attacker has the cert but not the private key, so the handshake fails. They could use their own key, but then they'd need a cert naming their key for `example.com`. They can't get one, because they can't pass the domain check above. A self-signed cert doesn't work either, because it isn't in the browser's trust list.

So the two key pairs answer different questions. The CA's key answers "is it true that K belongs to `example.com`?" The server's key answers "is the party I'm talking to the owner of K?" Trust needs both answers.

### Side note: what actually encrypts the data

The server's key pair doesn't encrypt the traffic itself. It's used to authenticate the server and to agree on a fresh symmetric session key, and the data is encrypted with that session key.

## In practice

This is deliberately simplified. It leaves out intermediate CAs (the chain of trust), revocation, ACME account keys, the CSR signature, and how the key exchange works.

The same model maps onto both legs in 0013. On leg 1 the browser is the client and the Cloudflare edge is the server. On leg 2 Cloudflare is the client and the Istio Gateway is the server. With Origin CA, Cloudflare's trusted CA list holds only Cloudflare's own CA, so nobody else accepts those certs.

## References

- https://letsencrypt.org/how-it-works/
- https://www.rfc-editor.org/rfc/rfc8446 (TLS 1.3, CertificateVerify proves possession of the private key)
- https://www.rfc-editor.org/rfc/rfc8555 (ACME)
