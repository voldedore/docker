#################################################
#        ICINGA2 + ICINGAWEB2 + MARIADB         #
#             for local use only                #
#                                  thevinh@MKD  #
#################################################

## Changelog
## Update date 27.04.2017 Graphite web integrated


# Use the main image of centos systemd base img
# https://hub.docker.com/_/centos/
# This Dockerfile deletes a number of unit files which might cause issues. 
# From here, you are ready to build your base image.
FROM centos:7
ENV container docker
# RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
# systemd-tmpfiles-setup.service ] || rm -f $i; done); \
# rm -f /lib/systemd/system/multi-user.target.wants/*;\
# rm -f /etc/systemd/system/*.wants/*;\
# rm -f /lib/systemd/system/local-fs.target.wants/*; \
# rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
# rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
# rm -f /lib/systemd/system/basic.target.wants/*;\
# rm -f /lib/systemd/system/anaconda.target.wants/*;
# VOLUME [ "/sys/fs/cgroup" ]
# CMD ["/usr/sbin/init"]

# Installation of Icinga2
# https://hub.docker.com/r/icinga/icinga2-demo/~/dockerfile/
RUN yum -y update; yum clean all; \
 yum -y install epel-release; yum clean all; \
 yum -y install https://packages.icinga.com/epel/7/release/noarch/icinga-rpm-release-7-1.el7.centos.noarch.rpm; yum clean all

# docs are not installed by default https://github.com/docker/docker/issues/10650 https://registry.hub.docker.com/_/centos/ # official docs are wrong, go for http://superuser.com/questions/784451/centos-on-docker-how-to-install-doc-files 
# we'll need that for mysql schema import for icingaweb2 
RUN [ -f /etc/rpm/macros.imgcreate ] && sed -i '/excludedocs/d' /etc/rpm/macros.imgcreate || exit 0
RUN [ -f /etc/yum.conf ] && sed -i '/nodocs/d' /etc/yum.conf || exit 0
RUN yum -y install vim hostname bind-utils cronie logrotate supervisor openssh openssh-server openssh-client rsyslog sudo passwd sed which vim-enhanced pwgen psmisc mailx \
 httpd nagios-plugins-all mariadb-server mariadb-libs mariadb mrtg; \
 yum -y install icinga2 icinga2-doc icinga2-ido-mysql icingaweb2 icingacli php-ZendFramework php-ZendFramework-Db-Adapter-Pdo-Mysql; \
 yum clean all;
# create api certificates and users (will be overridden later) 
RUN icinga2 api setup

# set icinga2 NodeName and create proper certificates required for the API 
RUN sed -i -e 's/^.* NodeName = .*/const NodeName = "docker-icinga2"/gi' /etc/icinga2/constants.conf; \
 icinga2 pki new-cert --cn docker-icinga2 --key /etc/icinga2/pki/docker-icinga2.key --csr /etc/icinga2/pki/docker-icinga2.csr; \
 icinga2 pki sign-csr --csr /etc/icinga2/pki/docker-icinga2.csr --cert /etc/icinga2/pki/docker-icinga2.crt;

# includes supervisor config 
ADD content/ /
# RUN chmod u+x /opt/icinga2/initdocker;

# no PAM # http://stackoverflow.com/questions/18173889/cannot-access-centos-sshd-on-docker 
RUN sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config && sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config; \
 echo "sshd: ALL" >> /etc/hosts.allow; \
 rm -f /etc/ssh/ssh_host_ecdsa_key /etc/ssh/ssh_host_rsa_key && \
 ssh-keygen -q -N "" -t dsa -f /etc/ssh/ssh_host_ecdsa_key && \
 ssh-keygen -q -N "" -t rsa -f /etc/ssh/ssh_host_rsa_key && \
 echo 'root:icingar0xx' | chpasswd; \
 useradd -g wheel appuser; \
 echo 'appuser:appuser' | chpasswd; \
 sed -i -e 's/^\(%wheel\s\+.\+\)/#\1/gi' /etc/sudoers; \
 echo -e '\n%wheel ALL=(ALL) ALL' >> /etc/sudoers; \
 echo -e '\nDefaults:root   !requiretty' >> /etc/sudoers; \
 echo -e '\nDefaults:%wheel !requiretty' >> /etc/sudoers; \
 echo 'syntax on' >> /root/.vimrc; \
 echo 'alias vi="vim"' >> /root/.bash_profile; \
 echo 'syntax on' >> /home/appuser/.vimrc; \
 echo 'alias vi="vim"' >> /home/appuser/.bash_profile;

