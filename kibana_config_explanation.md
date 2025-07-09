
# Kibana Configuration Settings Explained

This document explains some common Kibana configuration options related to server networking and SSL.

---

## `server.port`

- **Description:**  
  Specifies the TCP port where Kibana listens for incoming HTTP/HTTPS requests.

- **Default:** `5601`

- **Example:**  
  ```yaml
  server.port: 5601
  ```

- **Usage:**  
  Access Kibana in a browser via `http://<host>:5601` (or `https` if SSL is enabled).

---

## `server.host`

- **Description:**  
  Defines the IP address or hostname Kibana binds to for listening.

- **Example:**  
  ```yaml
  server.host: "192.168.1.14"
  ```

- **Usage:**  
  Binding to a specific IP restricts access to that interface.  
  Use `"0.0.0.0"` to listen on all interfaces.

---

## SSL/TLS Settings for Kibana Server

### `server.ssl.enabled`

- **Description:**  
  Enables HTTPS for the Kibana server.

- **Example:**  
  ```yaml
  server.ssl.enabled: true
  ```

---

### `server.ssl.certificate`

- **Description:**  
  Path to the SSL certificate file used by Kibana to serve HTTPS.

- **Example:**  
  ```yaml
  server.ssl.certificate: /path/to/kibana.crt
  ```

---

### `server.ssl.key`

- **Description:**  
  Path to the private key file corresponding to the SSL certificate.

- **Example:**  
  ```yaml
  server.ssl.key: /path/to/kibana.key
  ```

---

## Elasticsearch SSL Verification Mode

### `elasticsearch.ssl.verificationMode`

- **Description:**  
  Controls how Kibana verifies the SSL certificate of the Elasticsearch server when connecting over HTTPS.

- **Possible values:**  
  - `none`: Do **not** verify the Elasticsearch certificate (insecure, for testing).  
  - `certificate`: Verify the certificate but skip hostname verification.  
  - `full` (default): Full verification including hostname.

- **Example:**  
  ```yaml
  elasticsearch.ssl.verificationMode: none
  ```

- **Note:**  
  Setting this to `none` is useful in development or with self-signed certs, but not recommended for production.

---

# Summary

- `server.port` and `server.host` configure where Kibana listens for connections.
- SSL settings enable HTTPS for Kibana’s web interface.
- `elasticsearch.ssl.verificationMode` controls SSL trust verification between Kibana and Elasticsearch.

---

*This guide provides a general overview to help you understand and configure Kibana securely and appropriately.*
