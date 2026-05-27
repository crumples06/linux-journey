# Week-1 
*Started at 24/05/2026*

##  cd command:
`cd -` will take me to the last working directory i.e the last directory that i was in. I was able to go my notes folder directory to root as previously i went from root diecrtly to notes, so root was my last working directory  
`cd /` took me to the root of the filesystem, it took me at the absolute start, even before home diarerectory.  
`cd ~` took me to the home directory  
I first thought that `cd /` and `cd ~` were the same, but they are different `cd /` took me to the absoloute start from where the home directory is accessed but the `cd ~` took me to the home directory only  
  
## ls Command:   
There is a pattern for all commands, command options attributes. command is the actual command you are executing, options are the various different options of the command i.e. additional functioanities of the command, attribute is the things you are applying the command on.  
The options for the `ls` command are:  
1. `ls -l` displayed the contents of the directory in long format. At first i didnt understand what it all meant, too much information.  
* It first displays the read write permissions, the first char is the type of file, '-' means regular file, 'd' means directory  
* The next three chars represent the read, write, and execution rights respectively of the file's owner  
* The next three represent permission of the file's group, the next three represent the permissions granted to everybody else  
* The next two words represent the owner of the file and the name of the group  
* Next are a bunch of numbers representing the file size. At first i couldn't understnad them because they are in bytes. Late i found out that if you put `--human-readable` as an option then the size is represented in proper units.  
* Next shows the modification time of the file  
* Finally is the file name  
  
2. `ls --all` shows all file. It shows the hidden files like .git .gitignore etc.  
  
3. `ls --classify` basically classified the files, like a directory will show as directory_name followed by a `/`, similarly execuatable files will be followed by a `*`  
  
I also figured out the you can put multiple options at the same time, for example i can chain the long format, all, and the human readable as `ls -l --all --human-readable`.  
  
## cat command
* `cat filename` will display the file in terminal, it kind of a chore to go through a text file this way as you can't use the mouse to jump to lines, you have to use the keyboard which is annoying in big file  
* `>` is used to merge files, like for example `cat file1 file2 > file3` will merge the contents of file1 and file2 in file3. I also later found out that you can't really do this to an existing file, as the output file will get erased of it's previous data. This command is basically to create a new file containing the data of the input files. It can also be used to create a text file and write in it by `cat > filename.txt`  
* `>>` is for keeping keeping the data intact of the output file, it basically concatenates the data of input files to the output file so that the original data of the output file is not replaced. `cat file1 >> file2`  
* `--number` option will display the file with line numbers
  
## less command
* I stumbled upon the `less` command on pure accident when i was frustated by the tedious navigation of text files in terminal.   
* the `less` command is specifically designed for a smooth navigation of text files.  
* `less filename` opens a display in terminal which is interactive, i could finally scroll the text and even search for specific words.  
* This a way better alternative to `cat` if i just want to read files.  
  
## Wildcards  
* These are basically those patterns that i saw in commands that allow to select many file based some criteria  
* `*` means all file names  
* `g*` means all file name starting with g  
* `d??` means files starting with and having exactly 2 characters after it  
* `[characters]` are basically sets that i can make  
* for example `[abc]` means any file having a name a,b,or c only (i.e. a one character name). similarly `[abc]*` means any file starting with the character a,b or c  
* `[[:upper:]]` means any file that begins with an uppercase character.  
* some more of these in build wildcards are : `[:alnum:]`, `[:alpha:]`, `[:digit:]`, `[:lower:]`. I can also do 'not' by `[![:lower:]]`  
* It's kindof like regex  

## cp command 
* It's for copying files and directories
* So i though file copying would be different in `cp` than in `cat` but it's the same `cp file1 file2` behaves the same as `cat file1 > file2`  
* But one thing is that i could directly copy the file into a directory by `cp file1 directory1`  
* I also copied an entire directory by `cp -R dir1 dir2`. I first thought it will only copy the files within dir1 into dir2 but instead it created an new directory named dir1 inside dir2  
* `-R` means recursive  

## mv command
* renaming : `mv file1 file2`  
* moving : `mv file1 file2 dir1`  
* `mv dir1 dir2` if dir2 exists then move if not them rename  

## rm command
* `rm file1 file2`  
* `rm -r dir1 dir2`  

## I/O Redirection
* As soon as i saw this, i recognized that this is what makes the complicated long commands that i see online possible.  
* i had this problem befoe where `ls` command would give a very long ouptu in standard output (terminal) which was hard to navigate. With `>` i can redirect the standard output into another process/function or into a file.  
* i did `ls -l > list.txt` in usr/bin which made navigating all those files much easier through the text file.  
* `|` is a pipeline, it takes the standard output of a command and feeds it into the standard output of another command. Like `ls -l | less` my favoriate command for looking at big directories wihtout making a new text file using the command i mentioned above.  
  
  
I was playing with the `ls` command and i was trying to only get a select files printed based on a criteria. So i did `ls l*`, it instead printed the directoried starting with l and all the files and directories present in those directories respectively. It was a very interesting find.  

## Arithmetic
One more interesting thing i found out, i can do arithmetic operations in terminal.  
`echo $((2+3+7))` or `echo $(((5**2)+3))`  
  
## Brace Expansion
Used to create multipletext strings from a pattern.  
`echo F-{A,B,C}-B` will print "F-A-B F-B-B F-C-B"  
`echo {1..5}` will print "1 2 3 4 5"  

