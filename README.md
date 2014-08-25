#Using Linux command easier
The 10 tricks people should know to use linux command.
Note: these tricks apply to bash, which is the default shell on most Linux systems.



## 1. Quoting

To allow you to write awesome scripts, bash attaches a special meaning to many characters (like * & ; | { ! < [ # and 
a lot more). Sometimes, however, these characters are not to be interpreted in any special way, but just to be passed 
unmodified to some command. A commonly used example:

    find . -iname '*.conf'

Notice the single quotes around "*.conf". What if I forgot those? Bash would interpret the *.conf as a glob expression 
and expand it to all files ending in .conf in the current directory. This could prevent find from looking for all 
.conf files in subdirectories, causing unexpected results. Therefore, it's a good habit to always quote anything that 
might contain special characters.

Note that only single quotes prevent every character from being interpreted; double quotes still allow bash to 
interpret some characters. When working with variables, double quotes come in handy to prevent word splitting. 
This is often used in scripting. An example:

    #!/bin/bash
    # This script renames a file to lowercase
    newname="$(echo -n "$1" | tr '[A-Z]‘ ‘[a-z]‘)”
    mv -i “$1″ “$newname”

You can see quite a lot of double quotes in this script, and all of them are necessary to prevent trouble with certain 
filenames. You see, when a filename contains whitespace, bash splits the name on that whitespace when you leave 
variables unquoted. Suppose you have a file named “spaced name.txt”, you put it in a variable (“filename=’spaced 
name.txt’”), and then you try to move it to “unspacedname.txt” by executing “mv $filename unspacedname.txt”. You’ll get 
the error “mv: target `unspacedname.txt’ is not a directory”. This is because mv gets executed like this: “mv spaced 
name.txt unspacedname.txt”. In other words, mv will try to move two files, “spaced” and “name.txt” to “unspacedname.txt”, 
and fail because moving multiple files to a single destination is only allowed when the destination is a directory. 
Putting double quotes around “$filename” solves this issue.

So you see, quoting is a good habit to prevent your commands and scripts from doing unexpected things. Two final notes:
you can’t quote variables using single quotes, because the dollar sign loses its special meaning between single quotes,
and if you ever need to use a literal single quote in some command, you can do so by putting it between double quotes. 
Using \’ between single quotes will not work, because even the backslash it not interpreted between single quotes.

## 2. Process substitution

Ever wanted to diff the outputs of two commands quickly? Of course, you could redirect the output to a temporary file 
for both of them, and diff those files, like this:

    find /etc | sort > local_etc_files
    find /mnt/remote/etc | sort > remote_etc_files
    diff local_etc_files remote_etc_files
    rm local_etc_files remote_etc_files

This would tell you the differences between which files are in /etc on the local computer and a remote one. It takes
four lines, however. Using process substitution, we can do this is just a single line:

    diff <(find /etc | sort) <(find /mnt/remote/etc | sort)

What’s that <(…) syntax? It means “run the command inside it, connect the output to a temporary pipe file and give
that as an argument”. To understand this more thoroughly, try running this:

    echo <(echo test)

Instead of printing “test”, this will print something like “/dev/fd/63″. You see now that the <(…) part is actually 
replaced by a file. This file is a stream from which the output of the command inside <(…) can be read, like this:

    cat <(echo test)

Now this does print “test”! Bash redirects the output of “echo test” to /dev/fd/<something>, gives the path of that
file to cat, and cat reads the output of echo from that file. The shortened diff command above does the same, only for
two slightly more complicated commands. This technique can be applied in any place where a temporary file is needed, 
but it does have a limitation. The temporary file can only be read once before it disappears. There’s no use in saving 
the name of the temporary file. If you need multiple accesses to the output of a program, use an old-fashioned 
temporary file or see if you can use pipes instead.

## 3. The xargs command

Whenever you want to execute a command on multiple files, or for every line of a certain file, xargs is the first tool 
to look at. Here’s an example:

    find . -iname ‘*.php’ -print0 | xargs -0 svn add
Anyone who has ever worked with a version control system like svn probably knows the annoyance of having to svn add 
every newly created code file after a few hours of editing. This command does it for you in an instant. How does it 
work?

“find . -iname *.php -print0″ prints all files in the current directory (“.”) or its subdirectories that end in .php 
(case-insensitive) and separates them by null characters (“-print0″). The null character is never used in filenames, 
while a newline may be, so it is safer to separate by null characters.

“xargs -0 svn add” receives the output of find on its standard input through the pipe (“|”), separates it by null 
characters (“-0″) and feeds the filenames as arguments to svn, after the “add” argument. It minds the limits the system 
imposes on the amount and size of command line arguments, and will run svn multiple times as necessary while still 
invoking it as few times as possible.

To find out more about xargs, read the man page by executing “man xargs”. Using xargs is safer and more versatile than 
using the -exec option of find. For ultimate versatility, however, use the slightly less elegant for loop, which is 
described in the second part of this list.

## 4. Ctrl+U and Ctrl+Y

Do you know that moment when you’re typing a long command, and then suddenly realize you need to execute something else 
first? Especially when working over an SSH connection, when you can’t easily open a second terminal on the same machine,
this can be very annoying. Solution: ensure your cursor is at the end of your current command (shortcut: Ctrl+E), press 
Ctrl+U to get a clean line, type the other command you need to execute first, execute it, then press Ctrl+Y and voila! 
Your long command is back on the line. No mouse needed for copying, just quick hotkeys.

## 5. Using bash as a simple calculator

Sometimes you need to quickly do a calculation that is too large or too important to do using your head. When you’re 
working in a graphic environment, you might just fire up kcalc or gcalctool, but tools like that may not always be 
available or easy to find. Fortunately, you can do basic calculations within bash
itself. For example:

    echo $((3*37+12)) # Outputs 123
    echo $((2**16-1)) # Two to the power of sixteen minus one; outputs 65535
    echo $((103/10)) # Outputs 10, as all these operations are integer arithmetic
    echo $((103%10)) # Outputs 3, which is the remainder of 103 divided by 10

The $((something))-syntax also allows bitwise operations, and as such it interprets the caret (“^”) as a bitwise 
operator. That’s why “**” is used for “to the power of”. The syntax also supports showing the decimal equivalent of a 
hexadecimal or octal number. Here’s an example:

    echo $((0xdeadbeef)) # Outputs 3735928559
    echo $((0127)) # Outputs 87

For more information, read the bash man page using “man bash” and search (using the / key) for “arithmetic evaluation”.
If you want to do floating point calculations, you can use bc (might need to install first):

    echo ‘scale=12; 2.5*2.5′ | bc # Outputs 6.25
    echo ‘scale=12; sqrt(14)’ | bc # Outputs 3.741657386773

Note the setting of the scale variable. Using bc, you can perform floating point operations with any precision you like.
The scale variable controls the amount of decimals behind the dot that are calculated. I used 12 here because kcalc 
uses that amount by default, but you can increase or decrease it as you like. Find out more about what bc can do by 
executing “man bc”. It even supports more advanced mathematical functions, such as the arctangent or the natural 
logarithm.



