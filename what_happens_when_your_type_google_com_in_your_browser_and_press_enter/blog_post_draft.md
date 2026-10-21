# What Happens When You Type https://www.google.com in Your Browser and Press Enter

You type a web address, press Enter, and a fraction of a second later a page appears. It feels instant, but behind that moment, your computer and dozens of machines around the world cooperate through a long chain of steps. This is one of the most popular interview questions for software engineers because answering it shows how well you understand the whole web stack, not just one layer of it.

In this post I follow a single request for `https://www.google.com` from the keyboard to the screen. I will cover DNS, TCP/IP, the firewall, HTTPS/SSL, the load balancer, the web server, the application server and the database.

One honest note first: the real infrastructure behind google.com is private and heavily customized. What I describe here is the typical architecture of a large website, which is exactly the architecture an interviewer expects you to explain.

Here is the big picture before we zoom in:

![Flow of a request to https://www.google.com](https://raw.githubusercontent.com/oceex/holbertonschool-network/main/what_happens_when_your_type_google_com_in_your_browser_and_press_enter/diagram.png)

*(In Medium, upload `diagram.png` here instead of using the link above.)*

---

## 1. The browser reads the URL

Before anything touches the network, the browser splits the address into parts:

- **Scheme:** `https`, so the connection must be encrypted
- **Host:** `www.google.com`, the machine we want to talk to
- **Port:** not written, so the browser uses the default for HTTPS, which is **443**
- **Path:** `/`, the home page

The browser also checks its own memory. If it already has a fresh copy of the page, or it already knows the IP address of the host, it can skip some of the next steps.

## 2. The DNS request: turning a name into an IP address

Computers reach each other with IP addresses, not names. `www.google.com` is a name that humans can remember, and the **Domain Name System (DNS)** is the phone book that translates it into an address. The lookup goes through several layers, and each one is a cache that can answer early:

1. **The browser cache**, then the **operating system cache**, then the local `hosts` file.
2. **The recursive resolver.** This is a DNS server run by your internet provider, or a public one such as `8.8.8.8` or `1.1.1.1`. It does the hard work for you.
3. If the resolver does not know the answer, it asks a **root server**, which does not know the address but points to the servers in charge of `.com`.
4. The **`.com` (TLD) server** does not know it either, but it points to the **authoritative name servers** for `google.com`.
5. The **authoritative server** gives the final answer: an **A record** (IPv4 address) or an **AAAA record** (IPv6 address) for `www.google.com`.

The resolver stores the answer for a time set by the record's **TTL**, then hands it back to your computer. Big sites often answer with different IP addresses depending on where you are, so that you reach a data center near you.

Now the browser knows where to go, for example `203.0.113.10` (a documentation address I use as an example).

## 3. TCP/IP: building the road

The **Internet Protocol (IP)** gives every machine an address and moves packets across many routers, hop by hop, from your computer to the destination. On top of it, the **Transmission Control Protocol (TCP)** makes the connection reliable: packets are numbered, lost ones are sent again, and everything is put back in order at the other end.

TCP also uses **ports**, which tell the destination which program should receive the data. Our request is aimed at the server IP on **port 443**.

Before any data moves, the two machines perform the **three-way handshake**:

1. Your computer sends `SYN`: "I would like to talk."
2. The server answers `SYN-ACK`: "OK, I am ready."
3. Your computer replies `ACK`: "Great, let's go."

Your home router also plays a part here. Your computer has a private address, and the router translates it to a public one using NAT so replies can find their way back.

## 4. The firewall: the guard at the door

Before the packets reach the servers, they cross a **firewall**. A firewall is a security system that filters traffic according to rules: which IP addresses, ports and protocols are allowed. For a public website, the rules typically allow ports **443** (HTTPS) and **80** (HTTP) and drop everything else, which greatly reduces what an attacker can reach. There are usually firewalls at several places, including your own computer and your router.

## 5. HTTPS and SSL/TLS: making the conversation private

The connection exists, but anyone sitting on the path, such as a public Wi-Fi owner, could read the data. That is why we use **HTTPS**, which is HTTP carried inside an encrypted **TLS** tunnel. (SSL is the old name of the technology, and people still say "SSL certificate" out of habit.)

The **TLS handshake** works like this:

1. **ClientHello:** the browser lists the TLS versions and encryption methods it supports, and names the site it wants (`www.google.com`).
2. **ServerHello and certificate:** the server picks the options and sends its **certificate**, which contains its public key and is signed by a trusted **Certificate Authority (CA)**.
3. **Verification:** the browser checks that the certificate is valid, not expired, issued for `www.google.com`, and signed by an authority it trusts. If anything is wrong, you see a security warning.
4. **Key exchange:** both sides use a key agreement method (usually ephemeral Diffie-Hellman) to create the same secret key without ever sending it over the network.
5. From now on, everything is encrypted with that fast shared key. Only now does the browser send its real request, such as `GET / HTTP/1.1`.

HTTPS gives us three guarantees: **confidentiality** (nobody can read the data), **integrity** (nobody can change it unnoticed) and **authentication** (you are really talking to Google).

## 6. The load balancer: sharing the work

A site as busy as Google cannot be served by a single machine. The IP address you reached belongs to a **load balancer**, which receives all incoming requests and spreads them across many servers behind it.

- It uses an **algorithm** to choose a server, such as round robin (each server in turn) or least connections (the least busy server).
- It runs **health checks**, so a server that is down stops receiving traffic and users do not notice.
- It often **terminates TLS**, meaning it decrypts the traffic so the servers behind it do not have to.
- Load balancers are also run in pairs or clusters, so the load balancer itself is not a **single point of failure (SPOF)**.

## 7. The web server: answering the request

The chosen machine runs a **web server**, such as Nginx or Apache. Its job is to speak HTTP: it reads the request, finds out what is being asked, and decides what to do.

- If the answer is a **static file** (an image, a stylesheet, a script), it sends the file directly, very quickly.
- If the page must be **built on the fly**, it forwards the request to the application server and waits for the result. This is called acting as a reverse proxy.

## 8. The application server: building the page

The **application server** runs the code of the website, written in a language such as Python, Java, Go or PHP. This is where the business logic lives. For a page like Google's, it decides what to show you, for example the right language, personalized elements and the search box. It produces the final HTML for the page.

## 9. The database: where the data lives

Pages are rarely made of code alone. They need data, and data lives in a **database**, such as MySQL. The application server sends a query, for example a SQL `SELECT`, and the database returns the matching rows.

Large systems usually use a **Primary-Replica** setup: the Primary database accepts all writes and copies them to Replicas, which answer read queries. This spreads the load and keeps copies of the data safe.

## 10. The trip back, and the browser does its job

The page now travels back along the same road: database to application server, to the web server, through the load balancer and the firewall, across the internet, encrypted with TLS the whole way to your computer. The response starts with a **status code**, usually `200 OK`, followed by headers and the HTML.

Then the browser takes over:

1. It decrypts the response and reads the HTML to build the **DOM** (the tree of page elements).
2. It finds links to CSS, JavaScript and images, and requests them too, usually over the same open connection.
3. It combines the DOM and the styles, computes the layout and paints the pixels.
4. It runs the JavaScript, which can change the page or load more data.

And there is the Google home page, waiting for your search.

---

## Summary

| Step | What happens |
|------|--------------|
| URL | The browser reads the scheme, host, port and path |
| DNS | `www.google.com` is translated into an IP address |
| TCP/IP | A reliable connection to IP:443 is created with a three-way handshake |
| Firewall | Only allowed traffic gets through |
| HTTPS/TLS | The browser checks the certificate and both sides agree on an encryption key |
| Load balancer | The request is sent to a healthy server |
| Web server | Serves static files, forwards dynamic requests |
| Application server | Runs the code and generates the page |
| Database | Stores and returns the data the page needs |
| Response | The page travels back and the browser renders it |

If you are asked this question in an interview, remember to ask which area the interviewer wants to dig into. A front-end role may want more about rendering, and an SRE role may want more about load balancing and failure handling.

Thanks for reading!
