I needed to set up a development environment on Linux for a customer project. This needed to run on Linux since one of my dependencies (let's call it R) runs only on Linux and the VM needed an abundance of memory (R needs at least 7GB free to even start up). The way I set this up, though quite straightforward, may be useful to other developers with similar memory requirements. 

I needed 12Gb of VM RAM for this project because I also had to run a a different VM (call that S) with 4GB minimum RAM. Since my laptop has 16GB and the host OS also needs a heap of RAM, I didn't have room for a desktop environment in the Linux VM for R.

###Ingredients
-	VirtualBox (or something similar)
-	SSH with X forwarding
-	Eclipse (or whatever IDE you like)
-	X Server on host operating system (XQuartz on Mac OS X)

###Setup
1.	Create a Virtual Machine with 8GB of RAM and install CentOS 7 with GNOME desktop
1.	Install development tools:  yum groupinstall "Development tools"
1.	Install eclipse for C++ from <https://eclipse.org/downloads/packages/eclipse-ide-cc-developers/keplersr2>
1.	Disable the GUI at startup to free up memory for R: `systemctl set-default multi-user.target`

###Usage
To use this setup for development just start your VM headless, SSH into it and run:

```
$ ~/eclipse/cpp-neon/eclipse/eclipse &
```

And eclipse will start up on your host desktop. It will run as if it was running on your host machine. Because you’re not running a full desktop environment on the VM, there will be enough memory for R, S and your host operating system to run.