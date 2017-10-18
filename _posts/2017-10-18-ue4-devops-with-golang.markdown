---
layout: post
title: UnrealEngine 4 DevOps with Go
date: 2017-10-18 15:39:20 +0600
description: Learn how we made DevOps for our Indie Game project called Cold Space
img: cold-space-ue4-devops.jpg # Add image post (optional)
tags: [Cold Space, UE4, Golang, GameDev, IndieDev]
---
This is my first article in a series of posts about Game Development. My biggest dream of all times was to make games. And I'm making one now. I'm not doing this full time, because my main sphere of business is a [cloud computing](https://subutai.io). However, I do code games on my spare time. 

We (a few friends) are making a FPS game called [Cold Space](https://idea-x.com). It is already have a page on [Steam](http://store.steampowered.com/app/595120), but not yet available for play. Cold Space is not just FPS shooter game. It is a Multiplayer game. And this feature requires us to maintain servers. This post is about this - how we build and deploy game servers.

Of course at early steps of development everyone of us could start a local server and invite other developers to play. But once we reached Closed Beta Testing milestone this scheme became obsolete and we started to look for proper ways on server management. 

For this problem we have developed several tools that we use in our DevOps. Our infrastructure became big, but it is fully automated. Thanks to Go programming language!

## Hook Listener
The very first instrument that we have deployed to our servers was a hook-listener application, which was listening for incoming webhooks from github (where we store our devops tools) and execute different actions based on event type and repository. Generally speaking hook-listener is a devops for devops. It's accepts webhooks from our other tool's repositories, build them, deploy and run. 

It's also updates itself. 

## IxBuild
This is our main CI/CD application. It is huge and it's highly UE4 oriented. However, we are trying to get away from this and make it more generic. I even started to use this tool in Subutai for our C/C++ cross-platform applications - [Launcher](https://github.com/subutai/launcher) and [Tray](https://github.com/subutai/tray). 

Architecture of IxBuild is not so complex. 
Application itself is divided on two parts - daemon and client. 
Daemon is running on a dedicated server and it keeps track of our repositories. 
Hence speaking it accepts webhooks from our BitBucket repository (where our game code are kept) and analyzes it, whether this particular event needs to be handled or not. 
If daemon decides that this event needs to be handled (e.g. Push or Tag) it will send this command to all connected clients. 

Clients are connected to daemon over TCP and communicate with help of Google's Protobuf. Once client receive build command from daemon it updats local code from remote repository and build project using UE4 Automation Tool. Depending on platform it builds different targets, e.g. on Windows or MacOS it builds only game client, but on Linux a dedicated server is also being built. 

When build process finished (whether successfully or not) clients sent report to daemon, uploads logs and built artifacts. 

## IxServer
We are using [DigitalOcean](https://m.do.co/c/daf9b04dbc8e) as cloud service provider. We love their API and we love they love Golang! IxServer is another client/server application, but it have different purpose from IxBuild. IxServer daemon monitors servers we rent from DigitaOcean and wait for the moment when it needs to order new server. This event occurs whenever player calculator sees that soon another server maybe required for incoming players. Right after ixserver daemon has created new droplet, we upload ixserver to this machine too and start it in client mode. In client mode it connects to daemon and daemon assign unique ID to this server as part of handshake procedure. Once both parties are handshaked, client requests a new dedicated server package from ixserver daemon, downloads it and starts. Inside a Game Server we send a UDP packet to ixserver client, which reports current number of players on this particular server. Based on this information IxServer daemon may decide to order new droplets in some region or shutdown some droplets if they are empty. 

## Notifications
In order to notify us about what's happening and more easier monitoring of our infrastructure we are using some special channels on our Discord server. A few bots report us everything we need: repository events are reported by ixbuild bot and game server events reported by ixserver bot. It even notifies us about abnormal CPU or memory consumption so we can decide what to do with this droplet, collect server logs and analyze the problem.

## Futher reading
All tools listed above are hidden from public. Not because we want to hide our gems. We just want to give it a good shape and release for everyone to use. Soon we will start to clean up these projects and share with all of you. And my next articles will be more technical, describing every tool in details, so you can get better understanding on how you can use them together to make your life easier. Keep in touch.
