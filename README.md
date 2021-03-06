# Setup a secure CentOS 7

## Introduction

The new CentOS 7 server has to be customized before it can be put into use as a production system. In this article, will help you to increase the security and usability of your server and will give you a solid foundation for subsequent actions.

## Prerequisites

A newly activated CentOS 7 server, preferably setup with SSH keys. Log into the server as root with your server-ip-address.

```
ssh -l root xx.xx.xxx.xxx
```
or
```
ssh root@SERVER_IP_ADDRESS
```

Complete the login process by accepting the warning about host authenticity, if it appears, then providing your root authentication (password or private key). If it is your first time logging into the server, with a password, you will also be prompted to change the root password.

The root user is the administrative user in a Linux environment that has very broad privileges. Because of the heightened privileges of the root account, you are actually discouraged from using it on a regular basis. This is because part of the power inherent with the root account is the ability to make very destructive changes, even by accident.

The next step is to set up an alternative user account with a reduced scope of influence for day-to-day work. We'll teach you how to gain increased privileges during the times when you need them.


## Setup and Initial

### Step 1 — Create a New User

For security reasons, it is not advisable to be performing daily computing tasks using the root account. Instead, it is recommended to create a standard user account that will be using sudo to gain administrative privileges. For this tutorial, assume that we're creating a user named joe. To create the user account, type:

```
adduser example
```
Set a password for the new user. You'll be prompted to input and confirm a password.

```
passwd example
```
Add the new user to the wheel group so that it can assume root privileges using sudo.

```
gpasswd -a example wheel
```

Finally, open another terminal on your local machine and use the following command to add your SSH key to the new user's home directory on the remote server. You will be prompted to authenticate before the SSH key is installed.

```
ssh-copy-id example@server-ip-address
```
After the key has been installed, log into the server using the new user account.

```
ssh -l example server-ip-address
```

If the login is successful, you may close the other terminal. From now on, all commands will be preceded with sudo.


### Step 2 — Disallow Root Login and Password Authentication

Since you can now log in as a standard user using SSH keys, a good security practice is to configure SSH so that the root login and password authentication are both disallowed. Both settings have to be configured in the SSH daemon's configuration file. So, open it using nano.

```
sudo nano /etc/ssh/sshd_config
```
Look for the PermitRootLogin line, uncomment it and set the value to no.
```
PermitRootLogin     no
```
Do the same for the PasswordAuthentication line, which should be uncommented already:
```
PasswordAuthentication      no
```
Save and close the file. To apply the new settings, reload SSH.
```
sudo systemctl reload sshd
```

### Step 3 — Configure the Timezones and Network Time Protocol Synchronization

The next step is to adjust the localization settings for your server and configure the Network Time Protocol (NTP) synchronization.

The first step will ensure that your server is operating under the correct time zone. The second step will configure your system to synchronize its system clock to the standard time maintained by a global network of NTP servers. This will help prevent some inconsistent behavior that can arise from out-of-sync clocks.

#### Configure Timezones

Our first step is to set our server's timezone. This is a very simple procedure that can be accomplished using the timedatectl command:

First, take a look at the available timezones by typing:

```
sudo timedatectl list-timezones
```
Grep possible Asian timezones 
```
sudo timedatectl list-timezones | grep Asia
```

This will give you a list of the timezones available for your server. When you find the region/timezone setting that is correct for your server, set it by typing:
```
sudo timedatectl set-timezone region/timezone
```

For instance, to set it to United States eastern time, you can type:
```
sudo timedatectl set-timezone America/New_York
```
Your system will be updated to use the selected timezone. You can confirm this by typing:
```
sudo timedatectl
```

#### Configure NTP Synchronization

Now that you have your timezone set, we should configure NTP. This will allow your computer to stay in sync with other servers, leading to more predictability in operations that rely on having the correct time.

For NTP synchronization, we will use a service called ntp, which we can install from CentOS's default repositories:
```
sudo yum install ntp
```
Next, you need to start the service for this session. We will also enable the service so that it is automatically started each time the server boots:
```
sudo systemctl start ntpd
sudo systemctl enable ntpd
```
Your server will now automatically correct its system clock to align with the global servers.

#### How do I see the current time zone?

