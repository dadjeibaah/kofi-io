---
title: Setting Up Grunt for Frontend Development
date: 2017-03-30 14:12:12
tags:   
---

Back when I had my first job as a developer, I was introduced to the wonderful world of Grunt. At first I didn't understand it. I thought it was extra fluff that developers used to be cool. So I said, you know what, I don't need this! I began to not use grunt for various grad projects I developed but soon enought I saw how painful it was to develop JS web apps without it and I was like

![Nah](http://cdn.playbuzz.com/cdn/a9a0a9b0-a298-46b4-80ef-7fbabeb208ee/d4bcbde1-c22a-4521-879b-206c72669977.jpg)

I then realized Grunt made frontend development easier by doing a lot of the preprocessing needed to get javascript and other dependencies working in your HTML code. So I decided to set it up for at least one project I had.

### Getting Started
I am going to be using AngularJS and bunch of other dependencies I use often to demonstrate how grunt works. The first thing we will do create a new directory and a package.json
```!bash
mkdir demo-app
cd demo-app
npm init
```

We create a file called demo-app, cd into and run npm's initialization command. It should give you a bunch of prompts that I usually just accept whatever it gives me because I can always change it later.

Next, we install grunt's CLI globally so we can use it anywhere there is a local gruntfile (More on that later)

```npm
npm install -g grunt-cli
```