# http.server代码阅读
python内置一个简单的web server，可以用 `python -m http.server` 直接调用。直接调用运行的server是单线程的，而且只接受HTTP的GET和HEAD方法，若要使用POST等方法就要自己手动扩展了。源码在此 https://github.com/python/cpython/blob/master/Lib/http/server.py

代码结构很清晰，用`pyrevese`生成的类图如下，可以看出整个module大致分为两个部分，server类和处理http请求的handler类。
![](https://raw.githubusercontent.com/yuuzao/mypicserver/master/img/20200620230723.webp)
# main function
main函数主要负责直接使用`python -m`的执行。首先使用`argparse`解析传入的参数。
module是支持cgi扩展的，如果传入的参数有cgi的需求则使用`CGIHTTPRequestHandler`来处理，其它则用`SimpleHTTPRequestHandler`处理。`test()`函数使用解析后的参数来让server运行起来。
![](https://raw.githubusercontent.com/yuuzao/mypicserver/master/img/20200620230852.webp)

# HTTP请求处理
HTTP请求处理包括三个类，`BaseHTTPRequestHandler` `SimpleHTTPRequestHandler` 和 `CGIHTTPResquestHandler`。
- ## BaseHTTPRequestHandler
    这个类实现的是对HTTP请求的基本处理。HTTP(HyperText Transfer Protocol)是一个建立在可靠数据流传输服务（例如TCP/IP）之上的可扩展协议。该协议会识别出HTTP请求的三个部分：
    - 一行用于验证请求类型和路径的部分
    - 可选的RFC-822-style headers
    - 可选的data

    其中headers和数据使用一个空行分隔。
    http请求的第一行格式为
    `<command> <path> <version>`
 其中` <command>`是一个大小写敏感的关键字，例如`GET`和`POST`；`path`是包含路径信息的字符串，使用URL编码规则进行编码。；`version`表示HTTP协议版本，例如`HTTP/1.0`或者`HTTP/1.1`，要注意的是版本号的格式为`1.x`，后面小数不可省略。
 HTTP规定了使用`CRLF`进行换行，但为了兼容性的考虑，也需要处理`LF`换行的情况。http请求中的空格也需要相似的处理（允许相邻部分存在多个空格，也允许末尾存在空格）。
 同样地，对于服务器的输出，应当使用`CRLF`换行，但是也要保证兼容`LF`换行。
 如果请求的第一行格式不包括`version`，则默认为0.9，这种格式的请求没有headers和data部分，服务器的响应也只有data部分。
 对于HTTP/1.x协议，服务器响应也由三部分组成：
 - 第一行给出响应的状态码
 - 一组可选的RFC-822-style headers
 - data

 其中`headers`和`data`也用空行分隔。
 
 http状态码的格式为
 `<version> <responsecode> <responsestring>`
 其中`<version>`仍然表示http版本（`HTTP/1.0`或者`HTTP/1.1`）；`<responsecode>`是三个数字组成的HTTP状态码（例如200，500），用于表明请求是否成功；`<responsestring>`为面向用户的可选字符串，用于解释状态码含义。
 python内置的server的主要工作是解析http请求及其headers，然后根据请求类型（GET或POST）调用相关的函数处理该请求。需要指出的是，若请求被识别为SPAM，则调用`do_SPAM()`方法处理，如果没有该方法，server则返回一个错误响应。
 
 实例变量中会存储http请求的细节：
 - `client_address`为客户端的ip地址，格式为`(host, port)`
 - `command, path, version`是处理过后的请求行
 - `headers`是`email.message.Message`类的实例，带有header信息
 - `rfile`是一个可读的文件对象，位于可选data部分的开始，来自`socketserver`
 - `wfile`是一个可写的文件对象，来之`socketserver`
 
 
 根据HTTP协议，响应内容的第一行必须给出响应状态。然后是0行或者多行header。然后是一个空行。再然后才是data。header行的内容由server所执行的命令有关，在大多数情况下，如果有data返回，则header应当至少有一行包括`Content-type: <type>/<subtype>`，`type`和`subtype`为`MIME`类型，例如`text/html`或者`text/plain`。
 
 
- ## SimpleHTTPRequestHandler

    这个类的主要作用是处理`GET`请求。如果url代表的路径是一个文件夹的话，则生成一个该目录内容的的HTML格式的文件；若请求的是一个文件，则直接返回这个文件。使用`copyfile`将前面步骤生成的文件复制到`wfile`对象。
- `CGIHTTPResquestHandler`
    这个类处理CGI相关请求，`POST`的处理在这个类中，但是默认是不启用，需要在shell命令中添加 `--cgi`参数。
    
    
    #性能测试
    在了解了基本原理后我们来测试一下它的性能。
    首先用默认的单线程进行测试，自定义的代码如下：
    ```
    from http.server import BaseHTTPRequestHandler
    from http import HTTPStatus

    DEFAULT_MESSAGE = """\
    <!DOCTYPE html>
    <html>
        <head>
            <meta http-equiv="Content-Type" content="text/html;charset=utf-8">
            <title>Hello World</title>
        </head>
        <body>
        <p>hello, world</p>
        </body>
    </html>
    """
    class TestHandler(BaseHTTPRequestHandler):
        def do_GET(self):
            message = DEFAULT_MESSAGE.encode('utf-8')
            self.send_response(HTTPStatus.OK)
            self.send_header("Content-type", "text/html; charse='utf-8'")
            self.send_header('Content-Length', str(len(message)))
            self.end_headers()
    
            self.wfile.write(message)
    if __name__ == "__main__":
        from http.server import HTTPServer
        server = HTTPServer(('localhost', 8000), TestHandler)
        print('test starts')
        server.serve_forever()
    ```
用wrk进行测试 `wrk -t10 -d1m -c200 http://127.0.0.1:8000`结果如下：
![](https://raw.githubusercontent.com/yuuzao/mypicserver/master/img/20200620230910.webp)
单线程情况下平均每秒538个请求，嗯。。。还行。


