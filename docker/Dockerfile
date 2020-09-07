FROM alpine
RUN apk update && apk upgrade && apk --update --no-cache add vsftpd

COPY start_vsftpd.sh /bin/start_vsftpd.sh
COPY vsftpd.conf /etc/vsftpd/vsftpd.conf

EXPOSE 21 21000-21010
VOLUME /ftp
#VOLUME /var/log/vsftpd

ENTRYPOINT ["/bin/start_vsftpd.sh"]
