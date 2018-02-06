# GRPC-TO-FPM
Small reverse-proxy which translates gRPC unary methods calls to FastCGI requests (primarily for PHP-FPM).

## Why
Unfortunately, PHP can't serve as a gRPC server. But
it can generate client code and request / response structures. In this way,
for processing simple RPC calls it is enough to transfer body of the message,
which it will decode, execute business logic and return a serialized response.

### Work scheme:
* (proxy) Accept a gRPC request;
* (proxy) Get the body from it, which is serialized Protobuf message;
* (proxy) Send FastCGI request to PHP-FPM with serialized request body;
* (php) Read the body and route(PATH) from incoming request;
* (php) Prepare the result;
* (php) Serialize it and send it back;
* (proxy) Read the response, prepare gRPC response and send it to the gRPC request issuer;

## Installation

### Requirements
* Go 1.8+
* PHP-FPM

### Run
- `go get -u github.com/LTD-Beget/grpc-to-fpm/cmd/grpc-to-fpm`
- `./grpc-to-fpm`

### Debug mode
For verbose logging, you can send `SIGUSR2` signal
to the process to enable debug mode.

## Usage

### Configuration
By default, configuration must be stored in "grpc-to-fpm.yml".

Example:
```
instancename: my-php-service              // Service name; Primarily for logs
host: ":50051"                            // Host for our 'gRPC' server
debug: false                              // Default debug mode

target:
    host: localhost                       // PHP-FPM address
    port: 9000                            // PHP-FPM port
    scriptpath: /home/myuser/app/handlers // Path of PHP script
    scriptname: index.php                 // Name of PHP script
    returnerror: true                     // Enable passing of PHP errors to the gRPC request issuer

// Optional. Graylog configuration
graylog:
  host: graylog.localhost
  port: 12201

// Optional. Key and certificate if you want to use TLS
keyFile: localhost.key
crtFile: localhost.pem
```
All of these configuration variables can be overriden with ENV variables.

For example: `CONFIGOR_KEYFILE=localhost.key`

### PHP script example
If our service definition looks like this:
```proto
syntax = 'proto3';

package api.customer;

service CustomerService {
    rpc getSomeInfo(GetSomeInfoRequest) returns (GetSomeInfoResponse) {}
}

message GetSomeInfoRequest {
    string login = 1;
}

message GetSomeInfoResponse {
    string first_name = 2;
    string second_name = 3;
}
```

then our php handler will look like this:
```php
<?php
// Full gRPC method name in format:
// package.service-name/method-name
$route = $_GET['r'];

// Protobuf-serialized request message body
$body = file_get_contents("php://input");

try {
    // We can throw some errors here...
    if (rand(0, 1)) {
        throw new \RuntimeException("Some error happened!");
    }

    // Use structures generated by protobuf plugin to decode request and encode response
    $request = new GetSomeInfoRequest;
    $request->parse($body);

    $customer = findCustomer($request->getLogin());

    $response = (new GetSomeInfoResponse)
        ->setFirstName($customer->getFirstName());

    // Send response to the client
    echo $response->serialize();
} catch (\Throwable $e) {
    // Valid gRPC status code
    // All statuscodes is available in:
    // https://github.com/grpc/grpc-go/blob/master/codes/codes.go
    $errorCode = 13;

    header("X-Grpc-Status: ERROR");
    header("X-Grpc-Error-Code: {$errorCode}");
    header("X-Grpc-Error-Description: {$e->getMessage()}");
}
```