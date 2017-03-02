FROM drupal:7

RUN apt-get update && apt-get install -y mysql-client rsyslog bindfs
RUN echo "local0.* /var/log/drupal.log" > /etc/rsyslog.d/drupal_syslog.conf
