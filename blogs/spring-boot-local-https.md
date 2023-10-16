[Home](https://ngbsn.github.io/)

# How to securely run you localhost Java application on Windows with HTTPS

## Overview
This article is intended for Java applications running on Windows as localhost and exposing REST APIs to be consumed by other applications locally.
I will show how to set up SSL for a simple Hello World Spring Boot application with the objective of achieving end to end security for the REST APIs.
You should be able to run your application securely with the URL https://localhost:8080/hello-world

## Table of Contents
- Create a sample Spring Boot Application 
- Generate a Root CA certificate and SSL Keystore 
- Install the Root CA certificate into Windows Trusted Root Certification Authorities Certificate Store
- Use the SSL Keystore in your Spring Boot Application 
- Store the Keystore password in Windows Credential Manager
- Run the Spring Boot Application as a Windows Service

## Create a sample Spring Boot Application
Let's create a Spring Boot Rest Controller for Hello World.

```java
@RestController
public class HelloWorldController {

    @GetMapping("/hello-world")
    public String helloWorld(){
        return "Hello World";
    }
}
```
## Generate a Root CA certificate and SSL Keystore
1. Generate a private key for the Custom Root CA
    ```
    openssl genrsa -des3 -out rootCA.key 4096
    ```
2. Create a rootCA.cnf file

    ```
    [root_ca]
    basicConstraints = critical, CA:TRUE
    keyUsage = critical, cRLSign, keyCertSign, Certificate Signing, Off-line CRL Signing, CRL Signing (06)
    extendedKeyUsage = Client Authentication, Server Authentication
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid:always,issuer:always
    ```

3. Create the root CA certificate
    ```
    openssl req -x509 -new -nodes -key rootCA.key -subj "//C=IN\ST=Karnataka\O=ngbsn\CN=ngbsn.github.io" -sha256 -days 1024 -config rootCA.cnf -out rootCA.crt
    ```
4. Generate a private key for SSL
    ```
    openssl genrsa -out localhost.key 2048
    ```

5. Generate a Certificate Signing Request (CSR)
    ```
    openssl req -new -sha256 -key localhost.key -subj "//C=IN\ST=Karnataka\O=ngbsn\CN=localhost" -out localhost.csr
    ```
6. You can output the contents of the CSR.
    ```
    openssl req -in localhost.csr -noout -text
    ```
7. Create an openssl.cnf file
    ```
    [ req ]
    prompt                 = no
    days                   = 365
    distinguished_name     = req_distinguished_name
    x509_extension         = v3_req
    
    [ req_distinguished_name ]
    countryName            = IN
    stateOrProvinceName    = Karnataka
    localityName           =
    organizationName       = ngbsn
    organizationalUnitName =
    commonName             = localhost
    emailAddress           = contact@ngbsn.github.io
    
    [ v3_req ]
    keyUsage			   = critical, digitalSignature, keyAgreement
    extendedKeyUsage       = serverAuth
    subjectAltName         = @sans
    
    [ sans ]
    DNS.0 = localhost
    ```
8. Generate an SSL certificate signed by the Custom Root CA
    ```
    openssl x509 -req -in localhost.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out localhost.crt -days 500 -sha256 -extfile openssl.cnf -extensions v3_req
    ```
9. Output the contents of SSL certificate
    ```
    openssl x509 -in localhost.crt -text -noout
    ```
10. Create an SSL keystore having the SSL Key and Certificate
    ```
    openssl pkcs12 -export -in localhost.crt -inkey localhost.key -out localhost.p12 -name localhost
    ```
    
## Import the Root CA certificate into Windows Trusted Root Certification Authorities Certificate Store
Run the below command on Command Prompt as Administrator.
```
certutil.exe -addstore root rootCA.crt
```

## Use the SSL keystore in your Spring Boot application Main class

```java
@SpringBootApplication
class HelloWorldApplication{
    public static void main(String[] args) {
        Properties props = new Properties();
        setSSLProps(props);
        new SpringApplicationBuilder(HelloWorldApplication.class)
                .properties(props).run(args);
    }

    private static void setSSLProps(Properties props) {
        props.put("server.ssl.enabled", true);
        props.put("server.ssl.key-store-type", "PKCS12");
        props.put("server.ssl.key-store", "classpath:keystore/localhost.p12");
        props.put("server.ssl.key-alias", "localhost");
        props.put("server.ssl.key-store-password", "<keystore_password>");
    }    
}

```
## Store the Keystore password in Windows Credential Manager

Instead of hardcoding the keystore password inside the Java class or in Spring Boot Configuration, there is a better and more secure way to manage the password and retrieve it using Windows Credential Manager.
Use this command to store the keystore password in Credential Manager.
```
cmdkey /generic:KEYSTORE_PASSWORD /user:ngbsn /pass:password
```
This stores the password in Windows Credential Manager using account of Logged-In user. The Java application has to be run as the Logged-In user to be able to access the Windows Credential.

Use this maven dependency.
```xml
<dependency>
    <groupId>com.microsoft</groupId>
    <artifactId>credential-secure-storage</artifactId>
    <version>1.0.0</version>
</dependency>
```

Let's change our main class to retrieve this password during application startup.
```java   
@SpringBootApplication
class HelloWorldApplication {
    
   private static final SecretStore<StoredCredential> credentialStorage =
           StorageProvider.getCredentialStorage(true, StorageProvider.SecureOption.REQUIRED);
   
    public static void main(String[] args) {
        Properties props = new Properties();
        setSSLProps(props);
        new SpringApplicationBuilder(HelloWorldApplication.class)
                .properties(props).run(args);
    }

    private static void setSSLProps(Properties props) {
        //read password from Credential Manager
        Optional<char[]> password = getPassword();
        if (password.isPresent()) {
            props.put("server.ssl.enabled", true);
            props.put("server.ssl.key-store-type", "PKCS12");
            props.put("server.ssl.key-store", "classpath:keystore/localhost.p12");
            props.put("server.ssl.key-alias", "localhost");
            String pass = String.valueOf(password.get());
            props.put("server.ssl.key-store-password", pass);
        }
    }

    private static Optional<char[]> getPassword() {
        try {
            StoredCredential storedCredential = credentialStorage.get("KEYSTORE_PASSWORD");
            return storedCredential != null ? Optional.of(storedCredential.getPassword()) : Optional.empty();
        } catch (Exception e) {
            return Optional.empty();
        }
    }
}
```
Setting up our Java Application this way ensures HTTPS for our APIs with the added security of keystore password protected by the Windows vault.
Further to this, having some kind of API authentication for the APIs ensures an end to end security.

## Run the Spring Boot Application as a Windows Service
If you want to set up the application as a Windows Service, you can use a service manager such as [NSSM](https://nssm.cc/).
If the Windows Service is logged on as "Local System Account", then the Windows Credential needs to be stored as "Local System account", as "Local System account" cannot access the Credential that was created using the previous Logged-In user.

To do this, download [PSTools](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec) from the Windows Official Site and run the below command to store the keystore password.

```
psexec -s /accepteula cmdkey /generic:KEYSTORE_PASSWORD /user:ngbsn /pass:password
```
psexec will run the cmdkey as "Local System account".

### This concludes our article!
