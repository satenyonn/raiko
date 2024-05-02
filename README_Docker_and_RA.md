## Recommended Specs

We recommended 4 cores and 8GB memory for running Raiko. 8 cores and 16GB memory is ideal; the bare minimum is 2 cores and 4GB memory (tentative).


### Intel SGX-enabled CPU

Ensure that your machine has an [Intel SGX][sgx]-enabled CPU to run Raiko. You can verify if your CPU supports SGX (Software Guard Extensions) on Linux by using the [`cpuid`][cpuid] tool.

1. If `cpuid` isn't already installed, you can install it. On Ubuntu, use the following command:

        sudo apt-get install cpuid

2. Run `cpuid` and `grep` for `sgx`:

        cpuid | grep -i sgx

    If your CPU supports SGX, the output should resemble the following:

    ```
    SGX: Software Guard Extensions supported = true
    ```


[sgx]: https://www.intel.com/content/www/us/en/architecture-and-technology/software-guard-extensions.html
[cpuid]: https://manpages.ubuntu.com/manpages/noble/en/man1/cpuid.1.html

### Modern Linux kernel

Starting with Linux kernel version [`5.11`][kernel-5.11], the kernel provides built-in support for SGX. However, it doesn't support one of its latest features, [EDMM][edmm] (Enclave Dynamic Memory Management), which Raiko requires. EDMM support was first introduced in Linux `6.0`, so ensure that your Linux kernel version is `6.0` or above.

To check the version of your kernel, run:

```
uname -a
```

[kernel-5.11]: https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/linux-overview.html
[edmm]: https://gramine.readthedocs.io/en/stable/manifest-syntax.html#edmm

### Subscribing to Intel PCS Service

