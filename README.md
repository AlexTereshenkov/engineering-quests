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

### A peculiar country code in YAML

You are a data scientist and as part of your data munging operations 
you've got a task to write a simple Python script.
This script needs to read a YAML file you've got from a customer 
and you want to run some data sanity checks.

```text
country_codes:
- DK
- SE
- NO
- DE
- FR
- FI
- ES
```

You've got the [PyYAML](https://pypi.org/project/PyYAML/) package installed 
in your fresh Python virtual environment:

```bash
$ python3 -m venv sandbox
$ source sandbox/bin/activate
$ pip install pyyaml
```

You just finished writing the Python script:

```python
import yaml
with open('data.yaml') as fh:
    countries = yaml.load(fh, Loader=yaml.FullLoader)['country_codes']

# check number of countries read
print(f"Number of countries read is {len(countries)}")

# check that each country code is 2 characters
not_2_chars = [code for code in countries if len(str(code)) != 2]
print(
    f"Number of country codes that are not 2 characters is {len(not_2_chars)}"
)
```

You notice a thing that doesn't make any sense.
When checking that each country code in the input YAML file 
follows the [ISO 3166-1 alpha-2 standard](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2), 
one code appears to be of different length and you are not sure how this could happen.

```text
$ python readfile.py
Number of countries read is 7
Number of country codes that are not of 2 characters is 1
```

Can you figure out what is going on without running the Python script yourself 
and inspecting the variables?

<details>
  <summary>Hint 1</summary>
  Is it possible that YAML specification has any special treatments for any of the values?
</details>

<details>
  <summary>Hint 2</summary>
  
  Will `PyYAML` package read all country codes as strings? What if some of them are read into other data types?
</details>

<details>
  <summary>Hint 3</summary>
  Can you see any of the country codes that can represent a Boolean data type?
</details>

