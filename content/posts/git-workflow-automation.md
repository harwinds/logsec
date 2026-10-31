---
title: "GitHub Workflow Automation Script"
date: 2021-09-08
description: "A small bash script that stages, commits and pushes your changes to the current branch with one command."
tags: [git, github, bash, automation]
ShowToc: true
---

A quick blog post to share a bash script that I use almost daily to automate the basic GitHub "commit and push" process. Steps to reproduce: 

- Create a file, for example `gitpush.sh`, and copy/paste the content below:

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

- Move it to `/usr/local/bin` to make the script available everywhere. macOS has this folder in the `PATH` by default. You can verify this by typing `echo $PATH` in the terminal.
- Make the file executable using `chmod +x /usr/local/bin/gitpush.sh`
- Make your local changes
- Type `gitpush.sh` and voila! 

Feel free to comment below with questions or suggestions. 
