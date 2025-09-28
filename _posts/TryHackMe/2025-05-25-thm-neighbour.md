---
title: "TryHackMe - Neighbour Walkthrough"
#description: TryHackMe: Neighbour writeup.
date: 2025-05-25 00:00:00 +0800
last_modified_at: 2025-05-25 00:00:00 +0800
categories: [Writeups, TryHackMe]
tags: [TryHackMe,IDOR,Easy,CTF,Web App Pentesting]
image:
  path: https://tryhackme-images.s3.amazonaws.com/room-icons/5e9c5d0148cf664325c8a075-1737130517336
  alt: Neighbour room logo
---

Room: [Neighbour](https://tryhackme.com/room/neighbour)<br>
Author: [cmnatic](https://tryhackme.com/p/cmnatic)<br>
Difficulty: <span style="color: #03b303"> **Easy** </span>

In this challenge we will be exploiting **IDOR** or the **Insecure Direct Object Reference** vulnerability. IDOR allows the attacker to bypass authorization mechanisms to access information/resources directly by manipulating the value of a parameter used to point to that resource/object.

## Information Gathering / Initial Access

Once the machine has been fully deployed, we are then given a URL to access the web app - http://10.10.172.205. After accessing the URL we are greeted with a login page as shown below.

![image.png](/assets/img/thm-neighbour/login-page.png)

We don't have much information to access the contents of the website other than the note at the bottom to view the source code of this page. After reviewing the source code, notice there's a username and password (`guest:guest`) provided for us to login to the website.

![image.png](/assets/img/thm-neighbour/login-page-view-source.png)

We can then login using the username `guest` and password `guest`

![image.png](/assets/img/thm-neighbour/home-page.png)

## Exploiting IDOR

Notice the URL uses a `user` parameter with a value of the username `guest`. Since this room is about IDOR, we will then test for IDOR by changing the value of the parameter to another user in hopes of accessing the contents of the other user.

![image.png](/assets/img/thm-neighbour/home-page-url.png)

Since we already know that there's an admin user from the source code, we can try changing the value of the parameter from `user=guest` to `user=admin`. After changing, we are now able to access the admin account and the Flag for this challenge. 

![image.png](/assets/img/thm-neighbour/room-flag.png)

Thanks for reading my waklthrough, I hope you enjoyed it!

