# 服务器时间误差导致的google sign-in后台验证错误（远程调试java程序）


```
https://developers.google.com/identity/sign-in/web/backend-auth
import com.google.api.client.googleapis.auth.oauth2.GoogleIdToken;
import com.google.api.client.googleapis.auth.oauth2.GoogleIdToken.Payload;
import com.google.api.client.googleapis.auth.oauth2.GoogleIdTokenVerifier;

...

GoogleIdTokenVerifier verifier = new GoogleIdTokenVerifier.Builder(new NetHttpTransport(), JacksonFactory.getDefaultInstance())
    .setAudience(Collections.singletonList(CLIENT_ID))
    // Or, if multiple clients access the backend:
    //.setAudience(Arrays.asList(CLIENT_ID_1, CLIENT_ID_2, CLIENT_ID_3))
    .build();

// (Receive idTokenString by HTTPS POST)

GoogleIdToken idToken = verifier.verify(idTokenString);
if (idToken != null) {
  Payload payload = idToken.getPayload();

  // Print user identifier
  String userId = payload.getSubject();
  System.out.println("User ID: " + userId);

  // Get profile information from payload
  String email = payload.getEmail();
  boolean emailVerified = Boolean.valueOf(payload.getEmailVerified());
  String name = (String) payload.get("name");
  String pictureUrl = (String) payload.get("picture");
  String locale = (String) payload.get("locale");
  String familyName = (String) payload.get("family_name");
  String givenName = (String) payload.get("given_name");

  // Use or store profile information
  // ...

} else {
  System.out.println("Invalid ID token.");
}
```
后台验证完全按照google的文档来，却总是输出Invalid ID token，服务器是在香港，本机VPN调试通过。

进行调试了，查找问题逐步配合debug查看源码

```
public GoogleIdToken verify(String idTokenString) throws GeneralSecurityException, IOException {
    GoogleIdToken idToken = GoogleIdToken.parse(getJsonFactory(), idTokenString);
    return verify(idToken) ? idToken : null;
  }
  public boolean verify(GoogleIdToken googleIdToken) throws GeneralSecurityException, IOException {
    // check the payload
    if (!super.verify(googleIdToken)) {//super.verify(googleIdToken) false
      return false;
    }
    // verify signature, try all public keys in turn.
    for (PublicKey publicKey : publicKeys.getPublicKeys()) {
      if (googleIdToken.verifySignature(publicKey)) {
        return true;
      }
    }
    return false;
  }
```
```
  public boolean verify(IdToken idToken) {
    return (issuers == null || idToken.verifyIssuer(issuers))//idToken.verifyIssuer(issuers) true
        && (audience == null || idToken.verifyAudience(audience))//idToken.verifyAudience(audience) true
        && idToken.verifyTime(clock.currentTimeMillis(), acceptableTimeSkewSeconds);//idToken.verifyTime(clock.currentTimeMillis(), acceptableTimeSkewSeconds) false
  }
```
```
  public final boolean verifyTime(long currentTimeMillis, long acceptableTimeSkewSeconds) {
    return verifyExpirationTime(currentTimeMillis, acceptableTimeSkewSeconds)//verifyExpirationTime(currentTimeMillis, acceptableTimeSkewSeconds) true
        && verifyIssuedAtTime(currentTimeMillis, acceptableTimeSkewSeconds);//verifyIssuedAtTime(currentTimeMillis, acceptableTimeSkewSeconds) false
  }
  public final boolean verifyIssuedAtTime(long currentTimeMillis, long acceptableTimeSkewSeconds) {
    return currentTimeMillis
        >= (getPayload().getIssuedAtTimeSeconds() - acceptableTimeSkewSeconds) * 1000;
  }
```

```
验证ok，数据返回如下：
GoogleIdToken{header={"alg":"RS256","kid":"ce799fff481gg9bca45ccab64eart658029bc0aa"}, payload={"aud":"769999627631-0sei0a5lmsdfrty3l104jikeu2scibdp.apps.googleusercontent.com","azp":"768367627631-mxdfr2c27msedftyh3u2u5pftgvuive4.apps.googleusercontent.com","email":"ert@gmail.com","email_verified":true,"exp":1510827887,"iat":1510824287,"iss":"https://accounts.google.com","sub":"999989845523222955671","name":"jdok gan","picture":"https://lh6.googleusercontent.com/-jrMAzrNnTp0/AAAEEEEAAAI/AAABBBAAAAA/ANQ0kffefwJueALvmVPb16kvoX1MU3J7A/s96-c/photo.jpg","given_name":"jdok","family_name":"xxx","locale":"zh-CN"}}
```
最后GoogleIdTokenVerifier验证后获得的json中获得iat和的exp值是相差1个小时，恶心的是服务器时间配置超过了exp的时间。