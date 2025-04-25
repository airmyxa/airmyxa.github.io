---
date: '2025-03-11T9:03:02+04:00'
draft: true
title: 'Void Linux after MacOS'
---

---

I have had some experience using different linux distributions during my life. First I installed linux when I started studing
applied mathematics at University. I used it almost for every usual task as my main desktop system. I programmed, watched youtube
and did all kind of staff using it. Since then I've had some experience using different distros.
I used Ubuntu a lot, Archlinux, Manjaro and some others.

After uni when I started working on my first full time job I was offered to choose what laptop I want. There were linux or MacOS
variants on offer. I didn't believe that I could choose macbook because for me it was more like something for reach people and I wasn't
one of them. With thrill I checked macbook in forms. Since then I used macbook as my primarily laptop and I pretty get used to it.

But recently I found out the felling that I need new experience and more abilities to affect how my system work. I'm allowed to use
my working laptop for personal purpose but limited. And this limit didn't let me have as much fun as I wanted. So I purchesed
new lenovo thinkpad and started thinking about what distro I'd like to install this time.

As I mentioned, I've already had experience with several distros. But this time I wanted something different. Not mainstream and heavy
like Ubuntu, not so difficult to install like Archlinux. I was thinking about different variants like OpenSuse beacause of out of the
box snapshots feature and about Gentoo just to challenge myself. Also I knew that NixOS is popular now but
I wasn't sure why I need it. So finally I stumbled upon Void linux which was declared as minimal non-fork linux distribution.
It looked like something I needed: minimal installation, cool and fast package manager, not so heavy as Ubuntu and not so strange
as Arch, Gentoo or NixOS.

Since then I use it as my personal laptop OS and still happy with it. Finally I've got what I was looking for.

But here I'd like to share what is the difference to use linux and macos, what challenges I've met and what it feels like.

## Installation

I guess it's pretty clear that when you buy macbook you don't need to go through complex installation process like choosing filesystem
and partitions. Usually you press "next" button till the screen shows you desktop wallpapers. But when you install linux you can expect
everything from Archlike manual all kind of settings and ubuntu with more user frendly installation process. Void linux feels somewhere
in between. It uses mostly terminal to install itself but you almost don't need to go deep into linux installation process.

After installation is finished, you get really minimal setup. Nothing you'd expect from OS is included like bluetooth, wifi and even
audio or battery management. So next step is go through documentation and setup all you need. It took time for me because I had
no experience with such kind of linux setups.

## Configuration

When you think that you set up all you need you can find out many insteresting things like:
* Fonts are blury
* Display shows colors somehow I do not expect
* Audio works but awfully
* Some programs looks different from quality point of view
* Sleep doesn't work and s2idle drains battery more that when laptop is on

So I started to solve this "minor" problems. How I was looking for solutions? I tried to "vibe solve" (oh shit...) my problems with
different llms like chatgpt, deepseek, grok. And mostly no result. Grok was the most honest one, it said "I don't know the solution,
ask on forum and don't ask me". Other llms tried to feed me bullshit. So I found many solutions while reading arch wiki and gentoo wiki,
sometimes reddit. And I've spent quite a bit of time trying to understand the root of a problem.

But the significant upside of linux configuration is that most of configs are texts and you can just sync it with github or write down
in your notes system and if you need to reset os or setup new laptop the process will be much easier. I do it by syncing ~/.config directory
to my github and when I need I just pull configs from there. And another lifehack - you can manage two different main branches -
one for linux and one for macos.

## Battery life

I have M2 pro macbook and sometimes I don't believe that the laptop can work so long. I really get used to it and it's quite painfull
to see my thinkpad discharged percent by percent each minute. But I'd like to admit that in the past when I actively used non macbook laptops
the situation was much worse. With void linux installation it doesn't feel like I can't use it without charger but to the opposite to macbook
user experience - I wouldn't feel comfortable if I needed to go somewhere with my laptop and without charger.

## Display

Funny thing - I've chosen my model of laptop also because of the good quality screen it has. It's "2880x1800 @ 120.000 Hz" OLED Display
with non fucking standard pixels layout. First I couldn't understand why each letter shines with blurry aura around. But then I started googling
and found out that my model has some special pixels layout which is not supported by linux. I just tried all possible variants of
subpixel hinting and finally left vrgb variant.

So if you want to buy laptop with some cool display and use linux you'd better check first if it is supported buy os.

## Void specific

* `xbps` package manager feels so good. I haven't checked all the abbilities of it yet but it definitely a great thing.
* I get depper into linux because of minimalism philosofy of void. I needed to configure everything from scratch, so learnt a lot of that
I didn't expect.
* I don't feel any constraints like felt with other repos. This distro doesn't try to persuade you to do something in a particular way.
* It's not very popular distro so you won't find dirrect instructions on how to install apps in their official page. But most likely
there is a suitable package in repos so you just `xbps-query -Rs` package and most likely find it. There were just several times I could not
find package I needed but there was nothing really difficult to build them.
* You can convert `deb` package to `xbps` with the [script](https://github.com/xdeb-org/xdeb)! I tried it, it worked!

## My philosofy of using Linux

Just some bullet points that come to my mind:

- Do not try to make your setup perfect. *It's linux, it will never be perfect, you can either use it and be happy or try to reach
Ultima Thule of perfectness and stuck in continues setup with no result.*
- Measure your linux setup by it's usefullness. *Finally I managed to pay not so much attention to how my setup looks and how my
settings clean but how I can use it in an effective way and solve daily tasks.*
- Improve your setup step by step. *I tried first to fix everything, setup many things I'll use in the future (maybe) and so on.
But I do not need them now and it's time consuming to develop everything at once. So maybe I have audio not properly setup but I don't need
it to be, so I'm ok with it and don't spend time.*
- It's fun to use terminal for everything. *I didn't install wifi or bluetooth UI so far. Maybe later but now I'm having fun
from workflow of getting generic tasks done with my terminal*.
- Have control of your setup. *It's quite a difficult task for me, but I try to understand all the processes working on my machine,
all the daemons running, their purpose and why I can't turn them off. It helps to keep my setup clean and predictable.*

