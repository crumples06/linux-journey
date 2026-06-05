# Week-2
*Started at 31/5/26*

## Variables in scripts
I was playing around with variables in scripts, we can put variables is scripts by for examples title="System Information", Now title is a variable.  
I can call that variable using `$title`.  
One thing i found out was that when i put `title = "System Information"` and then tried to run the script, it showed me 'title command not found'. So basically i have to put the equal to sign just after the varible name, if not then instead of a variable name it is recognized as an command, and since 'title' is not a command it showed that error.  
I can also use environment variable like `$HOSTNAME`  

## Script to html
After writing my html output in a script, i wanted a proper html file that i can run on browser.  
To create a html file i did `./sysinfo > ./sysinfo.html`. Here the `>` is output redirection, it is redirecting the output of the script which is a html code to an html file which i can run.  

## Exit Status  
I learned that every command issue a value to the system when they terminate known as an exit status. It is a number, if it 0 then the command was successfull, if it is anything else than 0 then the command was unsuccessful.
You can if a command was successful or not after its execution by running `echo $?`, it will print the exit status if the previous command.
One confusing thing i learned was that the exit status for `true` is 0 and exit status for `false` is 1. 
So, in linux, 0 means successful/true, and anything else is a failure.

## test command
It's used test a command. It is usually used with 'if' command to perform true/false decisions.
It has two forms, `test command` and `[ expressions/command ]`. The space between the two square brackets is necessary.
Example:
'''
if [ -f .bash.profile ]; then
	echo "You have a bash_profile"
else 
	echo "You don't have a bash profile"
fi
'''
Alternate form,
'''
if [ -f .bash.profile]
then echo "You have a bash_profile"
else echo "You don't have a bash profile"
fi
'''

## evenOdd script
Made a basic script to check if an input is even or odd. The script first checks if the input is integer using regex then it checks if the number is 0 then finally it checks for even using modulo 2.
It's a basic script utilizing majority of what i've learned till now about scripts like reading input, test expressions, exit command, if statements, and basic regex.

## Positional Parameters
`command parameter1 parameter2 ...` this is the format of commands. in the script itself `$0` stores the command, `$1` stores the parameter1, `$2` stores the parameter2, and so on.
`$#` maintains the number of parameters put when command is called. It does not include command name i.e. `$0`.
`shift` is a shell built-in that shifts all positional parameters down by one. `$2` becomes `$1`, `$3` becomes `$2`, and so on.

## Command line processor in script
I added command line processor in my sysinfo script. It now has 3 different arguments it can accept.
`-i` makes it interactive, it basically asks for a file name for the html file.
`-h` is a help argument.
`-f` is a file argument that accepts directly the file name for the output html file.

## Flow Control
### for loop
```bash
for i in {1..5}; do
    echo $i
done
```

### case
```bash
case word in
    patterns ) commands ;;
esac
```

### while
```bash
while [ condition ]; do
	commands
done
```