Type the date command or the ls command:
```
date
ls -l /etc/localtime
```

### Step 4 - Enable the IPTables Firewall

By default, the active firewall application on a newly activated CentOS 7 server is FirewallD. Though it is a good replacement for IPTables, many security applications still do not have support for it. So if you'll be using any of those applications, like OSSEC HIDS, it's best to disable/uninstall FirewallD.

Let's start by disabling/uninstalling FirewallD:
```
sudo yum remove -y firewalld
```
Now, let's install/activate IPTables.

```
sudo yum install -y iptables-services
sudo systemctl start iptables
```

Configure IPTables to start automatically at boot time.
```
sudo systemctl enable iptables
```
IPTables on CentOS 7 comes with a default set of rules, which you can view with the following command.
```
sudo iptables -L -n
```
The output will resemble:

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

You can see that one of those rules allows SSH traffic, so your SSH session is safe.
Because those rules are runtime rules and will be lost on reboot, it's best to save them to a file using:
```
sudo /usr/libexec/iptables/iptables.init save
```
That command will save the rules to the /etc/sysconfig/iptables file. You can edit the rules anytime by changing this file with your favorite text editor.

### Step 5 - Allow Additional Traffic Through the Firewall

Since you'll most likely be going to use your new server to host some websites at some point, you'll have to add new rules to the firewall to allow HTTP and HTTPS traffic. To accomplish that, open the IPTables file:

```
sudo nano /etc/sysconfig/iptables
```

