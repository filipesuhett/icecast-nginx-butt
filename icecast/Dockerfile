FROM debian:bullseye

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get -qq -y update && \
    apt-get -qq -y install icecast2 python-setuptools sudo && \
    apt-get clean

COPY "icecast.xml" "/usr/share/icecast2/icecast.xml"

RUN chown -R icecast2 /usr/share/icecast2

VOLUME ["/var/log/icecast2", "/usr/share/icecast2"]

ADD ./start.sh /start.sh

EXPOSE 8000

CMD ["/start.sh"]
