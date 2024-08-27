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