Just after or before the SSH rule, add the rules for HTTP (port 80) and HTTPS (port 443) traffic, so that that portion of the file appears as shown in the code block below.
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```
Save and close the file, then reload IPTables.
```
sudo systemctl reload iptables
```
With the above step completed, your CentOS 7 server should now be reasonably secure and be ready for use in production.


## Update and Upgrade

To update installed packages
```
sudo yum -y update 
```
To upgrade installed packages
```
sudo yum -y upgrade
```

## Install Common Packages

### Extra Packages for Enterprise Linux (EPEL)

How do I install the extra repositories such as Fedora EPEL repo on a Red Hat Enterprise Linux server version 7.x or CentOS Linux server version 7.x?

You can easily install various packages by configuring a CentOS 7.x or RHEL 7.x system to use Fedora EPEL repos and third party packages. Please note that these packages are not officially supported by either CentOS or Red Hat, but provides many popular packages and apps.

#### How to install RHEL EPEL repository on Centos 7.x or RHEL 7.x

The following instructions assumes that you are running command as root user on a CentOS/RHEL 7.x system and want to use use Fedora Epel repos.

#### Method 1: Install EPEL repositories from Base Linux repository (recommended)

Install Epel repo using the following command:
```
sudo yum -y install epel-release
```
Refresh repo by typing the following commad: 
```
yum repolist
```

#### Method 2: Install EPEL repositories from dl.fedoraproject.org 

The command is as follows to download epel release for CentOS and RHEL 7.x using wget command:
```
cd /tmp
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo rpm -Uvh epel-release-latest-7*.rpm
```
Refresh repo by typing the following commad: 
```
yum repolist
```

#### Search and install package

To list all available packages under a repo called epel
```
sudo yum --disablerepo="*" --enablerepo="epel" list available
```

Example: Search and install htop package from epel repo on a CentOS/RHEL 7.x

## search it ##
```
sudo yum search htop
sudo yum info htop
sudo yum install htop
```

And, there you have it, a larger number of packages to install from EPEL repo on a CentOS and 
Red Hat Enterprise Linux (RHEL) version 7.x.

### OpenSSL(with ALPN support)

**Application-Layer Protocol Negotiation (ALPN)** is a Transport Layer Security (TLS) extension for application layer protocol negotiation. ALPN allows the application layer to negotiate which protocol should be performed over a secure connection in a manner which avoids additional round trips and which is independent of the application layer protocols. It is used by HTTP/2.

***Due to TLS False Start was disabled in Google Chrome from version 20 (2012) onward except for websites with the earlier Next Protocol Negotiation (NPN) extension. NPN was replaced with a reworked version, ALPN. On July 11, 2014, ALPN was published as RFC 7301.***

So to enable HTTP/2 on ALPN in chrome browser you need to be sure that you have already installed ***OpenSSL that supported ALPN** which is version >= 1.0.2.

You can verify openssl version by type
```
openssl version
```


### Nginx from source (with ALPN support)

Compiling NGINX from the sources provides you with more flexibility: you can add particular NGINX modules or 3rd party modules and apply latest security patches. 

For example of core NGINX Modules: 

**http_charset_module**	Adds the specified charset to the “Content-Type” response header field, can convert data from one charset to another.

**http_gzip_module**	Compresses responses using the gzip method, helping to reduce the size of transmitted data by half or more.

**http_ssi_module**	Processes SSI (Server Side Includes) commands in responses passing through it.

**http_userid_module**	Sets cookies suitable for client identification.

**http_access_module**	Limits access to certain client addresses.

**http_auth_basic_module**	Limits access to resources by validating the user name and password using the HTTP Basic Authentication protocol.

**http_autoindex_module**	Processes requests ending with the slash character (‘/’) and produces a directory listing.

**http_geo_module**	Creates variables with values depending on the client IP address.

**http_map_module**	Creates variables whose values depend on values of other variables.

**http_split_clients_module**	Creates variables suitable for A/B testing, also known as split testing.

**http_referer_module**	Blocks access to a site for requests with invalid values in the Referer header field.

**http_rewrite_module**	Changes the request URI using regular expressions and return redirects; conditionally selects configurations. Requires the PCRE library.

**http_proxy_module**	Passes requests to another server.

**http_fastcgi_module**	Passes requests to a FastCGI server

**http_uwsgi_module**	Passes requests to a uwsgi server.

**http_scgi_module**	Passes requests to an SCGI server.

**http_memcached_module**	Obtains responses from a memcached server.

**http_limit_conn_module**	Limits the number of connections per the defined key, in particular, the number of connections from a single IP address.

**http_limit_req_module**	Limits the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address.

**http_empty_gif_module**	Emits single-pixel transparent GIF.

**http_browser_module**	Creates variables whose values depend on the value of the “User-Agent” request header field.

**http_upstream_hash_module**	Enables the hash load balancing method.

**http_upstream_ip_hash_module**	Enables the IP hash load balancing method.

**http_upstream_least_conn_module**	Enables the least_conn load balancing method.

**http_upstream_keepalive_module**	Enables keepalive connections.

**http_upstream_zone_module**	Enables the shared memory zone.


For example of 3rd Party modules: 

**ngx_brotli module** - An open source data compression library introducing by Google, based on a modern variant of the LZ77 algorithm, Huffman coding and 2nd order context modeling. (By default nginx bundle with **gzip** compression library)
 
Brotli performance https://www.opencpu.org/posts/brotli-benchmarks/

Installing nginx brotli module https://github.com/google/ngx_brotli

**ngx_pagespeed** - Speeds up your site and reduces page load time by automatically applying web performance best practices to pages and associated assets (CSS, JavaScript, images) without requiring you to modify your existing content or workflow.  
Rewrites webpages and associated assets to reduce latency and bandwidth
 
Installing page speed module https://github.com/pagespeed/ngx_pagespeed

-------------------------------------------------------------------------------------------------------------------------

View more NGINX 3rd Party Modules: https://www.nginx.com/resources/wiki/modules/

Read more about extending nginx https://www.nginx.com/resources/wiki/extending/

Read more about dynamic modules https://www.nginx.com/blog/dynamic-modules-nginx-1-9-11/

-------------------------------------------------------------------------------------------------------------------------

Before we getting start you need to verify that your nginx was built with OpenSSL that support ALPN feature.
You can verify nginx version by type
```
nginx -V
```

### Choosing Between a Stable or a Mainline Version

NGINX Open Source is available in 2 versions:

**The mainline version** This version includes the latest features and bugfixes and is always up-to-date. It is reliable, but it may include some experimental modules, and it may also have some number of new bugs.

**The stable version** This version doesn’t have new features, but includes critical bug fixes that are always backported to the mainline version. The stable version is recommended for production servers.


### Prerequisites

### Step 1 Installing compiler and libraries

To compile nginx from source you need to install compiler and required libraries to build nginx.

1.1 Need to install epel repo for additional libraries, if not install by typing
```
sudo yum install -y epel-release
```
1.2 Then update the os environment
```
sudo yum update 
```
1.3 Install compiler 
```
sudo yum groupinstall 'Development Tools'
sudo yum -y install autoconf automake bind-utils wget curl unzip gcc-c++ pcre-devel zlib-devel libtool make nmap-netcat ntp pam-devel
```

### Step 2 Installing NGINX Dependencies

Prior to compiling NGINX from the sources, it is necessary to install its dependencies:

2.1 Install PCRE library
The PCRE library required by NGINX Core and Rewrite modules and provides support for regular expressions:
```
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz
tar -zxf pcre-8.39.tar.gz
cd pcre-8.39
./configure
make
sudo make install
```

2.2 Install ZLIB library
The zlib library required by NGINX Gzip module for headers compression:
```
wget http://zlib.net/zlib-1.2.8.tar.gz
tar -zxf zlib-1.2.8.tar.gz
cd zlib-1.2.8
./configure
make
sudo make install
```

2.3 Install OpenSSL library (lastest)
The OpenSSL library required by NGINX SSL modules to support the HTTPS protocol:
```
wget https://www.openssl.org/source/openssl-1.0.2j.tar.gz
tar -zxf openssl-1.0.2f.tar.gz
cd openssl-1.0.2f
./configure darwin64-x86_64-cc --prefix=/usr
make
sudo make install
```

### Step 3 Download NGINX source code

```
wget http://nginx.org/download/nginx-1.10.2.tar.gz
tar zxf nginx-1.xx.x.tar.gz
cd nginx-1.xx.x
```

3.1 Config the nginx for built
```
./configure \
--prefix=/etc/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib64/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--http-client-body-temp-path=/var/cache/nginx/client_temp \
--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
--user=nginx \
--group=nginx \
--with-pcre=/usr/local/src/pcre-8.39 \
--with-zlib=/usr/local/src/zlib-1.2.8 \
--with-openssl=/usr/local/src/openssl-1.0.2j \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-file-aio \
--with-threads \
--with-ipv6 \
--with-http_addition_module \
--with-http_auth_request_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_random_index_module \
--with-http_realip_module \
--with-http_secure_link_module \
--with-http_slice_module \
--with-http_sub_module \
--with-http_v2_module \
 --with-mail \
