arrest-jwt-bucket
=================

Forked from vivocha arrest repo : https://github.com/vivocha/arrest


REST framework for Node.js, Express and MongoDB

Arrest lets you write RESTful web services in minutes. It works with Express,
implements simple CRUD semantics on MongoDB and the resulting web services
are compatible with the $resource service of AngularJS.

It differs from the original repo by using jwt for accessing routes, and a variable in the token named bucket for db name.

## How to Install

```bash
npm install arrest-jwt-bucket
```

## Super Simple Sample

The following sample application shows how to attach a simple REST API to and express
application. In the sample, the path */api* is linked to a *data* collection
on a MongoDB instance running on *localhost* with the db name stored in the *jsontoken* token:

```js
var arrest = require('arrest')
  , express = require('express')
  , app = express();

var privateKey = "aaaaaaaahhhhhhhhhh";

app.use(express.bodyParser());

arrest.use(app, '/api', new arrest.RestMongoAPI('mongodb://localhost:27017', privateKey, 'data'));

app.listen(3000);
```

Now you can query your *data* collection like this:

```bash
curl --header "Authorization: Bearer jwttoken" "http://localhost:3000/api"
```

You can add a new item:

```bash
curl --header "Authorization: Bearer jwttoken" "http://localhost:3000/api" -d "name=Jimbo&surname=Johnson"
```

(for complex objects, just do a POST with a JSON body)

You can query a specific item by appeding the identifier of the record (the _id attribute):

```bash
curl --header "Authorization: Bearer jwttoken" "http://localhost:3000/api/51acc04f196573941f000002"
```

You can update an item:

```bash
curl --header "Authorization: Bearer jwttoken" "http://localhost:3000/api/51acc04f196573941f000002" "name=Jimbo&surname=Smith"
```

And finally you can delete an item:

```bash
curl --header "Authorization: Bearer jwttoken" "http://localhost:3000/api/51acc04f196573941f000002" -X DELETE
```

To use this REST service in an [AngularJS](http://angularjs.org) application, all you need to do is to include the
[ngResource](http://docs.angularjs.org/api/ngResource.$resource) service and, in a controller, create a $resource object:

```js
var api = new $resource('/api/:_id', { _id: '@_id' }, {});

$scope.data = api.query();
```

## Default API routes

By default, each time you call `arrest.use` specifying a different `path`, the following routes are
added to your Express `app`:

```js
app.get('/path', arrest.RestAPI._query);
app.get('/path/:id', arrest.RestAPI._get);
app.put('/path', arrest.RestAPI._create);
app.post('/path', arrest.RestAPI._create);
app.post('/path/:id', arrest.RestAPI._update);
app.delete('/path/:id', arrest.RestAPI._remove);
```

## Creating a custom API

To create a custom API, start by defining a sub class of RestMongoAPI (or RestAPI if you don't need
MongoDB support):

```js
var util = require('util')
  , arrest = require('arrest')
  
function MyAPI() {
  arrest.RestMongoAPI.call(this, 'mongodb://localhost:27017', privateKey, 'my_collection');
}

util.inherits(MyAPI, RestMongoAPI);
```

You can now customize, for example, how the queries on the entire collection are performed: the
following example checks that a query parameter `q` is passed to the web service:

```js
MyAPI.prototype._query = function(req, res) {
  var self = this;

  self.resolveAuthentification(req, self.privateKey, function () {
      if (!req.user && !req.user.bucket) {
          return arrest.sendError( res, 401, 'Unauthorized');
      }

      if (!req.query.q) {
          arrest.sendError(res, 400, 'q parameter is missing');
      } else {
        self.query.call( self, req.user.bucket, { name: req.query.q}, arrest.responseCallback(res));
      }
  });
}
```

To add a new web services, modify the `routes` array, adding the required entries.
The array contains objects with to following format:

```js
{
  method: 'get|post|put|patch|delete|any other valid http method',
  mount: '/path/to/the/new/webservice/:with/:needed/:paramenters',
  handler: this.handler_function }
}
```

The default routes are:

```js
[
  { method: 'get',    mount: '',     handler: this._query },
  { method: 'get',    mount: '/:id', handler: this._get },
  { method: 'put',    mount: '',     handler: this._create },
  { method: 'post',   mount: '',     handler: this._create },
  { method: 'post',   mount: '/:id', handler: this._update },
  { method: 'delete', mount: '/:id', handler: this._remove }
]
```

For example:

```js
function MyAPI() {
  arrest.RestMongoAPI.call(this, 'mongodb://localhost:27017', privateKey, 'my_collection');
  this.routes.push({ method: 'get', mount: '/greet/:name', handler: this._hello });
}

util.inherits(MyAPI, RestMongoAPI);

MyAPI.prototype._hello = function(req, res) {
  res.jsonp({ hello: req.param.name });
}
```
