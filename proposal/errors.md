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

* If a client sent a malformed request, that would require client dev attention
  (this would be HTTP 4XX).
* If the server encountered a panic, or some other bug, that would require
  server dev attention (HTTP 5XX).
* If the server encountered an environment error (permissions opening a file,
  out of mem, out of disk, etc.), that would require operator attention (also
  HTTP 5XX).
* If the server encountered a timeout while communicating with another host,
  that could be a transient condition, and the client may just want to retry.
  
There are some cases, however, where a client may want to match against a
specific error, such as if the Database does not exist, or if it already exists
on a CreateDatabase call, or if a frame does not exist, or already exists. In
these cases it will be useful for each error to have a unique code.

So while there are many types of potential errors, for the purposes of client
code, it really just needs to distinguish between errors where a retry will fix
the problem, errors which will require human intervention, and a few specific
errors. If the error requires human intervention, it should contain as much
relevant information as possible to allow it to be routed to the correct person,
and then localized to the exact code or component which is causing the issue.

This proposal will define a few broad categories of error messages which allow
client code to easily determine programmatically whether to retry or raise an
error. Once raised, the categories will further allow a human, or automated
system to route the error to the appropriate individual or set of individuals to
be handled. Each error will also include a message which will help to pinpoint
the exact location of the error by encoding information about the function call
stack which led to the error occuring.

## Error Codes

### Structure

A 5 character upper case alphanumeric ascii string [0-9A-Z]{5}.

* The first character is the component - 'S' for Server, 'P' for Plugin. '0' is
  reserved for success, and 'U' for unknown.
* The second two characters are the category - see the categories section below.
* The last 2 characters are the unique error code.

### Categories

Several categories are defined - they are meant to be mnemonic. 'O' means
operational, 'D' means developer, 'T' for temporary, 'P' for permanent, 'C' for
client, 'U' for unknown.

* OT - Temporary Operational/Infrastructure Error (network timeout) 
* OP - Permanent Operational/Infrastructure Error (file permissions)
* DT - Temporary Internal Error (too busy to service the request (not sure we'd ever use this))
* DP - Internal Error (panic, or other unexpected condition)
* CE - Client Error (malformed http request, invalid PQL)
* UU - Unknown (catch-all, this should be considered a bug if used)
* 00 - Reserved for indicating success if an error field is required for some reason.

Other categories may be added over time - if possible they should follow the
mnemonic conventions.

### Unique Code
The specific taxonomy of unique codes within each category will be left up to
the implementation, however two special codes are defined similarly to
categories. 

* "UU" is the unknown code which is used in the case of an error received by the
  handler which does not have a code.
* "00" Reserved for indicating success if an error field is required for some
  reason.

## Response

Example:
```json
{
  "code": "SIP23"", 
  "message":"doing something: couldn't do it",
  "detail":"doing something
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
Create a new package "github.com/pilosa/pilosa/errors"

Define an unexported code type with exported methods.
```go
package errors

type code struct {
	component string
	category  string
	id        string
}

func (c code) Component() string { return c.component }
func (c code) Category() string  { return c.category }
func (c code) ID() string        { return c.id }
func (c code) Code() string      { return c.Component() + c.Category() + c.ID() }
```

We then define all error codes within this package (they can't be defined elsewhere since `code` is not exported):
```go
var (
	// special codes
	Unknown code = build("UUUUU")
	Success      = build("00000")

	// client errors
	DatabaseDoesNotExist  = build("SCED1")
	DatabaseAlreadyExists = build("SCED2")
	FrameDoesNotExist     = build("SCEF1")
	FrameAlreadyExists    = build("SCEF2")

	// TODO define other errors
)
```

`build` is just a helper function which makes sure that all codes are valid at startup.
```go
func build(cd string) code {
	validate(cd)
	return code{
		component: cd[:1],
		category:  cd[1:3],
		id:        cd[3:5],
	}
}

var codeRegexp, _ = regexp.Compile("^[0-9A-Z]{5}$")

func validate(cd string) {
	if !codeRegexp.MatchString(cd) {
		panic(fmt.Sprintf("Tried to construct invalid error code: '%s'", cd))
	}
}
```

Now we create a library that is a thin wrapper around Dave Cheney's `errors`
library. It has all the same exported functions, except that each one takes a
`pilosa/errrors.code` as its first argument, and keeps that information along
with all the information that `pkg/errors` library adds:

```go

type pilosaError struct {
	code
	err error
}

func (p pilosaError) Error() string {
	return fmt.Sprintf("code: '%s' msg: %v", p.Code(), p.err)
}

func (e pilosaError) Format(f fmt.State, c rune) {
	e.err.(fmt.Formatter).Format(f, c)
}

func Cause(err error) error {
	if perr, ok := err.(pilosaError); ok {
		return errors.Cause(perr)
	} else {
		return errors.Cause(err)
	}
}

func Wrap(c code, err error, msg string) error {
	return pilosaError{
		code: c,
		err:  errors.Wrap(err, msg),
	}
} 

// Errorf, New, Wrapf,...etc.
```

We further provide a `Code(err error) string` function which gets an error code
from any error (returning the unknown code if it is not a pilosaError). This is
necessary since pilosaError is not an exported type. For convenience we also
provide functions to get the short message and stack trace.

```go
func Code(err error) string {
	if perr, ok := err.(pilosaError); ok {
		return perr.Code()
	}
	return Unknown.Code()
}

func Message(err error) string { return fmt.Sprintf("%v", err) }
func Detail(err error) string  { return fmt.Sprintf("%+v", err) }
```

At the handler level, an error returned can
  * be wrapped by the handler if it can determine what error it is.
  * already have been wrapped by something deeper (optimal).
  * neither - which should be considered a bug.
  
The handler will construct an error reply for any non-nil error as follows:
```go
type ErrResp struct {
    Code string
    Message string
    Detail string
}

err := e.Execute(blah...)
if err != nil {
    return ErrResp{
        Code: errors.Code(err)
        Short: errors.Message(err)
        Detail: errors.Detail(err)
    }
}
```

In general pilosa code, when an error condition is encountered, the code just needs to call one of the `errors` functions and pass it the appropriate code.
```go
db := getDB()
if db == nil {
    return errors.New(errors.DatabaseDoesNotExist, "couldn't find a database")
}

// OR

f, err := os.OpenFile(somename)
if err != nil {
    return errors.Wrapf(errors.UnableToLoadData, err, "loading data from %v", somename)
}

```
