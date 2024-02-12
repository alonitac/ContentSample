# Bash Project - Default Bash Profile for New Linux Users

**To be done on Ubuntu systems**

In this project, you will implement a bash script which applies default Bash profile configurations for new added Linux users.

## Preliminaries


1. Open (or clone if you didn't do it yet) [our shared git repo](https://github.com/alonitac/DevOpsAug23) in PyCharm and **pull** the repository (CTRL+T) to get an up-to-date version from GitHub.
2. In PyCharm, [create new branch](https://www.jetbrains.com/help/pycharm/manage-branches.html#create-branch) from branch `main`. Your branch must be named according to the following template:

```text
bash_project/<alias>
```

While changing `<alias>` to your name (choose some nickname if there is another student in the course with the same name as yours). e.g. `bash_project/alonit`.

The branch name must start with `bash_project/`.

Great. Let's get started...

## Background

In many Linux systems, the `/etc/skel` directory provides a way to assure new users are added to your Linux system with default Bash profile configurations.

When adding a new user with the `adduser` command, it will create the user's home directory (usually under `/home/<username>`), and copy files from `/etc/skel` to the new user's home directory. Typically, the `/etc/skel` directory contains a file called `.bash_profile` (you can create it if it doesn’t exist). This file, when copied to the user's home directory, can customize the Bash environment of the user.

The `/etc/skel` directory itself is owned by the `root` user and has restrictive permissions (mode `755`), which means that only the `root` user can modify its contents.

## Guidelines

1. In `/etc/skel` edit `.bash_profile` file using your favorite text editor (`nano`, `vi`, etc...) as detailed below (if the file doesn’t exist, create it). It's highly recommended to backup the original file before you start.
    - Greet the user. e.g. if the username is **john**, the message `Hello john` will be printed to stdout.
    -  Define an environment variable called `COURSE_ID` with a value equal to `DevOpsAug23`.
    - Given a file called `.token` in the home directory of the user, check the file permissions. If the octal representation of the permissions set is different from 600 (read and write by the user only), print the following warning message to the user:

      ```text
      Warning: .token file has too open permissions
      ```
    - Change the `umask` of the user such that the default permissions of new created files will be `r` and `w` for the user and the group only.
    - Add `/home/<username>/usercommands` (while `<username>` is the linux username) to the end of the `PATH` env var.
    - Print the current `date` on screen in ISO 8601 format. The precision should be seconds, and the timezone should be UTC. The correct format should look like: `2022-03-18T08:54:21+00:00`.
    - Define a [command alias](https://tldp.org/LDP/abs/html/aliases.html), as follows - whenever the user is executing `ltxt`, all files with `.txt` extension are printed (tip: wildcards).
    - If it doesn't exist, create the `~/tmp` directory on the user's home dir. If it exists, clean it (delete all the directory's content without removing the dir itself).
    - If it exists, kill the process that is bound to port `8080` (ports will be discussed later on in the course). [This resource](https://stackoverflow.com/questions/11583562/how-to-kill-a-process-running-on-particular-port-in-linux) may help.
    - Feel free to add any additional configurations of yours.
1. Create a new Linux user using `adduser` command. If everything is done well, the new user's Bash environment should have the custom script profile you created.
1. Login to the new user terminal session: `su -l <username>`. Replace `<username>` to the created user. Example for the user's terminal behavior might be:

```console
username@localhost:~$ su -l john
Password:
Hello john

Warning: .token file has too open permissions

The current date is: 2022-03-18T08:54:21+00:00

john@localhost:~$ ltxt
a.txt
john@localhost:~$ echo $COURSE_ID
DevOpsAug23
john@localhost:~$ ls -l tmp
total 0
john@localhost:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/john/usercommands
```

When done, copy the content of your `/etc/skel/.bash_profile` file into `bash_project/.bash_profile` in our shared repo.


## Submission 

Finally, **commit** and **push** your solution. The **only** file that has to be committed is `bash_project/.bash_profile`.

Bravo! 

Your solution has to pass automated tests. Go to **GitHub actions** and make sure your solution has passed the tests. You must see the following message:

```text
WELL DONE!!! you've passed all tests!
```

Otherwise, your solution has to be fixed. Do your changes, commit and push again.

Good Luck!
