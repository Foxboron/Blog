---
layout: post
title: "Mailpile, sendmail & procmail"
categories: mail
published: false 
---


set routes.default = {}
set routes.default.command = sendmail
set profiles.0.messageroute = default
set routes.default.command = /usr/bin/sendmail -i %(rcpt)s


What is Mailpile?
=================

[Mailpile] (https://www.mailpile.is) is mail client with a rather unusual goal 
in todays world. It wants to be free, open-source, privacy oritented and easy to
use with encryption. This all comes with the goal of being self-hosted.

This is a contrast to Protonmail who still keeps all your information on their
servers, making people with a slight trust issue look at you in a rather funny
way. However, protonmail and mailpile is among several email providers in the
wake of the NSA scandal to try and give you secure options to gmail, outlook and
yahoo. Which is in my opinion awesome.

This does not solve the fundemental problem in todays world of emails. People 
can use PGP to communicate securly, but most dosn't even know how it work (its
hard, remember?). Every email you send will get in the hands of gmail, outlook
or yahoo. Which makes NSA happy in the end. But this is all in the right step to
solve this problem.

Now, mailpile is supposed to be used with a local mailserver or an existing
one. You could host it on a local server only you got access to, with full disk
encryption. Or worse, host is on a VPS and let have your privat encryption keys.
For the latter there are other possible solutions, which i might take in another
blog post.


Installation
============

You will need 3 things:  
  1. sendmail  
  2. procmail  
  3. mailpile  

Sendmail will work as the Message Transfer Agent (MTA), it will controll
outgoing mail and incomming mail. Procmail works as a Mail Delivery Agent (MDA),
whos job is to deliver the mail correctly.

This setup is ment to work for one user. If you need several users, i recommend
looking into postfix. Tho sendmail chan achive this feature aswell with more
tweaking.

