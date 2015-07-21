# Fleek Validator

Middleware router that validates request against swagger doc specifications.

Quick reference:
- Best used in tandem with the [fleek-router](#koa-on-fleek)
- Validations are based on the swagger [parameter specifications](https://github.com/swagger-api/swagger-spec/blob/master/versions/2.0.md#parameter-object), but many additional validations have been added that are not supported by swagger directly
- A request may pass erroneously pass validation if the route is not recognized or the configuration is not correct. if no validation is run `this.validated` will be `false`
- The `validate` object/helper will be added to the request context and can be accessed during request handling. eg: `this.validate.one(foo, {/* parameter decription*/})`
- if koa-router is not used, paths will be mapped to swager docs using the [koa-router](https://github.com/koajs/route) module. If another routing module is used in the stack, unexpected results may appear (you have been warned!)
- A full list of errors an codes can be found in [error.json](https://github.com/gohart/fleek-validator/blob/master/errors.json)

## Key

- [Usage](#usage)
  - [Basic](#basic)
  - [Fully Custom (paths)](#fully-custom-paths)
  - [Fully Custom (objects)](#fully-custom-objects)
  - [Koa on Fleek](#koa-on-fleek)
- [Response Structure](#response-structure)
- [Static Utilities](#static-utilities)
  - [validator.validate](#validatorvalidate)
    - [`validate.one`](#validateone)
    - [`validate.object`](#validateobject)
    - [`validate.ctx`](#validatectx)
    - [`validate.error`](#validateerror)
    - [`validate.isError`](#iserror)
- [Configuration](#configuration)
  - [`config.swagger`](#configswagger)
  - [`config.success`](#configsuccess)
  - [`config.catch`](#configcatch)
  - [`config.strict`](#configstrict)
- [Authors](#authors)

## Usage

### Basic

- Swagger docs will be retrieved from `./api.json`, `/swagger.json`, `/config/api.json`, or `/config/swagger.json` in that order
- [**TODO** Full example]()

```javascript
var koa            = require('koa');
var router         = require('koa-router')();
var fleekValidator = require('fleek-validator');
var app = koa();

fleekValidator(app);

router.get('/', function *() {
  console.log('I passed validation!')
});

app.listen(3000);
```

### Fully Custom (paths)

- Swagger docs are pulled from `./custom/docs.json`
- error is run on validation failure
- success is run on validation pass (before next middlware)
- [**TODO** Full example]()

```javascript
var koa            = require('koa');
var router         = require('koa-router')();
var fleekValidator = require('fleek-validator');
var app = koa();

fleekValidator(app, {
  swagger : './custom/docs.json',
  strict  : true, // fail any route not in the swagger docs
  success : function *(next) {
    console.log('Passed validation!')
  },
  error  : function *(err, next) {
    console.log('Failed Validation :(');
    this.body = err;
  }
});

router.get('/', function *() {
  console.log('hello from root')
});

app.listen(3000);
```

### Fully custom (objects)

- Swagger docs are parsed from the object provided
- error is run on validation failure
- success is run on validation pass (before next middlware)
- [**TODO** Full example]()

```javascript
var koa            = require('koa');
var router         = require('koa-router')();
var fleekValidator = require('fleek-validator');
var app = koa();

fleekValidator(app, {
  swagger      : require('./some/swagger/generator')(),
  strict  : true, // fail any route not in the swagger docs
  success : function *(next) {
    console.log('Passed validation!')
  },
  error  : function *(err, next) {
    console.log('Failed Validation :(');
    this.body = err;
  }
});

router.get('/', function *() {
  console.log('hello from root')
});

app.listen(3000);
```

### Koa on fleek

- Controllers are pulled from the `./controllers` directory
- Swagger docs will be retrieved from `./api.json`, `/swagger.json`, `/config/api.json`, or `/config/swagger.json` in that order
- routes wil be generated by fleek-router
- [**TODO** Full example]()

```javascript
var koa         = require('koa');
var fleekRouter = require('fleek-router');
var app = koa();

fleekRouter(app, {
  authenicate : true,
  validate    : {
    error : function *(err, next) {
      console.log('oh no! Returning default errors');
    }
  }
});

app.listen(3000);
```

## Response Structure

- The following structure will be followed in errors passed to the error handler specified
- If no error handler is specified, the error will be sent to the client with the status `400`
- A full list of errors an codes can be found in [error.json](https://github.com/gohart/fleek-validator/blob/master/errors.json)

```javscript
{
  "error": "Validaiton Failed",
  "error_name": "VALIDATION_FAILED",
  "details": [{
    "name": "VALUE.FOO",
    "code": 204,
    "message": "Must be a valid foo string",
    "parameter": {
      "name": "some_val",
      "in": "body",
      "description": "fooget about it",
      "required": true,
      "type": "string",
      "foo": true
    }
  }]
}
```

## Static Utilities

### validator.validate

- A collection of static utilities

```javascript
var val    = require('fleek-validator').validate;
var errors = val.one(foo, {/* swagger parameter object foo must match */})

// OR - if using fleek-validor as middleware
function *(next) {
  var errors = this.validate.one(foo, {/* swagger parameter object foo must match */});
}
```

### validate.one

- validate a single object against a swagger parameter object

```javascript
var error = validator.validate.one(fooEmail, {
  type  : 'string',
  email : true
})
```

### validate.object

- validate an object of values against an array of swagger parameter objects
- also accepts accepts definition property objects
   - eg:

```javascript
      definitions : {
        UserModel : {
          parameters : {
            name : {
              type    : "string",
              required: true
            }
          }
        }
      }
```

#### parameters

- subject [String] - `.` delimited error property chain to identify which [error.json](https://github.com/gohart/fleek-validator/blob/master/errors.json) error to generate
- parameters [Object] - swagger parameter description which failed
- options [object] (optional)
  - trim [Boolean] - if true, trim properties from the result that aren't in the the parameters array
  - defaults [Boolean] - if true, apply any `defaultValue`'s from the parameters array
  - partial [Boolean] - if true, allow `required` parameters to pass through undefined (useful for partial updates)

#### returns

- [Array] - array of validation errors
- [Object] - the object which passed validation, after options are applied

```javascript
var parameters = [
  {
    name  : 'user_email',
    type  : 'string',
    email : true
  }, {
    name    : 'user_password',
    type    : 'string',
    pattern : passwordRegexp
  }, {
    name         : 'confirmed',
    defaultValue : false
  }
];

var subject = {
  user_email    : fooEmail,
  user_password : fooPwd,
  injection     : someMaliciousInjection
};

var result = validator.validate.object(subject, parameters, {
  trim     : true,
  defaults : true,
});
// result -> {
//   user_email    : fooEmail,
//   user_password : fooPwd,
//   confirmed     : false
// }
```

### validate.ctx

- validate a request context against a set of swagger params

```javascript
var errors = validator.validate.ctx(ctx, [
  {
    name  : 'user_email',
    type  : 'string',
    email : true
  }, {
    name    : 'user_password',
    type    : 'string',
    pattern : passwordRegexp
  }
])
```

### validate.error

- An error class for validation errors `[ValidationError]`

#### parameters

- errorName [String] - `.` delimited error property chain to identify which [error.json](https://github.com/gohart/fleek-validator/blob/master/errors.json) error to generate
- parameter [Object] - swagger parameter description which failed
- expected [String] (optional) - expected value for the parameter (replaces instances of `${expected}` in the message)

```javascript
var someError = new validator.validate.error('TYPE.STRING', someParameter, 'alphanumeric');
```

## validate.isError

- detects whether or not the object is a validation error

#### parameters

- result [Mixed] - result object to test for validation failure

#### alias

- isErr
- isValidationError
- isValidationErr
- isValError
- isValErr

```javascript
var failedValidationResult = validator.validate.isError(resultOfValidation);
```

## Configuration


### config.swagger

#### [optional]

#### summary

- sets the swagger documentation source for compiling routes.

#### accepts

- `Object` - javascript object mapping to the swagger doc format exactly
- `String` - path to the swagger json file (relative or absolute)
- `Undefined` - attempts to find the config in the following places `./api.json`, `./swagger.json`, `./config/api.json`, and `./config/swagger.json`

```javascript
config.swagger = undefined; // attempts to resolve internally
// OR
config.swagger = './some/relative/swag.json';
// OR
config.swagger = '/some/absolute/swagger.json';
// OR
config.swagger = require('./a/function/returning/swag')();
```

### config.success

#### [optional]

#### summary

- Success handler to run if validation succeeds (before the next middleware)
- The next middlewre funciton is provided

#### accepts

- `Function` - executed on validation success

```javascript
config.succcess = function *(next) {
  console.log('woo! validation passed!');
  yield next();
}
```

### config.catch

#### [optional]

#### summary

- Error handler to run if validation fails
- The validation error and the next middleware function are provided
- If not provided, the default validation error structure is returned to the client

#### accepts

- `Function` - executed on validation failure

```javascript
config.catch = function *(err, next) {
  console.log('oh no! Returning default errors');
}
```


### config.strict

#### [optional]

#### summary

- Determines whether or not to reject undocumented routes

#### accepts

- `Boolean` -  if true, undocumented routes will fail validation

```javascript
config.strict = true;
```



## Authors

- [John Hofrichter](https://github.com/johnhof)
- [Peter A. Tariche](https://github.com/ptariche)
