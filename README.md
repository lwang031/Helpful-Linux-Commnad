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

## 6. For loops

Using xargs, you can run a command on a list of files. However, sometimes you’ll want to use more advanced functionality
of bash on a list of files. For example, you might want to set up a pipe of commands, like this:

    # Count the number of while loops in each of the php files in the current directory
    for file in ./*.php; do echo -n “$file”:\ ; grep ‘while’ “$file” | wc -l; done

Let me explain what happens there part by part:

*  `for file in ./*.php`: make a list of all the files ending in “.php” in the current directory, and for each of them, 
run the following code with the variable “$file” set to the filename.

*  `do … done`: indicates the beginning and end of the code inside the for loop. Note that both of them have to start on 
a new line, which is why the semicolons are there. They count for starting a new line.

* `echo -n “$file”:\ `: print the filename, followed by a colon and a space, but not a newline (the “-n” option suppresses
the default extra newline). The backslash makes sure the space is actually printed and not eaten away by bash.

* `grep ‘while’ “$file” | wc -l`: grep reads the file and prints only the lines containing the word 
`while`. These lines are piped to
`“wc -l”`, which counts the amount of lines (word count with option lines).

As you can see, you can easily place multiple commands within the for loop; every bit of code between “do” and “done”
is executed for every file. Imagine the possibilities, especially when you get to know more of the built-in bash 
functionality. Read on for an example of that.

## 7. Ctrl+R

Speaking of long commands: when you need a previously used command again but don’t want to retype it because it’s long 
or complex, there’s a good chance it’s stored in your history file. The quickest way to retrieve and execute it is to 
press Ctrl+R and type a few characters that are part of your command. For example:

    for pid in $(pidof plugin-container); do file /proc/”$pid”/fd/* |
    fgrep /tmp/Flash | cut -d: -f1 | xargs mplayer; done
Now that’s a command that I wouldn’t like to type every time I use it. It’s a one-liner to play the temporary media 
files Flash secretly stores when you play, for example, a YouTube movie. When I want to execute this monster, I only
type the following:

    Ctrl+R
    “pidof”
    Enter
The stuff between quotes is literally typed, as in I press p, then i, and so on. The reason this works is that I almost 
never use the pidof command except in this one-liner, so the most recent command executed that contains “pidof” is almost
always the right one. However, suppose I did recently execute a different command containing “pidof”. By repeatedly pressing
Ctrl+R after typing “pidof” but before pressing Enter, I can cycle through the list of commands until I hit the one I 
meant to execute, and then press Enter. And, last but not least, you can still edit the command you found using Ctrl+R
before hitting Enter; just press the right or left arrow to get out of the history search mode, and edit away!

## 8. String manipulation

There comes a time in the life of every serious bash user when some string manipulation is needed. For example, you 
might have a number of photos of which the filenames start with “DSC”, and you’d like to replace that prefix with 
something more meaningful, like “Vacation2011″. Using a for loop as it was introduced above, you can do this in a flash
(pun intended):

    for file in DSC*; do mv “$file” Vacation2011″${file#DSC}”; done

How does this work? The for loop runs for every file in the current directory of which the name begins with “DSC”. For
every such file, mv is executed to move the file to “Vacation2011<name of the file with DSC stripped from the front>”.
The magic that strips “DSC” happens in “${file#DSC}”, where the “#” indicates that the string after it should be 
removed from the front of the contents of the variable before it. There also exist operations to strip from the back 
of the string (useful for removing file extensions), to search and replace in a string and to extract substrings. 
To learn more about string manipulation, visit Manipulating
Strings on TLDP.

## 9. Dynamic port forwarding

Sometimes you need to access a website that’s only accessible from computers inside a certain network. Or, the other 
way around, sometimes you are in a certain network and you need unobstructed access to the internet, but some 
hyperactive firewall is in the way. If you have ssh access to a computer that does have the internet access you need, 
you can use it as an anonymous tunneling proxy without any additional tools. Just add the following option when you 
ssh into the server:

    ssh -D <port number> user@remotehost

The -D option tells ssh to set up a dynamic port forward on local port <port number>. I like to use port number 1337,
but almost any port between 1024 and 65535 will do. When you’ve logged in on the remote server, you can configure your 
browser to use localhost with the port number you specified earlier as a SOCKS proxy (SOCKS versions 4 and 5 are both 
supported). This setting is usually found in the same place as the other options for using a web proxy. In Firefox, 
look under Preferences => Advanced => Network => Configure how Firefox connects to the Internet. Once you’re browsing 
over this SOCKS proxy, it will appear to the web as if the host you sshed into is browsing.

## 10. The screen command

When working over an ssh connection, commands that take long to execute can seriously get in your way. You have to keep 
the connection open to allow the command to complete, which means that you can’t turn off your computer, and you can’t 
execute a different command without opening a second ssh connection or (temporarily) terminating the running command. 
Both annoyances melt away when you use screen.

The screen command allows you to run multiple terminal sessions inside a single terminal session, and manage the multiple
sessions using hotkeys. Try it! Just execute “screen” in your terminal (install it first if necessary) and see an empty 
terminal opening. Now execute the command “sleep 9999″. This will take quite long and block your terminal. However, 
if you press Ctrl+A, let go of the keys, and then press C, you will get a fresh new terminal ready to take commands. 
The sleep command on the other terminal keeps running without being interrupted. To cycle between the two open terminals,
use Ctrl+A N for next and Ctrl+A P for previous (remember to let go of all keys after pressing Ctrl+A). Finally, to shut
down screen without interrupting the commands that are running, press Ctrl+A D for detach. You will return to your 
original terminal, and if it’s an ssh session, you can exit it without interrupting the commands running in screen. 
To get back to the screen terminals you opened before, execute “screen -R” for reattach. To exit a screen session, just 
exit all terminals in it as you would normally exit terminals (Ctrl+D).

For more information about screen, read its man page by executing `man screen` . It’s a very powerful tool that even 
allows multiple people to use a single terminal at the same time! 

For example: type `man ls`, [go to search informantion about ls](http://unixhelp.ed.ac.uk/CGI/man-cgi?ls) 


