# Triggering TidalCycles code with a Monome Grid (or any MIDI device)

Below lies my current (March 2019) über-hack to trigger code-blocks in Tidal with a controller. Not for the faint-hearted, and could definitely use some KISS help, but it is a ... workaround :-D . I hope it can be useful to your practice and please feel free to suggest, correct and comment!

# Overview

* [Background](#background)
* [Setting Up](#setting_up)
    * [You will need](#you-will-need)
    * [A simple method](#a-simple-method)
    * [Understanding the send-keys command](#understanding-the-send-keys-command)
    * [Transforming it into SC code](#transforming-it-into-sc-code) 
* [MIDI](#midi)
    * [Setup](#setup)
    * [Controller Layout](#controller-layout)
    * [MIDI Functions](#midi-functions)
* [Storing Patterns](#storing-patterns)

# Background  

In my live performance I use a Monome Grid 128 and [SuperCollider][sc] to store & trigger chuncks of code that I either live-code, or have already prepared. This is happening through a set of custom [Classes][elliLive] in SuperCollider that make use of the excellent [Grrr][grr] toolkit. Lately, I started using [TidalCycles][tidal] and I enjoy it so much that I want to integrate it in this setup.
Since I'm a total noob when it comes to Haskell I thought that the simplest way to accomplish that would be :  
>  press a button and the cursor is (a) placed on a specific line and (b) evaluates the block of code that lies there.

Simple as that. I thought..  

### Trying VSCode

My original intention was to use VSCode and trigger blocks of Tidal code there, by calling the `.unixCmd` method in SuperCollider. Simply put: when a button is pressed, SuperCollider sends a message which, through Terminal, reaches VSCode and controlls it.  

VSCode has a Command Line Interface ( to enable it look [here][vs] ) and I got as far as remotely placing the cursor in a window.
For example, in order to access line `84` in an `example.tidal` file, we can type in a Terminal window (after navigating to the file's directory):  
```bash
$ code -g example.tidal:84   
```
tadaaa! It opens the file in the VSCode editor and places the cursor on the line. But after days of research I could not find a feasible way of how to evaluate this line remotely, i.e. outside of VSCode.  
If you do know of a way, please share!   

### Enter TMUX

While researching I learned about the terminal multiplexer called [tmux][tmx].  
Very-simply put: it's a tiling window manager for the terminal.
I also [learned][send] that it can listen to a magic command:
```
tmux send-keys
```
Using that, we can send emulated keyboard key presses to a target window.
Voila! I thought, I should take advantage of this ability and use an editor that can be hosted in a terminal window.

### Enter VIM

While I cringed at first thought, I eventually found it the perfect excuse to finally get into [Vim][vim] and get to know it at first hand. I already knew about the existence of a Vim-port for TidalCycles, so the chances this could work were high. I understand that Vim has a steep learning curve and this approach can hinder the feasibility of this hack for some, I wholeheartly believe though, that is a fantastic editor, especially for live coding. In the meantime, I hope that CLI calls to editors like VSCode and Atom can soon become more sophisticated, or that there is indeed a simple way to `send-keys` to these editors, of which I still don't know of.   


# Setting Up 

## You will need
* SuperCollider 
* Tidalcycles
* Vim + Tmux
* vim tidal
* a controller (if Monome, you will need the SC3 Grrr Extension)


In case you are new to SuperCollider and TidalCycles, you can follow the installation [instructions][install].  
For setting up Vim with SuperCollider check this excellent [tutorial][mads]. I have to mention though, that I currently use the native SuperCollider IDE and use Vim only for Tidal.  
For installing Tmux and vim-tidal, follow the instructions [here][vimtdl].
Last, if you use the Monome Grid, you will need to download the [Grrr][grr] extension and place it in your `Extensions` folder which you can find by evaluating the following line in SuperCollider:

``` supercollider
Platform.systemAppSupportDir
```

## A "simple" method

It has to be clear that the main "negotiator" here will be SuperCollider. I guess it could be more efficient to "talk" directly to Haskell, but since SC is the environment that I'm comfortable with, I'll go with that.

Our goal is to __map__ controller-buttons to blocks of code on the Vim editor and use these buttons to evaluate our code. We will also want to live-code and assign the newly created blocks too. I opted for a simple solution. I will follow a system of commented-out expressions, like `tags`:

```
-- pat1
-- pat2
etc.
```
which i can send from SuperCollider to tmux and place them above the block of code where the cursor lies in Vim. Then I'll be able to evaluate that block by calling its "name" e.g. `-- pat1` (as in "pattern 1"), followed by the `evaluate` command, which in vim-tidal is `ctrl-e`. Our final code blocks in Vim will look like this:

``` haskell
-- pat1
d1 $ every 2 (# n (irand 18)) $ s "g:7" # cut (1)

-- pat2
d1 $ every 3 (within (0.5, 0.75) (striate 32). (|* speed 1.5))
$ every 2 (# n (irand 18)) $ s "g:7" # cut (-1) # speed 0.8

-- pat3
d1 $ every 3 (within (0.25, 0.75) (striate 2). (|* speed 1.5))
$ every 4 (# n "3 7 9")
$ s "g:5" # cut (-1) # speed 0.8
```


#### Understanding the `send-keys` command

Tmux is basically a `server` that can host multiple `sessions` of window-groups inside a terminal. It's very versatile. If we would like to have 2 window-splits in our session we would refer to them as `panes`. Tmux likes to number these sessions sequentially (starting from 0 - they can also be named) and the panes inside them too, by appending a second number after the session, e.g. `0:0`. Luckily for us, the tidal-vim developers have all these sorted out and when, in our Terminal, call
```bash
$ tidalvim example.tidal
```

a new tmux session starts (which is named `tidal`) with 2 panes stacked horizontally, above lies our Tidal code and below the Tidal interpreter, as you can see here (note the `panes` numbers, 0 and 1):


![](/pics/tmux_panes.png "The tidal tmux session showing the numbered panes")


Why are all these important?  
Because the basic structure of the `send-keys` command should follow this pattern:

```bash
$ tmux send-keys -t 0 "message in quotes" Enter
```

`-t` stands for `target`, which refers to the `session & pane` we want to send our message to and `0` is the session id, which can also be called by its name. So, in our case, the correct way to declare the target is:

```bash
$ tmux send-keys tidal.0
```
`tidal` is the session name and `0` the upper pane where we want to remotely type.  
We are now ready to send our first message to our `vim-tidal` window by typing in our Terminal:

```bash
$ tmux send-keys -t tidal.0 "i isn't that great?" Escape Enter
```

Attention to the details: Since Vim is a modal editor, in order to enter _Insert_ mode - which allows us to type - we'll have to start our string message with `i` ( the Vim shortcut for _Insert_ mode ).  
Hence, we typed `"i isn't that great?"`. Then we typed `Escape` to exit to _Normal_ mode and then `Enter` to send our message.
I guess, you can now see where this is going :-D


#### Transforming it into SC code

SuperCollider has numerous methods to communicate with the Unix system and the one we will use here is `.unixCmd`. As this is a `String` method we will have to enclose our whole message into `" "` double-quotes and escape any double-quote character that lies inside the message with backshlash `\`. The above command can now be evaluated from within SC like this:

```supercollider
"tmux send-keys -t tidal.0 \"i isn't that great? \" Escape Enter".unixCmd;
```

In case you are using a different shell than `bash` - I use `zsh`- you will have to declare it:

``` supercollider 
"source ~/.zshrc;  tmux send-keys -t tidal.0 \"i isn't that great? \" Escape Enter".unixCmd;
```

### Preparing Vim

Next we will need to setup Vim so 


# MIDI

#### Setup

After you connected you MIDI controller, you have to initialize the SuperCollider MIDI engine by evaluating:

``` supercollider
MIDIClient.init;
```

You will see something like this on the SC post window, depending on your connected devices:

``` supercollider
MIDI Sources:
    MIDIEndPoint("IAC Driver", "Bus 1", -111043670)
    MIDIEndPoint("APC Key 25", "APC Key 25", -855105762)
MIDI Destinations:
    MIDIEndPoint("IAC Driver", "Bus 1", -329508479)
    MIDIEndPoint("APC Key 25", "APC Key 25", 453773599)
-> MIDIClient
```

We are interested in the MIDI Sources, as we want to receive MIDI messages, so we eval

```
MIDIIn.connectAll;
```

to connect to all of our MIDI Input devices. In case you want to check which CC numbers your controller's buttons hold, you can enable/disable tracing by evaluating:

``` supercollider
MIDIFunc.trace(bool: true);

// disable it when done
MIDIFunc.trace(bool: false);
```
When `true`, any press on the MIDI controller will post its details on the Post window. In that way you can see the CC nr of the buttons you want to use. Don't forget to disable it when you're done.   
For a detailed overview of the MIDI system in SuperCollider search for the _Using MIDI_ tutorial, in the SC Help window.


#### Controller Layout

Let's say that we'll map 8 of our controller buttons to the first 8 patterns. We will need to:  

* store the patterns
* recall the patterns

In order to do that we will use an extra button as a `Shift` key. Then we could hold `Shift + Button` to store a pattern in a button and then simply press that  button to recall/evaluate the pattern. This allows us to re-assign blocks of code on the fly.

#### MIDI Functions

We will use a SuperCollider class called `MIDIdef()` and especially its `MIDIdef.cc()` subclass. This one listens for a specific CC input message and evaluates a given function. Shown below, are the type of values that it expects as its first 4 arguments:   
```
MIDIdef.cc( name, function, midi-CCnumber, midi-channel)
```
`name` is used to name this specific instance of `MIDIdef.cc()` and it can be anything we like, the rest are self explanatory.
In my case, the 8 buttons I want to use span the CC numbers range 32-39. I will have to remap that range back to 1-8 in a new function that I call `~renum`, like so:

```supercollider
~renum = { |x| x.linlin(32, 39, 1, 8).asInt };
```
If we pass it a number in the range 32-38 it will scale it to 1-8:

```supercollider
~renum.(34)

-> 3
```

> Hint: you can learn more about any of the SuperCollider methods by double-clicking on the method ( e.g. `linlin`) and hit cmd-d , or ctrl-d.


Our SuperCollider code will now look like this :

```supercollider
(
/* evaluate this block by placing the cursor anywhere between the
outer parentheses and hit cmd-enter or ctrl-enter. */

~renum = { |x| x.linlin(32,39,1,8).asInt }; // this maps nr. 32-39 to 1-8

// create 8 new MIDI CC responders - substitute the CC numbers (32..39) with yours

(32..39).do{ |cc|
	var patNum = ~renum.(cc); // transform the incoming numbers 
	MIDIdef.cc(("pat_"++patNum).asSymbol, // name each MIDIdef according to pattern number
    {
		("this is a test print for button  "++(cc)).postln;
	}, cc, 0); // assign CC number and Midi channel
};
);
```
By pressing any of the mapped buttons you should be able to see its CC number on the Post window.  
If you want to clear your assignments you can call `MIDIdef.freeAll`  

For the `shift` buttton, apart from its `MIDIdef.cc()`, we'll have to create a global variable and update its state when the button is pressed. Simplest way will be to toggle its value between `true` and `false` states.

```supercollider
(
~shift = false;

MIDIdef.cc(\shiftMomentary, {~shift = true }, 64, 0, argTemplate:{ |val| if(val==0) {~shift=false} {~shift=true}});

)
```
Here we named this new MIDIdef as `\shiftMomentary` and whenever the button ( CC=64) is pressed it sets `~shift` to `true`. We also used the special argument `argTemplate`, which passes the value of the button (0 or 127) to a boolean function. This function monitors whether the button is released `if(val==0)` and if so, sets`~shift` back to `false`.

# Storing Patterns

In the following scenario we have just live-coded ( in Tidal ) the following block of code shown below and we want to "tag" it with `-- pat3`  

```haskell

d1 $ every 3 (within (0.25, 0.75) (striate 2). (|* speed 1.5))
$ every 4 (# n "3 7 9")
$ s "g:5" # cut (-1) # speed 0.8

```

Vim's _Normal_ mode equips us with a ton of key-commands to navigate our document. We are going to use some of these in order to accomplish our goal. The only prerequisite here is that our cursor has to lie somewhere within the block we want to tag.  

When in _Normal_ mode, the left curly brace `{` places our cursor at the top of our paragraph (in the empty line) and the right parenthesis `)` moves it forward at the first line of our code-block. Then, capital o `O` places our cursor just above it and switches editing into _Insert_ mode and we are ready to type our `tag.` In a Terminal we could accomplish all this by typing: 

```bash

$ tmux send-keys -t tidal.0 Escape "{) O -- pat3" Escape Enter
```

Wonderful!  

By the way, the first `Escape` command before our string is needed to ensure that we are no more in _Insert_ mode, but in _Normal_ .  

Combining all the above in SuperCollider looks like this:  

```
(

/* evaluate this block by placing the cursor anywhere between the
outer parentheses and hit cmd-enter or ctrl-enter. */

MIDIdef.freeAll // clear any previous MIDI assignements
~renum = { |x| x.linlin(32,39,1,8).asInt }; // this maps nr. 32-39 to 1-8
~shift = false; // initialize shift state
~tmux_path = "tmux send-keys -t tidal.0 Escape "; // tmux global string

// assign Shift button
MIDIdef.cc(\shiftMomentary, {~shift = true }, 64, 1, argTemplate:{ |val| if(val==0) {~shift=false} {~shift=true}});

// assign Patterns to 8 buttons
(32..39).do{ |cc|
	var patNum = ~renum.(cc);
	MIDIdef.cc( ("pat_"++patNum).asSymbol,{
		if (~shift == true){
			(~tmux_path++("\"{) O-- "++patnum++" \" Escape Enter ")).unixCmd; // set pattern's name in TidalCycles 
		}{
			// here we will write code that triggers our pattern
		}
	}, cc, 0);

	cc.postln;
};
);

``` 

Note a few things:  
* We have now placed our global `tmux` string in the `~tmux_path` variable. Pay attention to any blank spaces (such as the one after the word `Escape`) as we will be concantnating strings and spaces will be important.
* Our 8-buttons `MIDIdef.cc()` main function now uses an `if` statement to track whether the `shift` button is pressed, hence it will swith between storing and triggering a pattern.
* asda


[elliLive]:https://github.com/tadklimp/ellilive_old
[grr]:https://github.com/antonhornquist/Grrr-sc
[tidal]:https://tidalcycles.org
[sc]:https://supercollider.github.io/
[vs]:https://code.visualstudio.com/docs/editor/command-line
[tmx]:https://github.com/tmux/tmux/wiki
[send]:https://blog.damonkelley.me/2016/09/07/tmux-send-keys/
[vim]:https://github.com/vim
[install]:https://tidalcycles.org/index.php/Userbase
[mads]:https://github.com/madskjeldgaard/talks-tutorials-workshops/blob/master/tutorials/scvim/scvim-installation.md
[vimtdl]:https://github.com/tidalcycles/vim-tidal
