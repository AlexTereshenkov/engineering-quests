## Introduction

I am pretty sure that over the time working as a software engineer, everyone had a few situations
when they have been presented with a problem that was tricky to solve - maybe it was a bug
that one was chasing for days, some weird operating system behavior, or something really silly
that one couldn't even think of.

In software development industry, the solutions to problems are not always obvious, and you often have to play Sherlock Holmes and investigate.
I think being on this journey and then getting that incredible feeling of joy after finally finding the root cause 
and ultimately fixing the problem is what many of us, engineers, really enjoy.

In this repository, I wanted to collect some of the problems I have faced myself or heard from others
to make it possible to let others try to figure out what was wrong.
Some problems may be rather simple to you; some may be more challenging.
These problems can also be used during the interviews for SRE, support engineers, or software
developers positions because I think they provide a fun and meaningful way to engage an interviewee
into a discussion about how they would approach tackling a problem.

Most of the time, the problem manifestations are very trivial; just a single line of code that breaks or a single error message that's shown.
However, I thought it would be more fun to convert them to little stories so that they are more like
puzzles and are more fun to work with.
Each problem consists of a plot, one or more hints, solution, and some comments and resources to learn more.
The problem plot contains some clues that may be of importance so make sure to pay attention to the details!
The problem's solution can also be very simple, so I had to complicate the plot to provide some distractions
to make your life a bit harder.
Hope you enjoy engaging with these quests and please do feel free to share the problems you have faced so that others could learn something, too!

### A mysteriously failing Python unit test

You are a senior programmer who works for an international software development company 
in the New York office. 
Starting your day, you've pulled the latest changes done by a teammate from the Moscow office some time earlier because
this colleague is facing a problem getting a unit test passed and you've agreed to look into this.
The part of the Python unit test looks like this:

```python
    def get_server_zone(server_name):
        return {"caesar-01": "ZONE1", "caesar-02": "ZONE2"}.get(server_name, "ZONE3")
    ...

    def test_get_server_zone():
        assert get_server_zone("caesar-01") == 'ZONE1'
        assert get_server_zone("сaesar-02") == 'ZONE2'
        assert get_server_zone("caesar-05") == 'ZONE3'
```

When you run `pytest`, one of the assertions fails:

```
    def test_get_server_zone():
        assert get_server_zone("caesar-01") == 'ZONE1'
>       assert get_server_zone("сaesar-02") == 'ZONE2'
E       AssertionError: assert 'ZONE3' == 'ZONE2'
E         - ZONE2
E         ?     ^
E         + ZONE3
E         ?     ^

test_server.py:7: AssertionError
```

What is going on, this doesn't make any sense?

<details>
  <summary>Hint 1</summary>
  The colleague is from Moscow, perhaps he/she is Russian?
</details>

<details>
  <summary>Hint 2</summary>
  Is there any chance the colleague is using multiple language inputs on their keyboard?
</details>

<details>
  <summary>Hint 3</summary>
  Maybe it could be worth inspecting the source code using a hex editor, maybe
  there is something wrong with any of the character encoding or something?
</details>

<details>
  <summary>Hint 4</summary>
  
  How is it possible?

  Python:

  ```
  > hash('c')
  1097105870894525455
  > hash('с')
  5664101110289827702
  > ord('c')
  99
  > ord('с')
  1089
  ```

  Bash:

  ```
  $ echo с | xxd               
  00000000: d181 0a

  echo c | xxd
  00000000: 630a                       
  ```

</details>


<details>
  <summary>Solution</summary>
  
  The unit test breaks because in the word "сaesar-02" the first character is not
  a Latin "c" but instead a Russian "с" (`CYRILLIC SMALL LETTER ES` in Unicode).
  This is why this string can't be found in the dictionary, falling back
  to the default value of `'ZONE3'`.

  If you look at the [Russian keyboard layout](https://en.wikipedia.org/wiki/JCUKEN),
  you'll see that the letter "c" is on the same key both in Russian and English key mapping.
  This means a person could have started to type in Russian, then realized they had to
  switch the input language, they did, and then continued typing in English leaving
  the first character in place.

  It can be useful to be able to use a hex editor to look at the program source code or any text at all really. The [`xxd`](https://linux.die.net/man/1/xxd) command can create 
  a hex dump of a given file or standard input.

  Resources:
  * Python [`ord`](https://docs.python.org/3.4/library/functions.html?highlight=ord#ord) and [`hash`](https://docs.python.org/3.4/library/functions.html?highlight=ord#hash) built-ins
  * Linux [`xxd`](https://linux.die.net/man/1/xxd) command
  * [IDN homograph attacks](https://en.wikipedia.org/wiki/IDN_homograph_attack) 

</details>