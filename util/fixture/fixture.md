@page can.util.fixture
@parent can.util

[can.util.fixture can.fixture] intercepts an AJAX request and simulates
the response with a file or function. They are a great technique
when you want to develop JavaScript
independently of the backend.

## Types of Fixtures

There are two common ways of using fixtures.  The first is to
map Ajax requests to another file.  The following
intercepts requests to `/tasks.json` and directs them
to `fixtures/tasks.json`:
    
    can.fixture("/tasks.json", "fixtures/tasks.json");

The other common option is to generate the Ajax response with
a function.  The following intercepts updating tasks at
`/tasks/ID.json` and responds with updated data:

    can.fixture("PUT /tasks/{id}.json", function(original, settings, respondWith){
       respondWith({ updatedAt : new Date().getTime() });
    })

We categorize fixtures into the following types:

  - __Static__ - the response is in a file.
  - __Dynamic__ - the response is generated by a function.

There are different ways to lookup static and dynamic fixtures.

## Static Fixtures

Static fixtures use an alternate url as the response of the Ajax request.

    // looks in fixtures/tasks1.json relative to page
    can.fixture("tasks/1", "fixtures/task1.json");

    // looks absolute to the page
    can.fixture("tasks/1", "//fixtures/task1.json");
    
## Dynamic Fixtures

Dynamic Fixtures are functions that get the details of the Ajax request and return the result of the mocked service
request from your server.

For example, the following returns a successful response with JSON data from the server:

    can.fixture("/foobar.json", function(original, settings, respondWith){
      respondWith(200, "success", { json: {foo: "bar" } }, {})
    })

The fixture function has the following signature:

    function( originalOptions, options, respond) {
      respond(status, statusText, responses, responseHeaders);
    }

where the fixture function is called with:

  - `originalOptions` - are the options provided to the ajax method, unmodified,
    and thus, without defaults from ajaxSettings
  - `options` - are the request options
  - `respondWith` - the response callback. It can be called with:
      - `status` - the HTTP status code of the response.
      - `statusText` - the status text of the response
      - `responses` - a map of dataType/value that contains the responses for each data format supported
      - `responseHeaders` - response headers
  - `headers` - a map of key/value request headers

However, can.fixture handles the common case where you want a successful response with JSON data.
The previous can be written like:

    can.fixture("/foobar.json", function(original, settings, respondWith){
      respondWith({ foo: "bar" });
    })

Since `respondWith` is called asynchronously you can also set a custom fixture timeout like this:

    can.fixture("/foobar.json", function(original, settings, respondWith){
      setTimeout(function() {
        respondWith({ foo: "bar" });
      }, 1000);
    })

If you want to return an array of data respond like this:

    can.fixture("/tasks.json", function(original, settings, respondWith){
      respondWith([ "first", "second", "third"]);
    })

__Note:__ A fixture function can also return its response directly like this:

    can.fixture("/foobar.json", function(original, settings){
      return { foo: "bar" };
    })

This is kept for backwards compatibility and should not be used.

can.fixture works closesly with [jQuery's ajaxTransport](http://api.jquery.com/extending-ajax/) system.

### Templated Urls

Often, you want a dynamic fixture to handle urls for multiple resources (for example a REST url scheme).
can.fixture's templated urls allow you to match urls with a wildcard.

The following example simulates services that get and update 100 todos.

    // create todos
    var todos = {};
    for(var i = 0; i < 100; i++) {
      todos[i] = {
        id: i,
        name: "Todo "+i
      }
    }
    can.fixture("GET /todos/{id}", function(orig, settings, respondWith){
      // return the JSON data
      // notice that id is pulled from the url and added to data
      respondWith(todos[orig.data.id]);
    })

    can.fixture("PUT /todos/{id}", function(orig, settings, respondWith){
      // update the todo's data
      can.extend(todos[orig.data.id], orig.data );
      respondWith({});
    })

Notice that data found in templated urls (ex: `{id}`) is added to the original data object.

## Simulating Errors

The following simulates an unauthorized request
to `/foo`.

    can.fixture("/foo", function(original, settings, respondWith) {
      respondWith(401,"{type: 'unauthorized'}");
    });

This could be received by the following Ajax request:

    can.ajax({
      url: '/foo',
      error : function(jqXhr, status, statusText){
        // status === 'error'
        // statusText === "{type: 'unauthorized'}"
      }
    })

## Turning off Fixtures

You can remove a fixture by passing `null` for the fixture option:

    // add a fixture
    can.fixture("GET todos.json","//fixtures/todos.json");

    // remove the fixture
    can.fixture("GET todos.json", null)

You can also set [can.util.fixture.on can.fixture.on] to false:

    can.fixture.on = false;

## Make

[can.util.fixture.make can.fixture.make] makes a CRUD service layer that handles sorting, grouping,
filtering and more.

## Testing Performance

Dynamic fixtures are awesome for performance testing.  Want to see what
10000 files does to your app's performance?  Make a fixture that returns 10000 items.

What to see what the app feels like when a request takes 5 seconds to return?  Set
[can.util.fixture.delay] to 5000.

## Organizing fixture

The __best__ way of organizing fixtures is to have a 'fixtures.js' file that steals
<code>can/util/fixture</code> and defines all your fixtures.  For example,
if you have a 'todo' application, you might
have <code>todo/fixtures/fixtures.js</code> look like:

    steal({
            path: '//can/util/fixture.js',
            ignore: true
          })
          .then(function(){

      can.fixture({
          type: 'get',
          url: '/services/todos.json'
        },
        '//todo/fixtures/todos.json');

      can.fixture({
          type: 'post',
          url: '/services/todos.json'
        },
        function(settings){
            return {id: Math.random(),
                 name: settings.data.name}
        });

    })

__Notice__: We used steal's ignore option to prevent
loading the fixture plugin in production.

Finally, we steal <code>todo/fixtures/fixtures.js</code> in the
app file (<code>todo/todo.js</code>) like:


    steal({path: '//todo/fixtures/fixtures.js',ignore: true});

    //start of your app's steals
    steal( ... )

We typically keep it a one liner so it's easy to comment out.

### Switching Between Sets of Fixtures

If you are using fixtures for testing, you often want to use different
sets of fixtures.  You can add something like the following to your fixtures.js file:

    if( /fixtureSet1/.test( window.location.search) ){
      can.fixture("/foo","//foo/fixtures/foo1.json');
    } else if(/fixtureSet2/.test( window.location.search)){
      can.fixture("/foo","//foo/fixtures/foo1.json');
    } else {
      // default fixtures (maybe no fixtures)
    }