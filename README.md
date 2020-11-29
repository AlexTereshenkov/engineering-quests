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

---

### A mysteriously failing Java integration test

You are a software developer contributing to a tiny open-source library 
for a file management system written in Java.
It's late in the evening, and you are pulling the latest changes done by a 
few other collaborators and starting working on adding a few integration tests.
The integration tests for the library you work on are quite simple. 
The library's module you are testing will generate a text file on disk based 
on the user configuration provided.
An existing file containing the expected output, has been pre-created 
and is stored with the project's test harness (the file is just a few lines of YAML).
The file produced during the test run is then compared to the expected file bit-for-bit 
to make sure they contain the same data. 

This is when it gets weird. 
You run the integration test you've just added and it fails and so does all other integration
tests where that file is being used (other tests using other files pass though).

Failing test Java source code (snippet only for brevity):

```java
    ...
    byte[] actual = Files.readAllBytes(
        Paths.get("src/test/resources/conversion/modules/conversion_actual.yaml"));
    byte[] expected = Files.readAllBytes(
        Paths.get("src/test/resources/conversion/modules/conversion_expected.yaml"));
    assertTrue(Arrays.equals(actual, expected));
```

The error message you get when running the test locally:

```
$ /opt/javautils/mvn test
...
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.249 sec <<< FAILURE!
testMain(com.ffmanage.conversion.ITYamlTest)  Time elapsed: 0.248 sec  <<< FAILURE!
java.lang.AssertionError
        at org.junit.Assert.fail(Assert.java:92)
        at org.junit.Assert.assertTrue(Assert.java:43)
        at org.junit.Assert.assertTrue(Assert.java:54)
        at com.ffmanage.conversion.ITYamlTest.testMain(ITYamlTest.java:51)
```

The line 51 is the line that asserts that two byte arrays are equal.

You take a look inside the actual and the expected files and they look identical.
After comparing the files contents quickly and not being able to see what can be the issue, 
you decide to replace the expected file in the test harness with the file 
you've just produced by running the test.
After this, all other tests and your test pass.
You think that the file was corrupted or something so you commit and push your changes.
A week after, a new project contributor posts a bug report in the tracking system claiming 
that the integration tests they wrote earlier that do comparison with the expected file 
you've recently re-created fail however they used to pass on their machine.

The error message the contributor gets when running the test locally:

```
C:\Users\robby> C:\MyDev\utils\mvn test
...
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.249 sec <<< FAILURE!
testMain(com.ffmanage.conversion.ITYamlTest)  Time elapsed: 0.248 sec  <<< FAILURE!
java.lang.AssertionError
        at org.junit.Assert.fail(Assert.java:92)
        at org.junit.Assert.assertTrue(Assert.java:43)
        at org.junit.Assert.assertTrue(Assert.java:54)
        at com.ffmanage.conversion.ITYamlTest.testMain(ITYamlTest.java:51)
```

What do you think is going on?

<details>
  <summary>Hint 1</summary>
  Can you spot any notable differences in your environment and the one of the contributor?
</details>

<details>
  <summary>Hint 2</summary>
  How did you compare the actual and the expected files contents? Maybe there are some
  differences in how text is represented in your environment and the one of the contributor?
</details>

