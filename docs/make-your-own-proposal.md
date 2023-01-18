# Make Your Own Proposal

Any Tezos baker can submit a proposal.

However, there is a convention in Tezos where proposals have been named after cities, in alphabetical order.

Every proposal is identified by its hash. Since the Babylon proposal in 2019, every proposal hash has been a vanity hash, where the first few letters of the hash match the city name of the protocol.

This guide explains how to create such vanity hashes.

## Vanity hash creation

Creating a vanity hash consists of editing a commented [random string inside the protocol proposal](https://gitlab.com/tezos/tezos/-/blob/master/src/proto_015_PtLimaPt/lib_protocol/main.ml#L417) and hashing it over and over, until the resulting hash starts with the string that you expect.

This requires powerful computers. We are leveraging the cloud to perform this task.

It works like this:

* create a file bucket (Amazon S3 or equivalent) with your protocol in it
* launch a Kubernetes cluster with several powerful machines
* install the `tezos-proto-cruncher` helm chart
* wait
* when the cluster finds results, it saves them in the same bucket.

## Download your protocol binary

Supposing that you are amending an existing proposal that is already snapshotted in the [Octez repository](https://gitlab.com/tezos/tezos):

Apply your changes in the Ocaml protocol code directly in the snapshotted protocol folder, for example `src/proto_016_PtMumbai/lib_protocol`.

Your modifications will change the protocol hash. Calculate the new hash:

```
$ ./octez-protocol-compiler -hash-only src/proto_016_*/lib_protocol
Psm5KsjAm73vQsdMpBfkTm9YmBjx3DHznR3HHhbnpLkpjCkf2WX

```
Compile and run Octez in private mode:

```
eval $(opam env)
make buil-deps
make
./octez-node run --synchronisation-threshold 0 --connections 0 --rpc-addr localhost
```

Download your new protocol in binary format:

```shell
curl -H "Accept: application/octet-stream" http://localhost:8732/protocols/Psm5KsjAm73vQsdMpBfkTm9YmBjx3DHznR3HHhbnpLkpjCkf2WX > mumbai_v
```

## Calculate the hash locally

This method will calculate the hash on your computer.

You will find a 4-character hash such as `PtMumb` in a few minutes. However, be advised that 6-character hashes such as `PtMumbai` will take a very long time to find on a single computer. For faster searches, please use the cloud method below.

Download the searcher script:

```
wget https://github.com/oxheadalpha/tezos-k8s/raw/06fbdc8f625e875b07300f8ba4f398261eb209a2/charts/tezos-proto-cruncher/scripts/proto-cruncher.py
```

Install `base58` pip package in a new virtual environment:

```
virtualenv venv
source venv/bin/activate
pip install base58
```

Set an environment variable to the string you are searching, then start the script with the binary proto file name as argument. It will display matches when found:

```
$ export VANITY_STRING="PtMum"
$ python proto-cruncher.py mumbai_v
Original proto nonce: b'(* Vanity nonce: 5020978286916341 *)\n' and hash: Psm5KsjAm73vQsdMpBfkTm9YmBjx3DHznR3HHhbnpLkpjCkf2WX
Found vanity nonce: b'(* Vanity nonce: 6031367764093451 *)\n' and hash: PtMumFNTQheTrXNAx9DaZhPeWjRMYquVnjxRF7daviopjFk2hFy
Found vanity nonce: b'(* Vanity nonce: 7792594028928256 *)\n' and hash: PtMumjvL2n1BGLK5HhGEodT7swSiLhqeuiRGnQr9DabWCthBjQW
```

## Calculate the hash in a Kubernetes cluster

You will need:

* a file bucket (S3 or compatible) to store your hashes,
* a Kubernetes cluster with as many machines as you like.

We use DigitalOcean as an example.

Create a bucket. For our example, we follow this [DigitalOcean Spaces guide](https://docs.digitalocean.com/products/spaces/reference/s3cmd/), create a bucket named `tezos-proto-cruncher` and install s3cmd.

Then transfer the proto binary file to the bucket:
```shell
$ s3cmd put mumbai_v s3://tezos-proto-cruncher
```

### Prepare a values file

```yaml
s3_access_key_id: DO00Y...
s3_secret_access_key: '/5CFZV...'
bucket_endpoint_url: 'ams3.digitaloceanspaces.com'
bucket_region: 'ams'
bucket_name: "tezos-proto-cruncher"
proto_name: "mumbai_v"
vanity_string: "PtMumbai"
```

Where:

* `s3_*`: the S3-compatible credentials
* `bucket_endpoint_url`: the endpoint of your S3-compatible service (for DigitalOcean, see [this tutorial](https://docs.digitalocean.com/products/spaces/reference/s3cmd/))
* `bucket_region`: the region of the bucket (for example `ams`)
* `bucket_name`: name of your bucket (so the s3 url is `s3://<name>`)
* `proto_name`: the name of the file containing your proto
* `vanity_string`: the string you want your proto to start with

### Create a cluster

Go to your cloud platform of choice and create a k8s cluster.

The proto-cruncher is configured as a kubernetes *daemon set* which means it runs on every virtual machine of your cluster.

So, you can pick the appropriate number of machines based on how fast you need the hashes.

In our example, we use a DigitalOcean k8s cluster. See [quick start](https://docs.digitalocean.com/products/kubernetes/quickstart/).

### Install the chart

Clone the [tezos-k8s repo proto-vanity-cruncher branch](https://github.com/oxheadalpha/tezos-k8s/tree/proto_vanity_cruncher).

NOTE: this will be merged later, at which point no more cloning will be needed.

Install the chart:

```
cd tezos-k8s/
helm install -f /path/to/mumbai_f.yaml cruncher charts/tezos-proto-cruncher --namespace cruncher --create-namespace
```

That's it! All vCPUS of all the nodes of your cluster are now searching for a vanity hash.

Go to bed, the next morning you should see a few results in your bucket. If not, get bigger VMs.
