# Jo

Jo (pronounced "Yo!") is a JavaScript library for web pages to (asynchronously) call the originating [Jolie](https://www.jolie-lang.org/) service.

# Usage

## Invoking an operation from the server

Syntax: `Jo.operation( data [, params] )` where
- the `data` to be sent is a JSON object;
- the optional `params` are parameters for the underlying `fetch` operation (https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), for example in case you need to specify special headers.

Suppose the originating Jolie service has an operation `greet` that returns a string given a tree with subnode `name`, as follows.
(Nanoservices are great for examples, not so much for production: do this at home!)

```jolie
greet( request )( response ) {
	response.greeting = "Hello " + request.name
}
```

You can invoke it with Jo as follows.

```javascript
Jo.greet( { name: "Homer" } )
	.then( response => console.log( response.greeting ) ) // Jo uses promises
	.catch( error => {		// an error occurred
		if ( error.isFault ) {	// It's an application error
			console.log( JSON.stringify( error.fault ) );
		} else {		// It's a middleware error
			console.log( error.message );
		}
	} );
```

## Catching errors, the alternative way

Distinguishing application and middleware errors might be boring.
Use `JoHelp.parseError` (`JoHelp` is pronounced "Yo! Help!") to get that part done for you automatically. You will get a string containing the error message, if it is a middleware error, or a JSON.stringify of the carried data, if it is an application error.

```javascript
Jo.greet( { name: "Homer" } )
	.then( response => console.log( response.greeting ) )
	.catch( JoHelp.parseError ).catch( console.log );
```

## Redirections

Jo supports [redirection](https://jolielang.gitbook.io/docs/architectural-composition/redirection), the Jolie primitive to build API gateways with named subservices. (Unnamed subservices in the gateway, obtained by [aggregation](https://jolielang.gitbook.io/docs/architectural-composition/aggregation), are available as normal operations, so they can be called with the previous syntax.)

Suppose that the originating Jolie service has a redirection table as follows.
```jolie
Redirects: Greeter => GreeterService
```

If `GreeterService` has our operation `greet`, we can invoke it as follows.

```javascript
Jo("Greeter").greet( { name: "Homer" } )
	.then( response => console.log( response.greeting ) )
	.catch( JoHelp.parseError ).catch( console.log );
```

If your API gateway points to another API gateway, you can nest them!

```javascript
Jo("SubGateway1/SubGateway2/Greeter").greet( { name: "Homer" } )
	.then( response => console.log( response.greeting ) )
	.catch( JoHelp.parseError ).catch( console.log );
```

# Installation

```html
<script type="text/javascript" src="https://cdn.rawgit.com/fmontesi/jo/master/lib/jo.js"></script>
```

Or, download Jo (https://cdn.rawgit.com/fmontesi/jo/master/lib/jo.js) and use it locally.

Pull requests with better ways to distribute Jo are welcome.

# Dependencies

There are no dependencies on other libraries. However, Jo uses some recent features offered by web browsers.

- Jo uses [fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to perform asynchronous calls. If you want to use Jo with older browsers, use a [polyfill for fetch](https://github.com/github/fetch). Check which browsers support fetch here: https://caniuse.com/#feat=fetch.

- Jo uses some modern JavaScript features. [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) is used to implement the magic of calling the operations of your Jolie server as if they were native methods (`Jo.operation`). We also use [Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions). Check which browsers support Proxy at https://caniuse.com/#feat=proxy, and which support arrow functions at https://caniuse.com/#feat=arrow-functions. If you want to use Jo in browsers that do not support these features, you can try compiling Jo with [Babel](https://babeljs.io/).

# FAQ

## What HTTP method are you using?

POST is the default method. To change it, you can use the optional parameters. For example, to use GET: `Jo.operation( data, { method: 'GET' } )`.

## How do I handle basic values in root nodes sent by Jolie? (AKA response.$)

In JSON, an element can either be a basic value (e.g., strings, numbers), an object, or an array.
In Jolie, there are no restrictions: an element is always a tree, and each node can contain _both_ a basic value and subnodes (similarly to markup languages).
For example, this is valid Jolie: `"Homer" { .children[0] = "Bart", .children[1] = "Lisa" }`. It gives a tree containing the string `Homer` in its root node, which has an array subnode `children` with two elements. If you receive this tree using `Jo` in variable `response`, you can access the value contained in the root node (`"Homer"` in our example) by `response.$`.
