Первый основной тезис node.js заключается в том, что операции ввода/вывода
очень дорогие:

<!-- The first basic thesis of node.js is that I/O is expensive: -->

﻿![][1]


В существующих технологиях, самые большые затраты получаются из-за необходимости
дожидаться завершения операции I/O. Вот несколько способов улучшения 
производительности (от [Sam Rushing][2])(WAT?):

<!-- So the largest waste with current programming technologies comes from waiting
for I/O to complete. There are several ways in which one can deal with the 
performance impact (from[Sam Rushing][2]): -->

*   **Синхронные операции**: за один раз вы можете обработать только один 
    запрос. 
    *Плюсы*: простота, 
    *минусы*: каждый запрос может зависнуть, пока не обработается предыдущий.

<!-- *   **synchronous**: you handle one request at a time, each in turn. *pros*:
    simple
   *cons*: any one request can hold up all the other requests -->

*   **Создание нового процесса**: вы запускаете новый процесс node.js для 
    каждого запросаа.
    *Плюсы*: простота
    *Минусы*: плохая масштабируемость. Сотни подключений — это сотни процессов.
    fork() — это молоток для unix-программистов. Он легко доступен, а каждая
    проблема выглядит как гвоздь. Но обычно так делать не нужно.

<!-- *   **fork a new process**: you start a new process to handle each request. 
    *pros*: easy 
    *cons*: does not scale well, hundreds of connections means hundreds
    of processes. fork() is the Unix programmer’s hammer. Because it’s available, 
    every problem looks like a nail. It’s usually overkill -->

*   **Потоки**: запускать новый поток для обработки каждого запуса.
    *Плюсы*: простота. Добрее к ядру, в отличии от fork(), так как для запуска
    потока необходимо меньше накладных расходов.
    *Минусы*: ваш комьютер может не поддерживать треды. Кроме того, 
    программирование потоков очень быстро становится тяжким трудом. Приходится
    постоянно беспокоиться о проблемах доступа к общим ресурсам.

<!-- *   **threads**: start a new thread to handle each request. *pros*: easy, and
    kinder to the kernel than using fork, since threads usually have much less 
    overhead
   *cons*: your machine may not have threads, and threaded programming can get
    very complicated very fast, with worries about controlling access to shared 
    resources. -->
   


Второй основной тезис заключается в том, что обработка каждого нового 
подключения в отдельном потоке обходиться дорого. [e.g. that graph everyone 
showns about Apache sucking up memory compared to Nginx] <!-- TODO: О чем это? -->

<!-- The second basis thesis is that thread-per-connection is memory-expensive: [e.g
. that graph everyone showns about Apache sucking up memory compared to Nginx
] -->

Apache — использует многопоточность: он создает новый [поток][3] или [процесс][4]
на каждый запрос (???), что именно создаст apache зависит от настроек. Вы можете
наблюдать как увеличивается количество потребляемой памяти, параллельно с ростом
количества одновременных подключений к вашему серверу. Nginx и Node.js не 
используют многопоточность, ботому как потоки и процессы обходятся бóльшими 
расходами памяти. Они работают в одном потоке, и основаны на событиях. Это 
исключает перерасход ресурсов, создаваемый тысячами потоков/процессов, 
благодаря использованию одного потока. 

<!-- Apache is multithreaded: it spawns a [thread per request][3] (or [process][4],
it depends on the conf). You can see how that overhead eats up memory as the 
number of concurrent connections increases and more threads are needed to serve 
multiple simulataneous clients. Nginx and Node.js are not multithreaded, because
threads and processes carry a heavy memory cost. They are single-threaded, but 
event-based. This eliminates the overhead created by thousands of threads/
processes by handling many connections in a single thread. -->


## **Node.js остается однопоточным для вашего кода**

<!-- ## **Node.js keeps a single thread for your code…** -->

Нода действительно запущена в одном потоке. Вы не можете ничего выполнить 
параллельно. К примеру, заставив интерпретатор немного подождать — вы 
заблокируете весь сервер.

<!-- It really is a single thread running: you can’t do any parallel code
execution; doing a “sleep” for example will block the server for one second: -->

    while(new Date().getTime() < now + 1000) {  
       // do nothing  
    }

Пока этот код выполняется, node.js не будет отвечать ни на один запрос 
от клиентов, это происходит потому что node.js использует только один
тред для выполнения вашего кода. Другой пример — если вы запустите код, серьезно
нагружающий процессор, например, изменение размера изображений, то это тоже,
заблокирует обработку других запросов.

<!-- So while that code is running, node.js will not respond to any other requests
from clients, since it only has one thread for executing your code. Or if you 
would have some CPU -intensive code, say, for resizing images, that would still 
block all other requests. -->


## **…тем не менее, все, кроме вашего кода, запускается параллельно**
<!-- ## **…however, everything runs in parallel except your code** -->

Нет ниодного способа выполнить код параллельно в рамках одного запроса. 
Однако, все операции ввода-вывода работают на основе событий и являются 
асинхронными. Следующий ниже пример не будет блокировать ваш сервер:

