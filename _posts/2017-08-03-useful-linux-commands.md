---
layout: post
title: >
  Master Linux CLI (part I). The most useful Linux commands.  
tags: [terminal]
---
Interestingly, when you start reading books about Linux or going through different tutorials, they often don't tell you about some of the cool commands that make your work sometimes so much easier. Maybe they are hiding them from you to make sure you still have to learn something in the future? ```¯\_(ツ)_/¯```

Anyway, today I decided to make a quick overview of some of the commands which I find useful, but which are sometimes hard to find out about.
<!--break-->

### tee

```tee``` command allows you to write to the stdout and a file (or files) at the same time.

This is useful when you want to store and view the output of any command.

![200x200](/public/img/linux/tee11.png)


And you can also use it to save the output to multiple files.

![200x200](/public/img/linux/tee22.png)

This command is incredibly useful when you want to store the output of a command to a file but also redirect it as an input to another command.

![200x200](/public/img/linux/tee33.png)

As you can see, we are ablt to take the snapshots of the data as it flows through the pipes.

### pbcopy (Mac) or xclip (Linux)

This allows you to copy a file's content to the clipboard.

![200x200](/public/img/linux/pbcopy.png)

Now if you try to paste, you'll get ```cucaracha```, which by the way means a cockroach in Spanish :)

This command makes copying and pasting a breeze. I find it especially useful when I need to copy SSH or GPG keys.

### watch

 ```watch``` runs a specified command repeatedly at regular intervals and displays its output on a console.

This is used when you need to continuously monitor some command's output.

Simple examples could be like monitoring who is logged in to the system with ```watch who``` command or watching for changes inside a directory with ```watch ls```

We should note the ```-d``` option that allows you to highlight the differences.

Just to show you how it works, we'll use it with a ```date``` command.

<script type="text/javascript" src="https://asciinema.org/a/wP3Hd9maxP6tgTYNKOxqYvIiM.js" id="asciicast-wP3Hd9maxP6tgTYNKOxqYvIiM" async></script>



### script & scriptreplay

These are 2 awesome commands which allow you to record and replay a shell session for you.

To use a ```script``` command, we can just type ```script``` in which case the session will be stored in a file with a default name ```typescript```. We can also specify the name of the file in which we want to store our sessions as the first argument to the ```script``` command.

<script type="text/javascript" src="https://asciinema.org/a/HVp4CIb8zU8JolXvzVY7dpyQk.js" id="asciicast-HVp4CIb8zU8JolXvzVY7dpyQk" async></script>

An alternative to the ```script``` command is ```history```, but it only keeps track of the commands you use and
not their outputs.

Another cool thing about the script command is that the shell session that you recorded can be replayed in your terminal with a ```scriptreplay``` command. This is particularly helpful if during the session you start interacting with some programs like htop in your terminal.

To be able to replay a recorded shell session, we need to specify a filename for storing the timing information.
~~~yml
$ script --timing=time.txt myshell.log
~~~
Then after we're done recording, we can use the ```scriptreplay```command to replay the session.

~~~yml
$ scriptreplay --timing=time.txt myshell.info
~~~
<script type="text/javascript" src="https://asciinema.org/a/vurga2WE8SpfRbjcXDfNKKONn.js" id="asciicast-vurga2WE8SpfRbjcXDfNKKONn" async></script>

It's also important to note the ```-c``` option to this command which allows to record the output of a single command. For example, this might come in handy when we need to record a command's output in our bash script. The syntax goes like this.
~~~yml
$ script -c 'ping -c 3 google.com' myshell3.log
~~~
<script type="text/javascript" src="https://asciinema.org/a/LBH7GXiAj7QF6PXCS9qAIyX5P.js" id="asciicast-LBH7GXiAj7QF6PXCS9qAIyX5P" async></script>

### jq

[jq](https://stedolan.github.io/jq/) is a handy JSON processor that allows you to extract necessary fields from a JSON object.

Let's take a simple JSON file and extract some specific fields.

~~~json
{
  "colors":
    {
      "color": "black",
      "category": "hue",
      "type": "primary",
      "code": {
        "rgba": [255,255,255,1],
        "hex": "#000"
      }
    }
}
~~~

![200x200](/public/img/linux/jq.png)
