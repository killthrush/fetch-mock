# fetch-mock [![Build Status](https://travis-ci.org/wheresrhys/fetch-mock.svg?branch=master)](https://travis-ci.org/wheresrhys/fetch-mock)
Mock http requests made using fetch (or [isomorphic-fetch](https://www.npmjs.com/package/isomorphic-fetch)). As well as shorthand methods for the simplest use cases, it offers a flexible API for customising all aspects of 	mocking behaviour.

## Installation and usage

`npm install fetch-mock` then `require('fetch-mock')` in most environments 

[Troubleshooting](#troubleshooting), [V4 changelog](#v4-changelog)

*To output useful messages for debugging `export DEBUG=fetch-mock`*

## Basic usage

**`require('fetch-mock')` exports a singleton with the following methods**

#### `mock(matcher, response)` or `mock(matcher, method, response)`  
Replaces `fetch()` with a stub which records its calls, grouped by route, and optionally returns a mocked `Response` object or passes the call through to `fetch()`. Calls to `.mock()` can be chained.

* `matcher` [required]: Condition for selecting which requests to mock Accepts any of the following
	* `string`: Either an exact url to match e.g. 'http://www.site.com/page.html' or, if the string begins with a `^`, the string following the `^` must begin the url e.g. '^http://www.site.com' would match 'http://www.site.com' or 'http://www.site.com/page.html'
	* `RegExp`: A regular  expression to test the url against
	* `Function(url, opts)`: A function (returning a Boolean) that is passed the url and opts `fetch()` is called with (or, if `fetch()` was called with one, the `Request` instance)
* `method` [optional]: only matches requests using this http method
* `response` [required]: Configures the http response returned by the mock. Can take any of the following values
	* `number`: Creates a response with this status
	* `string`: Creates a 200 response with the string as the response body
	* `object`: As long as the object does not contain any of the properties below it is converted into a json string and returned as the body of a 200 response. If any of the properties below are defined it is used to configure a `Response` object
		* `body`: Set the response body (`string` or `object`)
		* `status`: Set the response status (defaut `200`)
		* `headers`: Set the response headers. (`object`)
		* `throws`: If this property is present then a `Promise` rejected with the value of `throws` is returned
	* `Function(url, opts)`: A function that is passed the url and opts `fetch()` is called with and that returns any of the responses listed above

#### `restore()`
Restores `fetch()` to its unstubbed state and clears all data recorded for its calls

#### `reMock()`
Calls `restore()` internally then calls `mock()`. This allows you to put some generic calls to `mock()` in a `beforeEach()` while retaining the flexibility to vary the responses for some tests. `reMock()` can be chained.

#### `reset()`
Clears all data recorded for `fetch()`'s calls

#### `calls(matcher)`
Returns an object `{matched: [], unmatched: []}` containing arrays of all calls to fetch, grouped by whether fetch-mock matched them or not. If `matcher` is specified and is equal to `matcher.toString()` for any of the mocked routes then only calls to fetch matching that route are returned.

#### `called(matcher)`
Returns a Boolean indicating whether fetch was called and a route was matched. If `matcher` is specified and is equal to `matcher.toString()` for any of the mocked routes then only returns `true` if that particular route was matched.
		
##### Example

```
fetchMock
	.mock('http://domain1', 200)
	.mock('http://domain2', 'PUT', {
		affectedRecords: 1
	});

myModule.onlyCallDomain2()
	.then(() => {
		expect(fetchMock.called('http://domain2')).to.be.true;
		expect(fetchMock.called('http://domain1')).to.be.false;
		expect(fetchMock.calls().unmatched().length).to.equal(0);
		expect(JSON.parse(fetchMock.calls('http://domain2'[)[0][1].body)).to.deep.equal({prop: 'val'});
		fetchMock.restore();
	})
```

## Advanced usage

#### `mock(routeConfig)`

Use a configuration object to define a route to mock.
* `name` [optional]: A unique string naming the route. Used to subsequently retrieve references to the calls, grouped by name. If not specified defaults to `matcher.toString()` *Note: If a non-unique name is provided no error will be thrown (because names are optional, so auto-generated ones may legitimately clash)*
* `method` [optional]: http method
* `matcher` [required]: as specified above
* `response` [required]: as specified above

#### `mock(routes)`
Pass in an array of route configuration objects

#### `mock(config)`

Pass in an object containing more complex config for fine grained control over every aspect of mocking behaviour. May have the following properties
* `routes`: Either a single route config object or an array of them (see above).
* `greed`: Determines how the mock handles unmatched requests
	* 'none': all unmatched calls get passed through to `fetch()`
	* 'bad': all unmatched calls result in a rejected promise
	* 'good': all unmatched calls result in a resolved promise with a 200 status

#### `calls(routeName)`
Returns an array of arrays of the arguments passed to `fetch()` that matched the given route.

#### `called(routeName)`
Returns a Boolean denoting whether any calls matched the given route.

#### `useNonGlobalFetch(func)`
When using isomorphic-fetch or node-fetch ideally `fetch` should be added as a global. If not possible to do so you can still use fetch-mock in combination with [mockery](https://github.com/mfncooper/mockery) in nodejs. To use fetch-mock with with [mockery](https://github.com/mfncooper/mockery) you will need to use this function to prevent fetch-mock trying to mock the function globally.
* `func` Optional reference to `fetch` (or any other function you may want to substitute for `fetch` in your tests).

To obtain a reference to the mock fetch call `getMock()`.

##### Mockery example
```
var fetch = require('node-fetch');
var fetchMock = require('fetch-mock');
var mockery = require('mockery');
fetchMock.useNonGlobalFetch(fetch);

fetchMock.registerRoute([
 ...
])
it('should make a request', function (done) {
	mockery.registerMock('fetch', fetchMock.mock().getMock());
	// test code goes in here
	mockery.deregisterMock('fetch');
	done();
});
```
## Troubleshooting

### Environment doesn't support requiring fetch-mock?
* If your client-side code or tests do not use a loader that respects the browser field of package.json use `require('fetch-mock/es5/client')`.
* If you need to use fetch-mock without commonjs, you can include the precompiled `node_modules/fetch-mock/es5/client-browserified.js` in a script tag. This loads fetch-mock into the `fetchMock` global variable.
* For server side tests running in nodejs 0.12 or lower use `require('fetch-mock/es5/server')`

### Polyfilling fetch
* In nodejs `require('isomorphic-fetch')` before any of your tests.
* In the browser `require('isomorphic-fetch')` can also be used, but it may be easier to `npm install whatwg-fetch` (the module isomorphic-fetch is built around) and load `./node_modules/whatwg-fetch/fetch.js` directly into the page, either in a script tag or by referencing it your test runner config

## V4 changelog
* `registerRoute()` and `unregisterRoute()` have been removed to simplify the API. Since V3, calls to `.mock()` can be chained, so persisting routes over a series of tests can easily be achieved by means of a beforeEach or helper e.g.
	```
	beforeEach(() => {
		fetchMock
			.mock('http://auth.service.com/user', 200)
			.mock('http://mvt.service.com/session', {test1: true, test2: false})
	});

	afterEach(() => {
		fetchMock.restore();
	});

	it('should be possible to augment persistent set of routes', () => {
		fetchMock.mock('http://content.service.com/contentid', {content: 'blah blah'})
		page.init();
		expect(fetchMock.called('http://content.service.com/contentid')).to.be.true;
	});
	```
* Defining two routes with the same name will no longer throw an error (previous implementation was buggy anyway)