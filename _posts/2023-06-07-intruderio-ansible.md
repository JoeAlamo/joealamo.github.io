---
layout: post
title: "Automatically secure your servers with intruder.io via Ansible"
date: 2023-06-07
excerpt_separator:  <!--more-->
tags:
 - security
 - ansible
 - vulnerabilities
---

![intruderio and ansible](https://miro.medium.com/v2/resize:fit:640/format:webp/1*z9xjKBFsZvCYDWOJbSGgAA.png)

Running external vulnerability scanners against your infrastructure is a great way of bolstering your security 🔒

Whilst working at One Utility Bill, I came across and adopted [Intruder.io](https://intruder.io). 

I've really enjoyed their product, offering regularly scheduled scans and emergency scans whenever new CVEs are disclosed.

To make life easier whenever provisioning new servers, I produced an Ansible role to automate the enrollment and scanning of new servers.

Head over to the [OUB Engineering Blog](https://engineering.oneutilitybill.co/automatically-secure-your-servers-with-intruder-io-via-ansible-1f4e9bb62ffe) to see it in full!
