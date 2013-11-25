Первый основной тезис node.js заключается в том, что операции ввода/вывода
обходятся дорого:

﻿![][1]

Вообще, это самое большое расточительство в современном программировании — 
ждать, пока завешится операция ввода/вывода. Ниже я приведу список различных 
подходов, так или иначе влияющих на производительность 
(на основе статьи [Sam Rushing][2]):

*   **Синхронные операции**: за раз вы обрабатываете только один запрос, 
    обработка запросов происходит в порядке очереди.
    *Плюсы*: простота.
    *Минусы*: Любой запрос может заблокировать обработку всех остальных.

*   **Создание нового процесса**: каждый раз вы запускаете новый процесс 
    для обработки нового запроса.
    *Плюсы*: простота.
    *Минусы*: `fork()` — это молоток unix-программистов. Он прост 
    в применении, а каждая проблема всегда похожа на гвоздь. Обычно такой подход 
    приводит к плохой масштабируемости: сотни подключений — это сотни запущенных
    процессов.

*   **Потоки**: Вы обрабатываете каждый запрос в отдельном треде.
    *Плюсы*: Простота. В отличии от `fork()`, для запуска треда нужно меньше
    накладных расходов.
    *Минусы*: Ваш комьютер может не поддерживать работу с тредами. Кроме того, 
    программирование тредов очень быстро становится головной болью: приходится
    постоянно беспокоиться об проблемах доступа к общим ресурсам.


Второй основной тезис заключается в том, что обработка каждого нового 
подключения в отдельном треде приводит к большим расходам памяти: к примеру, 
Apache, работая с тредами, потребляет гораздо больше памяти, чем Nginx.

Apache использует многопоточность, в зависимости от настроек, для обработки 
нового запроса [запуская новый тред][3] или [поднимая еще один процесс][4]. 
Вы можете наблюдать как растущим параллельно с ростом количества одновременных 
подключений количеством потребляемой памяти. Nginx и Node.js не используют 
многопоточность, потому как потоки и процессы обходятся бóльшими расходами 
памяти. Они работают в одном потоке, и основаны на событиях. Однопоточность 
избавляет их от расходов на обработку тысяч потоков или процессов.


## **Node.js использует один поток для выполнения всего вашего кода**

Это действительно так. Node.js работает в одном потоке. Вы ничего не можете
выполнить параллельно. Пример ниже заблокирует весь сервер на одну секунду:

    while(new Date().getTime() < now + 1000) {  
       // do nothing  
    }

Пока этот код выполняется, Node.js не будет реагировать на любые запросы
от клиентов, потому что Node.js использует один поток для выполнения всего 
вашего кода. То же самое произойдет, если вы выполните полный тяжелых вычислений
код. Например, измененяющий размер изображения, точно так же заблокирует
обработку других запросов. 


## **…тем не менее, все, кроме вашего кода, запускается параллельно**



Нет ни одного способа выполнить код параллельно в рамках обработки запроса. 
Однако, для того, чтобы избежать блокировки сервера, все операции ввода-вывода 
асинхронны, и используют события для взаимодействия с сервером:

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

Если выполнить этот код обрабатывая один из поступивших запросов, то сервер
приступит к обработке следующих запросов не дожидаясь, пока ответа базы данных.


## 
## Почему такой подход хорош? Когда стоит использовать асинхронность вместо синхронности?

Синхронные операции хороши своей простотой. Особенно, по сравнению с тредами,
где проблема с параллельным доступом к ресурсам так и хочет свести ваш код
к состоянию «WTF?!».



В Node.js вам не нужно волноваться о том, что происходит на бэкэнде: просто, 
работая c I/O, используйте коллбеки. Это позволит вам быть уверенными в том, 
что ваш код не заблокирует работу сервера. В отличии от использования 
дополнительных тредов или процессов, такой подход положительно сказывается 
на производительности.


Асинхронный I/O — это замечательно, потому что операции ввода/вывода гораздо
затретнее, чем простое испольнение кода. К тому же, программа должна заниматься
более полезными делами, чем постоянно ждать I/O.

![][10]



Событийный цикл — это сущность которая перехватывает и обрабатывает внешние 
события и конвертирует их в функции обратного вызова. То есть, вызовы I/O — это
определенные точки, в которых node.js может переключаться от одного запроса 
к другому. В I/O вызове, ваш код сохраняет callback, и возвращает котроль 
рантайму node.js. Callback вызовется позднее, когда актуальные данные будут 
доступны.

<!-- An event loop is “an entity that handles and processes external events and
converts them into callback invocations”. So I/O calls are the points at which 
Node.js can switch from one request to another. At an I/O call, your code saves 
the callback and returns control to the node.js runtime environment. The 
callback will be called later when the data actually is available. -->

Конечно, на стороне бэкэнда, используются процессы и потоки для доступа к DB 
и для выполнения различних процессов. Однако, все это не присутствует явно в 
вашем коде, так что вы не можете беспокоиться об этом, никак кроме как зная, 
что операции I/O взаимодействия, например, с базой данных или с другими
процессами будут асинхронными с точки зрения каждого запроса, и результаты этих
взаимодействий  возвращается через цикл событий обратно в код. В сравнении с 
моделью работы Apache, здесь гораздо меньше тредов, и потраченных на них
ресурсов, потому что здесь нет необходимости создавать треды для каждого 
подключения. Просто когда вам действительно необходимо иметь что-то или выполнить 
что-то в параллельном потоке, то даже тут это обработает нода. 