<details>
  <summary>Solution</summary>
 
  Looking at the command line output, you can notice that the contributor is on a Windows machine
  and you are on a Linux machine (based on the path notion used to access the `mvn` executable). 
  There are differences in what control characters are used to signify the 
  end of a line in text files.
  The integration test failed because the byte representation for linefeed/carriage return is different on these
  operating systems and when comparing the bytecode arrays, the assertion fails.
  If your file hadn't have any line breaks, the test would have passed.

  To make the test more robust, you could have compared the lines instead of the bytes, like this:

  ```java
    ...
    List<String> actualStrings = Files.readAllLines(
        Paths.get("src/test/resources/conversion/modules/conversion_actual.yaml"));
    List<String> expectedStrings = Files.readAllLines(
        Paths.get("src/test/resources/conversion/modules/conversion_expected.yaml"));
    assertEquals(actualStrings, expectedStrings);
  ```

  Also, when constructing the text to write to files programmatically 
  (when you know that the files can be accessed in multiple operating systems), make sure
  to use a system dependent line separator that your programming language provides 
  such as `System.lineSeparator()` in Java.
  
  It is also possible to convert text files newline characters by using the `dos2unix` and `unix2dos`
  utilities or any general purpose programming language that supports reading and writing text files.
  Be prepared to see the [`^M`](https://unix.stackexchange.com/questions/32001/what-is-m-and-how-do-i-get-rid-of-it) carriage-return character
  as well when looking at the files originated in Windows.

  Resources:
  * [End Of Line Characters: Peter Benjamin](https://peterbenjamin.com/seminars/crossplatform/texteol.html)
  * [Newline: Wikipedia](https://en.wikipedia.org/wiki/Newline)
  * [`System.lineSeparator()` : Java](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#lineSeparator--)
  * [dos2unix](http://manpages.ubuntu.com/manpages/bionic/man1/dos2unix.1.html)
  * [unix2dos](http://manpages.ubuntu.com/manpages/bionic/en/man1/unix2dos.1.html)

</details>

### Fighting a Python f-string syntax error

You are a network engineer and you've got a task to write a simple Python script.
This script needs to be converted to an executable to make running it easier.
You have run `chmod +x script.py` to make it possible to launch the script from the command line
as an executable in the form of `./script.py arg1 arg2`.

You've verified it's an executable (the `x` permission is present):

```
$ ll
total 4.0K
-rwxr-xr-x 1 username username 138 Nov 29 13:57 script.py
```

This Python script takes a few arguments from the command line and then does some work.
When you run it, you get a `SyntaxError` though.
When you inspect the file in your IDE, no errors are shown and you have been able to successfully 
run a Python formatter on the file along with a couple of linters and no issues were found.

The file contents (truncated for brevity):

```python
#!/usr/bin/env python
import sys

print(f"Fist argument is \\ "
      f"{sys.argv[1]}")
print(f"Second argument is \\ "
      f"{sys.argv[2]}")

...
```

You run:

```
$ ./script.py hello world
  File "./script.py", line 4
    print(f"Fist argument is \\"
                               ^
SyntaxError: invalid syntax
```

You experiment running the `print` commands from the script above in a Python interactive
console and they run fine.
You are not very familiar with Python so you seek for help.
You push the file to a Git repository to make it available to a colleague 
to let them run the script, and they don't get this error, the script runs just fine.
This is what they shared back to you:

```
$ python3 script.py hello world
Fist argument is \ hello
Second argument is \ world
```

How come they can run it and you can't?

<details>
  <summary>Hint 1</summary>
  Can there be any difference in the environments that are used to run the Python script 
  (yours and your colleague's)?
</details>

<details>
  <summary>Hint 2</summary>
  Have the colleague and you run the script using the same command?
</details>

<details>
  <summary>Hint 3</summary>
  When Python script is run as an executable from a Bash shell, what Python interpreter is used to run it?
</details>

<details>
  <summary>Solution</summary>
  You could have thought first that there must be something weird with the escape characters
  in the f-string.
  However, this was just a distraction.
  You may have noticed that your colleague ran the script by specifying the Python interpreter explicitly:

  ```
  $ python3 script.py 
  ```

  You, in contrast, relied on something called a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) that defines what program will be used to run the file (using the program search path to find that program).

  So, with the shebang in the `script.py`, the Python script is run like this:
  
  ```
  $ python script.py hello world
  ```
  
  You may have noticed that the shebang refers to the `python` which by default points
  to Python 2 interpreter.
  The f-strings have been introduced in Python 3.6 and thus a Python script containing
  f-strings will fail to be parsed by a Python 2 interpreter.

  It's worth mentioning that when you have a Python virtual environment activated,
  the `python` directive may point to the `python3` symbolic link inside the virtual
  environment:

  ```
  $ python3 -m venv sampleenv       
  $ source sampleenv/bin/activate
  (sampleenv) $ ll sampleenv/bin | grep "py"              
  lrwxrwxrwx 1 username username    7 Nov 29 14:32 python -> python3
  lrwxrwxrwx 1 username username   47 Nov 29 14:32 python3 -> /usr/bin/python3
  ```
  
  This means that you could have run the script successfully 
  if your Python 3 virtual environment was activated.
  However, for extra clarity, it may be worth to change the shebang and use the `python3` instead.

  It is also important to note that `python3` can point to older Python 3 versions, for instance,
  the Python 3.5 interpreter would also fail to parse a Python program containing f-strings:

  ```
  $ python3.5 -c "var = 10; print(f'{var}')"
    File "<string>", line 1
    var = 10; print(f'{var}')
                           ^
  SyntaxError: invalid syntax
  ```

  To make sure that your programs are supported by multiple Python versions
  (for instance, you can't use [dataclasses](https://docs.python.org/3/library/dataclasses.html)
   that were added in Python 3.7 if your code will run on Python 3.6),
  you'd have to run the tests across multiple Python versions
  which can done using [tox](https://alextereshenkov.github.io/run-python-tests-with-tox-in-docker.html).

  Resources:
  * [Python f-strings](https://realpython.com/python-f-strings/)
  * [Shebang: Wikipedia](https://en.wikipedia.org/wiki/Shebang_(Unix))

</details>
