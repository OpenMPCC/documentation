# Networking in MP-SPDZ
All MPC protocols implemented in MP-SPDZ rely on point-to-point connections between all parties.
In a normal development setup this is realized via the localhost interface for all participating parties.
Under real evaluation and deployment scenarios this assumption no longer applies, as all participating parties are distributed with different IP addresses.

MP-SPDZ is therefore equipped with a mechanism that allows to also work in distributed environments.

There are two operating models, a coordination server and file based information knowledge where the IP addresses and port numbers of each participating party are defined. This file must be present for all participating parties.

## Development setup
In order to test this functionality we are first going to build a container containing all protocols or the protocol we want to evaluate.

```dockerfile
FROM debian:trixie-slim AS builddebian

RUN apt-get update && apt-get install -y --no-install-recommends \
                automake \
                build-essential \
                ca-certificates \
                clang \
		cmake \
                gdb \
                git \
                libboost-dev \
                libboost-filesystem-dev \
                libboost-iostreams-dev \
                libboost-thread-dev \
                libclang-dev \
                libgmp-dev \
                libntl-dev \
                libsodium-dev \
                libssl-dev \
                libtool \
                openssl \
                python3 \
                valgrind \
                vim \
        && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src

RUN git clone https://github.com/data61/MP-SPDZ.git

ENV MP_SPDZ_HOME /usr/src/MP-SPDZ
WORKDIR $MP_SPDZ_HOME

RUN make all -j $(nproc)
```

This container file can be build using the following command:

```bash
podman build --file containerfile --tag mpspdz:all --target builddebian .
```

Additionally, we are going to define a compose file which is going to be used to spin up the necessary containers.

```yaml
services:
  party0:
    image: "mpspdz:full"
    environment:
      - PARTY_ID=0
      - NUM_PARTIES=${NUM_PARTIES}
      - PROGRAM=${PROGRAM}
      - PROTOCOL=${PROTOCOL}
      - ENCRYPTION=${ENCRYPTION}
    command: /bin/bash -c "cp /opt/entrypoint.sh /usr/src/MP-SPDZ/entrypoint.sh && chmod +x ./entrypoint.sh && ./entrypoint.sh"
    volumes:
      - shared_data:/usr/src/MP-SPDZ/Player-Data
      - ./entrypoint.sh:/opt/entrypoint.sh:z
    depends_on:
      - setup
    
  party1:
    image: "mpspdz:full"
    environment:
      - PARTY_ID=1
      - NUM_PARTIES=${NUM_PARTIES}
      - PROGRAM=${PROGRAM}
      - PROTOCOL=${PROTOCOL}
      - ENCRYPTION=${ENCRYPTION}
    command: /bin/bash -c "cp /opt/entrypoint.sh /usr/src/MP-SPDZ/entrypoint.sh && chmod +x ./entrypoint.sh && ./entrypoint.sh"
    volumes:
      - shared_data:/usr/src/MP-SPDZ/Player-Data
      - ./entrypoint.sh:/opt/entrypoint.sh:z
    depends_on:
      - setup

  setup:
    image: "mpspdz:full"
    container_name: setup
    command: /bin/bash -c "cp /opt/setup.sh /usr/src/MP-SPDZ/setup.sh && chmod +x ./setup.sh && ./setup.sh"
    volumes:
      - shared_data:/usr/src/MP-SPDZ/Player-Data
      - ./setup.sh:/opt/setup.sh:z

volumes:
  shared_data:
```

This file creates a setup container which is going to be responsible for creating all necessary content to enable the running of the containers performing the MPC protocol.
The shared content will be mounted in a docker volume which is going to be accessible to the container representing a MPC party.
The setup script which is loaded could for example look like the following

```bash
#!/bin/bash
# setup.sh

rm -rf /usr/src/MP-SPDZ/Player-Data/*  &&
mkdir -p /usr/src/MP-SPDZ/Player-Data && 
cd /usr/src/MP-SPDZ/ &&
echo 'party0:5000' > Player-Data/Hosts.txt &&
echo 'party1:5000' >> Player-Data/Hosts.txt &&
echo '1 2 3 4' > Player-Data/Input-P0-0 &&
echo '1 2 3 4' > Player-Data/Input-P1-0 &&
./Scripts/setup-ssl.sh 2 &&
echo 'Setup complete'
```

This script is going to create the Hosts.txt file which is required for the file based networking approach of MP-SPDZ.
As we are running to separate containers with distinct IP addresses both instances are going to be using the port 5000, with their container name as the hostname resolved through the docker network.
We are also creating some dummy data for each which can be used during the communication.
Under normal circumstances this data is **not** shared with other parties, however as we want to simply the process for running protocols over the network this is sufficient.
The same applies to the generation of the necessary SSL/TLS certificates.
The script by MP-SPDZ will create two private keys and certificates for each party with the Common Name of the certificate set to the party name (P0, P1, etc.).
It is required that both parties have both certificates present, however under normal circumstances the private key that was generated should **not** be shared with the other party.

### Running the protocol
In order to run the MPC protocol on each instance a common entrypoint script has to be defined.

```bash
#!/bin/bash
# entrypoint.sh

PROTOCOL=${PROTOCOL:-mascot-party.x}
PARTY_ID=${PARTY_ID:-0}
NUM_PARTIES=${NUM_PARTIES:-2}
PROGRAM=${PROGRAM:-tutorial}
HOST_FILE=${HOST_FILE:-"Player-Data/Hosts.txt"}
ENCRYPTION=${ENCRYPTION:-""}

set -x

echo "Compiling $PROGRAM"
./compile.py $PROGRAM

echo "Starting $PROTOCOL $PARTY_ID for program $PROGRAM"
echo "Hosts file content:"
cat $HOST_FILE

# Run the MASCOT protocol
exec ./$PROTOCOL -p $PARTY_ID -N $NUM_PARTIES $PROGRAM --ip-file-name $HOST_FILE $ENCRYPTION
```

