# Fortigate Firewall Configuration for Tailscale Access

## Using Wildcards for Tailscale Domains

In many cases, you can simplify domain configurations by using wildcard entries such as `*.tailscale.com` and `*.tailscale.io` instead of specifying individual services like `controlplane.tailscale.com` or `log.tailscale.io`. However, the effectiveness of wildcard entries depends on the Fortigate firmware version and the specific DNS handling.

### Configuration Steps

### 1. Adding Tailscale Addresses

#### Option 1 (Preferred): Newer Firmware (Wildcard Domains) + SSL Inspection Exception
```bash
config firewall address
edit "Tailscale-WCDN"
set type fqdn
set fqdn "*.tailscale.com"
next
edit "Tailscale-IO-WCDN"
set type fqdn
set fqdn "*.tailscale.io"
next
edit "Tailscale-DERP-Relays"
set type fqdn
set fqdn "derp*.tailscale.com"
next
end

# Add the entire Tailscale group to SSL/SSH Inspection Exception
config firewall ssl-ssh-profile
edit "ckel-deep-inspection"  # Or your specific "custom-deep-inspection" profile
config ssl-exempt
edit 1
set address "tailscale"
next
end
next
end
```

#### Option 2: Older Firmware (Individual FQDNs with Dedicated Policies)
```bash
config firewall address
edit "Tailscale-FQDN"
set type fqdn
set fqdn "tailscale.com"
next
edit "Tailscale-Controlplane-FQDN"
set type fqdn
set fqdn "controlplane.tailscale.com"
next
edit "Tailscale-Login-FQDN"
set type fqdn
set fqdn "login.tailscale.com"
next
edit "Tailscale-Logs-FQDN"
set type fqdn
set fqdn "log.tailscale.com"
next
edit "Tailscale-IO-Logs-FQDN"
set type fqdn
set fqdn "log.tailscale.io"
next
edit "Tailscale-DERP-FQDN"
set type fqdn
set fqdn "derp*.tailscale.com"
next
end
```

### 2. Creating Address Group

```bash
config firewall addrgrp
edit "tailscale"
set member "Tailscale-WCDN" "Tailscale-IO-WCDN" "Tailscale-DERP-Relays"
next
end
```

#### Alternate Group for Older Firmware
```bash
config firewall addrgrp
edit "tailscale"
set member "Tailscale-FQDN" "Tailscale-Controlplane-FQDN" "Tailscale-Login-FQDN" "Tailscale-Logs-FQDN" "Tailscale-IO-Logs-FQDN" "Tailscale-DERP-FQDN"
next
end
```

### 3. Creating Firewall Policy

```bash
config firewall policy
edit <policy_id>
set name "Tailscale SSL Exception"
set srcintf "internal"
set dstintf "wan1"
set action accept
set srcaddr "all"
set dstaddr "tailscale"
set schedule "always"
set service "ALL"
set logtraffic all
set nat enable
set randomize-port enable
set ssl-ssh-profile "ckel-deep-inspection"  # Or use "reso-deep-inspection"
next
end
```

### 4. Configuring SSL Inspection Exception (Preferred Method)

- Go to **Security Profiles > SSL/SSH Inspection**.
- Edit your inspection profile These are just some of mine you can ignore these profiles (**ckel-deep-inspection** or **reso-deep-inspection**).
- I don't know if Toggling **Exempt Reputable Websites** to **On** fixes the problem for some people or what's categorized as reputable outside of banks and such.
- Add the **`tailscale`** group to the **SSL Exception List**, which automatically includes all related domains.

### 5. Verification

To verify that Tailscale connections work correctly:

```bash
exec ping controlplane.tailscale.com
exec ping derp1.tailscale.com
exec ping log.tailscale.com
```

### 6. Troubleshooting and Optimization

**Fortinet Specific Considerations:**

- **Direct Connections Issue:** Tailscale nodes might face connection issues, especially when more than 5 users are behind the same Fortigate firewall. This can cause fallback to DERP relays.
- **Solution:** Enable port randomization in the Tailnet policy file by adding:

```json
{
  "randomizeClientPort": true
}
```

- **FortiGate SSL Interception:** If FortiGate application control intercepts HTTPS connections to Tailscale’s control plane, you might encounter certificate trust errors.
- **Error Example:**
  ```
  fetch control key: Get "https://controlplane.tailscale.com/key?v=123": x509: "controlplane.tailscale.com" certificate is not trusted
  ```
- **Workaround:**
  - Access **FortiOS Admin Panel > Security Profiles > Application Control**.
  - adjust add an entry for the Tailscale signature and make sure it's set to allow, only applicable if blocking remote access category but I generally turn it on. 
  - Ensure the **Proxy** category isn't unintentionally blocking necessary traffic.

### 7. Best Practices & Additional Tips

- **Use Groups for Simplified Management**: Creating an address group (e.g., `tailscale`) and adding it to the SSL exemption list is more efficient than adding individual addresses.
- **Application Control**: Under **Security Profiles > Application Control**, add the **Tailscale** signature as **Allowed**.
- **VPN Blocking**: If desired, block other VPN categories while allowing only Tailscale for secure remote access.

### Additional Notes
- **Preferred Method**: Use wildcard domains with SSL inspection exception (Option 1). It's simpler and more efficient.
- **Wildcard Domains**: Use `Tailscale-WCDN` entries if supported by your firmware.
- **DNS Resolution**: Ensure that DNS resolution works for wildcard domains if switching.
- **Testing**: Fortigate versions may differ in wildcard domain handling. Test before applying in production.
- **SSL Inspection**: Prefer adding the `tailscale` group to the SSL exemption list to simplify management.
- **Profile Selection**: Use `ckel-deep-inspection` or `reso-deep-inspection` profiles, as available.

**Reference:** [Tailscale Firewall Documentation](https://tailscale.com/kb/1181/firewalls)
