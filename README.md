## Setting-Up-Nethack-3.4.3-Server
A guide on how to compile [Nethack 3.4.3](https://www.nethack.org/v343/release.html) and [Dgamelaunch](https://github.com/paxed/dgamelaunch), and to configure them to serve game sessions via telnet.

This guide details how to set up a Nethack 3.4.3 server on CentOS 7 or 8. If you are using a different distribution, the required packages may have different names, and the commands used to install them will differ depending on your package manager.

The steps to setup a Nethack 3.4.3 server are as follows:
1) [Installing Requisite Packages](#installing-requisite-packages)
2) [Configuring Nethack for the Chroot Environment](#configuring-nethack-for-the-chroot-environment)
3) [Compiling Nethack](#compiling-nethack)
4) [Compiling Dgamelaunch](#compiling-dgamelaunch)
5) [Creating the Chroot](#creating-the-chroot)
6) [Configuring Dgamelaunch](#configuring-dgamelaunch)
7) [Testing Nethack and Dgamelaunch](#testing-nethack-and-dgamelaunch)
8) [Setting up the Telnet Server](#setting-up-the-telnet-server)

Here are the directories we will be using in this example:

- Nethack will be configured before compiling in /home/nethack-temp/
- The chroot location will be /home/nethack/
- Nethack will be compiled into /home/nethack-compiled/
- Dgamelaunch will be compiled into /home/dgamelaunch/

### Installing Requisite Packages
The commands that you will run to install all required packages:
```
yum install gzip make gcc ncurses-libs ncurses-devel byacc flex autoconf automake git sqlite sqlite-devel xinetd telnet-server

dnf config-manager --set-enabled powertools #If CentOS8
yum install flex-devel
```

Required packages for compiling nethack:
```
yum install gzip make gcc ncurses-libs ncurses-devel byacc flex
```

Required packages for compiling dgamelaunch:
```
yum install git autoconf automake sqlite sqlite-devel

dnf config-manager --set-enabled powertools #If CentOS8
yum install flex-devel
```

Required packages for the telnet server:
```
yum install xinetd telnet-server
```

#### Downloading Nethack

Download the tarball and unpack it.
```
mkdir  /home/nethack-temp/ && cd  /home/nethack-temp/
wget http://downloads.sourceforge.net/project/nethack/nethack/3.4.3/nethack-343-src.tgz
tar -xf nethack-343-src.tgz
cd nethack-3.4.3
```

### Configuring Nethack for the Chroot Environment
##### 1) Edit src/cmd.c

Change function enter_explore_mode() so users cannot do that; comment it out.
Here's what that function will look like when commented out:
```
enter_explore_mode()
{
/*  if(!discover && !wizard) {
*           pline("Beware!  From explore mode there will be no return to normal game.");
*           if (yn("Do you want to enter explore mode?") == 'y') {
*                   clear_nhwindow(WIN_MESSAGE);
*                   You("are now in non-scoring explore mode.");
*                   discover = TRUE;
*           }
*           else {
*                   clear_nhwindow(WIN_MESSAGE);
*                   pline("Resuming normal game.");
*               }
*       }
*/      return 0;
}
```
##### 2) Edit include/config.h 

Change HACKDIR to /nh343 (there is more than one definition of HACKDIR)

Here are the lines (with line numbers visible) that need to be changed:
```
[root@server nethack-3.4.3]# grep -n HACKDIR include/config.h  | grep define | grep -v '*'
81:# define HACKDIR "/boot/apps/NetHack"
113:# define HACKDIR "\\nethack"
207:#  define HACKDIR "/usr/games/lib/nethackdir"
```

They should look like this when you are done:
```
[root@server nethack-3.4.3]# grep -n HACKDIR include/config.h  | grep define | grep -v '*'
81:# define HACKDIR "/nh343"
113:# define HACKDIR "\\nh343"
207:#  define HACKDIR "/nh343"
```

Change COMPRESS to /bin/gzip

Comment out these:
```
#define COMPRESS "/usr/bin/compress"
#define COMPRESS_EXTENSION ".Z"  
```
Uncomment these, and change to /bin/gzip :
```
/* #define COMPRESS "/usr/local/bin/gzip" */
/* #define COMPRESS_EXTENSION ".gz" */
```

##### 3) Edit sys/unix/Makefile.top

- Comment all lines that reference SHELLDIR (We don't need to install the shell script that is usually used to launch NetHack)
- Change PREFIX to the directory which will contain the compiled NetHack, in this case "/home/nethack-compiled/"
- Change GAMEDIR to $(PREFIX)/nh343 (This must match HACKDIR in include/config.h)
- Change VARDIR to $(GAMEDIR)/var  (This must match VAR_PLAYGROUND in include/unixconf.h)
- Change GAMEUID and GAMEGRP to the user and group you will run nethack as; use numbers, not names. In CentOS 8, they are 12 and 20 respectively. In CentOS 7, they are 12 and 100.

##### 4) Edit sys/unix/Makefile.src

Comment this line:
```
WINTTYLIB = -ltermlib
```

Uncomment This line: 
```
#WINTTYLIB = -lncurses
```

##### 5) Edit include/unixconf.h

Change VAR_PLAYGROUND to "/nh343/var"

### Compiling Nethack
##### 1) Run the setup script and `make all`.
```
chmod +x ./sys/unix/setup.sh
./sys/unix/setup.sh
make all
```

If you get an error involving win/tty/termcap.c, then

For this section of win/tty/termcap.c at about line 838:
```
#ifndef LINUX
 extern char *tparm();
#endif
```

Change it to instead read as follows, and run `make all` again.
```
#ifndef NCURSES_EXPORT
 extern char *tparm();
#endif
```

##### 2) Compile it.
```
make install
```

If you have to go back and fix anything in the makefiles, you should run `make clean` and `./sys/unix/setup.sh` after.

### Compiling Dgamelaunch

##### 1) Download the latest version from GitHub
```
mkdir /home/dgamelaunch/ && cd /home/dgamelaunch/
git clone https://github.com/paxed/dgamelaunch.git
```

##### 2) Run autogen.sh, making sure to use an etc/dgamelaunch.conf file location within the chroot directory you'll be using.
```
cd dgamelaunch
./autogen.sh --enable-sqlite --enable-shmem --with-config-file=/home/nethack/etc/dgamelaunch.conf
```

##### 3) Compile it.
```
make 
```

### Creating the Chroot
##### 1) Edit these lines in dgl-create-chroot, as follows:
```
  CHROOT "/home/nethack/"
  NETHACKBIN="/home/nethack-compiled/nh343/nethack"
  NH_PLAYGROUND_FIXED="/home/nethack-compiled/nh343"
```

##### 2) Setup the Chroot
```
./dgl-create-chroot
```

##### 3) Create These Files and Directories
```
cd /home/nethack/
touch nh343/perm
mkdir nh343/save
chmod 777 nh343/save
```

### Configuring Dgamelaunch
##### Edit these lines in /home/nethack/etc/dgamelaunch.conf as follows:
```
chroot_path  (enter full chroot path) "/home/nethack/" #The backslash at the end matters
maxusers     (maximum REGISTERED users, not simultaneous
SERVERID    (your server name)
shed_uid     (UID of user "games")
shed_gid     (GID of group "games")
menu_max_idle_time (uncomment)
```
### Testing Nethack and dgamelaunch
##### 1) Make sure that you can run nethack inside the chroot environment.
Try the following as root:
```
cd /home/nethack/
chroot ./ nh343/nethack
```

##### 2) Make sure that dgamelaunch works.
```
cd /home/nethack/
./dgamelaunch
```
If you do not get any output when running dgamelaunch, something is wrong.

### Setting up the Telnet Server
##### 1) Install the required packages if you have not already.
```
yum install xinetd telnet-server
```

##### 2) Set up the telnetd to accept incoming connections; if you're using xinetd like we are in this example, you can put this in /etc/xinetd.d/dgl:
```
service telnet
{
socket_type     = stream
protocol        = tcp
user            = root
wait            = no
server          = /usr/sbin/in.telnetd
server_args     = -h -L /home/nethack/dgamelaunch
rlimit_cpu      = 120
}
```

##### 3) Start the telnet server:
```
service xinetd start
chkconfig xinetd on
```

#### Testing the Telnet Server

Try connecting locally using 'telnet 127.0.0.1'.

If you get an immediate "connection closed by foreign host", then something is probably wrong with dgamelaunch. A common issue is that the dgamelaunch config file was not specified correctly, and dgamelaunch is looking for it in /etc/dgamelaunch.conf and not in the chroot.

#### Customizing the Dgamelaunch Menus
- Edit dgl-banner
- Edit dgl_menu_*

#### Cleaning Up
/home/nethack-temp/ and /home/nethack-compiled/ can be removed if you don't need them.

### External Links and References Used to Compile This Guide
https://nethackwiki.com/wiki/User:Paxed/HowTo_setup_dgamelaunch

https://web.archive.org/web/20170107151800/http://wiki.mc128k.info/index.php/Dgamelaunch_configuration
