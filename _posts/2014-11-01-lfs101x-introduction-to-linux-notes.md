---
layout: post
title:  "LFS101x Introduction to Linux 筆記"
date:   2014-11-01 10:26:00
comments: true
categories: [linux]
tags: [Linux]
---

### 重導向

- [孤島日誌: Unix 重新導向跟 2>&1](http://ibookmen.blogspot.tw/2010/11/unix-2.html)

- [【筆記整理】標準輸入輸出和管線 (Standard Input Output and Pipes) - ghoseliang- 點部落](http://www.dotblogs.com.tw/ghoseliang/archive/2013/05/29/105049.aspx)



Ch08: File Operations
==================

## Section 2: Filesystem Architecture

The /etc directory is the home for system configuration files. It contains no binary programs, although there are some executable scripts. For example, the file resolv.conf tells the system where to go on the network to obtain host name to IP address mappings (DNS). Files like passwd,shadow and group for managing user accounts are found in the /etc directory. 

**System run level scripts are found in subdirectories of /etc**. For example, /etc/rc2.d contains links to scripts for entering and leaving run level 2. The rc directory historically stood for Run Commands. Some distros extend the contents of /etc. For example, Red Hat adds the sysconfig subdirectory that contains more configuration files.

> Written with [StackEdit](https://stackedit.io/).

## Section 3: Comparing Files and File Types

###[8.3.2 Using diff3 and patch](https://courses.edx.org/courses/LinuxFoundationX/LFS101x/2T2014/courseware/be8401e6c3bb415a92ca42d2132d8cea/32348d6970e44913b948458f73dca0b4/)

透過 diff & patch 可用來修改服務的設定檔


## Section 4: Backing Up and Compressing Data


Ch11:  Local Security Principles
==========================

##Section 1: Understanding Linux security

Linux has four types of accounts:

- root
- System
- Normal
- Network


##Section 5: Securing the Boot Process and Hardware Resources

**Pluggable Authentication Modules (PAM)** can be configured to automatically verify that passwords created or modified using the passwd utility are strong enough (what is considered strong enough can also be configured).

You can secure the boot process with a secure password to prevent someone from bypassing the user authentication step. For systems using the GRUB boot loader, for the older GRUB version 1, you can invoke **grub-md5-crypt**  which will prompt you for a password and then encrypt as shown on the adjoining screen.


Chapter 12: Network Operations
==========================

## Section 1: Introduction to Networking

To view the IP address:
> $ /sbin/ip addr show*emphasized text*

To view the routing information:
> $ /sbin/ip route show

| Task | Command |
|------|---------|
| Show current routing table | $ route –n |
| Add static route | $ route add -net address |
| Delete static route | $ route del -net address |


Chapter 13: Manipulating Text
=========================

## Section 1: cat and echo

| Command | Usage |
|---------|-------|
| cat file1 file2 | Concatenate multiple files and display the output; i.e., the entire content of the first file is followed by that of the second file. |
| cat file1 file2 > newfile | Combine multiple files and save the output into a new file. |
| cat file >> existingfile | Append a file to the end of an existing file. |
| cat > file | Any subsequent lines typed will go into the file until CTRL-D is typed. |
| cat >> file | Any subsequent lines are appended to the file until CTRL-D is typed. |


## Section 2: sed and awk

### sed

> Its name is an abbreviation for **stream editor**.

> sed can filter text as well as perform substitutions in data streams, working like a churn-mill.

| Command | Usage |
|---------|-------|
| sed -e command `<filename>` | Specify editing commands at the command line, operate on file and put the output on standard out (e.g., the terminal) |
| sed -f scriptfile `<filename>` | Specify a scriptfile containing sed commands, operate on file and put output on standard out. |
| sed s/pattern/replace_string/ file | Substitute first string occurrence in a line |
| sed s/pattern/replace_string/g file | Substitute all string occurrences in a line |
| sed 1,3s/pattern/replace_string/g file | Substitute all string occurrences in a range of lines |
| sed -i s/pattern/replace_string/g file | Save changes for string substitution in the same file |


### awk

| Command | Usage |
|---------|-------|
| awk ‘command’ var=value file | Specify a command directly at the command line |
| awk -f scriptfile var=value file | Specify a file that contains the script to be executed along with f |
| awk '{ print $0 }' /etc/passwd | Print entire file |
| awk -F: '{ print $1 }' /etc/passwd | Print first field (column) of every line, separated by a space |
| awk -F: '{ print $1 $6 }' /etc/passwd | Print first and sixth field of every line |


## Section 3: File Manipulation Utilities

### sort

> sort is used to rearrange the lines of a text file either in ascending or descending order, according to a sort key. You can also sort by particular fields of a file. The default sort key is the order of the ASCII characters (i.e., essentially alphabetically).

| Syntax | Usage |
|--------|:------|
| sort `<filename>` | Sort the lines in the specified file |
| cat file1 file2 | sort | Append the two files, then sort the lines and display the output on the terminal |
| sort -r `<filename>` | Sort the lines in reverse order |


### uniq

> uniq is used to remove duplicate lines in a text file and is useful for simplifying text display. uniq requires that the duplicate entries to be removed are consecutive. 

> To remove duplicate entries from some files, use the following command: **sort file1 file2 | uniq > file3**
or
**sort -u file1 file2 > file3**


### paste

> paste can be used to create a single file containing all three columns. The different columns are identified based on delimiters (spacing used to separate two fields). For example, delimiters can be a blank space, a tab, or an Enter. In the image provided, a single space is used as the delimiter in all files.

> paste accepts the following options:

> - **-d** delimiters, which specify a list of delimiters to be used instead of tabs for separating consecutive values on a single line. Each delimiter is used in turn; when the list has been exhausted, paste begins again at the first delimiter.
> - **-s**, which causes paste to append the data in series rather than in parallel; that is, in a horizontal rather than vertical fashion.

![paste](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch12_screen27.jpg)


### join

![join](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch12_screen30.jpg)


### split

> split is used to break up (or split) a file into equal-sized segments for easier viewing and manipulation, and is generally used only on relatively large files. 

> By default split breaks up a file into 1,000-line segments.

![split](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch012_screen31.jpg)


##Section 4: grep

> grep is extensively used as a primary text searching tool. It scans files for specified patterns and can be used with regular expressions as well as simple strings as shown in the table.

| Command | Usage |
|---------|:------|
| grep [pattern] `<filename>` | Search for a pattern in a file and print all matching lines |
| grep -v [pattern] `<filename>` | grep -v [pattern] <filename> |
| grep [0-9] `<filename>` | Print the lines that contain the numbers 0 through 9 |
| grep -C 3 [pattern] `<filename>` | Print context of lines (specified number of lines above and below the pattern) for matching the pattern. Here the number of lines is specified as 3. |


## Section 5: Miscellaneous Text Utilities

### tr

> The tr utility is used to translate specified characters into other characters or to delete them. The general syntax is as follows:

> **$ tr [options] set1 [set2]**

| Command | Usage |
|---------|-------|
| $ tr abcdefghijklmnopqrstuvwxyz ABCDEFGHIJKLMNOPQRSTUVWXYZ | Convert lower case to upper case |
| $ tr '{}' '()' `<inputfile>` outputfile | Translate braces into parenthesis |
| $ echo `"This is for testing" \| tr [:space:] '\t'` | Translate white-space to tabs |
| $ echo '"This &nbsp; &nbsp;  is &nbsp; &nbsp; for &nbsp; &nbsp;  testing" \| tr -s [:space:]' | Squeeze repetition of characters using -s |
| $ echo "the geek stuff" \| tr -d 't' | Delete specified characters using -d option |
| $ echo `"my username is 432234" \| tr -cd [:digit:]` | Complement the sets using -c option |
| $ tr -cd [:print:] < file.txt | Remove all non-printable character from a file |
| $ tr -s '\n' ' ' < file.txt | Join all the lines in a file into a single line |


### cut

> cut is used for manipulating column-based files and is designed to extract specific columns. The default column separator is the tab character. A different delimiter can be given as a command option.


## Section 6: Dealing with Large Files and Text-related Utilities

### strings

> strings is used to extract all printable character strings found in the file or files given as arguments. It is useful in locating human readable content embedded in binary files: for text files one can just use **grep**.

## Summary

- **cat**, short for concatenate, is used to read, print and combine files.

- **echo** displays a line of text either on standard output or to place in a file.

- **sed** is a popular stream editor often used to filter and perform substitutions on files and text data streams.

- **awk** is a interpreted programming language typically used as a data extraction and reporting tool.

- **sort** is used to sort text files and output streams in either ascending or descending order.

- **uniq** eliminates duplicate entries in a text file.

- **paste** combines fields from different files and can also extract and combine lines from multiple sources.

- **join** combines lines from two files based on a common field. It works only if files share a common field.

- **split** breaks up a large file into equal-sized segments.

- **Regular expressions** are text strings used for pattern matching. The pattern can be used to search for a specific location, such as the start or end of a line or a word.

- **grep** searches text files and data streams for patterns and can be used with regular expressions.

- **tr** translates characters, copies standard input to standard output, and handles special characters.

- **tee** accepts saves a copy of standard output to a file while still displaying at the terminal.

- **wc** (word count) displays the number of lines, words and characters in a file or group of files.

- **cut** extracts columns from a file.

- **less** views files a page at a time and allows scrolling in both directions.

- **head** displays the first few lines of a file or data stream on standard output. By default it displays 10 lines.

- **tail** displays the last few lines of a file or data stream on standard output. By default it displays 10 lines.

- **strings** extracts printable character strings from binary files.

- The **z command family** is used to read and work with compressed files.


Chapter 14: Printing
=================

## Section 1: Configuration

### Introduction to Printing

The Linux standard for printing software is the Common UNIX Printing System (CUPS).


### CUPS Overview

CUPS is the software that is used behind the scenes to print from applications like a web browser or LibreOffice. It converts page descriptions produced by your application (put a paragraph here, draw a line there, and so forth) and then sends the information to the printer. It acts as a print server for local as well as network printers.


### How Does CUPS Work?

![CUPS Workflow](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch13_screen_05.jpg)


### Scheduler

CUPS is designed around a print scheduler that manages print jobs, handles administrative commands, allows users to query the printer status, and manages the flow of data through all CUPS components.


### Configuration Files

The print scheduler reads server settings from several configuration files, the two most important of which are **cupsd.conf** and **printers.conf**. These and all other CUPS related configuration files are stored under the **/etc/cups/** directory.

**cupsd.conf** is where most system-wide settings are located; it does not contain any printer-specific details. Most of the settings available in this file relate to network security, i.e. which systems can access CUPS network capabilities, how printers are advertised on the local network, what management features are offered, and so on.

**printers.conf** is where you will find the printer-specific settings. For every printer connected to the system, a corresponding section describes the printer’s status and capabilities. This file is generated only after adding a printer to the system and should not be modified by hand.


### Job Files

CUPS stores print requests as files under the **/var/spool/cups** directory.

![Job Files](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch13_screen_08.jpg)

After a printer successfully handles a job, data files are automatically removed. These data files belong to what is commonly known as the **print queue**.


### Log Files

Log files are placed in /var/log/cups and are used by the scheduler to record activities that have taken place. These files include access, error, and page records.

![CUPS Log Files](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch13_screen_09.jpg)


### Filters, Printer Drivers, and Backends

- CUPS uses **filters** to convert job file formats to printable formats.

- Printer drivers contain descriptions for currently connected and configured printers, and are usually stored under **/etc/cups/ppd/**.

- The print data is then sent to the printer through a filter and via a **backend** that helps to locate devices connected to the system.

![CUPS Filters, Printer Drivers, and Backends](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch13_screen_10.jpg)


## Section 2: Printing Operations

### Using lp

**lp** and **lpr** accept command line options that help you perform all operations that the GUI can accomplish. lp is typically used with a file name as an argument.

| Command | Usage |
|---------|-------|
| lp `<filename>` | To print the file to default printer |
| lp -d printer `<filename>` | To print to a specific printer (useful if multiple printers are available) |
| program \| lp<br />echo string \| lp | To print the output of a program |
| lp -n number `<filename>` | To print multiple copies |
| lpoptions -d printer | To set the default printer |
| lpq -a | To show the queue status |
| lpadmin | To configure printer queues |


### Managing Print Jobs

| Command | Usage |
|---------|-------|
| lpstat -p -d | To get a list of available printers, along with their status |
| lpstat -a | To check the status of all connected printers, including job numbers |
| cancel job-id<br />or<br />lprm job-id| To cancel a print job |
| lpmove job-id newprinter | To move a print job to new printer |


## Section 3: Manipulating Postscript and PDF Files

### Working with PostScript

PostScript is a standard page description language. It effectively manages scaling of fonts and vector graphics to provide quality printouts. It is purely a text format that contains the data fed to a PostScript interpreter.

![PostScript](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_Ch13_Screen_42.jpg)

Features of PostScript are:

- It can be used on any printer that is PostScript-compatible; i.e., any modern printer
- Any program that understands the PostScript specification can print to it
- Information about page appearance, etc. is embedded in the page


### Working with enscript

**enscript** is a tool that is used to convert a text file to PostScript and other formats. It also supports Rich Text Format (RTF) and HyperText Markup Language (HTML).

| Command | Usage |
|---------|-------|
| enscript -p psfile.ps textfile.txt | Convert a text file to PostScript (saved to psfile.ps) |
| enscript -n -p psfile.ps textfile.txt | Convert a text file to n columns where n=1-9 (saved in psfile.ps) |
| enscript textfile.txt | Print a text file directly to the default printer |


### Using Additional Tools

![Additional Tools](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch13_Screen_53.jpg)


Chapter 15 : Bash Shell Scripting
===========================

## Section 1: Features and Capabilities

### Introduction to Scripts

![Features of Shell Scripts](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch14_screen03.jpg)


### Command Shell Choices

Linux provides a wide choice of shells; exactly what is available on the system is listed in **/etc/shells**.


### Interactive Example Using bash Scripts

``` bash
#!/bin/bash
# Interactive reading of variables
echo "ENTER YOUR NAME"
read sname
# Display of variable values
echo $sname
```


### Return Values

All shell scripts generate a **return value** upon finishing execution; the value can be set with the <font color='blue'>**exit**</font> statement. 

![Return Values](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch14_screen10.jpg)


### Viewing Return Values

As a script executes, one can check for a specific value or condition and return success or failure as the result. By convention, success is returned as 0, and failure is returned as a non-zero value. An easy way to demonstrate success and failure completion is to execute ls on a file that exists and one that doesn't, as shown in the following example, where the return value is stored in the environment variable represented by <font color='red'>**$?</font>**:

``` bash
$ ls /etc/passwd
/etc/ passwd

$ echo $?
0
```


## Section 2: Syntax

### Basic Syntax and Special Characters

| Character | Description |
|:---------:|-------------|
| **#** | Used to add a comment, except when used as \#, or as #! when starting a script |
| **\\** | Used at the end of a line to indicate continuation on to the next line |
| **;** | Used to interpret what follows as a new command |
| **$** | Indicates what follows is a variable |


### Splitting Long Commands Over Multiple Lines

The concatenation operator ( <font color='red'>**\\**</font> ) is used to concatenate large commands over several lines in the shell.

Example: 

``` bash
scp abc@server1.linux.com:\
/var/ftp/pub/userdata/custdata/read \
abc@server3.linux.co.in:\
/opt/oradba/master/abc/
```


### Putting Multiple Commands on a Single Line

The <font color='red'>**;**</font> (**semicolon**) character is used to separate these commands and execute them sequentially as if they had been typed on separate lines.

The three commands in the following example will all execute even if the ones preceding them fail:

``` bash
$ make ; make install ; make clean
```

However, you may want to abort subsequent commands if one fails. You can do this using the <font color='red'>**&&**</font> (**and**) operator as in:

``` bash
$ make && make install && make clean
```

If the first command fails the second one will never be executed. A final refinement is to use the <font color='red'>**||**</font> (**or**) operator as in:
``` bash
$ cat file1 || cat file2 || cat file3
```


### Functions

The function declaration requires a name which is used to invoke it. The proper syntax is:

``` bash
function_name () {
	command...
}
```

The first argument can be referred to as <font color='red'>**\$1**</font>, the second as <font color='red'>**\$2**</font>, etc.

![function example](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch14_screen15a.jpg)


### Built-in Shell Commands

![Command Types](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_chapter14_screen_15.jpg)

**Compiled applications** are binary executable files that you can find on the filesystem. The shell script always has access to compiled applications such as rm, ls, df, vi, and gzip.

bash has many **built-in commands** which can only be used to display the output within a terminal shell or shell script.


### Command Substitution

At times, you may need to substitute the result of a command as a portion of another command. It can be done in two ways:

- By enclosing the inner command with backticks ( <font color='red'>**`**</font> )

- By enclosing the inner command in <font color='red'>**$( )**</font>


### Exporting Variables

By default, the variables created within a script are available only to the subsequent steps of that script. Any child processes (sub-shells) do not have automatic access to the values of these variables. To make them available to child processes, they must be promoted to environment variables using the export statement as in:

``` bash
export VAR=value
```

or

``` bash
VAR=value ; export VAR
```

While child processes are allowed to modify the value of exported variables, the parent will not see any changes; exported variables are not shared, but only copied.


### Script Parameters

Within a script, the parameter or an argument is represented with a $ and a number. The table lists some of these parameters.

| Parameter | Meaning |
|-----------|---------|
| $0 | Script name |
| $1 | First parameter |
| \$2, \$3, etc. | Second, third parameter, etc. |
| $* | All parameters |
| $# | Number of arguments |


## Section 3: Constructs

### The if Statement

``` bash
if TEST-COMMANDS; then CONSEQUENT-COMMANDS; fi
```

or

``` bash
if condition
then
	statements
else
	statements
fi
```

sample:

``` bash
if [ -f /etc/passwd ]
then
    echo "/etc/passwd exists."
fi
```


### Testing for Files

Note the very common practice of putting “**; then**” on the same line as the if statement.

| Condition | Meaning |
|-----------|---------|
| -e file | Check if the file exists. |
| -d file | Check if the file is a directory. |
| -f file | Check if the file is a regular file (i.e., not a symbolic link, device node, directory, etc.) |
| -s file | Check if the file is of non-zero size. |
| -g file | Check if the file has sgid set. |
| -u file | Check if the file has suid set. |
| -r file | Check if the file is readable. |
| -w file | Check if the file is writable. |
| -x file | Check if the file is executable. |

You can view the full list of file conditions using the command **man 1 test**.


### Example of Testing of Strings

``` bash
if [ string1 == string2 ] ; then
   ACTION
fi
```


### Numerical Tests

| Operator | Meaning |
|----------|---------|
| -eq | Equal to |
| -ne | Not equal to |
| -gt | Greater than |
| -lt | Less than |
| -ge | Greater than or equal to |
| -le | Less than or equal to |


### Arithmetic Expressions

Arithmetic expressions can be evaluated in the following three ways (spaces are important!):

Using the **expr** utility: expr is a standard but somewhat deprecated program. The syntax is as follows:

``` bash
expr 8 + 8
echo $(expr 8 + 8)
```

Using the **$((...))** syntax: This is the built-in shell format. The syntax is as follows:

``` bash
echo $((x+1))
```

Using the built-in shell command **let**. The syntax is as follows:

``` bash
let x=( 1 + 2 ); echo $x
```

> In modern shell scripts the use of expr is better replaced with var=$((...))


Chapter 16: Advanced Bash Scripting
==============================

## Section 1: String Manipulation

### String Manipulation

| Operator | Meaning |
|----------|---------|
| myLen1=${#mystring1} | Saves the length of string1 in the variable myLen1. |


### Parts of a String

**${string:0:1}** Here 0 is the offset in the string (i.e., which character to begin from) where the extraction needs to start and 1 is the number of characters to be extracted.

To extract all characters in a string after a dot (.), use the following expression: **${string#*.}**


## Section 3: The Case Statement

### Structure of the case Statement

``` bash
case expression in
   pattern1) execute commands;;
   pattern2) execute commands;;
   pattern3) execute commands;;
   pattern4) execute commands;;
   * )       execute some default commands or nothing ;;
