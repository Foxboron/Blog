---
title:  "The State of Hy"
date: 2014-01-17T00:00:00+01:00
---
<link rel="alternate" type="application/rss+xml" title="{{ site.name }}" href="{{ site.url }}/feed.xml">

So, with the recent hipster attitude of posting a "State of *" every year, I thought i'd try and do it for something I have been contributing to for the past 6 months, Hy.


Short introduction
------------------

Hy is a Lisp <del>leeching</del> living off the Python world. It compiles down to Python's AST and is completely bidirectional, you can import Hy into Python and vica versa seamlessly! It just works. Hy is also more portable then normal Python code. Any code you write with Hy can be run on Python 2.6, 2.7, 3.2, 3.3, even 3.4 and pypy! It's a rather young language but have hit a one year mark, but it does not mean Hy dosn't got neato features.


Probably worth mentioning:

* Hy got 1350 downloads from pypi last month alone!
* 36 different contributors in a year.
  * 16 are now core developers with a substantial contribution too the language
* 2 Lightning talks and 5 talks on different meetings/conferences
* tryHy got some attention in the past year. Still up and running; http://try-hy.appspot.com/


Speed
-----

About a year ago paultag (our supreme BDFL) posted a gist comparing Hy with several Python versions, pypy and clojure-py. Just to celebrate the fact that Hy is one year old, I decided to do the same. clojure-py was skipped this year as the project is dead as far as I know, thus nothing have changed on that part.

|           | Python 2.6  | Python 2.7  | Python 3.2  | Python 3.3  | Python 3.4  | Pypy |
| --------- | ----------- | ----------- | ----------- | ----------- | ----------- | ---- |
| Python    | 8.25        | 8.76        | 13.72       | 14.14       | 9.84        | 0.90 |
| Hy        | 14.20       | 14.68       | 15.06       | 15.06       | 10.51       | 3.81 |

As you can see, Python 3.4 is actually the fastest version for running Hy, and we got no clue why! Pypy is still taking time, and we also got no clue why. Help on that front is appreciated.

Original post: https://gist.github.com/paultag/4365555


Macros
------

Hy got fullfledged macros. What are macros? Its those spells Lisp evangelists talk about that you don't understand. Basically, it allows you to alter code with code itself. Thus code and data in Lisp have blurred lines, making Lisp the awesomeness it is.
Macros are also coupled with quote, unquote, splicing and gensyms for making your life easier.
ref: http://docs.hylang.org/en/latest/language/internals.html#hy-macros


Reader Macros
-------------

Hy also recently got reader macros! So you can now have neato ways of altering the syntax as you go!

```clojure
=> (defreader ^ [expr] (print expr))
=> #^(1 2 3 4)
(1 2 3 4)
=> #^"Hello"
"Hello"
=> #^1+2+3+4+3+2
1+2+3+4+3+2
```

What about a literal for tuples? Hy dosn't got that!