--with-mail_ssl_module \
--with-stream \
--with-stream_ssl_module \
--with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic'
```

3.2 Compile and install the build:
```
make
sudo make install
```

3.3 After the installation is finished, run NGINX Open Source:
```
sudo nginx
```

3.4 Config the firewall

```
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
systemctl restart firewalld.service
```

3.5 start nginx service by using below command
```
/usr/sbin/nginx -c /etc/nginx/nginx.conf
```

3.6 check nginx process 
```
ps -ef|grep nginx
```

3.7 to stop nginx service using below command
```
kill -9 PID-Of-Nginx
```

3.8 add nginx as systemd service by create a file "nginx.service" in /lib/systemd/system/nginx.service
 then copy below into the file
 
 ```
 [Unit]
 Description=The NGINX HTTP and reverse proxy server
 After=syslog.target network.target remote-fs.target nss-lookup.target
 
 [Service]
 Type=forking
 PIDFile=/var/run/nginx.pid
 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
 ExecReload=/bin/kill -s HUP $MAINPID
 ExecStop=/bin/kill -s QUIT $MAINPID
 PrivateTmp=true
 
 [Install]
 WantedBy=multi-user.target
 ```
 
3.9 then reload the system files
```
 systemctl daemon-reload
```

3.10 create startup script 
```
sudo ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx
sudo nano /etc/init.d/nginx then copy script below
```

```
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/sbin/nginx"
prog=$(basename $nginx)

NGINX_CONF_FILE="/etc/nginx/nginx.conf"

[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx

lockfile=/var/lock/subsys/nginx

make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -z "`grep $user /etc/passwd`" ]; then
       useradd -M -s /bin/nologin $user
   fi
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
       if [ `echo $opt | grep '.*-temp-path'` ]; then
           value=`echo $opt | cut -d "=" -f 2`
           if [ ! -d "$value" ]; then
               # echo "creating" $value
               mkdir -p $value && chown -R $user $value
           fi
       fi
   done
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

