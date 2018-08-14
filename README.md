# Spore Client Java
A Java implementation of the [Spore Protocol] (Add link to the REST API here)


## Installation
### JAR
Download the [JAR][1] file and add it to your project's path.
You will also need to following dependencies:
- [JSON in Java][2]
- [Java JWT][3]

### Maven
(Publish artifact to Maven?)

## Known Spore Servers
(Add address of known servers)

## Usage
### Choose a server
Any of the servers above offers good quality entropy through the Spore Protocol. Once the server is choosen, simple pass it's address to the `SporeClient` constructor.
```JAVA
String serverAddress = "http://127.0.0.41:8099/eaasp/";
SporeClient sporeClient = new SporeClient(serverAddress);
```

### Testing the Connection
Once the `SporeClient` is instantiated, a simple way of testing the connection is to simply send a `getInfo` request. This will also give us basic information on the server, namely it's name and the quantity of entropy it serves.
```JAVA
JSONObject infoResponse = sporeClient.doInfoRequest();
int entropySize = infoResponse.getInt("entropySize");
```

### Requesting Entropy
Now that we know the connection is working, we can request entropy from the server. If we care about verifying the validity of the response, we should send a challenge, however, this is not required.
- With a challenge
```JAVA
SecureRandom rbg = new SecureRandom();
byte[] randomBytes = new byte[16];
rbg.nextBytes(randomBytes);
String challenge = Base64.getUrlEncoder().encodeToString(randomBytes);
JSONObject entropyResponse = sporeClient.doEntropyRequest(challenge);
```
Here we generate a random challenge. This is the best practice to avoid repeating the challenge. If lots of requests are expected to be sent, increasing the length of the challenge would decrease the probability of challenge collisions.

- Without a challenge
```JAVA
JSONObject entropyResponse = sporeClient.doEntropyRequest(null);
```

### Adding the Entropy to our System
Now that we received good quality entropy, we can seed our local source of randomness. It is best practice to first perform some hash to combine the received entropy with the freshly received entropy. Indeed, if one was to simply seed it local entropy source with compromised entropy, his system would become deterministic to the malicious party.

Here a simple `XOR` is aplied with localy generated bytes. However, one could also use a more cryptographically secure hash function such as `SHA256`.

```JAVA
String b64entropy = entropyResponse.getString("entropy");
byte[] entropy = Base64.getUrlDecoder().decode(b64entropy);
randomBytes = new byte[entropySize];
rbg.nextBytes(randomBytes);
for (int i = 0; i < entropySize; i++) {
	entropy[i] ^= randomBytes[i];
}
rbg.setSeed(entropy);
```

Many IoT devices with extremely limited resources would stop here, now having an entropy pool at worst similar to before the operation and at best of high quality. However, some devices with more resources or with tasks critically relying on cryptographic security will want to verify and authenticate the entropy.

### Verifying the Challenge
The first obvious step is to confirm that the entropy was indeed generated for our request, and not anyone else's. To do that, we can simply make sure the returned challenge matches the one we sent.
```JAVA
String receivedChallenge = entropyResponse.getString("challenge");
if (challenge.equals(receivedChallenge)) {
	System.out.println("Challenges match");
} else {
	throw new Exception("Challenges do not match");
}
```

### Verifying the Freshness
Another verification that can be made is to verify that the entropy sent is fresh. This can easily be done with the help of the returned timestamp.
```JAVA
long timestamp = entropyResponse.getLong("timestamp");
long localTime = System.currentTimeMillis() / 1000L;
long freshnessWindowSec = 60;

if (Math.abs(timestamp - localTime) <= freshnessWindowSec) {
	System.out.println("Entropy response is fresh");
} else {
	throw new Exception("Entropy response is not fresh");
}
```
Here we allow a one minute window.

### Authenticating the Response
Using the JWT received in the response, we can authenticate it using the server's public signature. If it is not known, it can be obtained with a `getCertChain` request.
```JAVA
JSONObject certResponse = sporeClient.doCertificateChainRequest();
byte[] certChain = certResponse.getString("certificateChain").getBytes();
InputStream is = new ByteArrayInputStream(certChain);
CertificateFactory cf = CertificateFactory.getInstance("X.509");
X509Certificate certificate = (X509Certificate) cf.generateCertificate(is);
PublicKey publicKey = certificate.getPublicKey();

String token = entropyResponse.getString("JWT");
Algorithm algorithm = Algorithm.RSA256((RSAPublicKey) publicKey, null);
JWTVerifier verifier = JWT.require(algorithm)
		.withClaim("challenge", challenge)
		.withClaim("timestamp", timestamp)
		.withClaim("entropy", b64entropy)
		.build();
verifier.verify(token);
```
It is also recommended to make sure the certificate chain's root is a trusted CA.

## Issue Reporting
If you found a bug, have a feature request, or a design recommendation, please open a new issue at this repository's [issues section][4]. If you find a security vulnerablility, please do not report it on the public GitHub issue tracker but instead contact [Crypto4a][5] directly.

## Author
[Crypto4a][6]



[1]: https://github.com/crypto4a/spore-client-java/raw/master/publish/SporeClient.jar
[2]: https://mvnrepository.com/artifact/org.json/json/20180130
[3]: https://mvnrepository.com/artifact/com.auth0/java-jwt/3.4.0
[4]: https://github.com/crypto4a/spore-client-java/issues
[5]: https://crypto4a.com/contact-crypto4a/
[6]: https://crypto4a.com/

