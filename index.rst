########################
Draft IVOA JSON encoding
########################

.. abstract::

   Proposes a JSON network protocol encoding for IVOA web service protocols.

.. warning::

   This document is **not an IVOA standard** and is **not complete**.
   It is an experiment intended as input to ongoing IVOA discussions about future revisions to IVOA standards.
   The document is written as if it were an IVOA standard to save effort if parts of it can be cut and pasted into future documents, but it is not endorsed by the IVOA and is not part of any IVOA protocol.

.. seealso::

   :sqr:`91`: Draft IVOA web service standards framework
       The draft protocol structure for IVOA web services.
       This specifies what should be included in a network protocol specification such as this one.

   :sqr:`093`: Draft IVOA SODA web service specification
       An example network protocol specification for a simple SODA cutout service using this framework.

Overview
========

The network protocol encoding portion of an IVOA web service standard specifies how to send requests, receive responses, and encode data types.
This document specifies an encoding built on HTTP and JSON.

All requests are HTTP requests.
The version of the HTTP protocol is unspecified; any version that is compatible with standard HTTP verbs and headers may be used.
However, implementations of this network encoding should normally provide an HTTP/1.1 implementation, at least as a fallback, since support for newer versions of the HTTP protocol is still limited.

Clients and servers are expected to conform to the normal requirements of the HTTP protocol.
Only the portions that are specific to IVOA web protocols are specified here.

Data type encoding
==================

This network protocol provides two data type encodings: an encoding into JSON that should be used for requests via ``POST``, ``PUT``, or ``PATCH`` and all structured responses; and an encoding into query parameters that should be used for ``GET`` requests.
The JSON encoding is the default unless the web service specifies that ``GET`` is permitted for a given operation.

JSON encoding
-------------

This description of encodings uses the terminology of :rfc:`8259`.

object
    Encode as a JSON object.

null
    Encode as the JSON literal ``null``.

string
    Encode as a JSON string.

uri
    Encode as a JSON string.

integer
    Encode as a JSON number without a fractional part or exponent.

float
    Encode all values other than positive infinity, negative infinity, and NaN as a JSON number.
    Encode positive infinity as the JSON string ``"+Inf"``.
    Encode negative infinity as the JSON string ``"-Inf"``.
    Encode NaN as the JSON string ``"NaN"``.

boolean
    Encode as the JSON literal ``true`` or ``false``.

enum
    Encode as a JSON string.

timestamp
    Encode as an ISO 8601 string in the extended format.
    Use the calendar date format and include all components of the date.
    More than four digits are permitted for the year to specify dates after 9999 CE; however, these may not be supported in all implementations.
    Include at least hours, minutes, and seconds in the time.
    Milliseconds, if present, are encoded as a fractional part after the seconds using U+002E (``.``) as the decimal mark.
    For example, ``2024-08-23T14:42:47.043Z``.

duration
    Encode the number of seconds in the duration as a JSON number.

list
    Encode as a JSON array.
    Object labels whose value type is a list must use the plural form of the label.

.. _query-encoding:

Query encoding
--------------

The query encoding is used for ``GET`` requests.
It does not support objects as values, and therefore requests whose input contains such types cannot be sent via ``GET``.
Web service specifications should choose which operations allow ``GET`` with this in mind.

The encoding described below is for the value portion of the query parameter.
The entire query parameter, including the label, is then encoded following the rules for query parameters in :rfc:`3986`.
Note that this may involve additional percent-encoding of special characters and characters outside of the ASCII character set.

null
    Do not include the label or value if the value is null.

string
    Encode as a UTF-8 string.

uri
    Encode as a UTF-8 string.

integer
    Encode as a UTF-8 string using the normal representation of an integer and the base digit code points (U+0030 through U+0039).
    Do not use any digit grouping marks.
    Negative integers are prefixed with U+002D.

float
    Encode as a UTF-8 string using the normal representation of a floating point number.
    Use U+002E as the decimal point and do not use any digit grouping marks.
    Exponents can be introduced with either ``e`` (U+0065) or ``E`` (U+0045).
    Use U+002D to mark negative numbers.
    Encode positive infinity as the string ``+Inf``.
    Encode negative infinity as the string ``-Inf``.
    Encode NaN as the string ``NaN``.

