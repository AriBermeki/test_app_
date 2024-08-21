```rust


use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio::sync::{broadcast, mpsc, Mutex};
use std::collections::HashMap;
use std::sync::Arc;
use serde_json::to_string;

struct FromPythonToRust {
    sender: broadcast::Sender<String>,
}

impl FromPythonToRust {
    fn new(buffer_size: usize) -> Self {
        let (sender, _receiver) = broadcast::channel(buffer_size);
        Self { sender }
    }

    async fn start_receive_from_python_to_rust(&self, mut from_python: impl AsyncReadExt + Unpin) {
        let mut buffer = vec![0; 1024];
        loop {
            let n = match from_python.read(&mut buffer).await {
                Ok(n) if n == 0 => break, // Connection closed
                Ok(n) => n,
                Err(e) => {
                    eprintln!("Failed to read from TCP stream: {:?}", e);
                    break;
                }
            };

            let message = String::from_utf8_lossy(&buffer[..n]).to_string();

            if let Err(e) = self.sender.send(message) {
                println!("Error sending message: {:?}", e);
            }
        }
    }

    async fn receive_from_python(&self) {
        let mut receiver = self.sender.subscribe();
        while let Ok(message) = receiver.recv().await {
            message;
        }
    }
}

struct FromRustToPython {
    tx: mpsc::Sender<HashMap<String, String>>,
    rx: Mutex<mpsc::Receiver<HashMap<String, String>>>,
}

impl FromRustToPython {
    fn new(buffer_size: usize) -> Self {
        let (tx, rx) = mpsc::channel(buffer_size);
        Self {
            tx,
            rx: Mutex::new(rx),
        }
    }

    async fn send_from_rust(&self, data: HashMap<String, String>) {
        if let Err(e) = self.tx.send(data).await {
            eprintln!("Failed to send data: {:?}", e);
        }
    }

    async fn start_send_from_rust_to_python(&self, mut from_rust: impl AsyncWriteExt + Unpin) -> io::Result<()> {
        let mut rx = self.rx.lock().await;
        while let Some(received_map) = rx.recv().await {
            let serialized_data = to_string(&received_map).expect("Failed to serialize data");
            from_rust.write_all(serialized_data.as_bytes()).await?;
        }
        Ok(())
    }
}

struct Connection {
    rust: Arc<FromRustToPython>,
    python: Arc<FromPythonToRust>,
}

impl Connection {
    fn new(buffer_size: usize) -> Self {
        let python = Arc::new(FromPythonToRust::new(buffer_size));
        let rust = Arc::new(FromRustToPython::new(buffer_size));
        Self { rust, python }
    }



    async fn from_python(&self) {
        self.python.receive_from_python().await;
    }
    async fn to_python(&self, data: HashMap<String, String>) {
        self.rust.send_from_rust(data).await;
    }



    async fn start_tcp(&self, host:&str, port:i32) -> io::Result<()> {
        let stream = TcpStream::connect(format!("{}:{}", host, port)).await?;
        let (read_half, write_half) = stream.into_split();

        // Cloning Arcs to pass into async tasks
        let python_to_rust = Arc::clone(&self.python);
        tokio::spawn(async move {
            python_to_rust
                .start_receive_from_python_to_rust(read_half)
                .await;
        });

        let rust_to_python = Arc::clone(&self.rust);
        tokio::spawn(async move {
            if let Err(e) = rust_to_python.start_send_from_rust_to_python(write_half).await {
                eprintln!("Failed to send data from Rust to Python: {:?}", e);
            }
        });

        Ok(())
    }
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let connection = Connection::new(100);
    connection.start_tcp("127.0.0.1", 8080).await?;

    // Keep the main function running
    tokio::signal::ctrl_c().await?;
    println!("Received Ctrl-C, shutting down.");
    Ok(())
}





```