This script first reads some environment variables which have been passed from the compose file and originally have been defined in an .env file.

```
NUM_PARTIES=2
PROGRAM=tutorial
PROTOCOL=mascot-party.x
ENCRYPTION=--encrypted
```

This allows for an easy exchange of the underlying protocol and its parameters, as well as spawning easily new instances.
In our example we are going to run the mascot protocol for 2 parties with the `tutorial.mpc` called and doing the necessary computation on our provided input, which has been defined in our setup container.

If all necessary files are present one can run everything using `podman compose up`.
The output should look something like the following for party 0:
```
Compiling tutorial
+ echo 'Compiling tutorial'
+ ./compile.py tutorial
Default bit length for compilation: 64
Default security parameter for compilation: 40
Compiling file Programs/Source/tutorial.mpc
WARNING: Order of memory instructions not preserved, errors possible
Writing to Programs/Bytecode/tutorial-FPDiv(1)_31_15-1.bc
Writing to Programs/Bytecode/tutorial-Trunc(1)_30_15_True-3.bc
Writing to Programs/Bytecode/tutorial-EQZ(2)_64-5.bc
Writing to Programs/Bytecode/tutorial-LTZ(5)_65-7.bc
�WARNING: Probabilistic truncation leaks some information, see https://eprint.iacr.org/2024/1127 for discussion. Use 'sfix.round_nearest = True' to deactivate this for fixed-point operations.
Writing to Programs/Bytecode/tutorial-TruncPr(1)_47_16-9.bc
Writing to Programs/Bytecode/tutorial-FPDiv(1)_31_16-11.bc
Writing to Programs/Bytecode/tutorial-EQZ(2)_31-13.bc
Writing to Programs/Bytecode/tutorial-LTZ(5)_32-15.bc
Writing to Programs/Bytecode/tutorial-TruncPr(3)_47_16-17.bc
Writing to Programs/Bytecode/tutorial-EQZ(1)_31-18.bc
Writing to Programs/Bytecode/tutorial-TruncPr(6)_47_16-19.bc
Writing to Programs/Schedules/tutorial.sch
Writing to Programs/Bytecode/tutorial-0.bc
Hash: 76c34da788a6b7ecd0cb49118d238378b2d2834f5620d5db1547f4cc3e75057e
Program requires at most:
           4 integer inputs from player 0
           4 integer inputs from player 1
        2526 integer triples
        2514 integer simple multiplications
          90 integer opens
        5461 integer bits
           1 matrix multiplications (3x2 * 2x2)
           6 integer dot products
         298 virtual machine rounds
Starting mascot-party.x 0 for program tutorial
Hosts file content:
+ echo 'Starting mascot-party.x 0 for program tutorial'
+ echo 'Hosts file content:'
+ cat Player-Data/Hosts.txt
party0:5000
party1:5000
+ exec ./mascot-party.x -p 0 -N 2 tutorial --ip-file-name Player-Data/Hosts.txt --encrypted
Using statistical security parameter 40
got 1 from player 0
got 1 from player 1
expected 3, got 3
expected 2, got 2
expected -1, got -1
expected 2, got 2
expected 1, got 1
expected 1, got 1
expected 0, got 0
expected 0, got 0
expected 0, got 0
expected 1, got 1
expected 2, got 2
expected 9702, got 9702
expected 1.9, got 1.9
expected 2.1, got 2.1
expected -0.2, got -0.2
expected -20, got -20
expected 0, got 0
expected 0, got 0
expected 1, got 1
expected 1, got 1
expected 0, got 0
expected 1, got 1
expected -0.1, got -0.1
Party 0: please input three numbers not adding up to zero
Party 1: please input any three numbers
weighted average: 3.222
expected 2, got 2
expected 3, got 3
The following benchmarks are including preprocessing (offline phase).
Time = 1.45327 seconds 
Data sent = 176.567 MB in ~1245 rounds (party 0 only; use '-v' for more details)
Global data sent = 353.134 MB (all parties)
This program might benefit from some protocol options.
Consider adding the following at the beginning of your code:
        program.use_edabit(True)
```

## Using a coordination server
In order to use the more convenient coordination server we only need to adjust our `entrypoint.sh` and `setup.sh` to omit the necessary file and define a coordination server which is always party0.

```bash
#!/bin/bash
# entrypoint.sh

PROTOCOL=${PROTOCOL:-mascot-party.x}
PARTY_ID=${PARTY_ID:-0}
NUM_PARTIES=${NUM_PARTIES:-2}
PROGRAM=${PROGRAM:-tutorial}
HOST_FILE=${HOST_FILE:-"Player-Data/Hosts.txt"}
ENCRYPTION=${ENCRYPTION:-""}

set -x

echo "Compiling $PROGRAM"
./compile.py $PROGRAM

# Run the MASCOT protocol
exec ./$PROTOCOL -p $PARTY_ID -N $NUM_PARTIES $PROGRAM --hostname party0 $ENCRYPTION
```

```bash
#!/bin/bash

rm -rf /usr/src/MP-SPDZ/Player-Data/*  &&
mkdir -p /usr/src/MP-SPDZ/Player-Data && 
cd /usr/src/MP-SPDZ/ &&
echo '1 2 3 4' > Player-Data/Input-P0-0 &&
echo '1 2 3 4' > Player-Data/Input-P1-0 &&
./Scripts/setup-ssl.sh 2 &&
echo 'Setup complete'
```

The output from running this computation looks the same, as with the file based approach, with only the slight difference of using a different command.