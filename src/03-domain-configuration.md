# Domain configuration

Now we will set up DNS with Route53 to associate domains to the `sigstore-rekor`, `sigstore-fulcio` and `sigstore-oauth2` instances.

## Obtaining a static IP address

In the console, search for the `Elastic IP addresses` in `EC2`.
Click on `Allocate Elastic IP address`, select `Amazon's pool of IPv4 addresses`, optionally add a tag and click `Allocate`.
Repeat this process three times to get IP addresses for each one of the four services we will deploy.
The addresses should be ready shortly under the `Elastic IP addresses` menu.

Now let's associate the allocated addresses to each one of our instances.
Click on the first IP address, and select `Associate Elastic IP address`.
Choose `Instance` as a Resource type and select the `sigstore-rekor` instance in the search bar.
The private IP address of the instance to associate with the Elastic IP address should normally be present in the `Private IP address` search bar. You can verify the address corresponds to the instance in the `EC2` > `instances` menu.

Repeat this process for the `sigstore-fulcio` and `sigstore-oauth2` instances.

## Configuring DNS with Route53

Search for the `Route53` service in the console, and click `Create hosted zone` in the Route53` dashboard.
Enter `sigstore-aws-example.com` as a domain name, enter a description and select `Public hosted zone`. Click `Create hosted zone`.
Verify the hosted zone was created successfully, select it and click `Create record`.

For the Rekor instance, choose `rekor` as the subdomain, and `A - Route traffic to an IPv4 address and some AWS resources` as the Record type.
Enter the Elastic IP adress we associated to the Rekor instance in the previous step in the `Value` box, choose a TTL of 60 seconds and the `Simple routing` policy.


Repeat this process for the `sigstore-fulcio` and `sigstore-oauth2` instances with `fulcio` and `oauth2` as domain names.
