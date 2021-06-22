# django gunicorn gevent greenlet daphne What are they?

If you've written Python web application, you will find a dozen of server framework and deployment components on the Internet

Or If you take over other teammate's project, though they are all Python code, you may found they use various framework and W(A)SGI lib

"Oh, it's django app, I can start it with ```python manage.py runserver```, but why the deployment command is ```gunicorn xxx:xxx ``` instead of `python manage.py runserver`, what is the concurrency capabilities of our system? How many process and thread will the server start? Is it blocking IO or non-blocking IO ? ", you start to get confused

You need to understand how it works to choose the proper framework and tune the configuration

## django

If you visit the official site of [Django](https://github.com/django/django)

> Django is a high-level Python Web framework that encourages rapid development and clean, pragmatic design.

It's easy to write application interface in django, with the `orm` supported, as long as the code is manipulating the `orm` object istead of `SQL` directly, you can easily update your database engine or update your database field

Suppose you're familiar with django app, type in ```python manage.py runserver``` and server started

If there comes a new request, django starts a new thread and serve the coming request by default. Whenever there comes a new request, a new thread will be created

Refer to [my blog](https://blog.csdn.net/qq_31720329/article/details/90295027?spm=1001.2014.3001.5501) If you need the detail of the whole call stack

## gunicorn

Before knowing `gunicorn`, let's see the definition [WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface)

[WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) is defined as 

> The **Web Server Gateway Interface** (**WSGI**, pronounced *whiskey*[[1\]](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface#cite_note-1)[[2\]](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface#cite_note-2) or [*WIZ-ghee*](https://en.wikipedia.org/wiki/Help:Pronunciation_respelling_key)[[3\]](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface#cite_note-3)) is a simple [calling convention](https://en.wikipedia.org/wiki/Calling_convention) for [web servers](https://en.wikipedia.org/wiki/Web_server) to forward requests to [web applications](https://en.wikipedia.org/wiki/Web_application) or [frameworks](https://en.wikipedia.org/wiki/Web_framework) written in the [Python programming language](https://en.wikipedia.org/wiki/Python_(programming_language)). The current version of WSGI, version 1.0.1, is specified in [Python Enhancement Proposal](https://en.wikipedia.org/wiki/Python_Enhancement_Proposal) (PEP) 3333.[[4\]](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface#cite_note-:0-4)
>
> WSGI was originally specified as PEP-333 in 2003.[[5\]](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface#cite_note-5) PEP-3333, published in 2010, updates the specification for [Python 3](https://en.wikipedia.org/wiki/Python_3).

And 

[gunicorn](https://en.wikipedia.org/wiki/Gunicorn) is defined as 

> The **Gunicorn** "Green Unicorn" (pronounced jee-unicorn or gun-i-corn)[[2\]](https://en.wikipedia.org/wiki/Gunicorn#cite_note-2) is a [Python](https://en.wikipedia.org/wiki/Python_(programming_language)) [Web Server Gateway Interface](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) (WSGI) [HTTP server](https://en.wikipedia.org/wiki/Web_server). It is a pre-[fork](https://en.wikipedia.org/wiki/Fork_(operating_system)) worker model, [ported](https://en.wikipedia.org/wiki/Porting) from [Ruby's](https://en.wikipedia.org/wiki/Ruby_(programming_language)) [Unicorn](https://en.wikipedia.org/wiki/Unicorn_(web_server)) project. The Gunicorn server is broadly compatible with a number of [web frameworks](https://en.wikipedia.org/wiki/Web_framework), simply implemented, light on server resources and fairly fast.[[3\]](https://en.wikipedia.org/wiki/Gunicorn#cite_note-3)

