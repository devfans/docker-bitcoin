FROM ubuntu:18.04
MAINTAINER devfans <stefanliu@outlook.com>

RUN apt-get update
RUN apt-get -y install apt-utils wget curl
RUN mkdir /bitcoin
RUN cd /bitcoin && wget -q https://bitcoin.org/bin/bitcoin-core-0.18.0/bitcoin-0.18.0-x86_64-linux-gnu.tar.gz
RUN cd /bitcoin && tar xf bitcoin-0.18.0-x86_64-linux-gnu.tar.gz
RUN ln -s /bitcoin/bitcoin-0.18.0/bin/bitcoind /bin/bitcoind
RUN ln -s /bitcoin/bitcoin-0.18.0/bin/bitcoin-cli /bin/bitcoin-cli
RUN ln -s /bitcoin/bitcoin-0.18.0/bin/bitcoin-tx /bin/bitcoind-tx
RUN ln -s /bitcoin/bitcoin-0.18.0/bin/bitcoin-wallet /bin/bitcoin-wallet

ENTRYPOINT ["bitcoind", "-server"]
CMD ["-rpcuser=bit", "-rpcpassword=lkef5b389aa2xnf48b4500c81656", "-zmqpubhashtx=tcp://127.0.0.1:8331", "--rpcport=8332", "-listen=0"] 