<details>
  <summary>Solution</summary>
  
  Turns out YAML specification does have a special treatment for certain data types such as [Boolean](https://yaml.org/type/bool.html) and the Norway country code (`NO`) was read as `False`.

  If you print the `countries` variable, you'll see:
  ```
  ['DK', 'SE', False, 'DE', 'FR', 'FI', 'ES']
  ```
  
  This issue can be mitigated by using the quotes, `'NO'`, so that
  the value will be read as a string. 
  Alternative solution would be to use the `yaml.BaseLoader` so that everything will be a string by default:

  ```python
  yaml.load(fh, Loader=yaml.BaseLoader)
  ```
  
  There is a useful Python package, `strictyaml`, that parses and validates only a 
  restricted subset of the YAML specification which can be used instead of the `PyYAML`.

  This peculiar problem is a reminder of how fragile our programs 
  working with data can be when we are unaware of any implicit typing.
  
  Resources:
  * [The Norway problem](https://hitchdev.com/strictyaml/why/implicit-typing-removed/)
  * [YAML specification](https://yaml.org/)
  * [PyYAML](https://pypi.org/project/PyYAML/)
  * [strictyaml](https://pypi.org/project/strictyaml/)

</details>

### A failing conda build

You are a build system engineer and you have just finished migrating a Git repository to [Bazel](https://docs.bazel.build/versions/4.1.0/build-ref.html#intro) build system. That wasn't really difficult - you just had to create a bunch of build metadata text files throughout the directory tree. Some time after, a data scientist from another team asks why they can't build a Conda package locally on their Mac laptop in the monorepo after pulling in the latest. They send a snippet of the `conda-build` log:

```
...
running install
running build
running build_py
creating build
error: could not create 'build': File exists
...
```

You are very surprised - the CI build pipeline builds the artifacts with Bazel and Conda just fine already for a few days and there haven't been any problems. What's going on?

<details>
  <summary>Hint 1</summary>

  By following the link to the Bazel homepage, can you see what files are used to store build metadata information when using Bazel?
</details>

<details>
  <summary>Hint 2</summary>

  Your CI does build the artifacts both using Bazel and Conda fine. However, do you remember that your CI is running a Linux operating system and your colleague has a MacOS device?
</details>

<details>
  <summary>Hint 3</summary>

  Now you know that Bazel uses the `BUILD` file to store the build metadata, why does Conda say that the `build` file already exists in a directory that has the file named `BUILD` when run on MacOS?
</details>

<details>
  <summary>Solution</summary>
  
  Turns out that in file systems, filenames can be case-sensitive or case-insensitive. For instance, on Windows, if you create a file `data.txt`, you cannot create another file `DATA.txt` beside it; this is because Windows is case-insensitive. The same applies by default to MacOS as well even though it's a UNIX-like system. So when `conda` attempts to create a file `build`, an existing `BUILD` file doesn't let it.

  Linux, in contrast, has a case-sensitive file system, meaning that you can have `data.txt`, `Data.txt`, and `DATA.txt` stored in the same directory:

  ```
  $ touch data.txt Data.txt DATA.txt
  $ ls *.txt            
  DATA.txt  Data.txt  data.txt
  ```
  
  Case sensitivity is important to be aware of in many cases, particularly when working with the files that can be accessed on different operating systems such as within [source code repositories](https://www.hanselman.com/blog/git-is-casesensitive-and-your-filesystem-may-not-be-weird-folder-merging-on-windows). Having two files, `Makefile` and `makefile`, in a Git repository may cause a lot of confusion and it's best to not let this happen.
  
  Resources:
  * [Case sensitivity](https://en.wikipedia.org/wiki/Case_sensitivity#In_filesystems)
  * [How to check if my HD is case sensitive or not?](https://apple.stackexchange.com/questions/71357/how-to-check-if-my-hd-is-case-sensitive-or-not)
  * [MacOS file system formats](https://support.apple.com/en-gb/guide/disk-utility/dsku19ed921c/mac)
  
</details>

### A perfect terminal command

You are an engineer hacking in your terminal running a Python script. You think you've nailed it down and have found the right arguments that would do the job. However, when you run the script passing the arguments, you constantly get an error message. You've made sure the logic in Python code is right, and the file is free of syntax errors, but the annoying error persists.

In troubleshooting efforts, you stripped everything out of the script leaving only a few dummy arguments for testing:

```python
import argparse

parser = argparse.ArgumentParser(description='Sum integers.')
parser.add_argument('--value1', type=int)
parser.add_argument('--value2', type=int)
parser.add_argument('--value3', type=int)

args = parser.parse_args()
print(sum([args.value1, args.value2, args.value3]))
```

You run the script but the error persists:

```
$ python3 script.py \
    --value1=10 \
    --value2=20 \ 
    --value3=30
usage: script.py [-h] [--value1 VALUE1] [--value2 VALUE2] [--value3 VALUE3]
script.py: error: unrecognized arguments:  
zsh: command not found: --value3=30
```

When running the script from an IDE passing the arguments, it all works, but the terminal command fails somehow. What the heck is going on?

<details>
  <summary>Hint 1</summary>

  What happens if you run the script in a single line, `$ python3 script.py --value1=10 --value2=20 --value3=30`?
</details>

<details>
  <summary>Hint 2</summary>

  From the [bash manual](http://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html):

  > The backslash character `\` may be used to remove any special meaning for the next character read and for line continuation.
  
  Can there be anything peculiar about how the symbol is used at the end of the line?
</details>

<details>
  <summary>Hint 3</summary>

  Have you considered running [Shellcheck](https://www.shellcheck.net/) on your command to see if there are any issues?
</details>

<details>
  <summary>Solution</summary>
  
  If you run Shellcheck on the script, you'll see an issue reported:

  ```
  Line 3:
      --value2=20 \
                  ^-- SC1101 (error): Delete trailing spaces after \ to break line (or use quotes for literal space).
  ```

  This [issue](https://github.com/koalaman/shellcheck/wiki/SC1101) means that if there are spaces after the backslash, the escape will apply to them instead of the line break, and the command will not continue on the next line. To prevent this from happening, you have to delete the trailing spaces to make the line break work correctly.

  It's easy to leave some trailing whitespace which can be a nightmare to troubleshoot. Running a linter such as Shellcheck may save you a ton of time. If hacking in a terminal, watch out for those nasty ones!

  Resources:
  * [Bash manual](http://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html)
  * [Bash handbook](https://github.com/denysdovhan/bash-handbook)
  * [Shellcheck](https://www.shellcheck.net/)
  
</details>

### A very stubborn directory

It's time to write some shell scripts! A nice little Bash script that will save you some time, you think. You need to get a directory from user and delete all the files inside it (keeping the directory itself) as part of a larger workflow. That's easy, so you write:

```bash
#!/bin/bash

function delete_dir() {
    path=$1
    rm -r "${path}/*"
    tree ${path}
}

delete_dir datadir
```

When running `./run.sh` containing the code above (and having `datadir` directory in your `cwd`), you see that the directory still contains files as reported by the `tree` command. Trying to troubleshoot, you run the `rm` command in the terminal:

```
$ rm -r datadir/*
```

And this indeed deletes all the files inside the directory leaving the directory empty. Why on this green Earth doesn't this command clean up the directory when being run from the shell script?

<details>
  <summary>Hint 1</summary>

  What happens if you run the `rm -r "${path}/*"` command in a terminal with `path` variable being set to some path?
</details>

<details>
  <summary>Hint 2</summary>

  Perhaps it's worth reading more about [globbing](https://en.wikipedia.org/wiki/Glob_(programming)) and in particular how globbing works inside quotes, see [Bash: Quoting](https://linux.die.net/man/1/bash).
  
  Can there be anything peculiar about using a wildcard inside quotes?
</details>

<details>
  <summary>Hint 3</summary>

  If you can't put a wilcard to glob all the files in a directory inside quotes, what syntax should then be used?
</details>

<details>
  <summary>Solution</summary>
  
  Globbing doesn't work in either single or double quotes which is why the `rm -r "${path}/*"` command doesn't delete any files - they are not globbed! However, you may still want to use the quotes to be able to support paths containing whitespaces. One of the solutions is to place the wildcard after the quotes: 

   ```bash
   rm -r "${path}"/*
   ````

  Resources:
  * [Bash manual](http://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html)
  * [Bash handbook](https://github.com/denysdovhan/bash-handbook)
  
</details>