To use ECDSA Attestation, you need to subscribe to the Intel PCS service, following the steps in [Intel's how-to guide][intel-dcap-install-howto]. After subscribing to the service, you will get two keys: a primary API key and a secondary API key.

[intel-dcap-install-howto]: https://www.intel.com/content/www/us/en/developer/articles/guide/intel-software-guard-extensions-data-center-attestation-primitives-quick-install-guide.html

> **_NOTE:_** You do NOT need to follow the entirety of the linked guide, just the `Subscribe to the Intel PCS` section.


### Gramine

 Install your respective distribution [here](https://gramine.readthedocs.io/en/latest/installation.html).
```
sudo curl -fsSLo /usr/share/keyrings/gramine-keyring.gpg https://packages.gramineproject.io/gramine-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/gramine-keyring.gpg] https://packages.gramineproject.io/ $(lsb_release -sc) main" \
| sudo tee /etc/apt/sources.list.d/gramine.list

sudo curl -fsSLo /usr/share/keyrings/intel-sgx-deb.asc https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-sgx-deb.asc] https://download.01.org/intel-sgx/sgx_repo/ubuntu $(lsb_release -sc) main" \
| sudo tee /etc/apt/sources.list.d/intel-sgx.list

sudo apt-get update
sudo apt-get install gramine
```


[gramine-distros]: https://github.com/gramineproject/gramine/discussions/1555#discussioncomment-7016800
[gramine]: https://gramineproject.io/


## Raiko Docker

Once you have satisfied all the prerequisites, you can follow this section.

1. Prepare your system with some necessary installations

```
sudo apt-get update && sudo apt-get install -y build-essential wget python-is-python3 debhelper zip libcurl4-openssl-dev pkgconf libboost-dev libboost-system-dev libboost-thread-dev protobuf-c-compiler libprotobuf-c-dev protobuf-compiler
```

2. Generating PCCS Certificates

```
mkdir ~/.config
mkdir ~/.config/sgx-pccs
cd ~/.config/sgx-pccs
openssl genrsa -out private.pem 2048
chmod 644 private.pem  # Docker container needs access
openssl req -new -key private.pem -out csr.pem
openssl x509 -req -days 365 -in csr.pem -signkey private.pem -out file.crt
rm csr.pem
```

[pccs-readme]: https://github.com/intel/SGXDataCenterAttestationPrimitives/blob/master/QuoteGeneration/pccs/README.md
[pccs-cert-gen]: https://github.com/intel/SGXDataCenterAttestationPrimitives/tree/master/QuoteGeneration/pccs/container#2-generate-certificates-to-use-with-pccs

3. Install Intel lib & copy the config file

> **_NOTE:_** The library requires nodejs 18, but regardless if installation succeeds or not, we just need the `default.json` file it comes with.

```
apt install sgx-dcap-pccs
cd ~/.config/sgx-pccs
cp /opt/intel/sgx-dcap-pccs/config/default.json  .
```

```
hosts
ApiKey
echo -n "user_password" | sha512sum | tr -d '[:space:]-'

echo -n "admin_password" | sha512sum | tr -d '[:space:]-'

```

Ensure docker can use it by modifying permissions to the file:
 
```
chmod 644 default.json
```

[pccs-cert-gen-config]: https://github.com/intel/SGXDataCenterAttestationPrimitives/tree/master/QuoteGeneration/pccs/container#3-fill-up-configuration-file

4. Make some directories to prevent errors

```
mkdir ~/.config/raiko
mkdir ~/.config/raiko/config
mkdir ~/.config/raiko/secrets
```

5. Now, clone raiko and check out the `taiko/alpha-7` branch and navigate to the `docker` folder. From here you can build the docker images that we will be using.

```
git clone -b taiko/alpha-7 https://github.com/taikoxyz/raiko.git
cd raiko/docker
docker compose build
```

> **_NOTE:_** This step will take some time, sometimes ~5 minutes.

6. Check that the images have been built

```
docker image ls
```

You should see at least two images, `gcr.io/evmchain/raiko` and `gcr.io/evmchain/pccs`.

7. If both are present, bootstrap Raiko with the following command:

```
docker compose up init
```

If everything is configured correctly, raiko-init should run without errors and generate a `bootstrap.json`. Check that it exists with the following command:

```
cat ~/.config/raiko/config/bootstrap.json
```

It should look something like this:

```
{
  "public_key": "0x0319c008667385c53cc66202eb961c624481f7317bff679d2f3c7571e06e4d9877",
  "new_instance": "0x768691497b3e5de5c5b7a8bd5e0910eba0672992",
  "quote": "03000200000000000a....................................000f00939a7"
}
```


## On-Chain RA

1. Clone [taiko-mono](https://github.com/taikoxyz/taiko-mono/tree/main) and navigate to the scripts

```
git clone https://github.com/taikoxyz/taiko-mono.git
cd taiko-mono/packages/protocol
```

2. Install [`pnpm`](https://pnpm.io/installation#on-posix-systems) and [`foundry`](https://book.getfoundry.sh/getting-started/installation) so that you can install dependencies for taiko-mono.

```
curl -fsSL https://get.pnpm.io/install.sh | sh -
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
```

Once you have installed them, run the following:

```
pnpm install
pnpm compile
```

3. Prepare your prover's private key

```
export PRIVATE_KEY=
```

4. Ensure the values in the `script/config_dcap_sgx_verifier.sh` script match

`SGX_VERIFIER_ADDRESS`=0x532EFBf6D62720D0B2a2Bb9d11066E8588cAE6D9 
`ATTESTATION_ADDRESS`=0xC6cD3878Fc56F2b2BaB0769C580fc230A95e1398 
`PEM_CERTCHAIN_ADDRESS`=0x08d7865e7F534d743Aba5874A9AD04bcB223a92E 

5. If you've followed the Raiko Docker guide, you will have bootstrapped raiko and obtained a quote:

```
"public_key": "0x02ab85f14dcdc93832f4bb9b40ad908a5becb840d36f64d21645550ba4a2b28892",
"new_instance": "0xc369eedf4c69cacceda551390576ead2383e6f9e",
"quote": "0x030002......f00939a7233f79c4ca......9434154452d2d2d2d2d0a00"
```

Take that quote and replace `V3_QUOTE_BYTES` in the `script/config_dcap_sgx_verifier.sh` script.

6. In the `script/config_dcap_sgx_verifier.sh` script, replace `--fork-url https://any-holesky-rpc-url/` with any Holesky RPC URL.

7. Call the script with `./script/config_dcap_sgx_verifier.sh`.

> **_NOTE:_**  If you already have QE/TCB/Enclave already configured you can change `export TASK_ENABLE="1,1,1,1,1"` to `export TASK_ENABLE="0,0,0,0,1"` to only register the SGX instance.

8. If you've been successful, you will get a SGX instance `id` which can be used to run Raiko!

It should look like this:

```
emit InstanceAdded(id: 1, instance: 0xc369eedf4C69CacceDa551390576EAd2383E6f9E, replaced: 0x0000000000000000000000000000000000000000, validSince: 1708704201 [1.708e9])
```

## Running Raiko

Once you've completed the above steps, you can actually run a prover. Your `SGX_INSTANCE_ID` is the one emitted in the `InstanceAdded` event above.

```
cd ~/raiko/docker
export SGX_INSTANCE_ID={YOUR_INSTANCE_ID}
docker compose up raiko -d
```

If everything is working, you should see something like the following when executing `docker compose logs raiko`:

```
Start config:
Object {
    "address": String("0.0.0.0:8080"),
    "cache_path": Null,
    "concurrency_limit": Number(16),
    "config_path": String("/etc/raiko/config.sgx.json"),
    "log_level": String("info"),
    "log_path": Null,
    "max_log": Number(7),
    "network": String("taiko_a7"),
    "sgx": Object {
        "instance_id": Number(19), <--- sgx instance id
    },
}
Args:
Opt {
    address: "0.0.0.0:8080",
    concurrency_limit: 16,
    log_path: None,
    max_log: 7,
    config_path: "/etc/raiko/config.sgx.json",
    cache_path: None,
    log_level: "info",
}
2024-04-18T12:50:09.400319Z  INFO raiko_host::server: Listening on http://0.0.0.0:8080
```

## Verify that your Raiko instance is running properly

Once your Raiko instance is running, you can verify if it was started properly as follows:

```
 curl --location 'http://localhost:8080' \
--header 'Content-Type: application/json' \
--data '{
    "jsonrpc": "2.0",
    "method": "proof",
    "params": [
        {
            "proof_type": "sgx",
            "block_number": 31991,
            "rpc": "https://rpc.hekla.taiko.xyz/",
            "l1_rpc": "{HOLESKY_RPC_URL}",
            "beacon_rpc": "{HOLESKY_BEACON_RPC_URL}",
            "prover": "0x7b399987d24fc5951f3e94a4cb16e87414bf2229",
            "graffiti": "0x0000000000000000000000000000000000000000000000000000000000000000",
            "sgx": {
                "setup": false,
                "bootstrap": false,
                "prove": true
            }
        }
    ],
    "id": 0
}'
```

Replace `HOLESKY_RPC_URL` and `HOLESKY_BEACON_RPC_URL` with your Holesky RPC urls.

The response should look like this:

```
{"jsonrpc":"2.0","id":0,"result":{"proof":"0x000000149f....", "quote": "03000200000000000a"}}
```

If you received this response, then at this point, your prover is up and running: you can provide the raiko_host endpoint to your taiko-client instance for SGX proving!
