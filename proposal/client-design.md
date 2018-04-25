# Pilosa Client Library Design v1

## Introduction

This document contains the interface specification for client libraries for Pilosa v1.0 stable release. The interface described in this document only contains the common features which are expected from all client libraries and does not cover the full feature set of the Pilosa server.

## Conventions

The following conventions were used in this document:

- `name(argument1: type1, argument2: type2)`: Function or method that has two arguments and has no meaningful return value. `argument1` and `argument2` are both required.
- `name(argument1: type1, argument2?: type2)`: Function or method that has two arguments and has no meaningful return value. `argument1` is required, `argument2` is optional.
- `name() -> return_type`: Function or method that has no arguments with the given return type.
- `constructor name() -> return_type`: Function or method that creates an object of type `return_type`.
- `attribute name: type`: An attribute which can be read and written. The implementation varies between languages: some languages have the ability to define them directly, some languages would have getter and setter methods, yet others may simply use variables.
- `attribute get name: type`: An attribute which can only be read.
- `any`: Any type. For instance, it is `interface{}` in the Go client, and `Object` in the Java client.
- `int64`: Pilosa key type. It should ideally be 64bit integer, but if the language has no support for that, the client should use the biggest numeric type. For instance, Javascript supports a single 64bit floating point type, which can represent integers up to 53bits.
- `TimeStamp`: a type which contains both date and time.
- `Map<string, any>`: Map (or *dictionary*) with a string key and any type as the value type.
- `List<int64>`: List of 64bit unsigned integers.
- `serialize_type`: The type to serialize a PQL query into.
- `enum name`: An enumerated value. Some languages may define it as enumerated values or simply constants.

## Versioning

Client libraries have versions in the following format: `${MAJOR}.${MINOR}.${PATCH}`:
- Major: Corresponds to the highest supported Pilosa major version.
- Minor: Corresponds to the highest supported client feature level.
- Patch: Revision number, specific to the client library.

The scheme ensures that:
- Clients which have the same major version `A.x.x` can talk to Pilosa version `A` and below.
- Client `A.B.x` implements the same features and interface as client version `A.B.y`.
- It's safe for the users to upgrade from `A.B.1` to `A.B.2` without modifying their code.

This section will be modified when versioning specification of Pilosa server is complete.

## API

### Core

#### URI

The Pilosa URI is made up of three parts: scheme, host and port:
- The scheme is in the format: `$TRANSPORT[+$CONTENT_TYPE]`.Content type provides a hint to the client on how the request should be encoded and the response should be decoded, like `http+pb` would mean protobuf content over http. The default is `http`.
- The host is the domain name, host name, ipv4 or ipv6 address for a Pilosa host. The default is `localhost`.
- The port is the port of the Pilosa server. It is `10101` by default.

Pilosa URIs support a string form with the format: `$SCHEME://$HOST:$PORT`. All parts of that format are optional, but at least one part should be specified. The following are equivalent:
- http://localhost:10101
- http://localhost
- http://:10101
- localhost:10101
- localhost
- :10101
 
`URI` class supports following methods:
- `constructor defaultURI() -> URI`: Creates a new URI with the default address (`http://localhost:10101`) and options.
- `constructor fromAddress(address: string) -> URI`: Creates a new URI with the given string address and default options.
- `constructor fromHostPort(host: string, port?: int) -> URI`: Creates a new URI with the given host and port. Equivalent to: `URI.fromAddress("$HOST:$PORT")`.
- `attribute get scheme: string`: Returns the raw scheme.
- `attribute get host: string`: Returns the host.
- `attribute get port: int`: Returns the port.

#### Cluster

The `Cluster` class keeps the addresses of the hosts in the cluster.

