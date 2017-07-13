I'm a Quality Assurance Engineer. A big part of my job is to find problems,
then make sure that they are fixed and *automated* so they don't regress. If I
do my job well, then our process will identify new and potential problems early
*without manual intervention* from anybody in the team. It's like trying to
automate myself, everyday, until I'm no longer needed and have to jump to
another project.

However, as we work in the project, it's unavoidable that many small manual
tasks accumulate on my hands. This happens because I set up the continuous
integration infrastructure, so I'm the one who knows more about it and have
easier access, or because I'm the one who requested access to the build farm
so I'm the one with the password, or because I configured the staging
environment and I'm the only one who knows the details. This is a great way
to achieve job security, but it doesn't lead us to higher quality. It's a job
half done, and it's terribly boring to be a bottleneck and a silo of
information about testing and the release process. All of these tasks should be
shared by the whole team, as with all the other tasks in the project.

There are two problems. First, most of these tasks involve delicate credentials
that shouldn't be freely shared with everybody. Second, even if the task itself
is simple and quick to execute, it's not very simple to document how to set up
the environment to be able to execute them, nor how to make sure that the right
task is executed in the right moment.

Chatops is how I like to solve all of this. The idea is that every task that
requires manual intervention is implemented in a script that can be executed by
a bot. This bot joins the communication channel where the entire team is
present, and it will execute the tasks and report about their results as a
response to external events that happen somewhere in the project
infrastructure, or as a response to the direct request of a team member in the
channel. The credentials are kept safe, they only have to be shared with the
bot and the permissions can be handled with access control lists or membership
to the channel. And the operative knowledge is shared with all the team,
because they are all listening in the same channel with the bot. This means
that anybody can execute the tasks, and the bot assists them to make it simple.

In snapcraft we started writing our bot not so long ago. It's called snappy-m-o
([Microbe Obliterator](http://pixar.wikia.com/wiki/M-O)), and it's written
in python with [errbot](http://errbot.io/). We, of course, packaged it as a
snap so we have automated delivery every time we change it's source code, and
the bot is also autoupdated in the server, so in the chat we are always
interacting with the latest and greatest.

Let me show you how we started it, in case you want to get your own. But let's
call this one Baymax, and let's make a virtual environment with errbot, to
experiment.

[![drawing of the Baymax bot](https://upload.wikimedia.org/wikipedia/en/2/2c/Baymax_from_Disney%27s_Big_Hero_6.png)](https://upload.wikimedia.org/wikipedia/en/2/2c/Baymax_from_Disney%27s_Big_Hero_6.png)

    $ mkdir -p ~/workspace/baymax
    $ cd ~/workspace/baymax
    $ sudo apt install python3-venv
    $ python3 -m venv .venv
    $ source .venv/bin/activate
    $ pip install errbot
    $ errbot --init

The last command will initialize this bot with a super simple plugin, and
will configure it to work in text mode. This means that the bot won't be
listening on any channel, you can just interact with it through the command
line (the ops, without the chat). Let's try it:

    $ errbot
    [...]
    >>> !help
    All commands
    [...]
    !tryme - Execute to check if Errbot responds to command.
    [...]
    >>> !tryme
    It works !
    >>> !shutdown --confirm

`tryme` is the command provided by the example plugin that `errbot --init`
created. Take a look at the file `plugins/err-example/example.py`, errbot is
just lovely. In order to define your own plugin you will just need a class that
inherits from `errbot.BotPlugin`, and the commands are methods decorated with
`@errbot.botcmd`. I won't dig into how to write plugins, because they have an
amazing
[documentation about Plugin development](http://errbot.io/en/latest/user_guide/plugin_development/index.html).
You can also read the plugins we have in our snappy-m-o, one for
[triggering autopkgtests on GitHub pull requests](https://github.com/elopio/snappy-m-o/blob/master/plugins/autopkgtest_github/autopkgtest_github.py),
and the other for
[subscribing to the results of the pull requests tests](https://github.com/elopio/snappy-m-o/blob/master/plugins/snapcraft_github/snapcraft_github.py).

Let's change the config of Baymax to put it in an IRC chat:

    $ pip install irc

And in the `config.py` file, set the following values:

    BACKEND = 'IRC'
    BOT_IDENTITY = {
        'nickname' : 'baymax-elopio',  # Nicknames need to be unique, so append your own.
                                       # Remember to replace 'elopio' with your nick everywhere
                                       # from now on.
        'server' : 'irc.freenode.net',
    }
    CHATROOM_PRESENCE = ('#snappy',)

Run it again with the errbot command, but this time join the #snappy channel
in irc.freenode.net, and write in there `!tryme`. It works ! :)

So, this is very simple, but let's package it now to start with the good
practice of continuous delivery before it gets more complicated. As usual, it
just requires a `snapcraft.yaml` file with all the packaging info and metadata:

```
name: baymax-elopio
version: '0.1-dev'
summary: A test bot with errbot.
description: Chat ops bot for my team.
grade: stable
confinement: strict

apps:
  baymax-elopio:
    command: env LC_ALL=C.UTF-8 errbot -c $SNAP/config.py
    plugs: [home, network, network-bind]

parts:
  errbot:
    plugin: python
    python-packages: [errbot, irc]
  baymax:
    source: .
    plugin: dump
    stage:
      - config.py
      - plugins
    after: [errbot]
```

And we need to change a few more values in `config.py` to make sure that the
bot is relocatable, that we can run it in the isolated snap environment, and
that we can add plugins after it has been installed:

    import os

    BOT_DATA_DIR = os.environ.get('SNAP_USER_DATA')
    BOT_EXTRA_PLUGIN_DIR = os.path.join(os.environ.get('SNAP'), 'plugins')
    BOT_LOG_FILE = BOT_DATA_DIR + '/err.log'

One final try, this time from the snap:

    $ sudo apt install snapcraft
    $ snapcraft
    $ sudo snap install baymax*.snap --dangerous
    $ baymax-elopio

And go back to IRC to check.

Last thing would be to push the source code we have just written to a GitHub
repo, and enable the continuous delivery in build.snapcraft.io. Go to your
server and install the bot with `sudo snap install baymax-elopio --edge`.
Now everytime somebody from your team makes a change in the master repo in
GitHub, the bot in your server will be automatically updated to get those
changes within a few hours without any work from your side.

If you are into chatops, make sure that every time you do a manual task, you
also plan for some time to turn that task into a script that can be executed
by your bot. And get ready to enjoy tons and tons of free time, or just keep
going through those [400 open bugs](https://bugs.launchpad.net/snapcraft),
whichever you prefer :)
