---
title: Using Ruby ancestors to Execute Code via the String class
layout: posts 
---
## tldr/oneliner
`ruby -e '"".class.ancestors[3].system("cat /etc/passwd")'`
## Why?
So I was doing a bit of reading on [SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection), specifically that of [Jinja/python](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#jinja2---remote-code-execution) which looks like this:
{% raw %}
```
{{''.__class__.mro()[1].__subclasses__()[396]('cat flag.txt',shell=True,stdout=-1).communicate()[0].strip()}}{{config.__class__.__init__.__globals__['os'].popen('ls').read()}}
```
{% endraw %}
For an explanation on how these work in python, I recommend checking out [this writeup](https://medium.com/@nyomanpradipta120/ssti-in-flask-jinja2-20b068fdaeee). Essentially, it made me wonder if you could do the same thing in ruby, and you definitely can (at least the accessing other methods part). No pwnz yet from me (though I can't be the first to think of this, so it may be a nothingburger, or just a cool trick) 
## How it works
In ruby, you would start by delcaring a string:
```
somestring = "a string";
```
This would create an object from the [String](https://ruby-doc.org/core-2.7.2/String.html) class. Thus, it would have access to all the methods of the String class. It _also_ has some ancestor classes. You can access these like so:
```
somestring = "a string";
puts somestring.class.ancestors;
```
When you run that, you should see the output:
```
String
Comparable
Object
Kernel
BasicObject
```
This is an array of ancestor methods/classes of the String class. Thus, you can access them by index!
```
somestring = "a string";
puts somestring.class.ancestors[3];
```
When you run that, you should see the output:
```
Kernel
```
We are now accessing the [Kernel](https://ruby-doc.org/core-2.7.2/Kernel.html) module, which has some pretty interesting methods... The _most_ interesting to me is the [system](https://ruby-doc.org/core-2.7.2/Kernel.html#method-i-system) method. Using this method, you can execute arbitrary shell commands.
```
somestring = "a string";
somestring.class.ancestors[3].system("cat /etc/passwd");
```
## PoC
<figure class="image">
    <figcaption>Demo</figcaption>
    <img src="/assets/images/ruby-classes.gif" width="40%"/>
</figure>
So, just a fun learning experience for me on how you can access things in unique ways in Ruby!
