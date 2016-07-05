---
layout: post
title: "Scripting iTerm2 using AppleScript"
date: 2016-07-05 09:00:00 +0200
tags:
  - english
  - dev
---

As a developer and as a OSX user I use [iTerm2](https://www.iterm2.com) in a daily basis. It's open source, it's an awesome piece of software, and when you get familiar with features such as split panes, it's also an amazing productivity booster.

<!--more-->

On the other hand, from time to time I need to SSH to multiple severs to make some checks. For example, a few days ago I needed to SSH to 8 Varnish Cache and 3 Redis servers to check some log files, access to certain stats, etc. I needed to do that at 10:00 AM, then 20 minutes later again, then at 12:00 PM, etc. Every time I would need to create a new iTerm2 tab, then create enough panes, then execute SSH commands, etc. It looks like something a machine could do for me, isn't it?

Of course [the answer is yes](https://www.iterm2.com/documentation-scripting.html). The pleasant surprise here was how extremely simple was creating a disposable trivial script to do the job. All panes and SSH connections ready in less than five seconds anytime I needed them. Combine this with the iTerm2 broadcasting feature and you'll understand what love is :)️

    #! /usr/bin/osascript

    set commands to { ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "tail -f /var/log/messages | grep ..." ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "redis-cli -c -p 6379" ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "redis-cli -c -p 6379" ¬
        }, ¬
        { ¬
            connection: "ssh -o ServerAliveInterval=60 carlos@...", ¬
            action: "redis-cli -c -p 6379" ¬
        } ¬
    }

    tell application "iTerm"
        # Create tab.
        tell current window
            create tab with default profile
        end tell

        # Create sessions.
        tell current session of current window
            split horizontally with default profile
            split horizontally with default profile
            split horizontally with default profile
            split horizontally with default profile
        end tell
        tell session 1 of current tab of current window
            split vertically with default profile
        end tell
        tell session 3 of current tab of current window
            split vertically with default profile
        end tell
        tell session 5 of current tab of current window
            split vertically with default profile
        end tell
        tell session 7 of current tab of current window
            split vertically with default profile
        end tell
        tell session 9 of current tab of current window
            split vertically with default profile
            split vertically with default profile
        end tell

        # Establish connections & execute actions.
        repeat with i from 1 to count of commands
            tell session i of current tab of current window
                write text (connection of item i of commands)
                write text (action of item i of commands)
            end tell
        end repeat
    end tell