```clojure
=> (defreader t [expr] `(, ~@expr))
=> #t(1 2 3)
(1, 2, 3)
```

I should note that because of bad testing (or me), reader macros is currently lacking a few features in 0.9.12, but that will be solved in the upcoming release!
ref: http://docs.hylang.org/en/latest/language/readermacros.html


Error Handling
--------------

We went Extreme Makeover, Nerd Edition.
Before:
<img src="http://i.imgur.com/JmHLkll.png"/>
After:
<img alt="" src="http://i.imgur.com/k3jM0LD.png" />
You can now use Hy without getting long unreadable traceback blown in your face.
Obviously with its shortcomings:

{% highlight clojure %}
=> (map)
Traceback (most recent call last):
File "<input>", line 1, in 
TypeError: map() must have at least two arguments.
{% endhighlight %}
Hy got no way of currently interferring of Python's error handling, so the new error message are limited, but damn nice!


TCO...or just loop and recur it!
--------------------------------

Quite recently Hy got a implementation regarding loop/recur. loop/recur partly solves the problem of having no TCO. It thrives in Clojure because of a limitation in the JVM, and hopefully it will be great addition to Hy!

It is currently only in master, but will be in Hy for 0.9.13.

{% highlight clojure %}
(require hy.contrib.loop)

(defn factorial [n]
  (loop [[i n] [acc 1]]
        (if (zero? i)
          acc
          (recur (dec i) (* acc i)))))

(factorial 1000)
;=> 402387260077093773543702433923003985719374864210714632543799910429938512398629020592044208486969404800479988610197196058631666872994808558901323829669944590997424504087073759918823627727188732519779505950995276120874975462497043601418278094646496291056393887437886487337119181045825783647849977012476632889835955735432513185323958463075557409114262417474349347553428646576611667797396668820291207379143853719588249808126867838374559731746136085379534524221586593201928090878297308431392844403281231558611036976801357304216168747609675871348312025478589320767169132448426236131412508780208000261683151027341827977704784635868170164365024153691398281264810213092761244896359928705114964975419909342221566832572080821333186116811553615836546984046708975602900950537616475847728421889679646244945160765353408198901385442487984959953319101723355556602139450399736280750137837615307127761926849034352625200015888535147331611702103968175921510907788019393178114194545257223865541461062892187960223838971476088506276862967146674697562911234082439208160153780889893964518263243671616762179168909779911903754031274622289988005195444414282012187361745992642956581746628302955570299024324153181617210465832036786906117260158783520751516284225540265170483304226143974286933061690897968482590125458327168226458066526769958652682272807075781391858178889652208164348344825993266043367660176999612831860788386150279465955131156552036093988180612138558600301435694527224206344631797460594682573103790084024432438465657245014402821885252470935190620929023136493273497565513958720559654228749774011413346962715422845862377387538230483865688976461927383814900140767310446640259899490222221765904339901886018566526485061799702356193897017860040811889729918311021171229845901641921068884387121855646124960798722908519296819372388642614839657382291123125024186649353143970137428531926649875337218940694281434118520158014123344828015051399694290153483077644569099073152433278288269864602789864321139083506217095002597389863554277196742822248757586765752344220207573630569498825087968928162753848863396909959826280956121450994871701244516461260379029309120889086942028510640182154399457156805941872748998094254742173582401063677404595741785160829230135358081840096996372524230560855903700624271243416909004153690105933983835777939410970027753472000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000L
{% endhighlight %}


Astor
-----

One of our core devs took over the maintainer role for astor. Astor is a library designed to easily read/write Python AST. Our primary usage of the library is inside `hy2py`, where we use it to generate python code from the AST generated by Hy. A neat example is running `hy --spy`, it hands back the python equivalent of the Hy code you run.

{% highlight clojure %}
=> (defn hello [name] (print "Hello" name))
def hello(name):
    return print('Hello', name)

=> (hello "Hypsters!")
hello('Hypsters!')
Hello Hypsters!
{% endhighlight %}

Check it out!  
https://github.com/berkerpeksag/astor


Future
------

So whats next in store for Hy, except world dominance in the land of Python?

algernon is currently implementing a miniKanren in Hy. It's a neat project and shows just how far Hy can be taken!  
https://github.com/algernon/adderall
http://minikanren.org/


There will be a Hy talk on PyCon2014 by Paul Tagliamonte (Hy's creator and our supreme BDFL). Worth checking out if your interested in Pythons AST and how Hy works.
We are planning on porting Clojure's nrepl project for Hy, so we can <del>leech off</del> utilize all the amazing plugings created for nrepl.
There are small bugs around the place but Hy is coming along. Who knows what the future might bring? Join the Hyve!


Talks & Sources
---------------

Lightning talk on PyCon 2013:  
http://www.youtube.com/watch?feature=player_detailpage&amp;v=1vui-LupKJI#t=975

Boston Python Meetup (January 2013)  
http://www.youtube.com/watch?v=ulekCWvDFVI

PyCon Canada 2013  
http://www.youtube.com/watch?v=n8i2f6X0SkU

PyCon France 2013  
(Warning: French)  
http://www.youtube.com/watch?v=ah9fwabLD70

PyCon Spain 2013  
(Warning: Spanish)  
http://www.youtube.com/watch?v=dUBmaTZ8tpA

PyCon 2014
https://www.youtube.com/watch?v=AmMaN1AokTI

EuroClojure 2014
http://vimeo.com/100977462


Python User Group talk by Benjamin Vulpes
https://www.youtube.com/watch?v=v1aLxXw7bfw

Chicago Python User Group talk by Christopher Webber  
http://www.youtube.com/watch?v=SB9TWabor1k

Blog post explaining the basics of Hy  
http://engineersjourney.wordpress.com/2013/11/27/programming-can-be-fun-with-hy/

learnxinyminutes post about hy written by theanalyst.  
http://learnxinyminutes.com/docs/hy/


This concludes the end of my post, and hopefully I didn't ruin all your future hope for Hy, kinda like this:
<img src="http://i.imgur.com/9KXUOow.gif" alt="" />

Follow us on twitter! https://twitter.com/hylang
There is also a list of Hy developers: https://twitter.com/paultag/lists/hypsters

__Hy Society__  
https://github.com/hylang/hy  
http://docs.hylang.org/en/latest/
