---
author:
  name: "Pieter Jan Geutjens"
date: 2019-04-20
linktitle: Using python and tmuxinator for optimal docker logs monitoring
title: Using python and tmux(inator) for optimal docker logs monitoring
type:
- post
- posts
weight: 10
series:
aliases:
- /blog/tmuxinator-docker-logs/
---

I really enjoy using docker while developing. It's easy to spin up your database, api-server, frontend, and
whatever other components you need, saves a ton of hassle. For that same reason, I love docker-compose even more!  

Thinking back to when I used to develop on a LAMP stack that was just installed on my laptop, or later on
using virtual build environments and tools like vagrant or laravel homestead, this feels like the logical next step. 
I can spin up not just webservers but a whole plethora of configurations, including databases, proxies, 
redis cache, message queues... all with a single command

### The issue
There is one thing though that I find annoying when working with docker-compose: While coding I want to keep 
an eye on my container logs, but this proves to not always be so simple. See, I used to launch my containers with

{{< highlight bash >}}
docker-compose up
{{< /highlight >}}

but this just gives me a single shell with all of the container output jumbled together, very hard to follow as soon
as you have a couple of 'verbose' containers to keep an eye on. Not to mention you tie up that shell as the prompt 
does not release after spinning up the servers...

But ok, there's a solution for that last problem, in comes

{{< highlight bash >}}
docker-compose up -d
{{< /highlight >}}

this will launch your containers in _daemon_ mode, with the cursor returning once they have all launched. However, now I
have to call up the logs I want to see one by one :/

### The challenge

So what do I really want? Let me try to list my requirements here:

* A single command
* that gives me a view of all the container logs for the current project
* with separate, distinct log views per service
* in a flexible solution that handles varying numbers of services

Requirements 2 and 3 make me think I should split up my terminal into multiple panes, one for each log I want to see, and this leads me to [tmux][tmux wiki].
tmux should allow me to define horizontal and vertical panes that can be used individually to run commands (or show their output).
By using [tmuxinator][tmuxinator github] I should be able to simplify the creation and layout of all these windows and panes.

Requirements 1 and 4 look like something to script.

### Installing tmux

I do most of my coding work at home on a macbook, so fortunately getting started is just a matter of a couple homebrew installs.
{{< highlight bash >}}
brew install tmux
brew install tmuxinator
{{< /highlight >}}

the tmux package is also available on linux, and tmuxinator installs using a Ruby gem
{{< highlight bash >}}
gem install tmuxinator
{{< /highlight >}}

unfortunately I haven't yet found a way of getting this up and running in Windows.


### Flexibility, dynamically generating a tmuxinator config

As I recently started writing python scripts and I really enjoy the language, I will take any excuse to work with it :). So I wrote a little script, based on 2 examples you can find [here][tmux generator] and [here][yaml generator], that will read in a docker-compose.yml file and create
a tmuxinator config that, when executed, will spin up the containers in the background (first panel top left) and then create a pane for each
container's log file. ([gist][tmuxbuilder.py gist])

{{< highlight python >}}
import os
import yaml
import argparse

parser = argparse.ArgumentParser(
    description='Show docker logs in tmuxinator session')
parser.add_argument("-d", "--delay", help="delay in seconds before opening logs", default=5)
parser.add_argument("-o", "--out", help="output file path", default="tmuxinator.yml")
parser.add_argument("-l", "--layout", help="tmuxinator layout", default="tiled")
args = parser.parse_args()

def populateConfig(log_panes):
    configIn = {
        'name': 'docker-logs',
        'root': '.',
        'windows': [
            {'logs': {
                'layout': args.layout,
                'panes': log_panes}}
        ]
    }

    configOut = yaml.dump(configIn)
    return configOut


if __name__ == '__main__':
    try:
        docker_config = open('docker-compose.yaml').read()
    except IOError:
        docker_config = open('docker-compose.yml').read()

    config = yaml.load(docker_config)
    services = list(config['services'].keys())
    log_panes = list()

    # add a first pane to spin up the containers
    log_panes.append('clear && docker-compose up -d') 
    for s in services:
        # sleep some to allow containers to start, then show logs
        log_panes.append('sleep {} && clear && docker-compose logs -f {}'.format(args.delay,s))

    tmux_config = populateConfig(log_panes)
    outFile = open(args.out, "w")
    outFile.write(tmux_config)
    outFile.close()
{{< /highlight >}}


this file generates a tmuxinator config in yml format that can be used with the commandline

{{< highlight python >}}
tmuxinator start -p <configfile.yml>
{{< /highlight >}}

and the result looks like this (by the way, you can close the window by pressing C-b + &)

![tmux screenshot](/img/tmuxbuilder.png)

### Bonus: adding color

As you can see in the screenshot, the logs for the different containers are all colored in the same shade of blue. Since I noticed that when using docker-compose up,
the different container logs each get their own color, I would like to have the same here. After a bit of googling I found a script on [github][xcol] that might 
help me get where I need. Some small additions to the python script later, I get this result:

![tmux screenshot](/img/tmuxbuilder_colors.png)

I will not post the final python code here, as this variation requires you to install the xcolorize script, but you can find the gist [here][tmuxbuilder.py colors gist]

### References

[tmux man pages][tmux man]  
[the original gist containing a tmux-only generator script][tmux generator]  
[the original yaml generator][yaml generator]  
[xcolorize][xcol]  


[tmux generator]: https://gist.github.com/omaraboumrad/7fb7c60038591ee65961b85e7f8d2096
[yaml generator]: https://dev.to/jmarhee/example-of-yaml-generator-and-validator-in-python-1opk
[tmux wiki]: https://en.wikipedia.org/wiki/Tmux
[tmux man]: http://manpages.ubuntu.com/manpages/precise/en/man1/tmux.1.html#contenttoc6
[tmuxinator github]: https://github.com/tmuxinator/tmuxinator
[tmuxbuilder.py gist]: https://gist.github.com/pjgeutjens/84ec1ec4259e092addad98a5bc7998da
[xcol]: [https://github.com/nachoparker/xcol/blob/master/xcol.sh]
[tmuxbuilder.py colors gist]: https://gist.github.com/pjgeutjens/77eaeef25c136f6291f5a8450bfba8b3