# Fortinet Integration with Wazuh

This guide provides step-by-step instructions for integrating Fortinet firewall logs with Wazuh using rsyslog.

## Prerequisites
- Wazuh manager installed
- Wazuh agent installed
- Fortinet firewall with admin access
- rsyslog installed on the agent server

## 1. Fortinet Firewall Configuration

1. Log into Fortinet firewall web interface
2. Enable syslog forwarding
3. Configure syslog server IP (Wazuh agent IP)
4. Set syslog port to 514

## 2. Syslog Server Setup (Wazuh Agent)

### 2.1 Enable Port 514
```bash
# Open port 514 for TCP and UDP
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload
```

### 2.2 Install rsyslog (if not installed)
```bash
sudo yum install rsyslog   # For RHEL/CentOS
# or
sudo apt install rsyslog   # For Ubuntu/Debian
```

### 2.3 Configure rsyslog for Fortinet
Create `/etc/rsyslog.d/fortinet.conf`:
```bash
# Enable UDP and TCP syslog reception
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")

# Forward Fortinet logs to Wazuh
if $fromhost-ip == '115.247.177.186' then {    # Replace with your Fortinet IP
    action(type="omfwd" target="127.0.0.1" port="1514" protocol="tcp" template="WazuhFormat")
    stop
}
```

### 2.4 Restart rsyslog
```bash
sudo systemctl restart rsyslog
```

## 3. Wazuh Manager Configuration

### 3.1 Custom Rules Configuration
Create/edit `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="local,syslog,sshd,">
  <rule id="100001" level="5">
    <if_sid>5716</if_sid>
    <srcip>1.1.1.1</srcip>
    <description>sshd: authentication failed from IP 1.1.1.1.</description>
    <group>authentication_failed,pci_dss_10.2.4,pci_dss_10.2.5,</group>
  </rule>
</group>

<group name="fortigate,syslog,">
  <!-- Login Failed Rule -->
  <rule id="15001" level="4">
    <if_sid>81603</if_sid>
    <action>login</action>
    <status>failed</status>
    <description>Fortigate: Login failed.</description>
    <group>authentication_failed,gdpr_IV_32.2,gdpr_IV_35.7.d,gpg13_7.1,hipaa_164.312.b,invalid_login,nist_800_53_AC.7,nist_800_53_AU.14,pci_dss_10.2.4,pci_dss_10.2.5,</group>
  </rule>

  <!-- Multiple Failed Logins Rule -->
  <rule id="15002" level="7" frequency="18" timeframe="45" ignore="240">
    <if_matched_sid>15001</if_matched_sid>
    <same_source_ip />
    <description>Fortigate: Multiple failed login events from same source.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failures,gdpr_IV_35.7.d,gdpr_IV_32.2,gpg13_7.1,hipaa_164.312.b,nist_800_53_AU.6,nist_800_53_AU.14,nist_800_53_AC.7,pci_dss_10.6.1,pci_dss_10.2.4,pci_dss_10.2.5,</group>
  </rule>

  <!-- Additional rules omitted for brevity -->
</group>
```

## 4. Wazuh Agent Configuration

### 4.1 Configure ossec.conf
Add to `/var/ossec/etc/ossec.conf`:
```xml
<localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/fortinet.log</location>
</localfile>
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/messages</location>
</localfile>
```

## 5. Restart Services

### 5.1 On Wazuh Agent
```bash
sudo systemctl restart rsyslog
sudo systemctl restart wazuh-agent
```

### 5.2 On Wazuh Manager
```bash
sudo systemctl restart wazuh-manager
```

## Verification

1. Check rsyslog status:
```bash
sudo systemctl status rsyslog
```

2. Monitor log files:
```bash
sudo tail -f /var/ossec/logs/fortinet.log
sudo tail -f /var/log/messages
```

3. Check Wazuh alerts:
```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log
```

## Troubleshooting

1. Verify rsyslog is listening on port 514:
```bash
sudo netstat -tulnp | grep 514
```

2. Check Wazuh agent status:
```bash
sudo /var/ossec/bin/wazuh-control status
```

3. Verify log permissions:
```bash
sudo ls -l /var/ossec/logs/fortinet.log
sudo ls -l /var/log/messages
```
