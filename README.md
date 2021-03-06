# multifetch
Express middleware for performing internal batch GET requests. It allows the client to send a single HTTP request, which in turn can fetch multiple JSON resources in the app, without performing any further requests.
npm install multifetch

Developed and tested with node 0.10. Versions above 1.0.0 of this module are tested with express 4, while previous versions used express 3.
UsageIt can be used without any configuration.
var multifetch = require('multifetch');
var express = require('express');

var app = express();

app.get('/api/multifetch', multifetch());

app.get('/api/user', function(request, response) {
    response.json({
        name: 'user_1',
        associates: ['user_2', 'user_3']
    });
});

app.listen(8080);
Performing a GET request to /api/multifetch?user=/api/user, will return the user and some meta information. The query parameter should have a resource name as key and the relative path as value. The path can have its own query, as long it's encoded correctly. Furthermore the endpoint must return application/json or text/json (with or without character encoding) in the content-type header, or else the content will be ignored.
// Response JSON object
{
    user: {
        statusCode: 200,                            // Response code returned by the user route
        headers: {                                  // All response headers
            'content-type': 'application/json',
            ...
        },
        body: {                                     // The actual json body
            name: 'user_1',
            associates: ['user_2', 'user_3']
        }
    },
    _error: false                                   // _error will be true if one of the requests failed
}
This way we can fetch multiple resources, by adding them to the query. If we had more routes defined, it would be possible to do.
GET /api/multifetch?user=/api/user&albums=/api/users/user_1/albums&files=/api/files

And the response will contain all the resources as described above.
{
    user: {
        statusCode: 200,
        headers: { ... },
        body: { ... }
    },
    albums: {
        statusCode: 200,
        headers: { ... },
        body: [ ... ]
    },
    files: {
        statusCode: 200,
        headers: { ... },
        body: [ ... ]
    },
    _error: false
}
We don't perform any additional HTTP requests, instead express' internal routing is used to get the resources and send them back to client. The JSON is streamed to client one requests at the time.
It is also possible to configure multifetch to ignore some of the query parameters, or call a provided callback function before performing any internal routing, which makes it possible to set any required headers on the internal request, e.g. api access tokens (the cookie header is set by default).
// Ignore access_token and token in the query
app.get('/api/multifetch', multifetch({ ignore: ['access_token', 'token'] }));

// Callback function run before each internal request.
// The serverRequest argument, is the original request to multifetch,
// while internalRequest is the fake request generated to get the actual resource.
app.get('/api/multifetch', multifetch(function(serverRequset, internalRequest, next) {
    if(serverRequest.hasAccess) {
        // Calling next with a truthy value, skips this internal request.
        return next(true);
    }

    // Copy token
    internRequest.headers.token = serverRequest.headers.token || serverRequest.query.token;
    next();
}));
If request.body is available and is a JSON object, resources will also be included from there (body object with resource names as keys, and paths as values). This can de bone by using a post route with the bodyParsemiddleware.
Requesting non JSON resources, where content-type doesn't contain json, returns null as body.
Passing headers: false as an option, excludes statusCode and headers from the response, only the resource content is returned (the _error property is still available).
app.get('/api/multifetch', multifetch({ headers: false }));
Response with content only.
{
    user: {
        name: 'user_1',
        associates: ['user_2', 'user_3']
    },
    _error: false
}
ConcurrencyBy default, multifetch will process each request sequentially, waiting until a request has been processed and fully piped out before it processes the next request.
If the response to each of your requests is small but takes a long time to fetch (e.g. heavy database queries),multifetch supports concurrently processing requests.
Passing concurrency: N as an option allows you to control the number of concurrent requests being processed at any one time:
app.get('/api/multifetch', multifetch({concurrency: 5}));
In the above case, 5 requests would be routed through express concurrently, and the response of each is placed in a queue to be streamed out to the client sequentially.
LicenseMIT​
