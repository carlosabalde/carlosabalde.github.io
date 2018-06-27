---
layout: post
title: "Moving logic to the caching edge (and back)"
date: 2018-06-27 19:30:00 +0200
tags:
  - english
  - dev
---

There is no break for marketers. Any day is a good day to coin new bullshit buzzwords and make the world a worst place to live in. This is particularly painful in the technical arena. **You simply can't safely leave home without your bullshit detector**.

This post is about one of those popular buzzwords: **edge computing**. Did we need a new word to talk about decentralization in the cloud -fog? 🤷- computing era? Probably not, but that's not the point here :)

It looks like that term called edge computing is becoming more and more popular lately. In fact, it seems I've working on edge computing for the last few years and I wasn't even aware of that. I guess it's time to update my LinkedIn headline.

All kidding aside, **yes, edge computing is a highly recommended strategy to have in mind in your global architecture**. But, what logic should be moved to the edge? How to move that logic? There is not simple answer for those questions, but [this old post](http://highscalability.com/blog/2012/9/12/using-varnish-for-paywalls-moving-logic-to-the-edge.html) by Per Buer about moving access control logic to a caching layer built on top of [Varnish Cache](https://varnish-cache.org) is a nice starting point.

<!--more-->

You can embed logic in Nginx, Apache or in your ultra-expensive load balancer, but **Varnish Cache is particularly good at that**. Even when caching is not a key feature for you -really? :)-, Varnish Cache still is an element to consider. If you want to know more about that I'd suggest that you read about two powerful Varnish Cache weapons: the [Varnish Configuration Language](https://book.varnish-software.com/4.0/chapters/VCL_Basics.html) (i.e. VCL) and [Varnish Modules](https://book.varnish-software.com/4.0/chapters/Appendix_D__VMOD_Development.html) (i.e. VMODs). **If you think your beloved Nginx or your shiny load balancer can do that better than Varnish Cache, I must respectfully ask you to check some documentation because you're simply wrong :)**

However, although moving certain logic to the caching edge is a great idea, **sadly all that glitters is not gold. When moving logic to the edge, you're both changing where the logic is executed -hurray!- and where the logic is defined -meh!-**. Those facets are usually managed by [completely different beasts](https://blog.codinghorror.com/vampires-programmers-versus-werewolves-sysadmins/), and they have been confronted since ancient times: vampires (i.e. programmers), in charge of defining the logic, and werewolves (i.e. sysadmins), in charge of setting up infrastructure running in the edge.

An edge computing strategy based on Varnish Cache + VCL gives full responsibility to sysadmins, but **sysadmins don't want to worry about logic details -full disclosure: they don't understand it-, and, of course they'll never let developers change configuration (i.e. VCL in this particular case) of any infrastructure they are in charge of**. Yes, this is based on a true story, and that's why I created the [cfg VMOD](https://github.com/carlosabalde/libvmod-cfg) :)

I'm listening to you, my dear Varnish expert. I know what you are thinking about: “you don't need a VMOD for that! Simply let developers generate some VCL fragments and find a way to integrate that in the deployment workflow”. It's an option, but not sure if adding more complexity and dependencies is the way to go here.

So, **how does that cfg VMOD help mitigate the ancient conflict between vampires and werewolves? Let's go through a few examples.**

**Let's assume you're a sysadmin running an online shop. The release of a new game-changer product is approaching and the marketing team (yes, them again!) wants to create some expectation closing access to the shop one hour before the launch**. Not a problem! You could do that simply writing a few lines of VCL in order to redirect visitors to some static page during the closing window. Sadly the launch date is not yet defined and you want to go on holidays. Here comes the cfg VMOD to the rescue!

    sub vcl_init {
        # $ cat setting.ini
        # [expectation]
        # start: 1459468800
        # stop: 1459472400
        # ...
        new settings = cfg.file(
            "http://192.168.0.42/settings.ini",
            period=60);

        ...
    }

    sub vcl_recv {
        if (std.time(settings.get("expectation:start"), now) < now &&
            std.time(settings.get("expectation:stop"), now) > now) {
            # Generate 302 synthetic response.
        }

        ...
    }

Basically you've moved the closing logic to the caching edge, but input for that logic is still handled by developers. They need to keep the remote settings.ini file updated. Moreover, by writing some extra VCL you could give them control about when contents of that file will be refreshed. Time to leave to the beach! :)

**Let's assume now you're back from holidays. First thing in the morning, the head of development is waiting at your desk with new requirements**. This time the development team plans an incremental migration to a new infrastructure: some incoming URLs should be handled by new backend servers, some by the old ones. Of course, it's not trivial to identify what URLs should be routed to the new servers. Even worse, that's going to be changing every week during the next months, until the migration to the infrastructure is fully completed.

The routing logic needs to be executed in the caching edge, but you would like to make developers responsible of maintaining the URL matching logic and let them change it whenever they want. This is similar to the previous case, but this time a simple key-value scheme is not enough. No worries. The cfg VMOD can help here too!

    sub vcl_init {
        # $ cat backends.rules
        # (?i)\.(?:jpg|png|svg)(?:\?.*)?$      -> old
        # (?i)^www\.(?:foo|bar)\.com(?::\d+)?/ -> new
        # ...
        new backends = cfg.rules(
            "http://192.168.0.42/backends.rules",
            period=300);

        ...
    }

    sub vcl_backend_fetch {
        set bereq.http.X-Backend = backends.get(bereq.http.Host + bereq.url, "old");
        if (bereq.http.X-Backend == "old") {
            set bereq.backend = ...;
        } elsif (bereq.http.X-Backend == "...") {
            set bereq.backend = ...;
        } elsif ...

        ...
    }

So, somehow you've moved execution of the pattern matching logic to the caching edge, but definition of that logic is still handled by the origin servers.

**Of course this pattern matching strategy is limited. What if simple regular expressions are not enough? What if they are enough but the result is incomprehensible?** Again, the cfg VMOD is the solution to your problems: simply let developers write proper logic using [Lua](https://www.lua.org) and then run it inside Varnish Cache while processing HTTP requests.

    sub vcl_init {
        # $ cat backends.lua
        # local host = string.gsub(string.lower(ARGV[0]), ':%d+$', '')
        # local url = string.lower(ARGV[1])
        # 
        # varnish.log('Running Lua backend selection logic')
        # 
        # -- Remember that Lua's pattern matching is not equivalent to POSIX regular
        # -- expressions. Check https://www.lua.org/pil/20.2.html and
        # -- http://lua-users.org/wiki/PatternsTutorial for details.
        # if host == 'www.foo.com' or host == 'www.bar.com' then
        #     if string.match(url, '^/admin/') then
        #        return 'new'
        #     end
        # end
        # 
        # return 'old'
        new backends = cfg.script(
            "http://192.168.0.42/backends.rules",
            period=300);

        ...
    }

    sub vcl_backend_fetch {
        backends.init();
        backends.push(bereq.http.Host);
        backends.push(bereq.url);
        backends.execute();
        set bereq.http.X-Backend = backends.get_result();
        backends.free_result();
        if (bereq.http.X-Backend == "old") {
            set bereq.backend = ...;
        } elsif (bereq.http.X-Backend == "...") {
            set bereq.backend = ...;
        } elsif ...

        ...
    }

**That's it for today. Happy edge computing to everybody! :)**

Huge thank you to Cristian Marcote from [99languages](https://99languages.es) for his help with my broken English! :)
