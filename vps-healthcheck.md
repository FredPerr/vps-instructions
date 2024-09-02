# Automatic recurring health checks

## Introduction
Health checks can be used to monitor the health of your websites, APIs, and other services.

## Installation
Install the following packages:
    
```bash
sudo apt install mailutils -y
```
Select `Internet Site` and press `Enter`.


## Get started

Write a bash script that checks the health of your service. For example, you can use `curl` to check if your website is up:

```bash
#!/bin/bash

# Check if the website is up
curl -s -o /dev/null -w "%{http_code}" https://example.com
```

Save the script to a file, for example, `healthcheck.sh`.

Make the script executable:

```bash
chmod +x healthcheck.sh
```

Test the script to check the health of your service:

```bash
./healthcheck.sh
```

You should see the HTTP status code of your website.

## Schedule recurring health checks

You can use `cron` to schedule recurring health checks. Edit the crontab file:

For instance, schedule the health check to run every 5 minutes:
```bash
crontab -e '/5 * * * * /path/to/healthcheck.sh'
```



