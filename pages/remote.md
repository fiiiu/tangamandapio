# Canonistack hub

 1. Make an m1.tiny canonistack instance with ssh and mosh ports open.
 2. Go to gandi and add the `canonistack` DNS entry for this instance.
 3. ssh into the new machine and:

    sudo adduser elopio
    su elopio
    ssh-import-id elopio
    exit

 4. ssh into the machine again as elopio and:

    sudo deluser ubuntu
    sudo rm -rf ubuntu
    exit

 5. Copy the openstack configuration file from the host:

    scp ~/.canonistack/novarc_lcy02 elopio@canonistack.elopio.net:/tmp
    ssh elopio@canonistack.elopio.net
    mkdir ~/.canonistack
    mv /tmp/novarc_lcy02 ~/.canonistack/
    source ~/.canonistack/novarc_lcy02
    sudo snap install juju --beta --devmode

 6. On the canonistack machine, follow the CDO shared controllers instructions.

    sudo snap install charm --devmode
    mkdir workspace

# Remote communications

 1. On the canonistack machine:

    cd ~/workspace
    git clone https://github.com/elopio/remote-comms
    cd remote-comms
    charm build
    cd builds
    juju add-model remote-comms --credential canonistack
    juju deploy ./remote-comms
    juju expose remote-comms
    juju ssh remote-comms/0

 2. On the remote-comms machine:

    * replace $freenode_password in ~/.weechat/irc.conf
    * replace $personal_email_password in ~/.msmtprc and ~/.offlineimaprc
    * replace $canonical_google_password in ~/.msmtprc and ~/.offlineimaprc

# Remote devel

 1. On the canonistack machine:

   cd ~/workspace
   git clone https://github.com/elopio/remote-devel
   cd remote-devel
   charm build
   cd builds
   juju add-model remote-devel --credential canonistack
   juju deploy ./remote-devel --constraints instance-type=cpu4-ram18-disk50-ephemeral20
   juju expose remote-devel
   juju ssh remote-devel/0

 2. On the remote-devel machine:

   ssh-import-id elopio
