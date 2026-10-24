Machine: [TryHackMe - Battery](https://tryhackme.com/room/battery)

# Enumeration

first i start with a normal nmap scan:

```bash
nmap -sC -sV -oN scan.txt 10.130.181.25
```

<img width="1302" height="386" alt="image" src="https://github.com/user-attachments/assets/bdd3b46b-2552-4e32-8e5a-7fddd7dba973" />

as you can see here there is only two open ports

`22`: ssh
`80`: http

lets get started by investigating the web server on port `80`

<img width="1920" height="1040" alt="image" src="https://github.com/user-attachments/assets/e2f839b4-7785-48e0-ade3-00632598101e" />

as it appears it hosts a simple html page, and there is nothing interesting in the source code or in the js files

so lets fuzz the directories

```bash
gobuster dir -u http://10.130.181.25/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt 
```

<img width="1505" height="444" alt="image" src="https://github.com/user-attachments/assets/f4442c21-c4bc-46a3-94cf-adbc6ba1dd34" />


and as you can see here the output reveals two directories 

`scripts` and `report`

there is nothing interesting in `/scripts` i discovered it carefully

what caught my attention is `/report` which when i visit it, a binary file starts to download 

<img width="609" height="76" alt="image" src="https://github.com/user-attachments/assets/8ba448d9-203f-471d-988c-cdf4b0787175" />

<img width="1919" height="129" alt="image" src="https://github.com/user-attachments/assets/1cd10bb2-28d9-45c6-90f7-efbfff1ae29a" />

once it is downloaded i tried to decompile it online using https://dogbolt.org/

and i found some interesting things

<img width="915" height="882" alt="image" src="https://github.com/user-attachments/assets/8a2bb984-fdd1-4b06-a208-548a8deade7f" />

from the code it looks like a program to manage a bank system

<img width="742" height="187" alt="image" src="https://github.com/user-attachments/assets/e61d8726-6ba6-4bc9-91e2-174ad2f31fcf" />

the program has a login feature and tests the Username and the Password against `guest` and `guest`

<img width="727" height="330" alt="image" src="https://github.com/user-attachments/assets/23cbdfa7-5ba2-4083-acda-bc60effe4152" />

it also has a list of users' emails

<img width="932" height="388" alt="image" src="https://github.com/user-attachments/assets/1d7768e6-2356-45fc-9ed9-b321e8b009a8" />

and it seems to have a special email `admin@bank.a` so it uses it to bypass the `strcmp` condition and prints the message "Password Updated Successfully!"

after all of this i felt that i am at a dead end, nothing interesting and i can't do anything with the information i have and i can't go forward 

but something told me that there is something i haven't discovered yet about the content of that web server 

so i returned again to `gobuster` but this time trying to fuzz with some extensions like `php, js, py, conf, txt`

```bash
gobuster dir -u http://10.130.181.25/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,js,py,txt,conf
```

<img width="787" height="235" alt="image" src="https://github.com/user-attachments/assets/08fd0319-343b-4e65-9e77-2f4abe23de67" />

and yeah, this actually works and as you see i discovered many interesting new dirs

lets focus on `admin.php`

<img width="1920" height="1038" alt="image" src="https://github.com/user-attachments/assets/744c9049-680b-4de5-976a-ed4428c54960" />

a simple page lets you either login or register as a new user

to save your time i tried to register as a new user and examine the website for anything interesting but i didn't find anything

so i thought about the users i discovered in the binary app, specially the `admin@bank.a`

i tried this username with common passwords and it didn't work and also tried sql injection and there was none

so i had to think out of the box

---

# SQL Truncation

i started looking at the source code of that registration page

<img width="1066" height="68" alt="image" src="https://github.com/user-attachments/assets/522d0d96-8a29-4742-9e8a-e2931b8f96b8" />

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

<img width="687" height="56" alt="image" src="https://github.com/user-attachments/assets/b8f16c5c-8544-47b7-9e3a-7b1c4265baad" />

<img width="409" height="106" alt="image" src="https://github.com/user-attachments/assets/bb33c07a-135d-495d-87a1-a7e320fb0396" />

and yeah it works

<img width="518" height="692" alt="image" src="https://github.com/user-attachments/assets/5eeb350c-583e-4068-9105-14f26cafc462" />

now i am able to login as `admin@bank.a` with the password `Admin@123`

<img width="1920" height="811" alt="image" src="https://github.com/user-attachments/assets/1b94996d-95ac-483d-9f1e-dea1a20038f7" />

---
# XXE Injection

<img width="1920" height="1032" alt="image" src="https://github.com/user-attachments/assets/496c289c-d7f5-4536-abbf-5e70f1c45ff7" />

when investigating the command section, this shows a simple page that takes two inputs: `Account Number` and `Remark`. at first look it seems normal, but lets see what is behind it using burp

<img width="713" height="613" alt="image" src="https://github.com/user-attachments/assets/b5c65912-d2b8-4d81-b852-45cdc130ca07" />

this is the request that the command section made, as you can see it sends the data in XML form, so what about XML Injection?

<img width="697" height="505" alt="image" src="https://github.com/user-attachments/assets/88a68470-a96d-4121-b987-bed2074903c4" />

i tried this simple payload to read the content of `/etc/passwd`

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >]><foo>&xxe;</foo>
```

<img width="605" height="553" alt="image" src="https://github.com/user-attachments/assets/d4f9d6b1-8dd7-496c-a328-ab165e6af32f" />

then i tried to read the content of `acc.php` but it didn't work in the normal way

so i used a different technique using `php://filter`