- `constructor defaultCluster() -> Cluster`: Creates the default cluster with no hosts.
- `constructor withHost(uri: URI) -> Cluster`: Creates a cluster with a single host.
- `attribute get hosts: List(URI)`: Returns all hosts in the cluster.
- `addHost(uri: URI)`: Adds a host with the given URI to the cluster.
- `getHost() -> URI`: Returns the next host in the cluster. It's up to the cluster object to choose which host to return; currently the hosts are selected in a round robin fashion. If there are no hosts in the cluster, an exception is thrown.
- `removeHost(uri: URI)`: This method is used to remove bad hosts (down or consistenly returning connection errors) from the cluster object. Depending on the cluster object, the host may be removed for good, or it may be banned (unselectable by `getHost`) for a certain time.

#### Client

The Client class runs requests against a Pilosa server.

For HTTP connections, `Client` objects send a `User-Agent` header to the server, with the following format: `${LANGUAGE}-pilosa/${VERSION}`. Optionally, the clients may include auxiliary information in the header, with fields separated by semicolons (`;`). For instance, `java-pilosa/1.0.0;2017-01-01 15:16:17;Linux x86_64`.

Note that, functions of this class uses `Index` and `Frame` objects to receive index and frame values. The reason for that is twofold:
- Index and frame names should be validated. Passing those as raw strings requires validating them for each call.
- Index and frames have options, such as `columnLabel` and `rawLabel` besides their names. Using `Index` and `Frame` objects makes it more concise to pass these values.

- `constructor defaultClient() -> Client`: Creates a client with the default address and options.
- `constructor withURI(uri: URI) -> Client`: Creates a client with the given URI and default options.
- `constructor withCluster(cluster: Cluster, options?: ClientOptions) -> Client`: Creates a client with the given cluster and options.
- `query(query: PQLQuery, options?: QueryOptions) -> QueryResponse`: Runs a query with the given PQL query object and given query options. Pass `null` for default options.
- `schema() -> Schema`: Returns the schema (indexes, frames and fields) on the server.
- `syncSchema(schema: Schema)`: Ensures the indexes, frames and fields in the schema exists on the server side; loads indexes, frames and fields from the server. This method does not delete any index, frame or fields on the server side.
- `deleteIndex(index: Index)`: Deletes an index.
- `deleteFrame(frame: Frame)`: Deletes a frame.

#### ClientOptions (optional)

In order to customize how the `Client` works, users can pass a `ClientOptions` object. Since the underlying transport client of `Client` objects may have different capabilities, all of the attributes below are optional.

- *optional* `attribute socketTimeout: int`: The server should respond in the given time.
- *optional* `attribute connectTimeout: int`: The connection to the server should be established in the given time.
- *optional* `attribute retryCount: int`: After trying to operation with the given count, an error will be returned.
- *optional* `attribute connectionPoolSizePerRoute: int`: Number of connections in the pool for a host.
- *optional* `attribute connectionPoolTotalSize: int`: Number of total connections in the pool.

#### QueryOptions

It is possible to affect the response from the Pilosa server using query options.

- `attribute columns: bool`: Set to `true` to retrieve columns in the response for a query.
- `attribute slices: List<int64>`: List of slices to run the given query. It defaults to all slices if not specified.
- `attribute excludeAttrs: bool`: If true, attributes are excluded from the response.
- `attribute excludeBits: bool`: If true, bits are excluded from the response.

#### Validator

This class collects validation methods for various user input. In the Go client library, all of the following are individual functions:

- `validIndexName(name: string) -> bool`: Returns `true` if the given index name is valid, otherwise `false`. Valid names are between 1 and 64 in length and matches against the following regexp: `^[a-z][a-z0-9_-]*$`.
- `validFrameName(name: string) -> bool`: Returns `true` if the given frame name is valid, otherwise `false`. Valid names are between 1 and 64 in length and matches against the following regexp: `^[a-z][a-z0-9_-]*$`.
- `validLabel(label: string) -> bool`: Returns `true` if the given label is valid, otherwise `false`. Valid labels are between 1 and 64 in length and matches against the following regexp: `^[a-zA-Z][a-zA-Z0-9_-]*$`.

