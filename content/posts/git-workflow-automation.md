---
title: "GitHub Workflow Automation Script"
date: 2021-09-08
ShowToc: true
---

A quick blog post to share a bash script that I almost use daily to automate the basic GitHub "commit and push" prcoess. Steps to reproduce: 

- Create a file, for example `codecommit.sh` and copy/paste the content below:

~~~bash
#!/bin/bash

RESET="\033[0m"
BOLD="\033[1m"
YELLOW="\033[38;5;11m"

#Get the argument message
#read -p "Add Commit message: " message
read -p "$(echo -e $BOLD$YELLOW"Add Commit message: "$RESET)" message

#Stage all changes
git add .

#Commit the file(s)
git commit -m "$message"
echo "Added the commit with message: '$message'"

#Get current branch and push changes
current_branch=$(git branch --show-current)
git push origin "$current_branch"

#echo "Pushed changes to '$current_branch' branch"
echo -e $BOLD$YELLOW"Pushed changes to '$current_branch' branch"$RESET
~~~

- Move it to the location `/usr/local/bin` to make this shell script global. MacOS has it in the `PATH` by default. You can verify this by typing `echo $PATH` in the terminal.
- Make this file executable using `chmod +x codecommit.sh`
- Make needful local changes
- Type `codecommit.sh` and voila! 

Feel free to comment below for questions/suggestions. 