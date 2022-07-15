# THM-Internal-Writeup

This is my write up of the TryHackMe room "Internal" and will act as a walkthrough of the room as well as practise for my own reporting skills. This was completed using a Kali Linux machine connected to TryHackMe via Openvpn. My assigned Machine IP was 10.10.187.182 so whenever you see this value, replce it with your own. Any constructive feedback is appreciated.

1. The first step as mentioned in the Pre-engagement Briefing is to update the /etc/hosts file using the machines IP so that the subsequent webpages can be reached.

![Screenshot 2022-07-15 at 14 35 57](https://user-images.githubusercontent.com/109357285/179236159-e6076aff-cbad-4253-87f0-76b82e916154.png)

This simply requires any text editor to alter the file *Note* /etc/hosts is Read-Only for most users so running the text editor with sudo is advised.

![Screenshot 2022-07-15 at 14 33 41](https://user-images.githubusercontent.com/109357285/179236535-bdca4cd4-5acc-4fdd-8550-9ee548bb9a75.png)

2. Basic Enumeration: I ran an nmap scan looking for any scripts and versions available, -v to have a better idea of what is being scanned and -p- to check every port while i check out other points of interest. Specifically we find two open ports very quickly.

![Screenshot 2022-07-15 at 14 58 52](https://user-images.githubusercontent.com/109357285/179238358-51f0a6a1-fa3c-4f01-8df1-d22064ca68cb.png)

Both ports 80 (http) and 22 (ssh) are open so navigating to http://YourMachineIpHere results in the following:

![Screenshot 2022-07-15 at 15 01 48](https://user-images.githubusercontent.com/109357285/179238951-43f8de6a-3302-40ed-a882-2b19d0cfe363.png)

The Apache2 Ubuntu Default page, which by itself does not give a huge attack surface but is still useful to know the server is running on Linux (This is also found through the nmap scan).

The next script I ran to see what else might be hidden was to check for more directories. Personally I use Dirbuster but any busting tool works just as well, more importantly I used the Directory-list-2.3-medium.txt that can be commonly found on Kali Linux and the TryHackMe AttackBox.

![Screenshot 2022-07-15 at 15 16 02](https://user-images.githubusercontent.com/109357285/179241795-d0ef6395-cf9e-4d86-a97f-4ea9a42357c8.png)

The recursive nature of the scan returns a lot of results, which is Good! The results seem to be focused around one directory in particular which is /blog so let's try navigating there since it has a lot of potential avenues for attack (if you didn't alter the hosts file in step 1 this is where problems start to occur).

![Screenshot 2022-07-15 at 15 22 02](https://user-images.githubusercontent.com/109357285/179242968-5346073d-19e8-4e44-b5c1-02aaf6d5c9f7.png)
![Screenshot 2022-07-15 at 15 22 11](https://user-images.githubusercontent.com/109357285/179243418-bc92ec0b-23c5-4fff-a7f3-ceb4219780bd.png)

3. Walking the Website

Now that we have a surface to try to attack let's have a look at its functionality and check for any low-hanging fruit.
Firstly there is a search box near the top which does what you would expect, any terms related to the "Hello World" article featured will point you in its direction, anything else and you are shown nothing.

![Screenshot 2022-07-15 at 15 30 19](https://user-images.githubusercontent.com/109357285/179244630-99524655-99af-451e-8443-d0ac817f3a82.png)

A bit of playing around with this suggested to me that the search box is pretty well sanitised and not vulnerable to XSS (at least that I could find).

![Screenshot 2022-07-15 at 15 33 51](https://user-images.githubusercontent.com/109357285/179245334-372950ac-4413-4784-97fd-c9b799e746b5.png)

Looking around the Website more we can see various links that take you to different pages but none of them are of special interest except for 'Log in' which of course is a fun thing to see. As well as this, one of the first things you see on the website is that it is run on wordpress, which initially did'nt register as lodly as it should have how important this was.
Moving over to the login page we are presented with http://machineip/blog/wp-login.php/ Again I looked for some SQLi here but no luck, I was considering some brute forcing at this point but thought checking out wordpress first would be a little less invasive (treating this as a real test I would'nt want to potentially DoS attack the server with hydra straight away).

4. Log in

A fair bit of googling later landed me at the wpscan tool (if like me you had not used this until now, do some reading and check the usage syntax) I noticed that if a standard password attack is used then username enumeration will automatically be done as well so theoretically something as simple as:

![Screenshot 2022-07-15 at 15 53 46](https://user-images.githubusercontent.com/109357285/179248951-767bea68-c999-40d2-b51f-cf63478e48c9.png)

should get us some credentials... and what do you know! (Try finding them yourself as well!)

![Screenshot 2022-07-15 at 15 55 16](https://user-images.githubusercontent.com/109357285/179249238-421f2665-5c77-4be8-bb01-8a454f37285e.png)

Inputting these into the login page and we have access to the admin account for the wordpress site.

![Screenshot 2022-07-15 at 16 03 03](https://user-images.githubusercontent.com/109357285/179250945-8510615f-3cf8-419f-a4e8-7b0ac349ab03.png)

5. Initial access

Theres plenty more to look at here so walk the website again at your leisure, check for easy vulnerabilities in search bars or exposed comments not for our eyes etc. For our purposes though we can find that under Appearance -> Theme editor, we have access to multiple theme files, interestingly some of them are  .php files. The route here is obvious, grab your favourite php reverse shell (I used pentestmonkey) and replace the code already present in the 404 template with your shell. Make sure to edit any lines required to connect to your device (Host, port etc...) and set up a netcat listener.

From here all it takes is navigating to the correct webpage which should be easy to find from walking the website earlier, if not, try http://machineip/blog/wp-content/themes/twentyseventeen/404.php The listener should connect and we now have a shell!

![Screenshot 2022-07-15 at 16 16 08](https://user-images.githubusercontent.com/109357285/179253370-7a2cfbcd-b0b9-4395-96ef-dec20a9ee5bf.png)

You will quickly notice that we cant immediately cd to /home/aubreanna (the user in this machine) so no user.txt flag for us just yet. First yet again lets look around some, learn whats on the machine and get a feel for whats available, ideally we will stumble into something useful to escalate our privilieges.

6. Jenkins

We can look through everything interesting in the system: Bin, boot, dev, etc... but what is useful to us is found in /opt. 

![Screenshot 2022-07-15 at 16 21 36](https://user-images.githubusercontent.com/109357285/179254324-17f100a0-3457-4e23-8883-9e501568a00b.png)

More credentials with the same name as the user! Lets think back to the very start of the machine and look at the nmap scan again, we see that ssh is availabe on port 22, so maybe we can access Aubreannas account there.

![Screenshot 2022-07-15 at 16 24 26](https://user-images.githubusercontent.com/109357285/179255814-ddc7492d-5246-46fa-bf0b-9dd4c21a5523.png)

We're in. Lets grab user.txt while we're here and get ready to go for root.

7. Privilege Escalation

Along with user.txt we ave another interesting file available to us, "jenkins.txt" lets see what it says:

![Screenshot 2022-07-15 at 16 27 27](https://user-images.githubusercontent.com/109357285/179255874-4afd5af7-9e9e-4604-9862-6ad8d570a0b2.png)

This stumped me for some time and took a good amount of reading to fully understand what I was being told here, essentially we have a Jenkins server running "Internally" in the system on a given IP and port, the issue is how to access it. Eventually in "man ssh" I found the documentation for binding an address and a port like this

![Screenshot 2022-07-15 at 16 33 13](https://user-images.githubusercontent.com/109357285/179256929-d4ae5558-7ec2-4381-a702-3523b4389af0.png)

Using this we can reconnect to Aubreannas ssh account and link the jenkins service to our host with the following script:

![Screenshot 2022-07-15 at 16 36 56](https://user-images.githubusercontent.com/109357285/179257499-f2d853c2-bcd5-491e-9118-70ce41cb1dcb.png)

This brings us back to Aubreannas account in the terminal so at a glance nothing has happened, however we have now linked our own machine to the service running on port 8080 so by navigating to localhost:8080 (or 127.0.0.1:8080) we are given:

![Screenshot 2022-07-15 at 16 38 27](https://user-images.githubusercontent.com/109357285/179257825-95987434-86bd-45e8-9c4d-a0fe2b52934c.png)

Getting into Jenkins can be done in a number of different ways, the fact it runs on port 8080 which is the default port for most proxies such as burp or OWASP zap can be annoying but it can be bypassed nonetheless, choose your favourite fuzz attacker/ intruder and using "admin" as a username brute force your way in.

![Screenshot 2022-07-15 at 16 42 18](https://user-images.githubusercontent.com/109357285/179258666-fe06f3cc-9fc4-49e5-b760-97ef057dfde1.png)

In the Jenkins webpage, under the "Manage Jenkins" tab on the left, there is a script console option near the bottom, we are told that it is a groovy script so once again choose your preferred shell in this language and upload it:

![Screenshot 2022-07-15 at 16 47 09](https://user-images.githubusercontent.com/109357285/179259403-3cd12b84-acd5-4cd5-9e6b-c997192d9994.png)

nc -lvnp 4444 before pressing run and we should havve another shell, this time running as jenkins.
Knowing how this machine has treated us before, I looked at the /opt directory first and found note.txt

![Screenshot 2022-07-15 at 16 50 04](https://user-images.githubusercontent.com/109357285/179259944-6787bdf2-4655-4802-9768-76f67c412b2a.png)

Credentials for root! Switching to a different console tab lets try to ssh using these:

![Screenshot 2022-07-15 at 16 52 19](https://user-images.githubusercontent.com/109357285/179260395-5adb4928-de56-4c27-bab9-7bf2268762a0.png)

Success! From here grab root.txt and the room is complete!

Thank you for taking the time to read this, hopefully it was coherent enough to follow and maybe helped get through the room while also learning along the way, the main lesson for me was heavily based on enumeration, in other words do a lot of it to find the cracks available.
