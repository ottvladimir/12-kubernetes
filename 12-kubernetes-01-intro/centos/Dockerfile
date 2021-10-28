FROM centos:latest

RUN yum install -y openssh-server
RUN echo 'root:password' | chpasswd
RUN sed -ri 's/#Port 22/Port 22/g' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config
RUN sed -ri 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config
RUN sed -ri 's/#ListenAddress 0.0.0.0/ListenAddress 0.0.0.0/g' /etc/ssh/sshd_config
RUN echo 'PermitRootLogin yes' >>  /etc/ssh/sshd_config
RUN ssh-keygen -A
EXPOSE 222
ENTRYPOINT ["/usr/sbin/sshd"]
CMD ["-D"]