### Response

#### QueryResponse

`QueryResponse` is returned from a `Client.query` call.

- `attribute get success: bool`: `true` if the request was successful, otherwise `false`.
- `attribute get errorMessage: string`: If the request wasn't successful, the attribute contains the error message, otherwise the empty string (`""`).
- `attribute get results: List<QueryResult>`: All results in the response. If the request wasn't successful, it contains the empty list.
- `attribute get columns: List<ColumnItem>`: All columns in the response. If the request wasn't successful, or `QueryOptions.retrieveColumns` is `false`, it contains the empty list.
- *optional* `attribute get result: QueryResult`: Returns the first result if there are any otherwise `null`.
- *optional* `attribute get column: ColumnItem`: Returns the first column if there are any otherwise `null`.

Guaranteeing the return of a valid list object from `results` and `columns` attributes makes it more concise and less error-prone to write loops:
```
foreach (result in response.results) {
    // do something with the result
}
```

`result` and `column` attributes may be `null`, which makes it more concise to check the existence of results and columns for languages which consider `null` as a false value:
```
if (response.column) {
    // do something with response.column
}
```

#### QueryResult

- `attribute get changed: bool`: Returned from `SetBit` and `ClearBit` queries.
- `attribute get bitmap: BitmapResult`: Returned from bitmap queries, such as `Bitmap`, `Union`, etc.
- `attribute get countItems: List<CountResultItem>`: Returned from `TopN` queries.
- `attribute get count: int64`: Returned from `Count`, `Sum`, `Min` and `Max` queries.
- `attribute get value: int64`: Returnd from `Sum`, `Min` and `Max` queries.

#### ColumnItem

- `attribute get id: int64`
- `attribute get attributes: Map<string, any>`

#### BitmapResult

- `attribute get bits: List<int64>`
- `attribute get attributes: Map<string, any>`

#### CountResultItem

- `attribute get id: int64`: Returns row ID
- `attribute get count: int64`

### ORM

#### PQLQuery

- `attribute get index: Index`
- `serialize() -> serialize_type`
- *optional* `attribute get error: Error`

#### PQLBaseQuery <- PQLQuery

Implements `PQLQuery`.

#### PQLBitmapQuery <- PQLQuery

ORM methods which create calls with bitmap queries (such as `Bitmap`, `Union`) should have the `PQLBitmapQuery` return type.

Implements `PQLQuery`.

#### PQLBatchQuery <- PQLQuery

Implements `PQLQuery`.

- `add(query: PQLQuery)`

#### Schema

- `index(name: string) -> Index`: Returns the index with the given name.
- `attribute get indexes: List<Index>`: Returns the list of all indexes.

#### Index

- `attribute get name: string`
- `attribute get frames: List<Frame>`: Returns the list of all frames of this index.
- `frame(name: string, options?: FrameOptions) -> Frame`: Returns the Frame with the given name and options. The `options` argument is not used if the Frame already exists.
- `batchQuery() -> PQLBatchQuery`
- `union(queries: List<PQLBitmapQuery>) -> PQLBitmapQuery`: len(queries) >= 0
- `intersect(queries: List<PQLBitmapQuery>) -> PQLBitmapQuery`: len(queries) >= 1
- `difference(queries: List<PQLBitmapQuery>) -> PQLBitmapQuery`: len(queries) >= 1
- `xor(queries: List<PQLBitmapQuery>) -> PQLBitmapQuery`: len(queries) >= 1
- `count(query: PQLBitmapQuery) -> PQLBaseQuery`
- `setColumnAttributes(id: int64, attributes: Map<string, any>) -> PQLBaseQuery`
- `rawQuery(query: string) -> PQLBaseQuery`: Creates a query with the given string.

#### Frame

