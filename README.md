# TLS-LibreSSL Example [![Build Status](https://dev.azure.com/lganzzzo/lganzzzo/_apis/build/status/oatpp.example-libressl?branchName=master)](https://dev.azure.com/lganzzzo/lganzzzo/_build?definitionId=13&branchName=master)

Example project of how-to use [oatpp-libressl](https://github.com/oatpp/oatpp-libressl) module. 
- Serve via HTTPS 
- Make client calls via HTTPS. 
- Using oatpp Async API.

See more:

- [Oat++ Website](https://oatpp.io/)
- [Oat++ Github Repository](https://github.com/oatpp/oatpp)
- [Get Started](https://oatpp.io/docs/start)

## Overview

This project is using [oatpp](https://github.com/oatpp/oatpp) and [oatpp-libressl](https://github.com/oatpp/oatpp-libressl) modules.

### Project layout

```
|- CMakeLists.txt                        // projects CMakeLists.txt
|- src/
|    |
|    |- controller/                      // Folder containing Controller where all endpoints are declared
|    |- client/                          // HTTP client is here. Used in "proxy" endpoint /api/get
|    |- dto/                             // DTOs are declared here
|    |- AppComponent.hpp                 // Service config
|    |- App.cpp                          // main() is here
|    
|- test/                                 // test folder
|- utility/install-oatpp-modules.sh      // utility script to install required oatpp-modules.
|- cert/                                 // folder with test certificates 
```

---

### Build and Run

#### Using Meson


```
$ meson setup build
$ meson compile -C build
$ ./build/example-ssl  # - run application.
```

#### In Docker

```
$ docker build -t example-ssl .
$ docker run -p 8443:8443 -t example-ssl
```

---

### Configure AppComponent

Configure server secure connection provider

```c++

/**
 *  Create ConnectionProvider component which listens on the port
 */
OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, serverConnectionProvider)([] {
  /* non_blocking connections should be used with AsyncHttpConnectionHandler for AsyncIO */
  auto config = oatpp::libressl::Config::createDefaultServerConfig("cert/test_key.pem", "cert/test_cert.crt");
  return oatpp::libressl::server::ConnectionProvider::createShared(config, 8443, true /* true for non_blocking */);
}());

```

Configure client secure connection provider

```c++
OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ClientConnectionProvider>, sslClientConnectionProvider) ([] {
  auto config = oatpp::libressl::Config::createShared();
  tls_config_insecure_noverifycert(config->getTLSConfig());
  tls_config_insecure_noverifyname(config->getTLSConfig());
  return oatpp::libressl::client::ConnectionProvider::createShared(config, "httpbin.org", 443);
}());
```

### Endpoints
---
"Hello Async" root endpoint with json response
```c++
ENDPOINT_ASYNC("GET", "/", Root) {

  ENDPOINT_ASYNC_INIT(Root)
  
  Action act() override {
    auto dto = HelloDto::createShared();
    dto->message = "Hello Async!";
    dto->server = Header::Value::SERVER;
    dto->userAgent = request->getHeader(Header::USER_AGENT);
    return _return(controller->createDtoResponse(Status::CODE_200, dto));
  }

};
```

result:
```
$ curl -X GET "https://localhost:8443/" --insecure
{"user-agent": "curl\/7.54.0", "message": "Hello Async!", "server": "oatpp\/0.19.1"}
```
---
Async proxy endpoint to ```https://httpbin.org/get```

```c++
ENDPOINT_ASYNC("GET", "/api/get", TestApiGet) {

  ENDPOINT_ASYNC_INIT(TestApiGet)

  Action act() override {
    return controller->myApiClient->apiGetAsync().callbackTo(&TestApiGet::onResponse);
  }

  Action onResponse(const std::shared_ptr<IncomingResponse>& response){
    return response->readBodyToStringAsync().callbackTo(&TestApiGet::returnResult);
  }

  Action returnResult(const oatpp::String& body) {
    return _return(controller->createResponse(Status::CODE_200, body));
  }

};
```

result:
```
$ curl -X GET "https://localhost:8443/api/get" --insecure
{
  "args": {}, 
  "headers": {
    "Connection": "close", 
    "Host": "httpbin.org"
  }, 
  "origin": "176.37.47.230", 
  "url": "https://httpbin.org/get"
}
```
