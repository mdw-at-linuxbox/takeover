takeover - initial system setup
===============================

I use this just after doing a minimal OS install.
I used to do all this stuff by hand.
Using ansible gives me more accurate results faster.

Some of the logic is very specific to me - use at own risk!

Synopsis
--------

ansible-playbook take-over.yml

Prerequisites
-------------
The target system is assumed to have a minimal install.
Also, it should have some initial user configured,
and this user needs to have sudo root rights (without a password).

It is preferable that this user be neither root nor the
login you intend to use daily; it is sufficient (and best) if
this user can only authenticate with an ssh key.

What it does
------------

system,
update
adds a local repository (bull.repo)
entropy daemon (rngd or haveged).
timezone and hostname.
disable ssh with passwords (if disable_sshd_passwordauth set to true).
disable ssh with password for root (always).
add more stuff: standard system tools.
kerberos configuration: (for testing).
special commands: nvi, lynx, rcs as necessary.
add users -- my colleagues at work (and their ssh keys).
sudoers

for me (mdw),
dot files
bin directory with scripts, personal executables.

Configuration
-------------

Before using this script, it is necessary to populate
group_vars/all.yml
files/ssh-keys.tar.xz

For
group_vars/all.yml

required fields,

 * default_user_pass:
   hashed (crypt) password.
   You should use something like "md5crypt" or "sha256crypt" and not
   the ancient default 56-bit des "descrypt".
   But really, you want to use sha keys instead.

optional fields,

 * disable_sshd_passwordauth: true
 * disable all password based authentication
 * disable_root_login: false
   don't turn root login by password off in sshd.

at least one of,

 * mdw_tar_files_el7
 * mdw_tar_files_el8
 * mdw_tar_files_el9
 * mdw_tar_files_fc
 * mdw_tar_files_deb
 * mdw_tar_files_ubuntu

(should match your selected os).

For
files/ssh-keys.tar.xz
should have N files, one per user listed in roles/users/vars/main.yml
each file should be named <loginid>/.ssh/authorized_keys

In the ansible hosts file, you must provide a host group "linode"
and list your hosta after that.  For recent fedora | centos,
you also need to set ansible_python_interpreter
to /usr/libexec/platform-python .
For fc31, it might need to be /usr/bin/python3 .

It is permissble to initialize more than one host and they do not
all need to have the same OS.

Supported OSes
--------------

* Recent fedora (26 + in theory, 35 last tested).

* Some flavors of ubuntu (last tested with focal).

* Debian (recentish.)

Run Template
------------

On the target system as part of the initial server setup, as root
Copy & paste script:

.. code::

    useradd -m servus
    dnf install tar
    echo 'servus ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/servus
    base64 -di<<.|xz -d|tar xvf - -C /
    <<< PASTE IN OUTPUT FROM tar cf - home/servus/.ssh/authorized_keys | xz | base64 >>>
    .
    chown -R servus:servus /home/servus/.ssh
    chmod 700 /home/servus/.ssh

On a utility host:

Set up ssh key,

.. code::

    bash ; echo back to normal
    PS1="$HOSTNAME (servus)% "
    eval `ssh-agent`
    ssh-add ~/.ssh/servus-ansible
    export ANSIBLE_NOCOWS=1

Generate hashed password, fixed default password (Bad!) for all users.
Generally, use ssh keys not passwords.
For public installs, do disable passwords with disable_sshd_passwordauth .

.. code::

    while :; do
    CRYPT=`mkpasswd -m sha256crypt`
    SALT=`echo $CRYPT | sed 's%.*\$\([^$]*\)\$[^$]*$%\1%'`
    C2=`mkpasswd -m sha256crypt --salt $SALT`
    [ $CRYPT == $C2 ] && break
    echo try again
    [ -f /tmp/oops ] && break
    done

Set hostname, IP address.

.. code::

    MY_NAME=<<< name of test host.  You *are* using dns right? >>>
    MY_IP=`host $MY_NAME | sed 's%.* \([^ ]*\)*$%\1%'`

    echo $MY_NAME $MY_IP

Verify access, root rights.

.. code::

    ssh servus@$MY_NAME id
    ssh servus@$MY_IP sudo id

Setup ansible configuration,

.. code::

    cat <<.>ansible.cfg
    [defaults]
    remote_user = servus
    inventory = `pwd`/hosts
    .
    cat <<.>hosts
    [linode]
    $MY_NAME ansible_host=$MY_IP ansible_python_interpreter=/usr/libexec/platform-python
    .
    echo export ANSIBLE_CONFIG=`pwd`/ansible.cfg > ,setup
    . ,setup

Verify ansible configuration correct,

.. code::

    ansible all -m ping
    ansible linode -m command -a id -b

Configure takeover,

.. code::

    cd takeover
    mkdir -p group_vars files

    cat <<.>group_vars/all.yml
    default_user_pass: "$CRYPT"
    mdw_tar_files_el9:
    - mdw-dot-rhel9.tar
    - mdw-bin-rhel9.tar
    .
    ln -s <<< name of tar-file-with-ssh-keys >>> files/ssh-keys.tar.xz
    for X in bin dot; do
    ln -s ~/dot/,$X-fc files/mdw-$X-rhel9.tar; done
    wc files/*

Run the playbook,

.. code::

    ansible-playbook take-over.yml

Tear down environment,

.. code::

    ssh-agent -k
    exit

See Also
--------

Other examples of the same concept

 * digitalocean <https://www.digitalocean.com/community/tutorials/how-to-use-ansible-to-automate-initial-server-setup-on-ubuntu-20-04>
 * lukeharvey <https://github.com/lukeharvey/ansible-initial-server-setup>
 * vultr <https://www.vultr.com/docs/how-to-configure-a-new-ubuntu-server-with-ansible/>