<!-- There is no way of making code run in parallel within a single request. 
However, all I/O is evented and asynchronous, so the following won’t block the 
server: -->

    c.query(  
       'SELECT SLEEP(20);',  
       function (err, results, fields) {  
         if (err) {  
           throw err;  
         }  
         res.writeHead(200, {'Content-Type': 'text/html'});  
         res.end('<html><head><title>Hello</title></head><body><h1>Return from
    async DB query</h1></body></html
    >');  
         c.end();  
        }  
    ); 

Если вы выполните этот код в рамках обработки одного запроса, то следующие 
запросы, будут обработаны без проблем, пока база данных закончит обработку 
запроса.

<!-- If you do that in one request, other requests can be processed just fine
while the database is running it’s sleep. -->

## Что в этом хорошего в том, что мы переходим от синхронного к параллельному выполнению?

<!-- ## Why is this good? When do we go from sync to async/parallel execution? -->

Синхронные операции — это хорошо, потому что они очень простые, (по сравнению 
с тредами, где проблема с параллельностью стремится придти к состоянию «WTF»).

<!-- Having synchronous execution is good, because it simplifies writing code (
compared to threads, where concurrency issues have a tendency to result in WTFs
). -->

В node.js вам не нужно волноваться о том, что происходит на backend'е(плохой термин):
достаточно использовать фунции обратного вызова, когда вы работаете с I/O; 
Таким образом вы можете быть уверены, что такой код никогда не заблокирует другие
запросы к серверу. Кроме того, такой подход не имеет проблем с потреблением 
ресурсов, в отличии от тредов/процессов (К примеру, как у apache)


<!-- In node.js, you aren’t supposed to worry about what happens in the backend:
just use callbacks when you are doing I/O; and you are guaranteed that your code
is never interrupted and that doing I/O will not block other requests without 
having to incur the costs of thread/process per request (e.g. memory overhead in
Apache
). -->

Асинхронный I/O — это хорошо, потому что операции ввода/вывода более дороги,
чем простое выполнение кода. К тому же, нам стоит заниматься чем-то более полезным
чем просто ждать пока такая операция завершится.

<!-- Having asynchronous I/O is good, because I/O is more expensive than most code
and we should be doing something better than just waiting for I/O. -->

![][10]

An event loop is “an entity that handles and processes external events and
converts them into callback invocations”. So I/O calls are the points at which 
Node.js can switch from one request to another. At an I/O call, your code saves 
the callback and returns control to the node.js runtime environment. The 
callback will be called later when the data actually is available.

Of course, on the backend, there are 
[threads and processes for DB access and process execution][11]. However, these
are not explicitly exposed to your code, so you can’t worry about them other 
than by knowing that I/O interactions e.g. with the database, or with other 
processes will be asynchronous from the perspective of each request since the 
results from those threads are returned via the event loop to your code. 
Compared to the Apache model, there are a lot less threads and thread overhead, 
since threads aren’t needed for each connection; just when you absolutely 
positively must have something else running in parallel and even then the 
management is handled by Node.js.

Other than I/O calls, Node.js expects that all requests return quickly; e.g. 
[CPU-intensive work should be split off to another process][12] with which you
can interact as with events, or by using an abstraction like
 [WebWorkers][13]. This (obviously) means that you can’t parallelize your
code without another thread in the background with which you interact via events.
Basically, all objects which emit events (e.g. are instances of EventEmitter) 
support asynchronous evented interaction and you can interact with blocking code
in this manner e.g. using files, sockets or child processes all of which are 
EventEmitters in Node.js.[Multicore can be done][14] using this approach; see
also: node-http-proxy.

**Internal implementation**

[Internally][15], node.js relies on [libev][16] to provide the event loop,
which is supplemented by[libeio][17] which uses pooled threads to provide
asynchronous I/O. To learn even more,  have a look at the
[libev documentation][18].

## So how do we do async in Node.js?

Tim Caswell describes the patterns in his [excellent presentation][19]:

*   First-class functions. E.g. we pass around functions as data, shuffle them
    around and execute them when needed.
   
*   Function composition. Also known as having anonymous functions or closures
    that are executed after something happens in the evented I/O.
   
*   Callback counters. For evented callbacks, you cannot guarantee that I/O
    events are generated in any particular order. So if you need multiple queries to
    complete, usually you just keep count of any parallel I/O operations, and check 
    that all the necessary operations have completed when you absolutely must wait 
    for the result; e.g
   [by counting the number of returned DB queries][20] in the event callback
    and only going further when you have all the data. The queries will run in 
    parallel provided that the I/O library supports this (e.g. via connection 
    pooling
    ).
*   Event loops. As mentioned earlier, you can wrap blocking code into an
    evented abstraction e.g. by running a child process and returning data as it it 
    is processed.
   

It really is that simple!

 [1]: img/io-cost.png "io-cost"
 [2]: http://www.nightmare.com/medusa/async_sockets.html
 [3]: http://httpd.apache.org/docs/2.0/mod/worker.html
 [4]: http://httpd.apache.org/docs/2.0/mod/prefork.html
 [10]: img/bucket_3.gif "bucket_3"

 [11]: http://stackoverflow.com/questions/3629784/how-is-node-js-inherently-faster-when-it-still-relies-on-threads-internally

 [12]: http://stackoverflow.com/questions/3491811/node-js-and-cpu-intensive-requests
 [13]: http://blog.std.in/2010/07/08/nodejs-webworker-design/

 [14]: http://developer.yahoo.com/blogs/ydn/posts/2010/07/multicore_http_server_with_nodejs/
 [15]: https://github.com/ry/node/tree/master/deps
 [16]: http://software.schmorp.de/pkg/libev.html
 [17]: http://software.schmorp.de/pkg/libeio.html
 [18]: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod
 [19]: http://creationix.com/jsconf.pdf

 [20]: http://stackoverflow.com/questions/4631774/coordinating-parallel-execution-in-node-js