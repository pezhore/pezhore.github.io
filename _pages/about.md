---
title: "About Me"
layout: splash
classes: wide
permalink: /about/
date: 2018-04-12
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: /images/about/skippingrocks.jpg
excerpt: "Learning and experimenting shouldn't be limited to 9-5..."
intro: 
  - excerpt: "# Welcome to my little corner of the internet!"
feature_row2:
  - image_path: /images/about/homelab.jpg
    alt: "Home Lab"
    title: "Home Lab"
    excerpt: "On this site, you'll find I devote quite a bit of posts about my home lab. My current setup is a combination of HP DL360G5 servers and a 6TB Synology storage device. This lab lets me explore technology that is either unavailable to me at work, or with a freedom I would otherwise note enjoy."
    url: "home-lab"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row3:
  - image_path: /images/about/sourcecode.png
    alt: "placeholder image 2"
    title: "Code, Automation, Config Mgmt"
    excerpt: "The other chunk of content you'll find on this site revolves around code, automation, and configuration management. I find the best way to automate something tedious is to give it to the laziest engineer you can find. I was that engineer (and probably still am) for the bulk of my career. Wanting to find the right way to automate things has served me well."
    url: "automation"
    btn_label: "Read More"
    btn_class: "btn--primary"
---


{% include feature_row id="intro" type="center" %}

{% include feature_row id="feature_row2" type="left" %}

{% include feature_row id="feature_row3" type="right" %}

