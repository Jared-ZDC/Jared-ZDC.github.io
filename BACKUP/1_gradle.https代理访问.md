# [gradle https代理访问](https://github.com/Jared-ZDC/Jared-ZDC.github.io/issues/1)

## 背景
由于公司的网络无法直接访问外网，在用gradle的时候，必须配置代理

## 代理配置
gradle的代理配置网上教程较多，这里基本上也是复制网上的基础配置，谨以此作为备案，以便后续查看
```properties
systemProp.http.auth.ntlm.domain=CHINA
systemProp.http.keepAlive=true
systemProp.http.proxyHost=your proxy host
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=your acount if needed
systemProp.http.proxyPassword=your password if needed
systemProp.http.nonProxyHosts=*.xxxx.com|localhost|127.0.0.1
systemProp.https.proxyHost=your proxy host
systemProp.https.proxyPort=8080
systemProp.https.proxyUser=your acount if needed
systemProp.https.proxyPassword=your password if needed
systemProp.https.nonProxyHosts=*.xxxx.com|localhost|127.0.0.1
```
在gradle工程中，最好在以下两个文件中，均添加以上配置：
* ./gradle.properties
* ./gradle/wrapper/gradle-wrapper.properties

## 证书导入
由于gradle的maven源经常是https协议的，这样会导致https的证书错误：
```bash
sun.security.validator.ValidatorException: PKIX path validation failed: java.security.cert.CertPathValidatorException: signature check failed
javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path validation failed: java.security.cert.CertPathValidatorException: signature check failed
```
因此需要根据build.gradle中访问的网址将需要的证书逐个导入到jdk中：
```bash
keytool -import -alias your_alias_name -keystore $JAVA_HOME/lib/security/cacerts  -file  your_cer_file
```
证书可以通过使用浏览器访问该网站，然后导出到文件中，编码选择base64编码即可

**这里需要注意，证书有时候是多级认证的，因此需要逐个导出，此外，根据工程访问的repo源不同，需要的证书有可能不一样，因此都需要逐个导出，并导入到编译环境中，否则无法访问相关文件**

最后在gradle.properties（gradle-wrapper.properties）文件中指明cacerts路径：
```properties
systemProp.javax.net.ssl.trustStore=/home/paas/jdk-11.0.2/lib/security/cacerts
systemProp.javax.net.ssl.trustStorePassword=changeit
```