- `attribute name: string`
- `attribute options: string`
- `bitmap(rowID: int64) -> PQLBitmapQuery`
- `setBit(rowID: int64, columnID: int64) -> PQLBaseQuery`
- `clearBit(rowID: int64, columnID: int64) -> PQLBaseQuery`
- `topN(n: int64, bitmap?: PQLBitmapQuery) -> PQLBitmapQuery`
- `filterTopN(n: int64, bitmap: PQLBitmapQuery, field: string, values: List<any>) -> PQLBitmapQuery`
- `range(rowID: int64, start: TimeStamp, end: TimeStamp) -> PQLBitmapQuery`
- `setRowAttributes(rowID: int64, attributes: Map<string, any>) -> PQLBaseQuery`

#### FrameOptions (optional)

- `constructor defaultOptions() -> FrameOptions`
- `attribute timeQuantum: TimeQuantum`
- `attribute cacheType: CacheType`
- `attribute cacheSize: unsigned int`
- `attribute fields: List<RangeField>`

#### RangeField
- `LessThan(n: int64) -> PQLBitmapQuery`
- `LessThanOrEqual(n: int64) -> PQLBitmapQuery`
- `GreaterThan(n: int64) -> PQLBitmapQuery`
- `GreaterThanOrEqual(n: int64) -> PQLBitmapQuery`
- `Equals(n: int64) -> PQLBitmapQuery`
- `NotEquals(n: int64) -> PQLBitmapQuery`
- `NotNull() -> PQLBitmapQuery`
- `Between(a: int64, b: int64) -> PQLBitmapQuery`
- `Sum(bitmap: PQLBitmapQuery) -> PQLQuery`
- `Min(bitmap: PQLBitmapQuery) -> PQLQuery`
- `Max(bitmap: PQLBitmapQuery) -> PQLQuery`
- `SetIntValue(value: int64) -> PQLQuery`

#### TimeQuantum

- `enum NONE`: Equivalent to `""`.
- `enum YEAR`: Equivalent to `"Y"`.
- `enum MONTH`: Equivalent to `"M"`.
- `enum DAY`: Equivalent to `"D"`.
- `enum HOUR`: Equivalent to `"H"`.
- `enum YEAR_MONTH`: Equivalent to `"YM"`.
- `enum MONTH_DAY`: Equivalent to `"MD"`.
- `enum DAY_HOUR`: Equivalent to `"DH"`.
- `enum YEAR_MONTH_DAY`: Equivalent to `"YMD"`.
- `enum MONTH_DAY_HOUR`: Equivalent to `"MDH"`.
- `enum YEAR_MONTH_DAY_HOUR`: Equivalent to `"YMDH"`.

#### CacheType

- `enum DEFAULT`: Equivalent to `""`.
- `enum LRU`: Equivalent to `"lru"`.
- `enum RANKED`: Equivalent to `"ranked"`.

## Samples

### URI: constructor defaultURI() -> URI

#### Java

```java
URI uri = URI.defaultURI();
```

#### Go

```go
uri := pilosa.DefaultURI()
```

#### Python

```python
uri = URI()
```

#### Typescript

```typescript
const uri = new URI();
```

### URI: constructor fromAddress(address: string) -> URI

#### Java

```java
URI uri = URI.address("http://db1.pilosa.com:5555");
```

#### Go

```go
uri, err := pilosa.NewURIFromAddress("http://db1.pilosa.com:5555")
```

#### Python

```python
uri = URI.address("http://db1.pilosa.com:5555")
```

#### Typescript

```typescript
const uri = URI.address("http://db1.pilosa.com:5555");
```

### URI: constructor fromHostPort(host: string, port?: int) -> URI

#### Java

```java
URI uri = URI.fromHostPort("db1.pilosa.com", 5555);
```

#### Go

```go
uri := pilosa.NewURIFromHostPort("db1.pilosa.com", 5555)
```

#### Python

