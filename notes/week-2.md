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

