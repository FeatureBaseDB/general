# Pilosa Error Codes Proposal

## Overview

This proposal aims to define how Pilosa should handle error responses to clients. It should allow:

* client libraries to process errors appropriately,
* developers and/or operators to locate the source of the error precisely.
* errors to be transport agnostic.

While not strictly the same, how we respond to clients will influence how we handle errors internally. 
The general Go convention is to return errors up the stack, adding context as necessary,
and handling the error only once.
Pilosa will be able to handle some errors internally,
and others will get returned all the way to the HTTP handler;
it is at this point that we need to determine how to respond to the client.

Dave Cheney is a longtime gopher who has done a lot of thinking (and dev) on
error handling in Go. This presentation
https://dave.cheney.net/paste/gocon-spring-2016.pdf sums things up quite well.

The TLDR; is to assert for behavior rather than a specific error or error type.
Basically, the client needs to know whether to retry, or throw an exception, and
so it needs to know whether the error is transient, or a permanent condition
that will require intervention from a developer or an operator.

* If a client sent a malformed request, that would require client dev attention (this would be HTTP 4XX).
* If the server encountered a panic, or some other bug, that would require server dev attention (HTTP 5XX).
* If the server encountered an environment error (permissions opening a file,
  out of mem, out of disk, etc.), that would require operator attention (also
  HTTP 5XX).
* If the server encountered a timeout while communicating with another host,
  that could be a transient condition, and the client may just want to retry.

So while there are many types of potential errors, for the purposes of client
code, it really just needs to distinguish between errors where a retry will fix
the problem, and errors which will require human intervention. If the error
requires human intervention, it should contain as much relevant information as
possible to allow it to be routed to the correct person, and then localized to
the exact code or component which is causing the issue.

This proposal will define a few broad categories of error messages which allow
client code to easily determine programmatically whether to retry or raise an
error. Once raised, the categories will further allow a human, or automated
system to route the error to the appropriate individual or set of individuals to
be handled. Each error will also include a message which will help to pinpoint
the exact location of the error by encoding information about the function call
stack which led to the error occuring.

## Categories

- Temporary Infrastructure Error (network timeout)
- Permanent Infrastructure Error (file permissions)
- Internal Error (panic, or other unexpected condition)
- Temporary Internal Error (too busy to service the request (not sure we'd ever use this))
- Client Error (malformed http request, invalid PQL)
- Unknown (catch-all, this should be considered a bug if used)

## Response
Example:
```json
{"category": 1, "short":"doing something: couldn't do it", "long":"doing something
couldn't do it
main.doing
	/Users/path/go/src/play/main.go:29
main.main
	/Users/path/go/src/play/main.go:33
runtime.main
	/usr/local/go/src/runtime/proc.go:183
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:2086
"}
```

## Implementation
Define a category interface and constants for errors:
```go
type ErrorCategory int

var (
    Unknown ErrorCategory = iota
    TemporaryInfrastructure ErrorCategory
    PermanentInfrastructure
    Internal
    TemporaryInternal
    Client
)

type HasCategory interface {
     func Category() ErrorCategory
}
```

Define a helper which adds a Category to an error, and uses Dave Cheney's
errors package to wrap the stacktrace and context message into the error.

```go
package pilosa 

import (
	"github.com/pkg/errors"
)

type perror struct {
	category ErrorCategory
    err error
}

func NewError(cat ErrorCategory, err error, msg string) error {
	if err == nil {
    	return nil
    }
	return perror{
    	category: cat,
        errors.Wrap(err, msg)
    }
}

func (e perror) Category() ErrorCategory {
	return e.category
}

func (e perror) Error() string {
	return e.err.Error()
}

func (e perror) Cause() error {
	return errors.Cause(e)
}

func (e perror) Format(f fmt.State, c rune) {
    e.err.(fmt.Formatter).Format(f, c)
}
```

At the handler level, an error returned can
  * be wrapped by the handler if it can determine the category.
  * already have been wrapped by something deeper (optimal).
  * neither - which should be considered a bug.
  
The handler will construct an error reply for any non-nil error as follows:
```go
type ErrResp struct {
    Category int
    Short string
    Long string
}

err := e.Execute(blah...)
if err != nil {
    return ErrResp{
        Category: GetCategory(err),
        Short: fmt.Sprintf("%v", err),
        Long: fmt.Sprintf("%+v", err),
    }
}

func GetCategory(err error) Category {
   if pe, ok := err.(HasCategory); ok {
       return pe.Category()
   } else {
       return Unknown
   }
}
```
