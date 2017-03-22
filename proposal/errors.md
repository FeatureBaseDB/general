# Pilosa Error Codes Proposal

**Author**

* Yuce Tekol: yuce@pilosa.com

**Change Log**

* 2017-03-01: First draft

## Overview

This proposal aims to define a system of error responses for the Pilosa server which enables:

* client libraries to process errors appropriately,
* developers to locate the source of the error precisely.
* conducting errors to be propagated using transports other than HTTP.

## Current State

Currently, all errors are returned as HTTP status codes with an attached JSON object containing and `error: "MESSAGE"` field. This approach has the following shortcomings:

* HTTP status codes are not granular enough for client side error processing,
* It becomes harder to locate the source of the error when status codes and error messages are reused in many places in the code,
* Pilosa server maybe extended to serve other transport protocols in the future.

## Proposal

If a request results in failure, an error object is returned with the following fields:

* **code**: Error code (32 bit unsigned/signed integer)
* **message**: Text (utf-8 string)

If the response is in JSON, the response includes an `error` field which contains the error object above. If there are no errors, the `error` field is left out.

Other content types should encode the error object appropriately.

### Error Codes

The structure of error codes is summarized below:

```
All 0s: Success

0 R R R U U U U | C C C C C C C C | I I I I I I I I | I I I I I I I I
                                    ID
                  Category ID
        Unit
        0 0 0 0 - reserved
        0 0 0 1 - (0x1) server
        1 1 1 0 - (0xE) external/plugin
```

An error code is a 32 bits integer which maybe signed or unsigned depending on the programming language used. The bits are split into three parts:

* Byte 1: The most significant bit is always `0` in order to overcome possible integer sign problems. Next 3 bits are reserved. That part is followed by the *unit*, showing which component produced the error. Currently it is one of *server* (`0b0001`) or *plugin* (`0b1110`).

* Byte 2: *category* contains the category ID for the error. The category ID is used by clients to act with the correct action (like throwing the appropriate exception or retrying the request later).

* Bytes 3, 4: Unstructured sequential integer ID. This ID is unique per category.

It is assumed that an error code is used exactly once in the server codebase. This uniqueness property makes pinpointing source of errors easier.

### Client to Server Errors

These errors happen because of client mistakes, such as sending invalid queries or failing to supply proper credentials.

#### Errors During Reading a Request

In this stage, the request is not read fully or it wasn't vallidated.

##### TLS Certificate Verification Error

The server requires TLS verification, but the client did not provide one or provided an invalid one.

##### Usage Limit Exceeded

The client was forbidden to perform this action, since it exceeded the pre-defined usage limit for this action.

##### Request Read Error

Problem reading the request.

##### Request Decoding Error

The request was read but was not decoded.

##### Permission denied

The server requires authentication but the client failed to provide the correct credentials or the credentials is not adequate to perform this action.

##### Invalid argument

The request contains an invalid argument.

#### Errors During Request Validation

The request was read but its content is problematic.

##### Validation error

The database name or frame name did not pass validation.

##### Resource does not exist

The given database or frame does not exist.

###### Database does not exist

###### Frame does not exist

##### PQL Error

There is a problem with the given PQL.

###### PQL Syntax Error

###### Unknown PQL call

###### Invalid PQL argument

The query is syntactically correct, but it contains an invalid argument; such as an invalid timestamp.

#### Bad Input (Generic client error)

A catch all category for client to server errors.

### Server to Client Errors

The server fails to deliver the required response.

#### During Producing a Result

In this stage, the server starts producing the result, which involves interacting with files or calling other nodes.

##### Timeout

The server did not complete the required action in the predefined timeout.

##### I/O Error

An I/O error prevented completion of the required action.

###### Open Error

###### Write Error

###### Read Error

###### Delete Error

###### Close Error

##### Too Many Connections

##### Temporary Maintenance

The server is in maintenance, the client can check back later.

#### Errors During Creating a Response

The server produced a result but couldn't convert the result to the response.

##### Response Encoding Error

Failure during encoding of the result.

#### Generic server error

A catch all category for server to client errors.

## Implementation

In order to keep the error code, a new error type should be defined. Below is an example:

```go
type PilosaError struct {
	Code    uint32 `json:code`
	Message string `json:message`
}

func NewPilosaError(code uint32, message string) *PilosaError {
	return &PilosaError{
		Code:    code,
		Message: message,
	}
}

func (err *PilosaError) Error() string {
	return fmt.Sprintf("%s (0x%08x)", err.Message, err.Code)
}

func (err *PilosaError) Json() ([]byte, error) {
	return json.Marshal(err)
}
```

The error codes are kept in a separate module with their raw values, in order to make them easier to search:

```go
// codes.go

const (
	CodeServerError                     = 0x01000000
    // ...
	CodeRequestReadError                = 0x01100000
    // ...
	CodeTLSCertificateVerificationError = 0x01100001
)
```

One downside of this approach is, it maybe tedious to write all the error codes.

Here is an example which uses the `PilosaError` struct defined above and a sample error code:

```go
func checkClientCertificate(certificate *Certificate) error {
    if err := certificate.CheckSignature(...); err != nil {
        return NewPilosaError(CodeTLSCertificateVerificationError, err.Error())
    }
}
```

When the server returns `CodeTLSCertificateVerificationError`, the client may throw `CertificateValidationException` or `RequestException` depending on how granular its exceptions are. The thrown exception would contain the error code `0x01100001`, which can be used to find the matching error code constant `CodeTLSCertificateVerificationError`, then the exact location where the error code constant was used which in turn is the source of the error.
