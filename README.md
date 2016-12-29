# Ice with oauth2_proxy

This project is shamelessly forked from [docker-ice](https://github.com/jonbrouse/docker-ice) and add [oauth2_proxy](https://github.com/bitly/oauth2_proxy) as single repository which allow specific user browsing AWS usage Dashboard.


## Requirement
- docker-compose
- AWS iam for checking bills
- Google oauth account


## Getting Started

### Netflix Ice Setup 
- Enable Billing Reports to S3 bucket - [Details](https://console.aws.amazon.com/billing/home#/preferences)
 
- Create the configuration file that will be mounted to the container: 
```
cp ice/assets/sample.properties ice/assets/ice.properties
vi ice/assets/ice.properties
```

- Open ice.properties and configure a basic setup by updating the following:
```    
	    # s3 bucket name where the billing files are
	    ice.billing_s3bucketname=
	    
	    # Your company name
	    ice.companyName=
	    
	    # s3 bucket name where Ice can store output files
	    ice.work_s3bucketname=
	    
	    # Your AWS account number. You can also replace "production" with your own identifier 
	    ice.account.production=
```

- Create AWS IAM and Policy  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": [
                "arn:aws:s3:::yourcompany-billing-reports",
                "arn:aws:s3:::yourcompany-billing-reports/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObjectAcl",
                "s3:GetObjectAcl"
            ],
            "Resource": "arn:aws:s3:::yourcompany-ice/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:GetReservedInstancesExchangeQuote"
            ],
            "Resource": "*"
        }
    ]
}
```

- Add the AWS Access Key ID and Secret Key into docker-compose file from template 
```
cp docker-compose-template.yml docker-compose.yml
vi docker-compose.yml
```
More information and screenshots can be found on the [Netflix's git page](https://github.com/Netflix/ice) and [Docker-ice git page](https://github.com/jonbrouse/docker-ice). 


### Oauth2 Proxy Setup
- Setup Nginx proxy_pass to oauth2_proxy container - [Details](https://github.com/bitly/oauth2_proxy#ssl-configuration)  
  May consider to setup Let's Encrypt for HTTPS cert as well
```
server {
    listen 443 default ssl;
    server_name internal.yourcompany.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/cert.key;
    add_header Strict-Transport-Security max-age=2592000;

    location / {
        proxy_pass http://127.0.0.1:4180;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_connect_timeout 1;
        proxy_send_timeout 30;
        proxy_read_timeout 30;
    }
}
```

- Create a new oauth token on Google Cloud - [Details](https://github.com/bitly/oauth2_proxy#google-auth-provider)

- Generate `cookie_secret` and put `client_id` and `client_secret` into `./oauth2_proxy/mount/config`
```
python -c 'import os,base64; print base64.b64encode(os.urandom(16))'
cp ./oauth2_proxy/mount/config.template ./oauth2_proxy/mount/config
vi ./oauth2_proxy/mount/config
```

- Update allowed email addresses to the site
```
cp ./oauth2_proxy/mount/grant_emails.txt.template ./oauth2_proxy/mount/grant_emails.txt
vi ./oauth2_proxy/mount/grant_emails.txt
```

More information and screenshots can be found on the [oauth2 proxy git page](https://github.com/bitly/oauth2_proxy) and [dockerized git page](https://github.com/wingedkiwi/oauth2-proxy-container). 


## Docker Compose

 - When you have completed the previous steps, issue `docker-compose up` This will start the containers in the forground so you can see if there are any errors.
 - Once everything looks good and you can access the UI issue `docker-compose up -d` to run the containers in the background.


## Upstart Job

There is an upstart job in the `init` directory of this repository. This will allow you to start the containers with `start ice` and stop them by running `stop ice`.  This will also start your containers at boot.

1. Copy `init/ice.conf` to your host's `/etc/init/` directory
2. Edit the the job `vi /etc/init/ice.conf` and change the path to the docker-compose file
	
	    pre-start exec /usr/local/bin/docker-compose -f /path/to/your/docker-compose.yml up -d

		post-stop exec /usr/local/bin/docker-compose -f /path/to/your/docker-compose.yml stop

3. Reload the job controller `initctl reload-configuration`

## Credits
[jonbrouse](https://www.github.com/jonbrouse) for dockerize Netflix Ice and detail documentation :)
