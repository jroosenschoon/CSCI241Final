# CSCI241Final
This is a repo that stores my final for CSCI241 at MJC. It will go over how to create Debian 12 on VMware Workstation and then on that VM, install a debian 12 VM on kvm. We will also use a preseed file to automate most of the install of Debian 12.

## VMWare workstation Install
**Prerequisites**:
 - Installation of VMWare Workstation
 - Debian 12 iso file
 - Preseed files (found in this repo)
 - post run file (found in this repo)

First, we will use VMWare Workstation to create a GUI-based Debian 12 machine.
  1. Go to File -> New Virtual Machine - this will launch a wizard to create a VM.
  2. Click Custom install (we want this so we can use a virtual hard disk)
  3. In the Choose the Virtual Machine hardware compatibility, use the default workstation 17.5 or later and click Next
  4. In the Install operating system from option, select Use ISO Image and select your debian 12 iso.
  5. In the Name the Virtual Machine window, give the machine a name (we will call it debian12Nest0) and leave the Location as the default.
  6. For the Processor Configuration option, it depends on your PC's hardware. Since we are doing a nested VM, try for at least 1 processor and 3 cores.
  7. The memory page also depends on your hardware. We will do 8 GB
  8. For the network connection, we will use the bridged option so we can connect to the network with our VM. Our VM will behave as if it were directly connected to our network.
  9. For I/O Controller types, use the default LSI Logic
  10. For disk type, use the default SCSI
  11. For Disk, select Create a new virtual disk
  12. For Disk size, just use 10 GB or so. We can choose to store it as a single file (helpful when you are transferring the VM or we can choose to split it; we will just do the default)
  14. For the Disk file, we will just use the default file name.
  15. Finally click Finish
  16. On this first VM, we will do a graphical install so we can use KVM GUI to make the nested on. Select Graphical Install and go through the installer until you get to the Partition Disks section, accepting the defaults and creating a tux user account.
  17. On the Partition Disks section, select Guided - use entire Disk and set up LVM. Then continue through, accepting defaults.
  18. In the Software selection, install the web server and SSH server so we can set up a web server and SSH server.
  19. Finish off the installer and reboot and we will finally be in our first Debian 12 VM!


## Configure VMWare VM
Now that we have our VM, we will add some simple configurations to it. We will do this through a post run script (https://phillipsd.com/210/post_run).
First, we need to add our tux user to the sudoers file:
```
su -
apt install sudo
usermod -aG sudo tux
```
Now tux can use sudo.

Next, we will run `wget https://phillipsd.com/210/post_run` to get the post run script.
Finally, run `bash post_run` to run our post run script to set up our VM. 
Reboot your machine and it will launch in a CLI. To get back to a GUI, type `startx` as your tux user.

Let's look at our post run file in some more detail.

First We have some lines that updates our VM and installs some packages we will be using: 
```bash
sudo apt-get update && sudo apt-get upgrade -y

aptDepends=(
sudo
a2ps
acpid
aspell
astyle
autossh
bc
bind9-dnsutils
bsdgames
bsdmainutils
bsdutils
build-essential
byobu
bzip2
calendar
calc
coreutils
cryptsetup
cu
cups
curl
dc
dialog
dkms
ecl
elinks
fortune-mod
fortunes
gpm
jq
links
lua5.4
lynx
mailutils
minicom
mutt
ncal
ncat
net-tools
newlisp
nmap
nvi
pandoc
printer-driver-cups-pdf
python3-pdfminer
rsync
screen
shellcheck
spice-vdagent
sqlite3
sshfs
tmux
util-linux
vacation
vim-nox
vim-scripts
w3m
wget
whois
x11-apps
zlib1g-dev
virtinst 
ufw
qemu-kvm 
libvirt-clients 
libvirt-daemon-system 
libvirt-daemon 
bridge-utils 
qemu-system 
qemu-utils 
virt-manager 
)
sudo apt-get install -y "${aptDepends[@]}" 
```

Next, we have the code to set up our web sever: 
```
# web_ins
# install apache2
sudo tasksel install web-server 
# change default webpage
cd /var/www/html/
sudo mv index.html apache_default.org 
sudo wget https://phillipsd.com/basic.html 
sudo mv basic.html index.html
```
This will move a basic webpage (courtesy of the professor) into our home page of our web server

Next, we will add some fun functions in ~/.funcs:
```
cd

cat - > ~/.funcs << "FUN"
whdr() {
echo '<!DOCTYPE html><head><title>bash web</title></head><body><pre>'

}
wftr() {
echo '</pre></body></html>'

}
calcit() {
    printf "%s\n" "$@" | bc -lq ~/.bcrc  
}
FUN
```
We will add some handy aliases to our .bash_aliases, change our editor, add a MYIP variable, and add our functions from .funcs:
```
cat - > ~/.bash_aliases << "EOF"
#.bash_aliases 
# usually in .bashrc 
export PATH="$HOME/bin:$PATH" 
export EDITOR="vim"
export EDIT="vim"
export MYIP=$(hostname -I | cut -f1 -d" ")
alias h=' history'
alias l=' ls -sailF --color=auto'
alias la=' ls -A --color=auto'
alias lc=' ls -CF --color=auto'
alias ll=' ls -alF --color=auto'
alias lr=' ls -artlF --color=auto'
alias lr1=' ls -1rtlF --color=auto'
# source .funcs 
if [ -f ~/.funcs ]; then
    . ~/.funcs
fi
echo $MYIP
EOF
```
Next, we add /bin/upit - a handy file to update our VM. We will also make it executable:
```
mkdir bin
cat - > ~/bin/upit << "UPIT"
#!/bin/bash
# upit
#snap refresh - ubuntu desktop
sudo apt-get check
sudo apt-get clean
sudo apt-get -y autoclean
sudo apt-get remove
sudo apt-get -y autoremove
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y dist-upgrade
uname -a
cat /proc/version
cat /etc/os-release
#--
UPIT
chmod +x ~/bin/upit
```

And we will set up our serial port: 
```
sudo systemctl enable serial-getty@ttyS0.service
sudo systemctl start serial-getty@ttyS0.service
sudo systemctl daemon-reload
```
And allow ssh and web traffic through our firewall:
```
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow "WWW Full"
```
This line will boot us to a CLI so you may want to take it out:
```
sudo systemctl set-default multi-user
```

Finally we will rename our VM to its MAC address
```
MACNAME=$(ip a show enp4s0f0 | head -2 | tail -1 | awk '{print $2}' | awk -F":" '{print $4 $5 $6}')
echo "lab-"$MACNAME
echo "lab-"$MACNAME > tmp.txt
sudo mv tmp.txt /etc/hostname
hostname 
hostname -I
```

Our first VM is ready to go!

## Creating Debian 12 VM on KVM 

Now let's create our KVM VM. Part of the post run script installed all of the necessary packages to do this. We can just search for and run virt-manager.

Now go File -> New Virtual Machine
We will then go through a very similar process to what we did with VMWare.
Once we do this and reboot, we get to the installer for Debian. From here, we will do it a little different - go to Advanced -> Automated Install
Soon, we will be prompted for a preseed file - this selects the options for us. We will use https://phillipsd.com/210/f24vda.cfg to do this.

Once this is done, and we finish the installer, we will have a Debian 12 VM running on KVM which is itself running on a Debian 12 VM that is on the VMWare stack.