```python
uri = URI(host="db1.pilosa.com", port=5555)
```

#### Typescript

```typescript
const uri = new URI({host: "db1.pilosa.com", port: 5555});
```

### URI: attribute get scheme: string

#### Java

```java
String scheme = uri.getScheme();
```

#### Go

```go
scheme := uri.Scheme()
```

#### Python

```python
scheme = uri.scheme
```

#### Typescript

```typescript
const scheme = uri.scheme;
```

### Cluster: constructor defaultCluster() -> Cluster

#### Java

```java
Cluster cluster = Cluster.defaultCluster();
```

#### Go

```go
cluster := pilosa.DefaultCluster()
```

#### Python

```python
cluster = Cluster()
```

#### Typescript

```typescript
const cluster = new Cluster();
```

### Cluster: constructor withHost(uri: URI) -> Cluster

#### Java

```java
Cluster cluster = Cluster.withHost(uri);
```

#### Go

```go
cluster := pilosa.NewClusterWithHost(uri)
```

#### Python

```python
cluster = Cluster(uri)
```

#### Typescript

```typescript
const cluster = new Cluster(uri);
```

### Cluster: getHost() -> URI

#### Java

```java
URI host = cluster.getHost();
```

#### Go

```go
host := cluster.Host()
```

#### Python

```python
host = cluster.get_host()
```

#### Typescript

```typescript
const cluster = cluster.getHost();
```

### Client: constructor defaultClient() -> Client

#### Java

```java
PilosaClient client = PilosaClient.defaultClient();
```

#### Go

```go
client := pilosa.DefaultClient()
```

#### Python

```python
client = Client()
```

#### Typescript

```typescript
const client = new Client();
```

### Client: constructor withURI(uri: URI) -> Client

#### Java

```java
PilosaClient client = PilosaClient.withURI(uri);
```

#### Go

```go
client := pilosa.NewClientWithURI(uri)
```

#### Python

```python
client = Client(uri)
```

#### Typescript

```typescript
const client = new Client(uri);
```

### Client: constructor withCluster(cluster: Cluster, options?: ClientOptions) -> Client

#### Java

```java
PilosaClient client1 = PilosaClient.withCluster(cluster);

ClientOptions options = ClientOptions.builder()
    .setSocketTimeout(1000)
    .setConnectTimeout(2000)
    .build();
PilosaClient client2 = PilosaClient.withCluster(cluster, options);
```

#### Go

```go
client1 := pilosa.NewClientWithCluster(cluster, nil)
client2 := pilosa.NewClientWithCluster(cluster, pilosa.SocketTimeout(1000, pilosa.ConnectTimeout(2000)))
```

#### Python

```python
client1 = Client(cluster)

client2 = Client(cluster, socket_timeout=1000, connect_timeout=2000)
```

#### Typescript

```typescript
const client1 = new Client(cluster);

const client2 = new Client(cluster, {socket_timeout: 1000, connect_timeout: 2000});
```

### Client: query(query: PQLQuery, options?: QueryOptions) -> QueryResponse

#### Java

```java
QueryResponse response1 = client.query(frame.bitmap(1));

QueryOptions options = QueryOptions.builder()
    .setColumns(true)
    .build();
QueryResponse response1 = client.query(frame.bitmap(1), options);
```

#### Go

```go
response1, err := client.Query(frame.Bitmap(1), nil)
response2, err := client.Query(frame.Bitmap(1), pilosa.Columns(true))
```

#### Python

```python
response1 = client.query(frame.bitmap(1))

response2 = client.query(frame.bitmap(1), columns=True)
```

#### Typescript

```typescript
client.query(frame.bitmap(1)).then(response1 => {
    // act on the response
}).catch(err => {
    // deal with the error
});

client.query(frame.bitmap(1), {columns: true}).then(response2 => {
    // act on the response
}).catch(err => {
    // deal with the error
});
```