4.11 set permission to make this script be executable 
```
chmod +x /etc/init.d/nginx
sudo systemctl enable nginx
```

4.12 To make sure that Nginx starts and stops every time with the Droplet, add it to the default runlevels with the command:
```
sudo chkconfig nginx on
sudo service nginx restart
```

NOTE: If you want to install speedtest module (By Google) you must installed following libraries

gcc 
gcc-c++ 
pcre-devel 
zlib-devel 
make 
unzip 
openssl-devel 
libaio-devel
glibc 
glibc-devel 
glibc-headers
libevent
linux-vdso.so.1
libpthread.so.0
libcrypt.so.1
libstdc++.so.6
librt.so.1
libm.so.6
libpcre.so.0
libssl.so.10
libcrypto.so.10
libdl.so.2
libz.so.1
libgcc_s.so.1
libc.so.6
/lib64/ld-linux-x86-64.so.2
libfreebl3.so
libgssapi_krb5.so.2
libkrb5.so.3
libcom_err.so.2
libk5crypto.so.3
libkrb5support.so.0
libkeyutils.so.1
libresolv.so.2
libselinux.so.1


### PHP 7 (Incomplete)

If you have installed older version you must remove it first by following command.

```
sudo yum remove php-fpm php-cli php-common
```

Install the new PHP 7 packages from IUS. Again, press y and Enter when prompted.

```
sudo yum install php70u-fpm-nginx php70u-cli php70u-mysqlnd
```





## Set Up Firewall

### Introduction

After setting up the bare minimum configuration for a new server, there are some additional steps that are highly recommended in most cases.

### Prerequisites

In this guide, we will be focusing on configuring some optional but recommended components. This will involve setting our system up with a firewall and a swap file.

### Configuring a Basic Firewall

Firewalls provide a basic level of security for your server. These applications are responsible for denying traffic to every port on your server with exceptions for ports/services you have approved. CentOS ships with a firewall called firewalld. A tool called firewall-cmd can be used to configure your firewall policies. Our basic strategy will be to lock down everything that we do not have a good reason to keep open.

The firewalld service has the ability to make modifications without dropping current connections, so we can turn it on before creating our exceptions:
```
sudo systemctl start firewalld
```
Now that the service is up and running, we can use the firewall-cmd utility to get and set policy information for the firewall. The firewalld application uses the concept of "zones" to label the trustworthiness of the other hosts on a network. This labelling gives us the ability to assign different rules depending on how much we trust a network.

In this guide, we will only be adjusting the policies for the default zone. When we reload our firewall, this will be the zone applied to our interfaces. We should start by adding exceptions to our firewall for approved services. The most essential of these is SSH, since we need to retain remote administrative access to the server.

You can enable the service by name (eg ssh-daemon) by typing:
```
sudo firewall-cmd --permanent --add-service=ssh
```
or disable it by typing:
```
sudo firewall-cmd --permanent --remove-service=ssh
```
or enable custom port by typing:
```
sudo firewall-cmd --permanent --add-port=4200/tcp
```
This is the bare minimum needed to retain administrative access to the server. If you plan on running additional services, you need to open the firewall for those as well.

If you plan on running a conventional HTTP web server, you will need to enable the http service:
```
sudo firewall-cmd --permanent --add-service=http
```
If you plan to run a web server with SSL/TLS enabled, you should allow traffic for https as well:
```
sudo firewall-cmd --permanent --add-service=https
```
If you need SMTP email enabled, you can type:
```
sudo firewall-cmd --permanent --add-service=smtp
```
To see any additional services that you can enable by name, type:
```
sudo firewall-cmd --get-services
```
When you are finished, you can see the list of the exceptions that will be implemented by typing:
```
sudo firewall-cmd --permanent --list-all
```
When you are ready to implement the changes, reload the firewall:
```
sudo firewall-cmd --reload
```
If, after testing, everything works as expected, you should make sure the firewall will be started at boot:
```
sudo systemctl enable firewalld
```
Remember that you will have to explicitly open the firewall (with services or ports) for any additional services that you may configure later.

