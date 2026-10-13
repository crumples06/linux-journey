# Week-18
*12/10/26-18/10/26*

## CI for CA testing

I started making a CI pipeline for my segmented-bastion-ca project. The main problem is that the CI starts from nothing. A github runner is a blank machine, it has no data from `ca_data` volume, my CA keys, or my vault password. So the pipeline has to build a throwaway copy of the whole lab, test it, and discard it.

First step i added was a simple `docker compose up -d ca --wait` to start up the ca container. Then i added a step to show logs if the previous step failed. 
Now i had to copy the user and host public ca key to the host from the ca container. So i just copied them from `/host/step/certs/ssh_user_ca_key.pub` from ca container to the host at `ca/user_ca.pub` (same for host ca key). 

Next, i want to create a `known_hosts` file in the ca folder (in host). To configure clients to accept host certificates I need to add the host CA public key in `known_hosts` file with the following format:
`@cert-authority *example.com ecdsa-sha2-nistp256 AAAAE...=`
Where `*.example.com` is a pattern that matches the hosts and `ecdsa-sha2-nistp256 AAAAE...=` should be the contents of the host CA public key.
So i ran this command: `echo "@cert-authority [localhost]:2222,bastion,server1,server2 $(cat ca/host_ca.pub)" > ca/known_hosts` in the step.

Next i wanted to capture the CA root fingerprint, it was to be set up as an environment variable, 
```
FP=$(docker compose exec ca -T step certificate fingerprint /home/step/certs/root_ca.crt)
echo "CA_FINGERPRINT=$FP" >> "$GITHUB_ENV"
```