esac
```


## Section 4: Looping Constructs

### The 'for' Loop

``` bash
for variable-name in list
do
    execute one iteration for each item in the
            list until the list is finished
done
```


### The while Loop

``` bash
while condition is true
do
    Commands for execution
    ----
done
```


### The until loop

``` bash
until condition is false
do
    Commands for execution
    ----
done
```


## Section 5: Script Debugging

### More About Script Debugging

In bash shell scripting, you can run a script in debug mode by doing <font color='blue'>**bash –x ./script_file**</font>. Debug mode helps identify the error because:

It traces and prefixes each command with the <font color='blue'>**+**</font> character.
It displays each command before executing it.
It can debug only selected parts of a script (if desired) with:

``` bash
set -x    # turns on debugging
...
set +x    # turns off debugging
```


### Redirecting Errors to File and Screen

| File stream | Description | File Descriptor |
|-------------|-------------|-----------------|
| stdin | Standard Input, by default the keyboard/terminal for programs run from the command line | 0 |
| stdout | Standard output, by default the screen for programs run from the command line | 1 |
| stderr | Standard error, where output error messages are shown or saved | 2 |


## Section 6: Some Additional Useful Techniques

### Creating Temporary Files and Directories

Temporary files (and directories) are meant to store data for a short time. Usually one arranges it so that these files disappear when the program using them terminates. While you can also use **touch** to create a temporary file, this may make it easy for hackers to gain access to your data.

The best practice is to create random and unpredictable filenames for temporary storage. One way to do this is with the <font color='blue'>**mktemp**</font> utility as in these examples:

The <font color='blue'>**XXXXXXXX**</font> is replaced by the <font color='blue'>**mktemp**</font> utility with random characters to ensure the name of the temporary file cannot be easily predicted and is only known within your program.

| Command | Usage |
|---------|-------|
| TEMP=$(mktemp /tmp/tempfile.XXXXXXXX) | To create a temporary file |
| TEMPDIR=$(mktemp -d /tmp/tempdir.XXXXXXXX) | To create a temporary directory |


### Random Numbers and Data

Such random numbers can be generated by using the <font color='blue'>**$RANDOM**</font>  environment variable, which is derived from the Linux kernel’s built-in random number generator, or by the OpenSSL library function, which uses the FIPS140 algorithm to generate random numbers for encryption


Chapter 17: Processes
==================

## Section 1: Introduction to Processes and Process Attributes

### Process Typs

| Process Type | Description | Example |
|--------------|-------------|---------|
| Interactive Processes | Need to be started by a user, either at a command line or through a graphical interface such as an icon or a menu selection. | bash, firefox, top |
| Batch Processes | Automatic processes which are scheduled from and then disconnected from the terminal. These tasks are queued and work on a FIFO (First In, First Out) basis. | updatedb |
| Daemons | Server processes that run continuously. Many are launched during system startup and then wait for a user or system request indicating that their service is required. | httpd, xinetd, sshd |
| Threads | Lightweight processes. These are tasks that run under the umbrella of a main process, sharing memory and other resources, but are scheduled and run by the system on an individual basis. An individual thread can end without terminating the whole process and a process can create new threads at any time. Many non-trivial programs are multi-threaded. | gnome-terminal, firefox |
| Kernel Threads | Kernel tasks that users neither start nor terminate and have little control over. These may perform actions like moving a thread from one CPU to another, or making sure input/output operations to disk are completed. | kswapd0, migration, ksoftirqd |


### Process Scheduling and States

When a process is in a so-called <font color='blue'>**running**</font> state, it means it is either currently executing instructions on a CPU, or is waiting for a share (or time slice) so it can run. A critical kernel routine called the <font color='blue'>**scheduler**</font> constantly shifts processes in and out of the CPU, sharing time according to relative priority, how much time is needed and how much has already been granted to a task. All processes in this state reside on what is called a <font color='blue'>**run queue**</font> and on a computer with multiple CPUs, or cores, there is a run queue on each.

However, sometimes processes go into what is called a <font color='blue'>**sleep**</font> state, generally when they are waiting for something to happen before they can resume, perhaps for the user to type something. In this condition a process is sitting in a <font color='blue'>**wait**</font> queue.


### Process and Thread IDs

At any given time there are always multiple processes being executed. The operating system keeps track of them by assigning each a unique process ID (**PID**) number. The PID is used to track process state, cpu usage, memory use, precisely where resources are located in memory, and other characteristics.

New PIDs are usually assigned in ascending order as processes are born. Thus PID 1 denotes the init process (initialization process), and succeeding processes are gradually assigned higher numbers.

| ID Type | Description |
|---------|-------------|
| Process ID (PID) | Unique Process ID number |
| Parent Process ID (PPID) | Process (Parent) that started this process |
| Thread ID (TID) | Thread ID number. This is the same as the PID for single-threaded processes. For a multi-threaded process, each thread shares the same PID but has a unique TID. |


### User and Group IDs

![User and Group IDs](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch16_screen07.jpg)


### More About Priorities

The priority for a process can be set by specifying a **nice value**, or **niceness**, for the process. The lower the nice value, the higher the priority. Low values are assigned to important processes, while high values are assigned to processes that can wait longer.

![nice value](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch16_screen08.jpg)

You can also assign a so-called **real-time priority** to time-sensitive tasks, such as controlling machines through a computer or collecting incoming data. This is just a very high priority and is not to be confused with what is called **hard real time** which is conceptually different, and has more to do with making sure a job gets completed within a very well-defined time window.


## Section 2: Listing Processes

### The ps Command (System V Style)

ps provides information about currently running processes, keyed by PID. If you want a repetitive update of this status, you can use top or commonly installed variants such as htop or atop from the command line, or invoke your distribution's graphical system monitor application.


### The ps Command (BSD Style)

ps has another style of option specification which stems from the BSD variety of UNIX, where options are specified without preceding dashes. For example, the command <font color='blue'>**ps aux**</font> displays all processes of all users. The command ps axo allows you to specify which attributes you want to view.


### The Process Tree

**pstree** displays the processes running on the system in the form of a tree diagram showing the relationship between a process and its parent process and any other processes that it created. Repeated entries of a process are not displayed, and threads are displayed in curly braces.


### Third Line of the top Output

The third line of the top output indicates how the CPU time is being divided between the users (<font color='blue'>**us**</font>) and the kernel (<font color='blue'>**sy**</font>) by displaying the percentage of CPU time used for each.

The percentage of user jobs running at a lower priority (<font color='blue'>**niceness - ni**</font>) is then listed. Idle mode (<font color='blue'>**id**</font>) should be low if the load average is high, and vice versa. The percentage of jobs waiting (wa) for I/O is listed. Interrupts include the percentage of hardware (hi) vs. software interrupts (si). Steal time (st) is generally used with virtual machines, which has some of its idle CPU time taken for other uses.


### Process List of the top Output

Each line in the process list of the top output displays information about a process. By default, processes are ordered by highest CPU usage. The following information about each process is displayed:

- Process Identification Number (**PID**)
- Process owner (**USER**)
- Priority (**PR**) and nice values (**NI**)
- Virtual (**VIRT**), physical (**RES**), and shared memory (**SHR**)
- Status (**S**)
- Percentage of CPU (**%CPU**) and memory (**%MEM**) used
- Execution time (**TIME+**)
- Command (**COMMAND**)


### Interactive Keys with top

| Command | Output |
|:-------:|--------|
| t | Display or hide summary information (rows 2 and 3) |
| m | Display or hide memory information (rows 4 and 5) |
| A | Sort the process list by top resource consumers |
| r | Renice (change the priority of) a specific processes |
| k | Kill a specific process |
| f | Enter the top configuration screen |
| o | Interactively select a new sort order in the process list |


## Section 3: Process Metrics and Process Control

### Load Averages

Load average is the average of the load number for a given period of time. It takes into account processes that are:

- Actively running on a CPU.
- Considered runnable, but waiting for a CPU to become available.
- Sleeping: i.e., waiting for some kind of resource (typically, I/O) to become available.

The load average can be obtained by running <font color='blue'>**w**</font>, <font color='blue'>**top**</font> or <font color='blue'>**uptime**</font>.


### Interpreting Load Averages

The load average is displayed using three different sets of numbers.

The last piece of information is the average load of the system. Assuming our system is a single-CPU system, the 0.25 means that for the **past minute**, on average, the system has been 25% utilized. 0.12 in the next position means that over the **past 5 minutes**, on average, the system has been 12% utilized; and 0.15 in the final position means that over the **past 15 minutes**, on average, the system has been 15% utilized. If we saw a value of 1.00 in the second position, that would imply that the single-CPU system was 100% utilized, on average, over the past 5 minutes; this is good if we want to fully use a system. A value over 1.00 for a single-CPU system implies that the system was over-utilized: there were more processes needing CPU than CPU was available.


### Background and Foreground Processes

The background job will be executed at lower priority, which, in turn, will allow smooth execution of the interactive tasks, and you can type other commands in the terminal window while the background job is running.

By default all jobs are executed in the foreground. You can put a job in the background by suffixing <font color='red'>**&**</font> to the command, for example: **updatedb &**

You can either use **CTRL-Z** to suspend a foreground job or **CTRL-C** to terminate a foreground job and can always use the <font color='blue'>**bg**</font> and <font color='blue'>**fg**</font> commands to run a process in the background and foreground, respectively.


### Managing Jobs

The **jobs** utility displays all jobs running in background. The display shows the job ID, state, and command name, as shown here.

**jobs -l** provides a the same information as jobs including the PID of the background jobs.


## Section 4: Starting Processes in the Future

### Scheduling Future Processes using at

![at](https://courses.edx.org/c4x/LinuxFoundationX/LFS101x/asset/LFS01_ch16_screen41b.jpg)


### cron

<font color='blue'>**cron**</font> is a time-based scheduling utility program. It can launch routine background jobs at specific times and/or days on an on-going basis. cron is driven by a configuration file called <font color='blue'>**/etc/crontab**</font> (cron table) which contains the various shell commands that need to be run at the properly scheduled times. There are both system-wide crontab files and individual user-based ones. Each line of a crontab file represents a job, and is composed of a so-called CRON expression, followed by a shell command to execute.

The **crontab -e** command will open the crontab editor to edit existing jobs or to create new jobs. Each line of the crontab file will contain 6 fields:

| Fields | Description | Values |
|--------|-------------|---------|
| MIN | Minutes | 0 to 59 |
| HOUR | Hour field | 0 to 23 |
| DOM | Day of Month | 1-31 |
| MON | Month field | 1-12 |
| DOW | Day Of Week | 0-6 (0 = Sunday) |
| CMD | Command | Any command to be executed |

Examples:
1. The entry "**<font color='red'>* * * * * /usr/local/bin/execute/this/script.sh</font>**" will schedule a job to execute 'script.sh' every minute of every hour of every day of the month, and every month and every day in the week.

2. The entry "**<font color='red'>30 08 10 06 * /home/sysadmin/full-backup**</font>" will schedule a full-backup at 8.30am, 10-June irrespective of the day of the week.


### sleep

sleep suspends execution for at least the specified period of time, which can be given as the number of seconds (the default), minutes, hours or days. After that time has passed (or an interrupting signal has been received) execution will resume.

Syntax: 
> sleep NUMBER[SUFFIX]...

where SUFFIX may be:

- **s** for seconds (the default)
- **m** for minutes
- **h** for hours
- **d** for days