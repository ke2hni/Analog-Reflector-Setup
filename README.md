# Analog_Reflector Setup Notes


## What Analog_Reflector Does

Analog_Reflector provides the DVSwitch HTML client (hUC) and connects clients to configured AllStar and Analog_Bridge endpoints. Installing the package does not finish the node-specific setup: the JSON configuration and TLS certificates must also be prepared.

## Configuration

Edit:

```text
/opt/Analog_Reflector/Analog_Reflector.json
```

The relevant settings on this node are:

| Setting | Value / purpose |
| --- | --- |
| `dmrID` | `XXXX` — default DMR ID |
| `bridges.asl[].node` | `XXXX` — AllStar node number |
| `amiUserName` | `XXXX` — AMI username; the section name in `/etc/asterisk/manager.conf` |
| `amiSecret` | `XXXX` — matching AMI secret from `manager.conf` |
| `wsPort` | `8090` — hUC HTTPS/WebSocket port; Apache already uses 443 |
| `keyDir` | `./ssl/` — location of the TLS certificate files |

Use the AMI credentials configured in Asterisk's `manager.conf`. Do not assume they are the Allmon3 web login credentials. Keep the remaining bridge addresses and ports consistent with the actual Analog_Bridge and AllStar configuration. The reflector's USRP ports and Analog_Bridge's `[USRP]` ports must be paired correctly; changing the web port does not change those audio/bridge ports.

The JSON must remain valid: do not add comments, and preserve its commas, quotes, and brackets.

## TLS Certificates

The certificates are not generated automatically just by installing Analog_Reflector. Its included script creates a self-signed certificate for the current host. On this node, `mkcerts.sh` was not executable initially, so make it executable and run it from the SSL directory:

```bash
sudo chmod 755 /opt/Analog_Reflector/ssl/mkcerts.sh && cd /opt/Analog_Reflector/ssl && sudo ./mkcerts.sh
```

The script may be unable to look up the country, state, and city and use placeholders; that did not prevent certificate creation here. Verify the three files expected by Analog_Reflector:

```bash
sudo ls -l /opt/Analog_Reflector/ssl/dvswitch.crt /opt/Analog_Reflector/ssl/dvswitch.key /opt/Analog_Reflector/ssl/dvswitch-ca.crt
```

Keep `dvswitch.key` private. Because these are self-signed certificates, a browser may show a security warning unless the certificate authority is trusted on that device.

## Port Conflict and Client URL

The default `wsPort` is 443, but Apache was already listening on TCP port 443 on this node. Analog_Reflector therefore exited with a port-in-use error. Changing `wsPort` to 8090 resolved the conflict and allowed the HTML client to load.

Open the client at:

```text
https://XXXX:8090
```

Include `:8090` in the URL. For access from outside the LAN, forward TCP port 8090 through the router and allow it in any active host firewall. UFW was not installed on this node, which does not rule out other firewall rules.

## Start and Check the Service

After editing the JSON or creating the certificates, start the service and inspect its status:

```bash
sudo systemctl reset-failed analog_reflector.service && sudo systemctl start analog_reflector.service && sudo systemctl status analog_reflector.service --no-pager -l
```

For later checks:

```bash
sudo systemctl status analog_reflector.service --no-pager -l
sudo journalctl -u analog_reflector.service -b --no-pager -n 80
sudo tail -n 80 /var/log/dvswitch/Analog_Reflector.log
```

The service was enabled for startup on this node. A successful foreground test also confirmed that the certificate loaded, the bridges initialized, and the AMI login succeeded before the port 443 conflict was fixed.

## Error Clues Confirmed Here

| Exit status / message | Cause and fix |
| --- | --- |
| `3` / SSL error | TLS certificate/key files were missing. Generate them with `mkcerts.sh`. |
| `4` / no permission, followed by “port 443 is already in use” in foreground output | Apache owned port 443. Set `wsPort` to an unused port; this node uses 8090. |

The browser may prompt for a callsign and prefill `N0CALL`, the built-in receive-only guest account. That prompt is separate from service startup.

## Official Documentation

- [DVSwitch Analog_Reflector and HTML Client guide](https://dvswitch.org/Analog%20Reflector/Analog_Reflector.html)
