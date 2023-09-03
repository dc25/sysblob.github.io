---
title: Everything Bookstack
image: bookstack.png
img_path: /images/
date: 2023-08-21
categories: [homelabbing]
tags: [bookstack,documentation,self-hosted]
pin: false
comments: true
---

Bookstack is a self-hosted wiki website that makes editing and storing your documentation in an organized and secure fashion fast, efficient, and easy.

Link: [https://www.bookstackapp.com/](https://www.bookstackapp.com/)

Refer to the Table of Contents if you are looking for something specific about Bookstack.

## What is Bookstack?

Bookstack is first and foremost a self-hosted wiki. A place to store all your knowledge and documentation so you don't lose it or forget it. Unlike static website generators, however, Bookstack is a fully fledged app. You create, edit, publish, and admin over, all your content directly from Bookstack's Web GUI. This makes it more like a Word Press designed for documentation.

Bookstack can be setup as a stand alone apache web server, or a docker container. Luckily, Bookstack is an easy installation -- as the creator has made many scripts available to the community. I personally run both of my Bookstack instances (one local, one remote) as stand alones as I gave them their own virtual machines. 

Let's take a look at the interface of Bookstack and see what makes it unique.

![bookstack gui](bookstack-gui.png){: w="840" h="400" }

You'll notice right away the unique way Bookstack displays information. Your documentation is laid out in a hierarchy of Shelfs > Books > Chapters > Pages. You can use as much or as little of this hierarchy as you want to. To give you an idea of my layout I have two shelves, "Applications", and "Commands and Processes". Under My applications shelf for example I have a Book Docker. I do not use Chapters, but I then have several pages on Docker, i.e. "Docker setup", and "Docker logging".

You'll also notice the customization. Bookstack has a settings area which allows for colors and themes. You can upload logos, and if you're really ambitious, you can add CSS overrides or edit the code as you wish. 

![bookstack settings](bookstack-settings.png){: w="840" h="400" }

Bookstack also has built in user management which is nice. You can add users, authenticate them via email, delegate what rights they have and over what pages, and much more. Other features Bookstack supports are Webhooks to send out data on actions such as posting, or audit logging for login and wiki changes.

![bookstack users](bookstack-users.png){: w="840" h="400" }

Where Bookstack really shines though, is the ease at which you can create content. Just go to your content page, click new page, and everything you need to create and publish a new post is right there. From font customization and coloring, code blocks with syntax highlighting, tables, graphs, images, videos, the whole tool kit is there. 

![bookstack content](bookstack-content.png){: w="840" h="400" }

The best part is Bookstack uses a database and a few files which can easily be backed up so you can take your content on the go pretty easily. The developer even included a script to backup and restore which I'll cover.

I highly recommend Bookstack as an open source free project to store your documents or blog.

## Setting up Bookstack

## Usage and Tips

### Locating your .env file

Your .env file handles a lot of configuration for Bookstack. This is located at /var/www/bookstack/.env

### Bookstack inside an iFrame

By default BookStack will only allow itself to be embedded within iframes on the same domain as you’re hosting on. This is done through a CSP: frame-ancestors header. You can add additional trusted hosts by setting an ALLOWED_IFRAME_HOSTS option in your .env file like the example below:

```yaml
# Adding a single host
ALLOWED_IFRAME_HOSTS="https://example.com"
# Multiple hosts can be separated with a space
ALLOWED_IFRAME_HOSTS="https://a.example.com https://b.example.com"
```

> When this option is used, all cookies will be served with SameSite=None (info) set so that a user session can persist within the iframe.
{: .prompt-info }