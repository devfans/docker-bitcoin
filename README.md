# devfans/bitcoin

bitcoin-core docker image

[![devfans/bitcoin][docker-pulls-image]][docker-hub-url] [![devfans/bitcoin][docker-stars-image]][docker-hub-url]

## Tags

- `0.18.0`, `0.18`, `latest` ([0.18/Dockerfile](https://github.com/devfans/docker-bitcoin/blob/master/0.18/Dockerfile))

**Picking the right tag**

- `devfans/bitcoin:latest`: points to the latest stable release available of Bitcoin Core. Use this only if you know what you're doing as upgrading Bitcoin Core blindly is a risky procedure.
- `devfans/bitcoin:<version>`: based on a slim Debian image, points to a specific version branch or release of Bitcoin Core. Uses the pre-compiled binaries which are fully tested by the Bitcoin Core team.

## What is Bitcoin Core?

Bitcoin Core is a reference client that implements the Bitcoin protocol for remote procedure call (RPC) use. It is also the second Bitcoin client in the network's history. Learn more about Bitcoin Core on the [Bitcoin Developer Reference docs](https://bitcoin.org/en/developer-reference).

## Usage

### How to use this image

The first way is to pass bitcoind parameters in the command line

```
docke run -d --name bitcoin-server devfans/bitcoin -testnet -rpcuser=xx -rpcpassword=xxx
docker exec -it bitcoin-server bitcoin-cli -rpcuser=xx -rpcpassword=xxx getwalletinfo
```

_Note: default entrypoint is `bitcoind -server`, default extra paramters (if no other paramters are specified)_:
```
CMD ["-rpcuser=bit", "-rpcpassword=lkef5b389aa2xnf48b4500c81656", "-zmqpubhashtx=tcp://127.0.0.1:8331", "--rpcport=8332", "-listen=0"]
```

The second way is to save the configuation into `bitcoin.conf`

/data/bitcoin/bitcoin.conf
```
rpcuser=bitcoinrpc
# generate random rpc password:
#   $ strings /dev/urandom | grep -o '[[:alnum:]]' | head -n 30 | tr -d '\n'; echo
rpcpassword=xxxxxxxxxxxxxxxxxxxxxxxxxx
rpcthreads=4
rpcbind=0.0.0.0
rpcallowip=172.16.0.0/12
rpcallowip=192.168.0.0/16
rpcallowip=10.0.0.0/8

# use 1G memory for utxo, depends on your machine's memory
dbcache=1000

# set maximum BIP141 block weight
blockmaxweight=4000000
```

```
docker run -d -v /data/bitcoin:/root/.bitcoin -p 8332:8332 \
   --name bitcoin-server devfans/bitcoin -conf=bitcoin.conf
docker exec -it bitcoin-server bitcoin-cli getbalance

curl -u bitcoinrpc:xxxxxxxxxxxxxxxxxxxxxxxxxx --data-binary '{"jsonrpc":"1.0","id":"1","method":"getblockchaininfo","params":[]}' http://127.0.0.1:8332

```

_Note: [learn more](#using-rpcauth-for-remote-authentication) about how `-rpcauth` works for remote authentication._
You can optionally create a service using `docker-compose`:

```yml
bitcoin-core:
  image: devfans/bitcoin
  command:
    -testnet
```

To deploy on a kubernetes cluster:

```bash
# first login the bastion server which has helm client ready
# then execute below commands
npm i -g git-to-k8s
git-to-k8s https://github.com/devfans/docker-bitcoin
```

### Using RPC to interact with the daemon

#### Using rpcauth for remote authentication

Before setting up remote authentication, you will need to generate the `rpcauth` line that will hold the credentials for the Bitcoind Core daemon. You can either do this yourself by constructing the line with the format `<user>:<salt>$<hash>` or use the official [`rpcauth.py`](https://github.com/bitcoin/bitcoin/blob/master/share/rpcauth/rpcauth.py)  script to generate this line for you, including a random password that is printed to the console.

_Note: This is a Python 3 script. use `[...] | python3 - <username>` when executing on macOS._

Example:

```sh
❯ curl -sSL https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcauth/rpcauth.py | python - <username>

String to be appended to bitcoin.conf:
rpcauth=foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc
Your password:
qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=
```

Note that for each run, even if the username remains the same, the output will be always different as a new salt and password are generated.

Now that you have your credentials, you need to start the Bitcoin Core daemon with the `-rpcauth` option. Alternatively, you could append the line to a `bitcoin.conf` file and mount it on the container.

```sh
❯ docker run --rm --name bitcoin-server -it devfans/bitcoin \
  -testnet \
  -rpcbind=0.0.0.0 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc'
```

Two important notes:

1. Some shells require escaping the rpcauth line (e.g. zsh), as shown above.
2. It is now perfectly fine to pass the rpcauth line as a command line argument. Unlike `-rpcpassword`, the content is hashed so even if the arguments would be exposed, they would not allow the attacker to get the actual password.

You can now connect via `bitcoin-cli` or any other [compatible client](https://github.com/devfans/docker-bitcoin). You will still have to define a username and password when connecting to the Bitcoin Core RPC server.

To avoid any confusion about whether or not a remote call is being made, let's spin up another container to execute `bitcoin-cli` and connect it via the Docker network using the password generated above:

```sh
❯ docker run -it --link bitcoin-server --rm devfans/bitcoin \
  bitcoin-cli \
  -rpcuser=foo\
  -stdinrpcpass \
  getbalance
```

Enter the password `qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=` and hit enter:

```
0.00000000
```

Note: under Bitcoin Core < 0.16, use `-rpcpassword="qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0="` instead of `-stdinrpcpass`.

Done!

### Exposing Ports

Depending on the network (mode) the Bitcoin Core daemon is running as well as the chosen runtime flags, several default ports may be available for mapping.

Ports can be exposed by mapping all of the available ones (using `-P` and based on what `EXPOSE` documents) or individually by adding `-p`. This mode allows assigning a dynamic port on the host (`-p <port>`) or assigning a fixed port `-p <hostPort>:<containerPort>`.

Example for running a node in `regtest` mode mapping JSON-RPC/REST (18443) and P2P (18444) ports:

```sh
docker run --rm -it \
  -p 18443:18443 \
  -p 18444:18444 \
  devfans/bitcoin \
  -rpcbind=0.0.0.0 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc'
```

To test that mapping worked, you can send a JSON-RPC curl request to the host port:

```
curl --data-binary '{"jsonrpc":"1.0","id":"1","method":"getnetworkinfo","params":[]}' http://foo:qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=@127.0.0.1:18443/
```

#### Mainnet

- JSON-RPC/REST: 8332
- P2P: 8333

#### Testnet

- Testnet JSON-RPC: 18332
- P2P: 18333

#### Regtest

- JSON-RPC/REST: 18443 (_since 0.16+_, otherwise _18332_)
- P2P: 18444

## Docker

This image is officially supported on Docker version 17.09, with support for older versions provided on a best-effort basis.

## License

[License information](https://github.com/bitcoin/bitcoin/blob/master/COPYING) for the software contained in this image.

[License information](https://github.com/devfans/docker-bitcoin/blob/master/LICENSE) for the [devfans/docker-bitcoin][docker-hub-url] docker project.

## Others

The readme doc is taking [ruimarinho/docker-bitcoin-core](https://github.com/ruimarinho/docker-bitcoin-core/edit/master/README.md) as reference. Thanks to @ruimarinho for the detailed explaination of the `bitcoin-core` and `docker` usages.

This image is based on ubuntu:18.04.

[docker-hub-url]: https://hub.docker.com/r/devfans/bitcoin
[docker-layers-image]: https://img.shields.io/imagelayers/layers/devfans/bitcoin/latest.svg?style=flat-square
[docker-pulls-image]: https://img.shields.io/docker/pulls/devfans/bitcoin.svg?style=flat-square
[docker-size-image]: https://img.shields.io/imagelayers/image-size/devfans/bitcoin/latest.svg?style=flat-square
[docker-stars-image]: https://img.shields.io/docker/stars/devfans/bitcoin.svg?style=flat-square