```plain
php://filter/read=convert.base64-encode/resource=acc.php
```

<img width="708" height="569" alt="image" src="https://github.com/user-attachments/assets/6a845b39-7e79-4aee-923f-3cfdf1423364" />

do you think it will work?

<img width="740" height="662" alt="image" src="https://github.com/user-attachments/assets/ec762bcc-00e3-4911-952d-f074dc06bf41" />

yah boy it worked, let's decode the content

<img width="668" height="124" alt="image" src="https://github.com/user-attachments/assets/f0a75d6c-f817-48aa-974a-d6f93b4bcf38" />

once i decoded it i found creds in the code and it seems to work with ssh

<img width="743" height="527" alt="image" src="https://github.com/user-attachments/assets/19d8cdcf-2678-47dd-aa9b-79245957d3bf" />

here we go, these creds worked and now i have registered as `cyber` on this machine

<img width="492" height="127" alt="image" src="https://github.com/user-attachments/assets/825163e3-4993-40bb-a1af-e37ad6231759" />

and here we got the first flag

---

# Privilege Escalation 

<img width="1003" height="169" alt="image" src="https://github.com/user-attachments/assets/5200545a-a7eb-4a93-8787-46260d32ec7b" />

as you can see here in our directory there is a file named `run.py`, this file belongs to root and we can execute it with sudo rights

so lets execute it 

<img width="1004" height="92" alt="image" src="https://github.com/user-attachments/assets/9768aa63-d8bc-489c-a7f6-020c99848c86" />

nothing odd, it writes a message in the terminal, since i can't open this file and read the source code, i have to think of something special 

so i thought about it, the file is in my own directory and since i can make changes in this dir, so why not make my own run.py and execute it with sudo rights and get a privileged shell

<img width="358" height="27" alt="image" src="https://github.com/user-attachments/assets/b84c2fcc-d5c2-4f2b-87b6-4801917201af" />

i changed the file name so i can make my own one

<img width="641" height="413" alt="image" src="https://github.com/user-attachments/assets/a826d618-e1f2-4d28-b031-bf9aa8c3e9de" />



<img width="311" height="53" alt="image" src="https://github.com/user-attachments/assets/4d1ffe72-22ec-490c-a964-cb74a5befebe" />

and lets execute it

<img width="529" height="165" alt="image" src="https://github.com/user-attachments/assets/fd2d02aa-0de8-488c-80c5-65667b7d10a8" />

pretty cool dude we are root

<img width="476" height="139" alt="image" src="https://github.com/user-attachments/assets/1415414b-fc45-4da8-b65c-db3f954845c6" />

and here is the rest of your flags

beautiful ain't it?
