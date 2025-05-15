# Security Groups for Inlets Cloud

If you want to restrict access to your exposed services, you can set up a Security Group in the Inlets Cloud dashboard.

You can add IP addresses or CIDR ranges into a Security Group which will be used to filter incoming traffic to your tunnel server.

### Security Groups for dynamic IPs

If you want to put a dynamic IP address into the Security Group, then you can use the following script to update an existing group by API.

* Find the `PROJECT_ID` through the dashboard and update it in the script
* Then take the `GROUP_NAME` from the address bar when viewing your given Security Group
* Create a file called `token.txt` and put your API token in it. You can request the API token from the Inlets Discord server.

```bash
#!/bin/bash

# File to store the last known IP
IP_FILE=".last_known_ip"

# Get current IP
CURRENT_IP=$(curl -s -f -L -S https://checkip.amazonaws.com)

# Exit if we couldn't get the IP
if [ -z "$CURRENT_IP" ]; then
    echo "Failed to fetch current IP address"
    exit 1
fi

# Check if we need to update
if [ -f "$IP_FILE" ] && [ "$(cat $IP_FILE)" == "$CURRENT_IP" ]; then
    echo "IP hasn't changed. No update needed."
    exit 0
fi

PROJECT_ID=1
GROUP_NAME="dynamic-ip"
TOKEN=$(cat ./token.txt)

# Update the security group
if curl -i -f -s -S -L https://cloud.inlets.dev/api/security-groups/$GROUP_NAME?project=$PROJECT_ID \
    -X PATCH  \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    --data-binary '{"allow": ["'"$CURRENT_IP"'"]}'; then
    
    # Only store the IP if the update was successful
    echo "$CURRENT_IP" > "$IP_FILE"
    echo "Successfully updated IP to: $CURRENT_IP"
else
    echo "Failed to update security group"
    exit 1
fi
```

You can create a cron expression on a Linux server using `crontab -e` to run this script every 5 minutes:

```bash
*/5 * * * * /home/alex/update-security-group.sh
```

This will check your current IP address and update the Security Group if it has changed.

**Important note on rate limiting**

The script will check your current IP address and update the Security Group if it has changed before making an API call.
