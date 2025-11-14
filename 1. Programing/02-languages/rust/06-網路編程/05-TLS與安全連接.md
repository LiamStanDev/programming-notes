# TLS 與安全連接

## TLS 基礎

### 使用 rustls (客戶端)

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-rustls = "0.25"
rustls = "0.22"
webpki-roots = "0.26"
```

```rust
use tokio::net::TcpStream;
use tokio_rustls::{TlsConnector, rustls};
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut root_store = rustls::RootCertStore::empty();
    root_store.extend(
        webpki_roots::TLS_SERVER_ROOTS
            .iter()
            .cloned()
    );
    
    let config = rustls::ClientConfig::builder()
        .with_root_certificates(root_store)
        .with_no_client_auth();
    
    let connector = TlsConnector::from(Arc::new(config));
    let domain = rustls::pki_types::ServerName::try_from("www.rust-lang.org")?.to_owned();
    
    let stream = TcpStream::connect("www.rust-lang.org:443").await?;
    let mut tls_stream = connector.connect(domain, stream).await?;
    
    // 使用 TLS 連接
    use tokio::io::{AsyncWriteExt, AsyncReadExt};
    tls_stream.write_all(b"GET / HTTP/1.1\r\nHost: www.rust-lang.org\r\n\r\n").await?;
    
    let mut buf = vec![0; 1024];
    let n = tls_stream.read(&mut buf).await?;
    println!("{}", String::from_utf8_lossy(&buf[..n]));
    
    Ok(())
}
```

### TLS 服務端

```rust
use tokio::net::TcpListener;
use tokio_rustls::{TlsAcceptor, rustls};
use std::sync::Arc;
use std::fs::File;
use std::io::BufReader;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 加載證書和私鑰
    let cert_file = File::open("cert.pem")?;
    let key_file = File::open("key.pem")?;
    
    let certs = rustls_pemfile::certs(&mut BufReader::new(cert_file))
        .collect::<Result<Vec<_>, _>>()?;
    let key = rustls_pemfile::private_key(&mut BufReader::new(key_file))?
        .ok_or("No private key found")?;
    
    let config = rustls::ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)?;
    
    let acceptor = TlsAcceptor::from(Arc::new(config));
    let listener = TcpListener::bind("127.0.0.1:8443").await?;
    
    loop {
        let (stream, _) = listener.accept().await?;
        let acceptor = acceptor.clone();
        
        tokio::spawn(async move {
            match acceptor.accept(stream).await {
                Ok(tls_stream) => {
                    // 處理 TLS 連接
                    println!("TLS connection established");
                }
                Err(e) => eprintln!("TLS handshake failed: {}", e),
            }
        });
    }
}
```

## HTTPS 客戶端 (reqwest)

```rust
use reqwest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 默認使用系統證書
    let client = reqwest::Client::new();
    let response = client.get("https://www.rust-lang.org")
        .send()
        .await?;
    
    println!("Status: {}", response.status());
    
    // 自定義證書
    let cert = std::fs::read("custom_cert.pem")?;
    let cert = reqwest::Certificate::from_pem(&cert)?;
    
    let client = reqwest::Client::builder()
        .add_root_certificate(cert)
        .build()?;
    
    Ok(())
}
```

## 雙向 TLS (mTLS)

```rust
use tokio_rustls::rustls;
use std::sync::Arc;
use std::fs::File;
use std::io::BufReader;

async fn mtls_server() -> Result<(), Box<dyn std::error::Error>> {
    // 加載 CA 證書用於驗證客戶端
    let mut client_auth_roots = rustls::RootCertStore::empty();
    let ca_file = File::open("ca.pem")?;
    for cert in rustls_pemfile::certs(&mut BufReader::new(ca_file)) {
        client_auth_roots.add(cert?)?;
    }
    
    let client_cert_verifier = rustls::server::WebPkiClientVerifier::builder(Arc::new(client_auth_roots))
        .build()?;
    
    // 加載服務端證書
    let cert_file = File::open("server_cert.pem")?;
    let key_file = File::open("server_key.pem")?;
    
    let certs = rustls_pemfile::certs(&mut BufReader::new(cert_file))
        .collect::<Result<Vec<_>, _>>()?;
    let key = rustls_pemfile::private_key(&mut BufReader::new(key_file))?
        .ok_or("No private key")?;
    
    let config = rustls::ServerConfig::builder()
        .with_client_cert_verifier(client_cert_verifier)
        .with_single_cert(certs, key)?;
    
    // 使用 config...
    Ok(())
}
```

---

## 參考資料

1. [rustls Documentation](https://docs.rs/rustls/)
2. [tokio-rustls Documentation](https://docs.rs/tokio-rustls/)
3. [TLS 1.3 RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)
