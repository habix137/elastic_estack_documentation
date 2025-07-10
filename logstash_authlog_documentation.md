
# Logstash Configuration: Auth Log Pipeline

This configuration sets up a Logstash pipeline to receive `auth.log` data from remote Linux systems over TCP and forward it securely to an Elasticsearch instance.

## 🔧 How to Use This in Logstash

1. Save this configuration file as `logstash_authlog.conf` in your Logstash configuration directory, e.g., `/home/user01/logstash/config/`.
2. Run Logstash with the following command:

```bash
./bin/logstash -f /home/user01/logstash/config/logstash_authlog.conf
```

Ensure Logstash has access to the `ca-cert.pem` file used to verify Elasticsearch's SSL certificate.

---

## 📥 Input Section

```logstash
input {
  tcp {
    port => 5000
    codec => line { charset => "UTF-8" }
  }
}
```

- Listens on TCP port 5000 for incoming logs from remote servers.
- Each line is treated as a separate message with UTF-8 encoding.

---

## 🔍 Filter Section

```logstash
filter {
  grok {
    match => {
      "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname} %{WORD:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:auth_message}"
    }
  }

  date {
    match => ["timestamp", "MMM d HH:mm:ss", "MMM dd HH:mm:ss"]
    target => "@timestamp"
  }

  mutate {
    remove_field => ["timestamp"]
  }
}
```

- `grok`: Parses syslog lines into structured fields.
- `date`: Parses the timestamp into Logstash's `@timestamp` field.
- `mutate`: Removes the intermediate `timestamp` field.

---

## 📤 Output Section

```logstash
output {
  elasticsearch {
    hosts => ["https://213.233.184.217:9200"]
    index => "authlog-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "********"

    ssl_enabled => true
    ssl_certificate_authorities => ["/home/user01/logstash/config/certs/ca-cert.pem"]
  }

  stdout {
    codec => rubydebug
  }
}
```

- Sends structured logs to Elasticsearch over HTTPS.
- Uses SSL with a custom certificate authority file.
- Password is hidden here for security.
- Logs are printed to console as well for debugging.

---

## 🛰️ Remote Syslog Configuration (on remote Linux server)

To send `/var/log/auth.log` to Logstash over TCP, configure `rsyslog`:

1. Create a file `/etc/rsyslog.d/60-logstash.conf` or edit `/etc/rsyslog.conf`:

```bash
auth,authpriv.*  @@213.233.184.217:5000
```

2. Restart `rsyslog`:

```bash
sudo systemctl restart rsyslog
```

> Note: `@@` specifies TCP (single `@` would mean UDP).

---

## ✅ Summary

| Component | Description |
|----------|-------------|
| Input     | TCP on port 5000, receives logs from remote syslog |
| Filter    | Extracts timestamp, hostname, program, pid, and message |
| Output    | Forwards logs to Elasticsearch securely and prints to console |
| Remote    | `rsyslog` sends logs to Logstash via TCP |

---

**Security Tip:** Store sensitive data such as Elasticsearch credentials in the Logstash keystore or use environment variables.

