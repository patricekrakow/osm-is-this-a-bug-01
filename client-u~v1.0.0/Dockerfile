FROM ubuntu:18.04
RUN apt-get -y update
RUN apt-get install -y curl
RUN apt-get install -y jq
WORKDIR /app
COPY client-u.sh /app
RUN chmod 755 /app/client-u.sh
CMD /app/client-u.sh