# fixes at build time (we can't do that at user's runtime) # setuid problem https://github.com/docker/docker/issues/6828 
# 4755 ping is required for icinga user calling check_ping # can be circumvented for icinga2.cmd w/ mkfifo and chown 
# (icinga2 does not re-create the file) 
RUN mkdir -p /var/log/supervisor; \
 chmod 4755 /bin/ping /bin/ping6; \
 chown -R icinga:root /etc/icinga2; \
 mkdir -p /etc/icinga2/pki; \
 chown -R icinga:icinga /etc/icinga2/pki; \
 mkdir -p /var/run/icinga2; \
 mkdir -p /var/log/icinga2; \
 chown icinga:icingacmd /var/run/icinga2; \
 chown icinga:icingacmd /var/log/icinga2; \
 mkdir -p /var/run/icinga2/cmd; \
 mkfifo /var/run/icinga2/cmd/icinga2.cmd; \
 chown -R icinga:icingacmd /var/run/icinga2/cmd; \
 chmod 2750 /var/run/icinga2/cmd; \
 chown -R icinga:icinga /var/lib/icinga2; \
 usermod -a -G icingacmd apache >> /dev/null; \
 chown root:icingaweb2 /etc/icingaweb2; \
 chmod 2770 /etc/icingaweb2; \
 mkdir -p /etc/icingaweb2/enabledModules; \
 chown -R apache:icingaweb2 /etc/icingaweb2/*; \
 find /etc/icingaweb2 -type f -name "*.ini" -exec chmod 660 {} \; ; \
 find /etc/icingaweb2 -type d -exec chmod 2770 {} \;

# configure PHP timezone TODO: Need to check this.
RUN sed -i 's/;date.timezone =/date.timezone = Asia\/Ho_Chi_Minh/g' /etc/php.ini



#############################
#  SSH keygen               #
# for remote monitoring     #
#############################
RUN yes | cp -rfp /.ssh /var/lib/icinga2/; \
	mkdir /var/spool/icinga2/.ssh; \
	ssh-keyscan -H 192.168.1.9 > /var/spool/icinga2/.ssh/known_hosts; \
	ssh-keyscan -H 192.168.1.3 >> /var/spool/icinga2/.ssh/known_hosts; \
	chmod 744 /var/lib/icinga2/.ssh/id_rsa;

#############################
#  Install graphite module  #
#############################
ARG ICINGA2_FEATURE_GRAPHITE
ARG ICINGA2_FEATURE_GRAPHITE_HOST
ARG ICINGA2_FEATURE_GRAPHITE_PORT
ARG ICINGA2_FEATURE_GRAPHITE_WEB_PORT
ADD icingaweb2-graphite-modules/ /usr/share/icingaweb2/modules/graphite/

#############################
#  Config MRTG             #
#############################
RUN mkdir /var/www/html/mrtg;

ADD mrtg-crontab /etc/cron.d/mrtg-crontab
RUN chmod 0644 /etc/cron.d/mrtg-crontab; \
 crontab /etc/cron.d/mrtg-crontab;



#############################
# _initdocker_ init script  #
#            goes here      #
#############################
RUN  chmod u+x /opt/icinga2/init; \
 /opt/icinga2/init;


# ports (icinga2 api & cluster (5665), mysql (3306)) 
EXPOSE 22 80 443 5665 3306 
# volumes 
VOLUME ["/etc/icinga2", "/etc/icingaweb2", "/var/lib/icinga2", "/usr/share/icingaweb2", "/var/lib/mysql", "/usr/share/mrtg", "/var/www/html/mrtg"]
# change this to entrypoint preventing bash login 
# CMD ["/opt/icinga2/initdocker"]
# ENTRYPOINT ["/opt/icinga2/initdocker"]
RUN chmod u+x /opt/icinga2/run; 
ENTRYPOINT ["/opt/icinga2/run"]
