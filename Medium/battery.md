Machine: [TryHackMe - Battery](https://tryhackme.com/room/battery)

# Enumeration

first i start with a normal nmap scan:

```bash
nmap -sC -sV -oN scan.txt 10.130.181.25
```

![[Pasted image 20261023205607.png]]

as you can see here there is only two open ports

`22`: ssh
`80`: http

lets get started by investigating the web server on port `80`

![[Pasted image 20261023205820.png]]

as it appears it hosts a simple html page, and there is nothing interesting in the source code or in the js files

so lets fuzz the directories

```bash
gobuster dir -u http://10.130.181.25/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt 
```

![[Pasted image 20261023212956.png]]

and as you can see here the output reveals two directories 

`scripts` and `report`

there is nothing interesting in `/scripts` i discovered it carefully

what caught my attention is `/report` which when i visit it, a binary file starts to download 

![[Pasted image 20261023213251.png]]

![[Pasted image 20261023213328.png]]

once it is downloaded i tried to decompile it online using https://dogbolt.org/

and i found some interesting things

![[Pasted image 20261023213611.png]]

from the code it looks like a program to manage a bank system

![[Pasted image 20261023213729.png]]

the program has a login feature and tests the Username and the Password against `guest` and `guest`

![[Pasted image 20261023213829.png]]

it also has a list of users' emails

![[Pasted image 20261023213953.png]]

and it seems to have a special email `admin@bank.a` so it uses it to bypass the `strcmp` condition and prints the message "Password Updated Successfully!"

after all of this i felt that i am at a dead end, nothing interesting and i can't do anything with the information i have and i can't go forward 

but something told me that there is something i haven't discovered yet about the content of that web server 

so i returned again to `gobuster` but this time trying to fuzz with some extensions like `php, js, py, conf, txt`

```bash
gobuster dir -u http://10.130.181.25/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,js,py,txt,conf
```

![[Pasted image 20261023232511.png]]

and yeah, this actually works and as you see i discovered many interesting new dirs

lets focus on `admin.php`

![[Pasted image 20261023232630.png]]

a simple page lets you either login or register as a new user

to save your time i tried to register as a new user and examine the website for anything interesting but i didn't find anything

so i thought about the users i discovered in the binary app, specially the `admin@bank.a`

i tried this username with common passwords and it didn't work and also tried sql injection and there was none

so i had to think out of the box

---

# SQL Truncation

i started looking at the source code of that registration page

![[Pasted image 20261023234354.png]]

and immediately something caught my eye, there was a `maxlength` attribute in the username input, and i had focused on scenarios like this before.

i used to, when i saw a `maxlength` in a registration form, specially in the username or the email field, think about `SQL Truncation`

this is an issue that occurs when the value being entered into a database is too large to fit into the column where it is intended to be stored.

#### lets imagine it:

We have a website that has an `Admin` account

Imagine that the database contains this table:

| Username | Password   |
| -------- | ---------- |
| admin    | Admin@123  |
| beshoy   | Beshoy@123 |

for example if the database is configured so that the username length is limited to just `8` characters

then the attacker comes to register a new account, he will register with this username `admin   XYZ`

Let's count the letters.

`admin` = 5 letters
3 spaces = 3 characters
`XYZ` = 3 letters

The total is `11` characters, but the column only allows for `8`.

If the database truncates the excess data as expected in this example, and that is what is called `SQL Truncation`, the result might be:

`admin   XYZ` (11 characters) -> `admin   ` (5 letters + 3 spaces)

####  So, where’s the problem?  

Some databases handle spaces at the end of a text in such a way that: `admin` = `admin   `

## Exploitation

since the maximum length of the username is `14` characters and `admin@bank.a` is `12` characters.

so we need at least two additional characters to spoof it

the user name that we will use will be `admin@bank.a  XYZ`

![[Pasted image 20261024000953.png]]

![[Pasted image 20261024001018.png]]

and yeah it works

![[Pasted image 20261024001119.png]]

now i am able to login as `admin@bank.a` with the password `Admin@123`

![[Pasted image 20261024001204.png]]

---
# XXE Injection

![[Pasted image 20261024124107.png]]

when investigating the command section, this shows a simple page that takes two inputs: `Account Number` and `Remark`. at first look it seems normal, but lets see what is behind it using burp

![[Pasted image 20261024125251.png]]

this is the request that the command section made, as you can see it sends the data in XML form, so what about XML Injection?

![[Pasted image 20261024125803.png]]

i tried this simple payload to read the content of `/etc/passwd`

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >]><foo>&xxe;</foo>
```

![[Pasted image 20261024130524.png]]

then i tried to read the content of `acc.php` but it didn't work in the normal way

so i used a different technique using `php://filter`

```plain
php://filter/read=convert.base64-encode/resource=acc.php
```

![[Pasted image 20261024130801.png]]

do you think it will work?

![[Pasted image 20261024130854.png]]

yah boy it worked, let's decode the content

![[Screenshot From 2026-10-24 13-30-30(1).jpg]]

once i decoded it i found creds in the code and it seems to work with ssh


![[Pasted image 20261024133451.png]]

here we go, these creds worked and now i have registered as `cyber` on this machine

![[image.jpg]]

and here we got the first flag

---

# Privilege Escalation 

![[Pasted image 20261024140229.png]]

as you can see here in our directory there is a file named `run.py`, this file belongs to root and we can execute it with sudo rights

so lets execute it 

![[Pasted image 20261024140414.png]]

nothing odd, it writes a message in the terminal, since i can't open this file and read the source code, i have to think of something special 

so i thought about it, the file is in my own directory and since i can make changes in this dir, so why not make my own run.py and execute it with sudo rights and get a privileged shell

![[Pasted image 20261024140732.png]]

i changed the file name so i can make my own one

![[Pasted image 20261024141025.png]]

![[Pasted image 20261024141046.png]]

and lets execute it

![[Pasted image 20261024141137.png]]

pretty cool dude we are root

![[Pasted image 20261024141217.png]]

and here is the rest of your flags

beautiful ain't it?
