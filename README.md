# Enterprise Standalone Middleware Architecture: Apache HTTPD to Tomcat Reverse Proxy Bridge

A production-grade, standalone infrastructure implementation demonstrating a Layer 7 Reverse Proxy architecture. This project isolates an Apache Tomcat 11 application server runtime safely behind a hardened Apache HTTPD 2.4 front-end gateway. The gateway captures all traffic on the standard HTTP port (80) and tunnels it securely back to the servlet engine on port 8080.

---

## 🏗️ Architectural Topology

```text
       [ Public Client Request ]
                  |
                  v (Port 80)
    +------------------------------+

    |  Apache HTTPD Web Server     |  <- Gateway Edge (C:\Apache24)
    |  (Acting as a Reverse Proxy) |
    +------------------------------+
                  |
                  v (Internal Tunnel)
    +------------------------------+

    |  Apache Tomcat Servlet Core  |  <- Application Runtime Engine
    |  (HTTP-NIO Core Socket)      |     (C:\ApacheTomcat, Port 8080)
    +------------------------------+
                  |
                  v
       [ Java 25 Runtime (JDK) ]      <- Decoupled Core (C:\JDK\jdk-25.0.2)
```

### Core Design Decisions & Mechanics
*   **Perimeter Decoupling:** Client browsers never interface with the application engine directly. Apache HTTPD terminates the initial connection at the network edge, protecting Tomcat from direct exploitation.
*   **Proxy Protocol Handshake:** Implements the `ProxyPass` and `ProxyPassReverse` directives. The proxy maps incoming URLs and interceptively rewrites HTTP response location headers so internal ports (`:8080`) are never leaked to the client's address bar.
*   **Decoupled Environment Management:** Bypasses unpredictable global OS system variable blocks by injecting a local `setenv.bat` runtime script hook inside Tomcat to explicitly manage the Java 25 JDK path context cleanly on boot.

---

## 📂 Server Filesystem Structure

The local environment is configured using standard standalone enterprise installation directories directly on the root drive:

```text
C:\
├── JDK\
│   └── jdk-25.0.2\                 <-- Core Java 25 Development Kit Runtime
├── Apache24\
│   ├── bin\httpd.exe               <-- Web Server Daemon Binaries
│   └── conf\httpd.conf             <-- Global configuration containing Proxy rules
└── ApacheTomcat\
    └── apache-tomcat-11.0.27\
        └── bin\
            ├── setenv.bat          <-- Local Java environment hook override
            └── startup.bat         <-- Tomcat engine bootstrap control script
```

---

## ⚙️ Middleware Configuration Signatures

### 1. Front-End Gateway Architecture (`C:\Apache24\conf\httpd.conf`)
The following configuration parameters were injected at the base of the global web server configuration file to dynamically load the network proxy pipelines and define the bridging loops:

```apache
# Initialize Layer 7 routing and text header engines
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule headers_module modules/mod_headers.so

# Hardening Directives against rogue forward proxying
ProxyRequests Off
ProxyPreserveHost On

# Establish the HTTP bridge loop between Port 80 and Port 8080
ProxyPass / http://localhost:8080/
ProxyPassReverse / http://localhost:8080/
```

### 2. Standalone Application Target Hook (`C:\ApacheTomcat\...\bin\setenv.bat`)
To ensure complete process isolation and reliable start behaviors under strict runtime checks, environment values are explicitly set using Tomcat's local boot interceptor batch:

```cmd
set "JAVA_HOME=C:\JDK\jdk-25.0.2"
```

---

## 🚀 Deployment & Operational Verification

### Execution Sequence
1.  **Boot the Application Engine:** Open an administrative command shell, step into the Tomcat control house, and start the engine:
    ```cmd
    cd C:\ApacheTomcat\apache-tomcat-11.0.27\bin
    startup.bat
    ```
    *Tomcat initializes its connectors and listens internally on port 8080.*

2.  **Boot the Reverse Proxy Gateway:** Open a second, independent administrative command shell and fire up the proxy daemon loop:
    ```cmd
    cd C:\Apache24\bin
    httpd.exe
    ```
    *The web server actively anchors port 80 and begins proxying traffic.*

### Infrastructure Verification Vector
*   Open an absolute clean web browser window and route traffic to the naked address: **`http://localhost`** (leaving out any port references).
*   **Expected Behavior:** The request hits Apache on Port 80, cleanly traverses the internal connection pool proxy bridge, grabs the servlet payload from Tomcat on Port 8080, and delivers the dynamic dashboard interface seamlessly back to the client browser.
