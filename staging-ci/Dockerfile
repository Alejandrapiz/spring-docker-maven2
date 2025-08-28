FROM eclipse-temurin:17-jre-jammy
RUN apt-get update && apt-get install -y openssh-server curl && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash deploy && mkdir -p /home/deploy/.ssh /opt/springboot/app \
    && chown -R deploy:deploy /home/deploy /opt/springboot
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config \
 && sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/g' /etc/ssh/sshd_config
EXPOSE 22 8080
CMD ["/usr/sbin/sshd","-D"]
