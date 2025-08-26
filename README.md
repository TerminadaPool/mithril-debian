# Build mithril as deb packages
This document outlines one way to build the mithril software into Debian packages. Building the software as deb packages makes it easier to install on any Debian or Ubuntu system and have the package management system take care of everything.

These are not official deb packages and this document does not do things the official Debian/Ubuntu way. Furthermore, these deb packages do not deliver the proper copyright documents or any manuals.

These deb packages will install the software in the following standard locations:

* Libraries are placed in /usr/lib/
* Executables are placed in /usr/bin/
* Configuration files are placed in /etc/cardano/
* Cardano blockchain database is stored in /var/lib/cardano
* Systemd service file is placed in /lib/systemd/system/

## Setup your build environment
I usually create a virtual machine for building software and do everything on it. This allows me to keep everything separated from my desktop PC.

I also create a separate user ('builder') for building. This is advisable, even if you choose to build on your desktop PC, so that the installation of any local compilers, or any specific configurations, won't interfere with anything in your normal user account.
```
sudo adduser --gecos '' --disabled-password builder
```

Install build dependencies and some extra requirements
```
sudo apt install build-essential m4 fakeroot devscripts debhelper autoconf-archive d-shlibs pkg-kde-tools git curl automake pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make jq libncursesw5 libtool autoconf libncurses-dev libncurses5 libnuma-dev binutils-gold
```

Recommend install llvm and set cc and c++ to use clang
```
sudo apt install llvm clang; \
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100; \
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang++ 100
```
Ubuntu users might need to skip the previous optional step if their Ubuntu version of clang doesn't support optimization flag '-ffat-lto-objects'.  Revert the previous changes with:
```
sudo apt purge llvm clang; \
sudo apt autoremove;
```
Or, if you want to keep clang and llvm on your system but not have clang as preferred cc/c++ compiler:
```
sudo update-alternatives --remove cc /usr/bin/clang; \
sudo update-alternatives --remove c++ /usr/bin/clang++;
```

## Install Node.js
As root:
```
curl -fsSL https://deb.nodesource.com/setup_23.x -o nodesource_setup.sh; \
bash nodesource_setup.sh; \
apt-get install -y nodejs; \
node -v;
```

## Switch to your builder account
```
sudo su - builder
```
The rest of this document assumes you are using your 'builder' account:
builder@build:~$

## Install rust
As builder:
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
Press enter to proceed with default installation.

Source your .bashrc to configure your $PATH and update
```
source ${HOME}/.bashrc; \
rustup install stable; \
rustup default stable; \
rustup update; \
rustup component add clippy rustfmt
```

## Install wasm-pack
```
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
```

### Determine latest mithril release
Set variable $MITHRIL_RELEASE to latest version with:
```
MITHRIL_RELEASE="$(curl -s https://api.github.com/repos/input-output-hk/mithril/releases/latest | jq -r .tag_name)"; \
echo "Latest MITHRIL_RELEASE is: ${MITHRIL_RELEASE}";
```

Familiarise yourself with the following commands before running them. You can simply copy and paste the entire list of commands below into a bash terminal to run them in sequence.

## Build mithril deb packages
```
deb_build_instructions_repo='https://github.com/TerminadaPool/mithril-debian.git'; \

mithril_repo='https://github.com/input-output-hk/mithril.git'; \

echo "Building MITHRIL_RELEASE=${MITHRIL_RELEASE}"; \

package='mithril'; \
basedir="${HOME}/src/${package}"; \

mkdir -p "${basedir}"; \
rm -rf "${basedir}/${package}-${MITHRIL_RELEASE}"; \
cd "${basedir}"; \

git clone "${mithril_repo}" "${package}-${MITHRIL_RELEASE}"; \
cd "${package}-${MITHRIL_RELEASE}"; \
git fetch --all --recurse-submodules --tags; \
git checkout "${MITHRIL_RELEASE}"; \

git clone "${deb_build_instructions_repo}" debian; \

debuild --prepend-path "$HOME/.cargo/bin" --set-envvar RUSTUP_HOME="${HOME}/.rustup" --set-envvar CARGO_HOME="${HOME}/.cargo" -us -uc -b; \

unset deb_build_instructions_repo mithril_repo MITHRIL_RELEASE package basedir;
```

Nb. debuild switches:  
* -b: build binary packages only
* -uc: No signing of chainlog
* -us: No signing of source code


Expect to see lintian errors like the following at the end of the build process:  
> E: mithril-aggregator: embedded-library gmp [usr/bin/mithril-aggregator]  
> E: mithril-client: embedded-library gmp [usr/bin/mithril-client]  
> E: mithril-signer: embedded-library gmp [usr/bin/mithril-signer]  
> E: mithril-aggregator: no-copyright-file  
> E: mithril-client: no-copyright-file  
> E: mithril-signer: no-copyright-file  
> W: mithril-aggregator-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/30/174131b0ed65294810e467360067bc6fc9ece3.debug]  
> W: mithril-client-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/25/d7525ae8fe1ca7f4a86a5a5fd9a856aeb457c9.debug]  
> W: mithril-signer-dbgsym: debug-file-with-no-debug-symbols [usr/lib/debug/.build-id/81/93741a9afcce084dfb58f3f9a2357429ab4a93.debug]  
> W: mithril-aggregator: no-manual-page [usr/bin/mithril-aggregator]  
> W: mithril-client: no-manual-page [usr/bin/mithril-client]  
> W: mithril-signer: no-manual-page [usr/bin/mithril-signer]  

****

## Use mithril-client for fast bootstrapping
1. Change to user cardano
    ```
    su - cardano;
    ```
2. Create subvol /var/lib/cardano/mainnet/db (Separate btrfs subvol so it is not included in rootfs snapshots.)
    ```
    mkdir -p mainnet; \
    btrfs subvolume create mainnet/db; \
    cd mainnet;
    ```
3. List available cardano-db snapshots for mainnet
    ```
    export GENESIS_VERIFICATION_KEY=$(curl -s https://raw.githubusercontent.com/input-output-hk/mithril/main/mithril-infra/configuration/release-mainnet/genesis.vkey); \
    export AGGREGATOR_ENDPOINT=https://aggregator.release-mainnet.api.mithril.network/aggregator; \
    mithril-client --run-mode mainnet cardano-db snapshot list;
    ```
    >| 487   | 5785      | mainnet | 7c3f669f3beb38e21bbb9bed76950e0790db9be1959c27f8f8a9f130741f555e | 48082064212 | 1         | 2024-05-25 01:00:29.133503778 UTC |
4. Pick the latest snapshot from the list and ask mithril-client to download and verify it.  Eg:
    ```
    mithril-client --run-mode mainnet cardano-db download e7139f6fd587338b0b5b007cc12bcc3c10b8a17024ba83cac84ede2a8463d02b;
    ```
    Nb. Cut and paste the correct snapshot.

This will place the Cardano blockchain data in the db subdirectory.

#### Start cardano-node
(Back as root user)

Check that the db directory is correct in the systemd service file
```
cat /lib/systemd/system/cardano-node.service | grep 'database-path'
```
>--database-path /var/lib/cardano/mainnet/db

Start cardano-node
```
systemctl enable cardano-node; \
systemctl start cardano-node;
```

Check current tip
```
cardano-cli query tip --mainnet --socket-path /run/cardano/mainnet/node.socket
```