boolean
    Encode as the string ``true`` or the string ``false``.

enum
    Encode as a UTF-8 string.

timestamp
    Encode as an ISO 8601 string in the extended format.
    Use the calendar date format and include all components of the date.
    More than four digits are permitted for the year to specify dates after 9999 CE; however, these may not be supported in all implementations.
    Include at least hours, minutes, and seconds in the time.
    Milliseconds, if present, are encoded as a fractional part after the seconds using U+002E (``.``) as the decimal mark.
    The encoded timestamp must end in ``Z`` and must not include time zone offset information in any other format.
    For example, ``2024-08-23T14:42:47.043Z``.

duration
    Encode the number of seconds in the duration as a UTF-8 string.
    Milliseconds, if present, are encoded as a fractional part after the seconds using U+002E (``.``) as the decimal mark.

list
    Repeat the query parameter for each value, encoding each value following the regular rules for its data type.

.. _requests:

Requests
========

All client requests are HTTP requests.
The HTTP verb is mostly determined by the type of the operation.
When a web service supports this network protocol, it must choose the appropriate verb in several cases discussed in detail below.

query
    ``POST`` by default.
    The web service specification can also allow ``GET``, but should keep in mind the constraints discussed in :ref:`query-encoding`.

create
    ``PUT`` by default.
    In some cases, ``POST`` is more appropriate.
    The web service specification can select either.

modify
    ``PATCH``.
    The web service may also allow ``PUT`` for complete replacement of the object.

delete
    ``DELETE``.

action
    ``POST``.

In all cases except ``GET`` and ``DELETE``, the client must provide a body in JSON format, consisting of an encoding of the request object described in the web service specification.
The client must send the ``Content-Type: application/json`` header to indicate that this request body is in JSON format.

Responses
=========

The response from the server uses standard HTTP response codes.

Successful responses
--------------------

A successful response is indicated by a 2xx response code.
Normally this will be 200, but 201 is appropriate after create operations, and 204 should be used when there is no accompanying response body, such as in response to a delete operation.
Servers should choose a 2xx response approrpriate to the nature of the operation and response.

The web service specification will describe whether a response is a structured response encoded according to this document or a data response.
Structured responses following the encoding rules in this document must include ``Content-Type: application/json`` in the response headers.
Data responses must include a ``Content-Type`` header with an appropriate MIME type describing the format of the response.
For example, responses that return a VOTable in XML format must include ``Content-Type: application/x-votable+xml``.

Redirect responses
------------------

The response may be an appropriate 3xx redirect response code for the type of request (, in which case the client should retrieve the location of the response from the ``Location`` header and make a new ``GET`` request to the provided URL.
Servers should normally use the 303 response code for this purpose, but clients should support any of 301, 302, 303, 307, or 308.

Error responses
---------------

Errors in performing the operation must result in a 4xx or 5xx HTTP response code appropriate to the nature of the error.

A 4xx response code should be used when the client's request is invalid for some reason.
Using 422 as the HTTP response code for requests that do not pass input validation is recommended, since that makes it easier to distinguish protocol errors from semantic errors in the request.
A 5xx response code should be used for apparently valid requests that cannot be handled due to some server problem.
For client or server errors without a more specific assigned HTTP response code, use 400 or 500, respectively.

The server should attempt to include a structured error message as described in :sqr:`091` as the body of an error response whenever possible.
If this is possible, the error response must include the header ``Content-Type: application/json``, as with any other structured response.

Clients must support appropriate error reporting using only the information in the HTTP status code and fall back on that behavior if there is no structured error body, if the MIME type of the response is not ``application/json``, or if the error body is in an unknown format.
HTTP errors are frequently generated by other intermediate components outside of the web service, and those components will generally not follow IVOA conventions.

Web service specifications
==========================

Web service specifications that support this network encoding must include an OpenAPI 3.0 schema for the web service as a supplemental part of the specification.

Web service specifications must also choose the HTTP verbs following the rules in :ref:`requests`, and specify which operations are supported via ``GET``.
This information must be included in the schema, and should also be included in the body of the standard since implementors may miss it in the schema.
