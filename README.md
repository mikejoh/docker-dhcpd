# docker-dhcpd
Yet another Docker container running dhcpd!

## More info!
Here's the [gist](https://gist.github.com/mikejoh/04978da4d52447ead7bdd045e878587d) describing what i've done and why. You do **need** to read this!

## Getting started

If you have read the gist above you can proceed with the following:

1. Clone this repo.
2. Build the Docker image.
```
docker build . -t dhcpd
```
3. Create the network needed.
```
docker network create -d macvlan --subnet=<your subnet> --gateway=<your GW> -o parent=<parent if> macvlan0
```
4. Run the container with docker-compose
```
docker-compose up -d
```
5. As soon as you've tested the with a dhcp-client you should be able to see some leases in the lease-file
```
docker cp <Container ID>:/var/lib/dhcp/dhcpd.leases .
```

