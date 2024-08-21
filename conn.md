


```rust



use std::collections::HashMap;
use std::sync::Arc;
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio::sync::Mutex;
use serde_json;

struct Connection {
    write: Arc<Mutex<tokio::io::WriteHalf<TcpStream>>>,
    read: Arc<Mutex<tokio::io::ReadHalf<TcpStream>>>,
    callback_store: Arc<Mutex<HashMap<String, Box<dyn Fn(HashMap<String, String>) + Send + Sync>>>>,
}

impl Connection {
    async fn emit(&self, event: &str, data: HashMap<String, String>) -> io::Result<()> {
        let mut json_data = data.clone();
        json_data.insert("event".to_string(), event.to_string());
        let json = serde_json::to_string(&json_data)?;
        let bytes = json.as_bytes();

        let mut write_handle = self.write.lock().await;
        write_handle.write_all(bytes).await?;
        Ok(())
    }

    async fn on<F>(&self, event: &str, callback: F) -> io::Result<()>
    where
        F: Fn(HashMap<String, String>) + Send + Sync + 'static,
    {
        let mut store = self.callback_store.lock().await;
        store.insert(event.to_string(), Box::new(callback));
        Ok(())
    }

    async fn start_tcp(host: &str, port: i32) -> io::Result<Self> {
        let stream = TcpStream::connect(format!("{}:{}", host, port)).await?;
        let (read, write) = tokio::io::split(stream);
        Ok(Connection {
            read: Arc::new(Mutex::new(read)),
            write: Arc::new(Mutex::new(write)),
            callback_store: Arc::new(Mutex::new(HashMap::new())),
        })
    }

    async fn listen_for_events(self: Arc<Self>) {
        let mut buf = [0; 1024];
        loop {
            let mut read_handle = self.read.lock().await;
            let n = read_handle.read(&mut buf).await.expect("Failed to read from socket");
            if n == 0 {
                break; // Connection closed
            }
            let message = String::from_utf8_lossy(&buf[..n]);
            
            if let Ok(parsed_message) = serde_json::from_str::<HashMap<String, String>>(&message) {
                if let Some(event) = parsed_message.get("event") {
                    let callback_store = self.callback_store.lock().await;
                    if let Some(callback) = callback_store.get(event) {
                        callback(parsed_message);
                    }
                }
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let connections = Arc::new(Connection::start_tcp("127.0.0.1", 8080).await.unwrap());

    connections.on("message", |data| {
        println!("Received message: {:?}", data);
    }).await.unwrap();

    let connections_clone = connections.clone();
    tokio::spawn(async move {
        connections_clone.listen_for_events().await;
    });

    let mut data = HashMap::new();
    data.insert("content".to_string(), "Hello, world!".to_string());
    connections.emit("message", data).await.unwrap();
}


```
