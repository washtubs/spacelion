### Setup

https://medium.com/@limichelle21/connecting-google-domains-to-amazon-s3-d0d9da467650

### To refresh certs

Generate certs as follows

    mkdir .letsencrypt
    docker run -it -v $PWD/.letsencrypt:/etc/letsencrypt certbot/certbot certonly --manual --preferred-challenges dns --email david.milum@gmail.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d 'spacelion.dev' -d '*.spacelion.dev'

Go through thr prompts until you get to the challenge
Add a TXT record in route 53 as they descibe
If it prompts you with two separate challenges on the same subdomain, just try again, I have no idea why it does this
It could take a while to update so just run this until it shows you what you expect

    dig _acme-challenge.spacelion.dev TXT

In cert manager, import a cert copying the contents of the cert, privkey, and chain
Make sure the region is N. Virginia us-east

    sudo cat .letsencrypt/live/spacelion.dev/cert.pem
    sudo cat .letsencrypt/live/spacelion.dev/privkey.pem
    sudo cat .letsencrypt/live/spacelion.dev/chain.pem

Setting up cloudfront for https

Create a distribution in cloudfront
    Put the S3 bucket in as the origin domain name
    Make sure there are alternate CNAMEs
        spacelion.dev
        www.spacelion.dev
    Set the custom SSL cert you just imported (must be us-east), you can use the ARN if it doesn't show up in the drop down

Configure route 53
    Use the cloudfront domain name as an A record in ALIAS mode (replace existing A records if needed)

### Updating - Invalidating

Should use a version scheme so you only have to invalidate index.html in the CDN

After uploading new s3 objects, go into cloudfront, and invalidate

    /
    /index.html

