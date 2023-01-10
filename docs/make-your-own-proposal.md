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

### Create a File Bucket with your protocol

After applying your protocol changes, create a tar archive of the protocol folder.

For example, from Octez repo:

```shell
$ cd src/proto_016_PtMumbai
$ tar czf mumbai_f.tar.gz lib_protocol/
```

Create a bucket. For our example, we follow this [DigitalOcean Spaces guide](https://docs.digitalocean.com/products/spaces/reference/s3cmd/), create a bucket named `tezos-proto-cruncher` and install s3cmd.

Then transfer the archive to the bucket:
```shell
$ s3cmd put mumbai_f.tar.gz s3://tezos-proto-cruncher
```

### Prepare a values file

Create a new yaml file named `mumbai_f.yaml`:

```yaml
s3_access_key_id: DO00Y...
s3_secret_access_key: '/5CFZV...'
host_bucket: '%(bucket)s.ams3.digitaloceanspaces.com'
host_base: 'ams3.digitaloceanspaces.com'
bucket_name: "tezos-proto-cruncher"
proto_name: "mumbai_f"
vanity_string: "Mumbai"
```

Where:

* `s3_*`: the S3-compatible credentials
* `host_*`: the endpoints of your S3-compatible service (for DigitalOcean, see [this tutorial](https://docs.digitalocean.com/products/spaces/reference/s3cmd/))
* `bucket_name`: name of your bucket (so the s3 url is `s3://<name>`)
* `proto_name`: the name of the `.tar.gz` file containing your proto
* `vanity_string`: the string you want your proto to start with (excluding the first two chars `Ps` or `Pt`).

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
