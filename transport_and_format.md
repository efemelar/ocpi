# Transport and format

## JSON / HTTP implementation guide

The OCPI protocol is based on HTTP and uses the JSON format. It follows a RESTful architecture for webservices where possible.

### Security and authentication

The interfaces are protected on HTTP transport level, with SSL and token based authentication. Please note that this mechanism does **not** require client side certificates for authentication, only server side certificates in order to setup a secure SSL connection.

### Request format

Each HTTP request must add a 'Authorization' header. The header looks as following:

```
    Authorization: Token IpbJOXxkxOAuKR92z0nEcmVF3Qw09VG7I7d/WCg0koM=
```

The literal 'Token' indicates that the token based authentication mechanism is used. Its parameter is a string consisting of printable, non-whitespace ASCII characters. The token must uniquely identify the requesting party. The server can then use this to link data and commands to this party's account.

The request method can be any of [GET](#get), [PUT](#put), [PATCH](#patch) or DELETE. The OCPI protocol uses them in a similar way as REST APIs do.

<div><!-- ---------------------------------------------------------------------------- --></div>
| Method          | Description                                        |
|-----------------|----------------------------------------------------|
| [GET](#get)     | Fetches objects or information.                    |
| POST            | Creates new objects or information.                |
| [PUT](#put)     | Updates existing objects or information.           |
| [PATCH](#patch) | Partially updates existing objects or information. |
| DELETE          | Removes existing objects or information.           |
<div><!-- ---------------------------------------------------------------------------- --></div>

The mimetype of the request body is `application/json` and may contain the data as documented for each endpoint.


#### GET
All GET methods that return a list of objects have pagination.
To enable pagination of the returned list of objects extra URL parameters are allowed for the GET request and extra headers need to be added to the response.


##### Paginated Request
The following table is a list of all the parameters that have to be supported, but might be omitted by a client request.

<div><!-- ---------------------------------------------------------------------------- --></div>
| Parameter | Description                                            |
|-----------|--------------------------------------------------------|
| offset    | The offset of the first object returned. Default is 0. |
| limit     | Maximum number of objects to GET. Note: the server might decided to return less objects, because there are no more objects or the server limits the maximum amount of object to return to prevent, for example, overloading the system |
<div><!-- ---------------------------------------------------------------------------- --></div>


##### Paginated Response
HTTP headers that have to be added to any paginated get response.

<div><!-- ---------------------------------------------------------------------------- --></div>
| HTTP Parameter | Description                                                                 |
|----------------|-----------------------------------------------------------------------------|
| link           | Link to the 'next' page should be provided, see example below               |
| X-Total-Count  | (Custom HTTP Header) Total number of objects available in the server system |
| X-Limit        | (Custom HTTP Header) Number of objects that are returned. Note that this is an upper limit, if there are not enough remaining objects to return, then less than this number will be returned. |
<div><!-- ---------------------------------------------------------------------------- --></div>

Example of a required OCPI pagination link header
```   
   Link: <https://www.server.com/ocpi/cpo/2.0/cdrs/?offset=5&limit=50>; rel="next"
```   


#### PUT
A PUT request must specify all required fields of an object (similar to a POST request). Optional fields that are not included will revert to their default value which is either specified in the protocol or NULL.


#### PATCH
A PATCH request must only specify the object's identifier and the fields to be updated. Any fields (both required or optional) that are left out remain unchanged.

The mimetype of the request body is `application/json` and may contain the data as documented for each endpoint.

In case a fails, the client is expected to call the GET method to check the state of the object in the other parties system. If the object doesn't exist, the client should do a [PUT](#put). 


### Client owned object push
Normal client/server RESTful services work in a way that the Server is the owner of objects that are created. The client request a POST method with an object to the end-point URL. The response send by the server will contain the URL to the new object. The client will only request 1 server to create a new object, not multiple servers.
 
Many OCPI modules work differently: The client is the owner of the object and only pushes the information to 1 or more servers for information sharing purposes.   
For Example: The CPO owns the Tariff objects, and pushes them to a couple of eMSPs, so each eMSP gains knowledge of the tariffs that the CPO will charge them for their customers' sessions. eMSP might receive Tariff objects from multiple CPOs. They need to be able to make a distinction between the different tariffs from different CPOs. 
POST is not supported for these kind of modules.
PUT is used to send new objects to the servers. 
The distinction between objects from different CPOs/eMSPs is made, based on a {[country-code](credentials.md#credentials-object)} and {[party-id](credentials.md#credentials-object)}.
Client owned object URL definition: {base-ocpi-url}/{end-point}/{country-code}/{party-id}/{object-id}


Example of a URL to a client owned object
```   
   https://www.server.com/ocpi/cpo/2.0/tariffs/NL/TNM/14
```   
 

### Response format

When a request cannot be accepted, an HTTP error response code is expected including a JSON object that contains more details. HTTP status codes are described on [w3.org](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html).

The content that is sent with all the response messages is a 'application/json' type and contains a JSON object with the following properties:

<div><!-- ---------------------------------------------------------------------------- --></div>
| Property       | Type                                  | Card. | Description                                               |
|----------------|---------------------------------------|-------|-----------------------------------------------------------|
| data           | object                                | *     | Contains the actual response data object or list of objects from each request. |
| status_code    | Integer                               | 1     | Response code, as listed in [Status Codes](status_codes.md#status-codes), indicates how the request was handled. To avoid confusion with HTTP codes, at least four digits are used.                              |
| status_message | String                                | ?     | An optional status message which may help when debugging. |
| timestamp      | [DateTime](types.md#12_datetime_type) | 1     | The time this message was generated.                      |
<div><!-- ---------------------------------------------------------------------------- --></div>

For brevity's sake, any further example used in this specification will only contain the value of the "data" field. In reality, it will always have to be wrapped in the above response format.

#### Example: Version details response

```json
{
	"data": [{
		"version": "2.0",
		"endpoints": [{
			"identifier": "credentials",
			"url": "https://example.com/ocpi/cpo/2.0/credentials/"
		}, {
			"identifier": "locations",
			"url": "https://example.com/ocpi/cpo/2.0/locations/"
		}]
	}],
	"status_code": 1000,
	"status_message": "Success",
	"timestamp": "2015-06-30T21:59:59Z"
}
```

#### Example: Tokens GET Response with one Token object. (CPO end-point)

```json
{
	"data": [{
		"uid": "012345678",
		"type": "RFID",
		"auth_id": "FA54320",
		"visual_number": "DF000-2001-8999",
		"issuer": "TheNewMotion",
		"valid": true,
		"allow_whitelist": true
	}],
	"status_code": 1000,
	"status_message": "Success",
	"timestamp": "2015-06-30T21:59:59Z"
}
```


#### Example: Tokens GET Response with list of Token objects. (eMSP end-point)

```json
{
	"data": [{
		"uid": "100012",
		"type": "RFID",
		"auth_id": "FA54320",
		"visual_number": "DF000-2001-8999",
		"issuer": "TheNewMotion",
		"valid": true,
		"allow_whitelist": true
	}, {
		"uid": "100013",
		"type": "RFID",
		"auth_id": "FA543A5",
		"visual_number": "DF000-2001-9000",
		"issuer": "TheNewMotion",
		"valid": true,
		"allow_whitelist": true
	}, {
		"uid": "100014",
		"type": "RFID",
		"auth_id": "FA543BB",
		"visual_number": "DF000-2001-9010",
		"issuer": "TheNewMotion",
		"valid": false,
		"allow_whitelist": true
	}],
	"status_code": 1000,
	"status_message": "Success",
	"timestamp": "2015-06-30T21:59:59Z"
}
```


## Interface endpoints

As OCPI contains multiple interfaces, different endpoints are available for messaging. The protocol is designed such that the exact URLs of the endpoints can be defined by each party. It also supports an interface per version.

The locations of all the version specific endpoints can be retrieved by fetching the API information from the versions endpoint. Each version specific endpoint will then list the available endpoints for that version. It is strongly recommended to insert the protocol version into the URL.

For example: `/ocpi/cpo/2.0/locations` and `/ocpi/emsp/2.0/locations`.

The URLs of the endpoints in this document are descriptive only. The exact URL can be found by fetching the endpoint information from the API info endpoint and looking up the identifier of the endpoint.


<div><!-- ---------------------------------------------------------------------------- --></div>
| Operator interface         | Identifier  | Example URL                                   |
| -------------------------- | ----------- | --------------------------------------------- |
| Credentials                | credentials | https://example.com/ocpi/cpo/2.0/credentials  |
| Charging location details  | locations   | https://example.com/ocpi/cpo/2.0/locations    |
<div><!-- ---------------------------------------------------------------------------- --></div>


<div><!-- ---------------------------------------------------------------------------- --></div>
| eMSP interface             | Identifier  | Example URL                                   |
| -------------------------- | ----------- | --------------------------------------------- |
| Credentials                | credentials | https://example.com/ocpi/emsp/2.0/credentials |
| Charging location updates  | locations   | https://example.com/ocpi/emsp/2.0/locations   |
<div><!-- ---------------------------------------------------------------------------- --></div>

