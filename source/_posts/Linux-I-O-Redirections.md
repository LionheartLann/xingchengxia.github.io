---
title: Linux I/O Redirections
date: 2017-08-17 12:06:15
tags:
---
# Linux I/O Redirections — Stdin Stdout Stderr
#lannister
## standard output & standard error output
* stdin: code:0, < or <<
* stdout: code:1, > (overwrite file) or >> (append)
(also 1>, 1>>)
* stderr: code:2, 2> (overwrite file) or 2>>(append)

## Examples
* Save the output on screen to a file.
`ll / > log` 
if the file `log` does not exists, the system will create the file;
else the system will overwrite the file with the output of your command `ll /` 

* save the result of stdout and stderr respectively
`find /etc -name redis.cnf > stdout_right 2> stderr_error`

* the `/dev/null`, when you want to ignore the error output
`find /home -name something 2> /dev/null`

* save stdout and stderr to the same file
this operation require a special way to write commands:  
`find /home -name .bashrc > result 2>&1` 
or  `find /home -name .bashrc &> result`
`2>&` and `&>` without space in between
`2>& 1` is ok

* `stdin`: < & <<
replace keyboard input with the content of a file: 
`cat > catfile < ~/.bashrc`
ends your input with `<<` :
`cat > catfile << "eof(or anything you like)"`


	
