## start with an example

```python
from wsgiref.simple_server import make_server


SERVER_ADDRESS = (HOST, PORT) = 'localhost', 2017
RESPONSE = b"Hello from server."


def my_app(environ, start_response):
	response_headers = [('content-type', 'text/plain')]
	start_response('200 OK', response_headers)
	return [RESPONSE]


httpd = make_server(HOST, PORT, my_app)
httpd.serve_forever()
```

## step 1:  `httpd = make_server(HOST, PORT, my_app)` - `server`

```python
def make_server(host, port, app, server_class=WSGIServer, handler_class=WSGIRequestHandler):
    """Create a new WSGI server listening on `host` and `port` for `app`"""
    server = server_class((host, port), handler_class)
    server.set_app(app)
    return server
```

So it just 
* create an **`WSGIServer`** instance with `host`, `port` and `handler_class`(talk about this later)
* call **`set_app()`** to pass `my_app` to the server.
* and return the server.

Let's see **`WSGIServer`**.

```python
class WSGIServer(HTTPServer):

    """BaseHTTPServer that implements the Python WSGI protocol"""

    application = None

    def server_bind(self):
        """Override server_bind to store the server name."""
        HTTPServer.server_bind(self)
        self.setup_environ()

    def setup_environ(self):
        # Set up base environment
        env = self.base_environ = {}
        env['SERVER_NAME'] = self.server_name
        env['GATEWAY_INTERFACE'] = 'CGI/1.1'
        env['SERVER_PORT'] = str(self.server_port)
        env['REMOTE_HOST']=''
        env['CONTENT_LENGTH']=''
        env['SCRIPT_NAME'] = ''

    def get_app(self):
        return self.application

    def set_app(self,application):
        self.application = application
```

We can see that **`WSGIServer`** is just a **`HTTPServer`** with extra methods to do extra stuff.

* It first setup some environ to a dict in **`setup_environ()`** called by **`server_bind()`**
* and in **`make_server()`**, we called **`set_app()`** to pass our `my_app` to `self.application`

## step 1:  `httpd = make_server(HOST, PORT, my_app)` - `handler`

Now the server side is done. Lets see how about handler class.

```python
class WSGIRequestHandler(BaseHTTPRequestHandler):

    server_version = "WSGIServer/" + __version__

    def get_environ(self):
        env = self.server.base_environ.copy()
        env['SERVER_PROTOCOL'] = self.request_version
        env['SERVER_SOFTWARE'] = self.server_version
        env['REQUEST_METHOD'] = self.command
        if '?' in self.path:
            path,query = self.path.split('?',1)
        else:
            path,query = self.path,''

        env['PATH_INFO'] = urllib.parse.unquote(path, 'iso-8859-1')
        env['QUERY_STRING'] = query

        host = self.address_string()
        if host != self.client_address[0]:
            env['REMOTE_HOST'] = host
        env['REMOTE_ADDR'] = self.client_address[0]

        if self.headers.get('content-type') is None:
            env['CONTENT_TYPE'] = self.headers.get_content_type()
        else:
            env['CONTENT_TYPE'] = self.headers['content-type']

        length = self.headers.get('content-length')
        if length:
            env['CONTENT_LENGTH'] = length

        for k, v in self.headers.items():
            k=k.replace('-','_').upper(); v=v.strip()
            if k in env:
                continue                    # skip content length, type,etc.
            if 'HTTP_'+k in env:
                env['HTTP_'+k] += ','+v     # comma-separate multiple headers
            else:
                env['HTTP_'+k] = v
        return env

    def get_stderr(self):
        return sys.stderr

    def handle(self):
        """Handle a single HTTP request"""

        self.raw_requestline = self.rfile.readline(65537)
        if len(self.raw_requestline) > 65536:
            self.requestline = ''
            self.request_version = ''
            self.command = ''
            self.send_error(414)
            return

        if not self.parse_request(): # An error code has been sent, just exit
            return

        handler = ServerHandler(
            self.rfile, self.wfile, self.get_stderr(), self.get_environ()
        )
        handler.request_handler = self      # backpointer for logging
        handler.run(self.server.get_app())
```

from [socketserver](https://github.com/alexthemonkey/read_source_code_python_01_socketserver/blob/master/01_non_threading_TCPServer.md) and [httpserver](https://github.com/alexthemonkey/read_source_code_python_02_http_server/blob/master/01_http_server.md) we know that, eventually we will call **`handle()`** in whatever hanlder class we defined. And if we look at the **`handle()`** method above, it eventually uses a **`ServerHandler`** instance to handle the request via **`handler.run(self.server.get_app())`**.


![alt text](WSGIServer_function_calls.png)


## We already have `HTTPSever` and `BaseHTTPHandler`, why bother WSGI?

The reason to to create a standard of how application will run so all knids of web servers: `waitress`, `Nginx uwsgi`, `Gunicorn` etc can work with framworks such as `Flask`, `Django` or `Pyramid`.

how do you then make sure that you can run your Web server with multiple Web frameworks without making code changes either to the Web server or to the Web frameworks?

And the answer to that problem became the :

**Python Web Server Gateway Interface (or WSGI for short, pronounced)**


1. WSGI allowed developers to separate choice of a Web framework from choice of a Web server.
2. Now you can actually mix and match Web servers and Web frameworks and choose a pairing that suits your needs.
3. You can run Django, Flask, or Pyramid, for example, with Gunicorn or Nginx/uWSGI or Waitress.

```
|---------------|
| web servers:  |
| waitress,     |     <======  WSGI  ======>   Pyramid, Flask, Django
| Nginx uwsgi,  |
| Gunicorn      |
|---------------|
```

WSGI is a protocol for :    web server   and    application

There are 2 roles in WSGI protocol:

	1. "server" or "gateway" side
	2. "application" or "framework" side
	

Framework like: `Django` or `Flask`, does the same as our `my_app`:

* **define a callable** (in our case, `my_app`):
	* a function
	* or a method
	* or an instance of a class with `__call__` implemented so the instance is callable.
* **this callable takes 2 arguments**:
	* a dictionary containing CGI like variables (`environ`)
	* a callback function (in our case: `start_response`) that will be used by the application to send:
		* HTTP status code/message
		* HTTP headers
* **return the response body to the server as strings wrapped in an iterable**
	* that is why we return `['hello there']` as a list in `my_app`.


## middleware

Between the request and your final `my_app` to provide response, you can apply multiple layers of process to:
* check if a certain request is satified
* if user is authorised
* ...

All those layers are called: **`middleware`**.

There are a lot of ways to design the middleware patterm. Checked how Django did it maybe? 



