---
layout: post
title:  "Ops Tools - Awk, Sed, Grep"
date:   2014-10-20 12:14:19
categories: devops unix
---
These three commands (```awk```, ```sed``` and ```grep```) center around text manipulation and transformations. They are useful for every developer; oftentimes for simple tasks, it is easier to use one of these three tools rather than write a Python or Ruby script.

###**Awk**

####What does it stand for?
The initials of the creators' names- Aho, Weinberger and Kernighan.

####What does it do?
Awk is a text pattern scanning and processing language. An awk program operates on each line of an input file. For each line of the input file, it sees if there are any pattern-matching instructions (usually regular expressions), if yes, it only operates on lines that match that pattern; otherwise it operates on all lines.

The operations can be sophisticated math and string manipulations, and awk also supports associative arrays (otherwise known as a hash or dictionary).

Awk sees each line as being made up of a number of fields, each separated by a "field separator". Within awk, the first field is referred to as ```$1```, second as ```$2```, etc. The whole line is referred to as ```$0```. So you can output or manipulate specific fields in each line.

####What is the most common syntax used to invoke it?
As a stand-alone command:
```awk '/search_pattern/ { action_to_take_on_matches; another_action; }' file_to_parse```

Or using a pipe:
```cat file.txt | awk '/search_pattern/ { action_to_take_on_matches; another_action; }'```

####What is it useful for?
- Parsing complicated log files so that only the fields you care about are outputted
- Parsing files and outputting one particular field to pipe into another command
- Basic-to-intermediate number processing (using associative arrays)

####More details:
 By default, a field separator is one or more space characters. You can customize what the field separator is by setting the internal variable ```FS```. So ```FS=":"``` will divide a line up according to the position of the ```:```.

Awk can operate on any file; most commonly people use it to operate on std-in through the ```|``` pipe operator.

One other awk pattern to note is:
```
BEGIN { Actions}
{ACTION} # Action for every line in a file
END { Actions }
```
In this case, actions in the BEGIN block will get executed before any of the lines are processed, and the actions in the END block will get executed after all of the lines are processed.

####Example:
```cat ~/.ssh/config | grep Hostname | awk '{print $2}'```
- Print out the 2nd column (the urls) for all of the hosts in the ssh config file.

####Further reading:
- http://gregable.com/2010/09/why-you-should-know-just-little-awk.html
- https://www.digitalocean.com/community/tutorials/how-to-use-the-awk-language-to-manipulate-text-in-linux


###**Sed**

####What does it stand for?
**S** tream **Ed** itor

####What does it do?
Sed performs basic text transformations (replacement, insertion, deletion) on an input stream very quickly and efficiently.

####What is the most common syntax used to invoke it?
```sed -e '/search_pattern/ command' file```
```command``` can be (most commonly):
- ```s``` for search & replace
- ```p``` for print
- ```d``` for delete
- ```i``` for insert
- ```a``` for append

But the most common use for sed is as a search and replace, and the syntax for that is slightly simpler:
```sed 's/old_value/new_value/g' file_name```
The ```s``` in the beginning is for search & replace. And by default it only replaces the first ```old_value``` *on each line* it finds. The ```g``` at the end makes it global (does it for all ```old_value```s in a line).

Another common pattern is searching for a pattern and then replacing that pattern by adding extra characters to it. In this case, use ```&``` to represent the matched string:
```
sed 's/unix/{&}/' file.txt
```
Will replace ```unix``` with ```{unix}```

####What is it useful for?
- It's great for simple text transformations
- Massaging text so that a particular string is removed or changed

####Further reading:
- http://www.grymoire.com/Unix/Sed.html
- http://www-users.york.ac.uk/~mijp1/teaching/2nd_year_Comp_Lab/guides/grep_awk_sed.pdf
- http://sed.sourceforge.net/sed1line.txt (sed one-liners)


###**Grep**

####What does it stand for?
**G** lobal **R** egular **E** xpression **P** rint

####What does it do?
Grep searches input files for a particular search string line by line. If it finds a match, by default it will copy that line to standard output.

####What is the most common syntax used to invoke it?
The simplest way:
```grep "search_pattern" file_name```

Using a pipe:
```cat file_name | grep "search_pattern"```

Output with line numbers:
```grep -n "search_pattern" file_name```

Finding all files in a directory (and its sub-directories) that match a search string:
```grep -nR "search_pattern" directory```

One common/useful flag is ```-C [num]``` which prints out [num] lines before and after a matching line.

####What is it useful for?
- Parsing large files to output only the lines that you need
- Finding files in a directory that contain a search string

####Examples:
```grep -nR "User" /rails/app```
- find all lines that contain ```User``` in the ```/rails/app``` directory and its sub-directories. Output will include line number.
``` tail -f ./development.log | grep "TwitterWorker"```
- tail (stream) the development log file and output only lines that match ```TwitterWorker```

####Further reading:
- http://www.thegeekstuff.com/2009/03/15-practical-unix-grep-command-examples/
- http://www-users.york.ac.uk/~mijp1/teaching/2nd_year_Comp_Lab/guides/grep_awk_sed.pdf