<!-- Of course, on the backend, there are 
[threads and processes for DB access and process execution][11]. However, these
are not explicitly exposed to your code, so you can’t worry about them other 
than by knowing that I/O interactions e.g. with the database, or with other 
processes will be asynchronous from the perspective of each request since the 
results from those threads are returned via the event loop to your code. 
Compared to the Apache model, there are a lot less threads and thread overhead, 
since threads aren’t needed for each connection; just when you absolutely 
positively must have something else running in parallel and even then the 
management is handled by Node.js. -->

Кроме I/O, node.js предполагает, быстрый ответ на входящие запросы. К примеру, 
[сложные расчеты, загружающие CPU должны быть вынесены в отдельные процессы][12], 
с которыми вы сможете взаимодействовать на основе событий. Либо тяжелые операции
можно вынести из основного потока, используя такие абстракции, как [WebWorkers][13].
Очевидно, это(вебворкеры?) значит, что вы не сможете распараллелить работу вашего кода, 
без использования отдельного фонового потока, с которым вы сможете 
взаимодействовать с помощью событий. В основном, все объекты, которые могут
триггерить события (к примеру, инстансы EventEmitter) поддерживают асинхронное 
взаимодействие с помощью событий(WAT?). Вы можете использовать это в работе 
с блокирующим кодом. Взаимодействие можно организовать с помощью файлов, 
сокетов, или дочерниз процессов, каждый из которых является инстансом EventEmitter
в Node.JS. Поддержка многоядерности так же может быть реализована с помощью 
такого подхода. Советую посмотреть `node-http-proxy`.

<!-- Other than I/O calls, Node.js expects that all requests return quickly; e.g. 
[CPU-intensive work should be split off to another process][12] with which you
can interact as with events, or by using an abstraction like
 [WebWorkers][13]. This (obviously) means that you can’t parallelize your
code without another thread in the background with which you interact via events.
Basically, all objects which emit events (e.g. are instances of EventEmitter) 
support asynchronous evented interaction and you can interact with blocking code
in this manner e.g. using files, sockets or child processes all of which are 
EventEmitters in Node.js.[Multicore can be done][14] using this approach; see
also: node-http-proxy.
 -->

**Внутренняя реализация**
<!-- **Internal implementation** -->

[Внутри себя][15], nodejs использует [libev][16] для реализации событийного цикла.
В приложение к libev nodejs использует [libeio][17], который использует очереди
потоков для обеспечения асинхронного ввода/вывода. Чтобы узнать об этом больше — 
обратитесь к [документации libev][18]

<!-- [Internally][15], node.js relies on [libev][16] to provide the event loop,
which is supplemented by[libeio][17] which uses pooled threads to provide
asynchronous I/O. To learn even more,  have a look at the
[libev documentation][18]. -->

## Так как же мы реализуем асинхронность в node.js?

<!-- ## So how do we do async in Node.js? -->

Tim Caswell определил следующие паттерны в его [потрясающей презентации][20]:

<!-- Tim Caswell describes the patterns in his [excellent presentation][19]: -->

*   Основа — функции. К примеру, мы подходим к функциям, как к данным, 
    разбрасывая их вокруг и выполняя по необходимости

<!-- *   First-class functions. E.g. we pass around functions as data, shuffle them
    around and execute them when needed. -->

*   Композиции функций. Так же известны как анонимные функции, или замыкания, 
    которые выполняются после того, как что-то произойдет в I/O.

<!-- *   Function composition. Also known as having anonymous functions or closures -->
    <!-- that are executed after something happens in the evented I/O. -->

*   Счетчики коллбеков. Повесив функции обратного вызова к определенным событиям,
    вы не можете гарантировать порядок их выполнения. Так что, если вам необходимо
    дождаться выполнения нескольких запросов, то самый простой способ решения
    такой задачи — считать каждую выполненную операцию, и таким образом проверять, 
    все ли необходимые операции были завершены. Это пригодиться, если вам
    обязательно нужно дождаться результатов. Например, считая количество 
    [выполненных запросов к базе данных][20] в коллбеке, мы можем определить,
    когда все наши запросы будут выполненны, и только тогда пойти дальше. 
    Запросы к базам данных запустятся параллельно, потому что I/O библиотека
    поддерживает это (e.g. via connection pooling).

<!-- *   Callback counters. For evented callbacks, you cannot guarantee that I/O
    events are generated in any particular order. So if you need multiple queries to
    complete, usually you just keep count of any parallel I/O operations, and check 
    that all the necessary operations have completed when you absolutely must wait 
    for the result; e.g
   [by counting the number of returned DB queries][20] in the event callback
    and only going further when you have all the data. The queries will run in 
    parallel provided that the I/O library supports this (e.g. via connection 
    pooling
    ). -->

*   Событийные циклы. Как упоминалось раньше, вы можете обернуть блокирующий код
    в событийную абстракцию, запустив его в дочернем процессе, и получив из этого
    процесса данные по окончанию обработки. 

<!-- *   Event loops. As mentioned earlier, you can wrap blocking code into an
    evented abstraction e.g. by running a child process and returning data as it it 
    is processed. -->

Все это действительно очень просто!

<!-- It really is that simple! -->

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