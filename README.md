# Error

```bash

[dependencies]
image = "0.25.1"
mime_guess = "2.0.5"
serde = { version = "1.0.204", features = ["derive"] }
serde_json = "1.0.120"
wry = { version = "0.41.0", features = [
	"devtools",
	"transparent",
	"fullscreen",
	"protocol",
	
] }
allocator_api = "0.6.0"
tao = "0.28.1"
ureq = { version = "2.10.0", default-features = false, features = [
	"native-tls",
	"gzip",
	"json",
] }
base64 = { version = "0.22.1", features = ["alloc"] }
native-tls = "0.2.12"
os_info = "3.8.2"
directories = "5.0.1"
opener = "0.7.1"
rfd = { version = "0.14.1", default-features = false, features = [
	"xdg-portal",
] }
fs_extra = "1.3.0"
anyhow = "1.0.86"
flate2 = "1.0.30"
png = "0.17.13"
sys-locale = "0.3.1"
url = "2.5.2"
glob = "0.3.1"
tauri-winres = "0.1.1"
serde-pyobject = "0.4.0"

[target.'cfg(target_os = "macos")'.dependencies]
cocoa = { version = "0.25.0" }
objc = "0.2.7"
active-win-pos-rs = "0.8.3"

[target.'cfg(target_os = "windows")'.dependencies]
winapi = "0.3.9"
windows = { version = "0.58.0", features = ["Win32_Foundation"] }




```


### first idea


```rust
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};
use serde_json::Value;
use tao::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow,EventLoop, EventLoopBuilder, EventLoopProxy},
    window::{Window, WindowAttributes, WindowBuilder, WindowId},
};
use wry::{http::Request, WebView, WebViewAttributes, WebViewBuilder};



pub type ArcMut<T> = Arc<Mutex<T>>;

#[allow(dead_code)]
pub fn arc<T>(t: T) -> Arc<T> {
    Arc::new(t)
}

#[allow(dead_code)]
pub fn arc_mut<T>(t: T) -> ArcMut<T> {
    Arc::new(Mutex::new(t))
}

#[allow(dead_code)]
macro_rules! unsafe_impl_sync_send {
    ($type:ty) => {
        unsafe impl Send for $type {}
        unsafe impl Sync for $type{}
    };
}

pub enum PythonEventAPI {
    CloseWindow(WindowId),
    NewTitle(WindowId, String),
    NewWindow,
    PythonEvent(HashMap<String, Value>),
    NewMessageReceived(String),
}









pub struct PythonEventlister {
    window_id: WindowId,
    proxy: EventLoopProxy<PythonEventAPI>,
}

impl PythonEventlister {
    pub fn new(
        window_id: WindowId,
        proxy: EventLoopProxy<PythonEventAPI>,
    ) -> PythonEventlister {
        Self {
            window_id,
            proxy,
        }
    }

    pub fn handle_receive_message(self) -> impl Fn(Request<String>) + 'static {
        move |req: Request<String>| {
            let body = req.body();
            if body == "close" {
                let _ = self.proxy.send_event(PythonEventAPI::CloseWindow(self.window_id));
            }
        }
    }
}








struct PythonEventHandler;

impl PythonEventHandler {
    pub fn new() -> Self {
        PythonEventHandler
    }

    pub fn handle_incoming_event(&self, event: Event<PythonEventAPI>, _proxy:Arc<EventLoopProxy<PythonEventAPI>>) {
        match event {
            Event::UserEvent(PythonEventAPI::CloseWindow(window_id)) => {
                println!("Close window event received for window ID: {:?}", window_id);
                // let event = "";
                // proxy.send_event(PythonEventAPI::PythonEvent(event));
                // Handle window close logic here
            },
            Event::UserEvent(PythonEventAPI::NewTitle(window_id, title)) => {
                println!("New title event received for window ID: {:?}, Title: {}", window_id, title);
                // Handle title change logic here
            },
            Event::UserEvent(PythonEventAPI::NewWindow) => {
                println!("New window event received");
                // Handle new window logic here
            },
            Event::UserEvent(PythonEventAPI::PythonEvent(data)) => {
                println!("Python event received: {:?}", data);
                // Handle Python event logic here
            },
            Event::UserEvent(PythonEventAPI::NewMessageReceived(message)) => {
                println!("New message received: {}", message);
                // Handle new message logic here
            },
            _ => {
                // Handle other events if needed
            }
        }
    }
}




struct PyWindowFrame {
    win_attrs: WindowAttributes,
}

impl PyWindowFrame {
    pub fn new() -> PyWindowFrame {
        Self {
            win_attrs: Default::default(),
        }
    }

    pub fn with_title(&mut self, title: &str) {
        self.win_attrs.title = title.into();
    }
}

pub struct PyWebView {
    pub webview_attrs: WebViewAttributes,
}

impl PyWebView {
    pub fn new() -> PyWebView {
        Self {
            webview_attrs: Default::default(),
        }
    }

    pub fn with_url(&mut self, url: &str) {
        self.webview_attrs.url = Some(url.into());
    }

    pub fn with_html(&mut self, html: &str) {
        self.webview_attrs.html = Some(html.into());
    }
}

pub struct PyFrame {
    event_loop: EventLoop<PythonEventAPI>,
    window: Window,
    webview: WebView,
}

impl PyFrame {
    pub(crate) fn new(
        window: PyWindowFrame,
        webview: PyWebView,
        _web_context: Option<bool>,
    ) -> PyFrame {
        let event_loop = EventLoopBuilder::<PythonEventAPI>::with_user_event().build();
        let my_event_proxy = event_loop.create_proxy();
        let mut window_builder = WindowBuilder::new();

        window_builder.window = window.win_attrs.clone();
        let main_window = window_builder.build(&event_loop).unwrap();
        let event_listner = PythonEventlister::new(
            main_window.id(),
            my_event_proxy,
        );

        #[cfg(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        ))]
        let mut webview_builder = WebViewBuilder::new(&main_window);

        #[cfg(not(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        )))]
        let mut webview_builder = {
            use tao::platform::unix::WindowExtUnix;
            use wry::WebViewBuilderExtUnix;
            let vbox = main_window.default_vbox().unwrap();
            WebViewBuilder::new_gtk(vbox)
        };

        let attributes = webview.webview_attrs;
        webview_builder.attrs = attributes;
        webview_builder = webview_builder.with_ipc_handler(event_listner.handle_receive_message());

        let main_webview = webview_builder.build().unwrap();

        Self {
            event_loop,
            window: main_window,
            webview: main_webview,
        }
    }

    pub fn set_title(&self, title: &str) {
        let _ = self.window.set_title(title);
    }

    pub fn evaluate_script(&self, js: &str) {
        let _ = self.webview.evaluate_script(js);
    }

    pub(crate) fn run(self, py:PythonEventHandler) {
        let proxy = Arc::new(self.event_loop.create_proxy());
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;

            if let Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } = event
            {
                *control_flow = ControlFlow::Exit;
            }

            // Clone proxy to pass to the event handler
            py.handle_incoming_event(event, Arc::clone(&proxy));
        });
    }

}


unsafe_impl_sync_send!(PyFrame);
unsafe_impl_sync_send!(PyWebView);

fn main() {

    let backend_api = PythonEventHandler::new();
    let mut window_frame = PyWindowFrame::new();
    window_frame.with_title("Rust Application");



    let mut webview_frame = PyWebView::new();
    //webview_frame.with_url("https://www.example.com");
    let html_code = r#"
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Mein Button</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                text-align: center;
                margin-top: 50px;
            }
            button {
                padding: 10px 20px;
                font-size: 16px;
                cursor: pointer;
            }
        </style>
    </head>
    <body>
        <h1>Willkommen auf meiner Seite</h1>
        <button onclick="close_event()">Klick mich!</button>
        <script>

            function close_event(){
                window.ipc.postMessage('close');
            }
        </script>
    </body>
    </html>
    "#;
    webview_frame.with_html(html_code);




    let py_frame = PyFrame::new(window_frame, webview_frame, None);
    py_frame.evaluate_script("alert('Hallo from Rust')");
    py_frame.set_title("Hallo JS from Rust");
    py_frame.run(backend_api);
}

```



### second idea


```rust


use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};
use serde_json::Value;
use tao::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow,EventLoop, EventLoopBuilder, EventLoopProxy},
    window::{Window, WindowAttributes, WindowBuilder, WindowId},
};
use wry::{http::Request, WebView, WebViewAttributes, WebViewBuilder};



pub type ArcMut<T> = Arc<Mutex<T>>;

#[allow(dead_code)]
pub fn arc<T>(t: T) -> Arc<T> {
    Arc::new(t)
}

#[allow(dead_code)]
pub fn arc_mut<T>(t: T) -> ArcMut<T> {
    Arc::new(Mutex::new(t))
}

#[allow(dead_code)]
macro_rules! unsafe_impl_sync_send {
    ($type:ty) => {
        unsafe impl Send for $type {}
        unsafe impl Sync for $type{}
    };
}



pub enum PythonEventAPI {
    CloseWindow(WindowId),
    NewTitle(WindowId, String),
    NewWindow,
    PythonEvent(HashMap<String, Value>),
    NewMessageReceived(String),
}









pub struct PythonEventlister {
    window_id: WindowId,
    proxy: EventLoopProxy<PythonEventAPI>,
}

impl PythonEventlister {
    pub fn new(
        window_id: WindowId,
        proxy: EventLoopProxy<PythonEventAPI>,
    ) -> PythonEventlister {
        Self {
            window_id,
            proxy,
        }
    }

    pub fn handle_receive_message(self) -> impl Fn(Request<String>) + 'static {
        move |req: Request<String>| {
            let body = req.body();
            if body == "close" {
                let _ = self.proxy.send_event(PythonEventAPI::CloseWindow(self.window_id));
            }
        }
    }
}








struct PythonEventHandler;

impl PythonEventHandler {
    pub fn new() -> Self {
        PythonEventHandler
    }

    pub fn handle_incoming_event(&self, event: Event<PythonEventAPI>, _proxy:Arc<EventLoopProxy<PythonEventAPI>>) {
        match event {
            Event::UserEvent(PythonEventAPI::CloseWindow(window_id)) => {
                println!("Close window event received for window ID: {:?}", window_id);
                // let event = "";
                // proxy.send_event(PythonEventAPI::PythonEvent(event));
                // Handle window close logic here
            },
            Event::UserEvent(PythonEventAPI::NewTitle(window_id, title)) => {
                println!("New title event received for window ID: {:?}, Title: {}", window_id, title);
                // Handle title change logic here
            },
            Event::UserEvent(PythonEventAPI::NewWindow) => {
                println!("New window event received");
                // Handle new window logic here
            },
            Event::UserEvent(PythonEventAPI::PythonEvent(data)) => {
                println!("Python event received: {:?}", data);
                // Handle Python event logic here
            },
            Event::UserEvent(PythonEventAPI::NewMessageReceived(message)) => {
                println!("New message received: {}", message);
                // Handle new message logic here
            },
            _ => {
                // Handle other events if needed
            }
        }
    }
}




struct PyWindowFrame {
    win_attrs: WindowAttributes,
}

impl PyWindowFrame {
    pub fn new() -> PyWindowFrame {
        Self {
            win_attrs: Default::default(),
        }
    }

    pub fn with_title(&mut self, title: &str) {
        self.win_attrs.title = title.into();
    }
}

pub struct PyWebView {
    pub webview_attrs: WebViewAttributes,
}

impl PyWebView {
    pub fn new() -> PyWebView {
        Self {
            webview_attrs: Default::default(),
        }
    }

    pub fn with_url(&mut self, url: &str) {
        self.webview_attrs.url = Some(url.into());
    }

    pub fn with_html(&mut self, html: &str) {
        self.webview_attrs.html = Some(html.into());
    }

    pub fn build(&self, window:Window, event_listner:PythonEventlister)->WebView{
        
        #[cfg(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        ))]
        let mut webview_builder = WebViewBuilder::new(&window);

        #[cfg(not(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        )))]
        let mut webview_builder = {
            use tao::platform::unix::WindowExtUnix;
            use wry::WebViewBuilderExtUnix;
            let vbox = main_window.default_vbox().unwrap();
            WebViewBuilder::new_gtk(vbox)
        };

        let attributes = &self.webview_attrs;
        webview_builder.attrs = attributes; // Error
        webview_builder = webview_builder.with_ipc_handler(event_listner.handle_receive_message());

        let main_webview = webview_builder.build().unwrap();
        main_webview
    }
}

pub struct PyFrame {
    event_loop: EventLoop<PythonEventAPI>,
    window: Window,
    webview: WebView,
}









impl PyFrame {
    pub(crate) fn new(
        window: &PyWindowFrame,
        webview: &PyWebView,
        _web_context: Option<bool>,
    ) -> PyFrame {
        let event_loop = EventLoopBuilder::<PythonEventAPI>::with_user_event().build();
        let my_event_proxy = event_loop.create_proxy();
        let mut window_builder = WindowBuilder::new();

        window_builder.window = window.win_attrs.clone();
        let main_window = window_builder.build(&event_loop).unwrap();
        let event_listner = PythonEventlister::new(
            main_window.id(),
            my_event_proxy,
        );
        
        Self {
            event_loop,
            window: main_window,
            webview: webview.build(main_window,event_listner), // Error
        }
    }

    pub fn set_title(&self, title: &str) {
        let _ = self.window.set_title(title);
    }

    pub fn evaluate_script(&self, js: &str) {
        let _ = self.webview.evaluate_script(js);
    }

    pub(crate) fn run(self, py:PythonEventHandler) {
        let proxy = Arc::new(self.event_loop.create_proxy());
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;

            if let Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } = event
            {
                *control_flow = ControlFlow::Exit;
            }

            // Clone proxy to pass to the event handler
            py.handle_incoming_event(event, Arc::clone(&proxy));
        });
    }

}


unsafe_impl_sync_send!(PyFrame);
unsafe_impl_sync_send!(PyWebView);








fn main() {

    let backend_api = PythonEventHandler::new();
    let mut window_frame = PyWindowFrame::new();
    window_frame.with_title("Rust Application");



    let mut webview_frame = PyWebView::new();
    //webview_frame.with_url("https://www.example.com");
    let html_code = r#"
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Mein Button</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                text-align: center;
                margin-top: 50px;
            }
            button {
                padding: 10px 20px;
                font-size: 16px;
                cursor: pointer;
            }
        </style>
    </head>
    <body>
        <h1>Willkommen auf meiner Seite</h1>
        <button onclick="close_event()">Klick mich!</button>
        <script>

            function close_event(){
                window.ipc.postMessage('close');
            }
        </script>
    </body>
    </html>
    "#;
    webview_frame.with_html(html_code);




    let py_frame = PyFrame::new(&window_frame, &webview_frame, None);
    py_frame.evaluate_script("alert('Hallo from Rust')");
    py_frame.set_title("Hallo JS from Rust");
    py_frame.run(backend_api);
}




```



# compiled code
```rust



use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};
use serde_json::Value;
use tao::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow,EventLoop, EventLoopBuilder, EventLoopProxy},
    window::{Window, WindowAttributes, WindowBuilder, WindowId},
};
use wry::{http::Request, WebView, WebViewAttributes, WebViewBuilder};



pub type ArcMut<T> = Arc<Mutex<T>>;

#[allow(dead_code)]
pub fn arc<T>(t: T) -> Arc<T> {
    Arc::new(t)
}

#[allow(dead_code)]
pub fn arc_mut<T>(t: T) -> ArcMut<T> {
    Arc::new(Mutex::new(t))
}

#[allow(dead_code)]
macro_rules! unsafe_impl_sync_send {
    ($type:ty) => {
        unsafe impl Send for $type {}
        unsafe impl Sync for $type{}
    };
}

pub enum PythonEventAPI {
    CloseWindow(WindowId),
    NewTitle(WindowId, String),
    NewWindow,
    PythonEvent(HashMap<String, Value>),
    NewMessageReceived(String),
}









pub struct PythonEventlister {
    window_id: WindowId,
    proxy: EventLoopProxy<PythonEventAPI>,
}

impl PythonEventlister {
    pub fn new(
        window_id: WindowId,
        proxy: EventLoopProxy<PythonEventAPI>,
    ) -> PythonEventlister {
        Self {
            window_id,
            proxy,
        }
    }

    pub fn handle_receive_message(self) -> impl Fn(Request<String>) + 'static {
        move |req: Request<String>| {
            let body = req.body();
            if body == "close" {
                let _ = self.proxy.send_event(PythonEventAPI::CloseWindow(self.window_id));
            }
        }
    }
}








struct PythonEventHandler;

impl PythonEventHandler {
    pub fn new() -> Self {
        PythonEventHandler
    }

    pub fn handle_incoming_event(&self, event: Event<PythonEventAPI>, _proxy:Arc<EventLoopProxy<PythonEventAPI>>) {
        match event {
            Event::UserEvent(PythonEventAPI::CloseWindow(window_id)) => {
                println!("Close window event received for window ID: {:?}", window_id);
                // let event = "";
                // proxy.send_event(PythonEventAPI::PythonEvent(event));
                // Handle window close logic here
            },
            Event::UserEvent(PythonEventAPI::NewTitle(window_id, title)) => {
                println!("New title event received for window ID: {:?}, Title: {}", window_id, title);
                // Handle title change logic here
            },
            Event::UserEvent(PythonEventAPI::NewWindow) => {
                println!("New window event received");
                // Handle new window logic here
            },
            Event::UserEvent(PythonEventAPI::PythonEvent(data)) => {
                println!("Python event received: {:?}", data);
                // Handle Python event logic here
            },
            Event::UserEvent(PythonEventAPI::NewMessageReceived(message)) => {
                println!("New message received: {}", message);
                // Handle new message logic here
            },
            _ => {
                // Handle other events if needed
            }
        }
    }
}




struct PyWindowFrame {
    win_attrs: WindowAttributes,
}

impl PyWindowFrame {
    pub fn new() -> PyWindowFrame {
        Self {
            win_attrs: Default::default(),
        }
    }

    pub fn with_title(&mut self, title: &str) {
        self.win_attrs.title = title.into();
    }
}

pub struct PyWebView {
    pub webview_attrs: WebViewAttributes,
}

impl PyWebView {
    pub fn new() -> PyWebView {
        Self {
            webview_attrs: Default::default(),
        }
    }

    pub fn with_url(&mut self, url: &str) {
        self.webview_attrs.url = Some(url.into());
    }

    pub fn with_html(&mut self, html: &str) {
        self.webview_attrs.html = Some(html.into());
    }
}

pub struct PyFrame {
    event_loop: EventLoop<PythonEventAPI>,
    window: Window,
    webview: WebView,
}

impl PyFrame {
    pub(crate) fn new(
        window: PyWindowFrame,
        webview: PyWebView,
        _web_context: Option<bool>,
    ) -> PyFrame {
        let event_loop = EventLoopBuilder::<PythonEventAPI>::with_user_event().build();
        let my_event_proxy = event_loop.create_proxy();
        let mut window_builder = WindowBuilder::new();

        window_builder.window = window.win_attrs.clone();
        let main_window = window_builder.build(&event_loop).unwrap();
        let event_listner = PythonEventlister::new(
            main_window.id(),
            my_event_proxy,
        );

        #[cfg(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        ))]
        let mut webview_builder = WebViewBuilder::new(&main_window);

        #[cfg(not(any(
            target_os = "windows",
            target_os = "macos",
            target_os = "ios",
            target_os = "android"
        )))]
        let mut webview_builder = {
            use tao::platform::unix::WindowExtUnix;
            use wry::WebViewBuilderExtUnix;
            let vbox = main_window.default_vbox().unwrap();
            WebViewBuilder::new_gtk(vbox)
        };

        let attributes = webview.webview_attrs;
        webview_builder.attrs = attributes;
        webview_builder = webview_builder.with_ipc_handler(event_listner.handle_receive_message());

        let main_webview = webview_builder.build().unwrap();

        Self {
            event_loop,
            window: main_window,
            webview: main_webview,
        }
    }

    pub fn set_title(&self, title: &str) {
        let _ = self.window.set_title(title);
    }

    pub fn evaluate_script(&self, js: &str) {
        let _ = self.webview.evaluate_script(js);
    }

    pub(crate) fn run(self, py:PythonEventHandler) {
        let proxy = Arc::new(self.event_loop.create_proxy());
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;

            if let Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } = event
            {
                *control_flow = ControlFlow::Exit;
            }

            // Clone proxy to pass to the event handler
            py.handle_incoming_event(event, Arc::clone(&proxy));
        });
    }

}


unsafe_impl_sync_send!(PyFrame);
unsafe_impl_sync_send!(PyWebView);

fn main() {

    let backend_api = PythonEventHandler::new();
    let mut window_frame = PyWindowFrame::new();
    window_frame.with_title("Rust Application");



    let mut webview_frame = PyWebView::new();
    //webview_frame.with_url("https://www.example.com");
    let html_code = r#"
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Mein Button</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                text-align: center;
                margin-top: 50px;
            }
            button {
                padding: 10px 20px;
                font-size: 16px;
                cursor: pointer;
            }
        </style>
    </head>
    <body>
        <h1>Willkommen auf meiner Seite</h1>
        <button onclick="close_event()">Klick mich!</button>
        <script>

            function close_event(){
                window.ipc.postMessage('close');
            }
        </script>
    </body>
    </html>
    "#;
    webview_frame.with_html(html_code);




    let py_frame = PyFrame::new(window_frame, webview_frame, None);
    py_frame.evaluate_script("alert('Hallo from Rust')");
    py_frame.set_title("Hallo JS from Rust");
    py_frame.run(backend_api);
}






```
# This will be the result on the Python side


```python


class PyWebViewBuilder:
    def __init__(self):
        pass


    def with_accept_first_mouse(self):
        pass

    def with_asynchronous_custom_protocol(self):
        pass

    def with_autoplay(self):
        pass

    def with_back_forward_navigation_gestures(self):
        pass

    def with_background_color(self):
        pass

    def with_bounds(self):
        pass

    def with_clipboard(self):
        pass

    def with_custom_protocol(self):
        pass

    def with_devtools(self):
        pass

    def with_document_title_changed_handler(self):
        pass

    def with_download_completed_handler(self):
        pass

    def with_download_started_handler(self):
        pass

    def with_drag_drop_handler(self):
        pass

    def with_focused(self):
        pass

    def with_headers(self):
        pass

    def with_hotkeys_zoom(self):
        pass

    def with_html(self):
        pass

    def with_incognito(self):
        pass

    def with_initialization_script(self):
        pass

    def with_ipc_handler(self):
        pass

    def with_navigation_handler(self):
        pass

    def with_new_window_req_handler(self):
        pass

    def with_on_page_load_handler(self):
        pass

    def with_proxy_config(self):
        pass

    def with_transparent(self):
        pass

    def with_url(self):
        pass

    def with_url_and_headers(self):
        pass

    def with_user_agent(self):
        pass

    def with_visible(self):
        pass

    def with_web_context(self):
        pass





class PyWindowBuilder:
    def __init__(self):
        pass

    def with_always_on_bottom(self):
        pass

    def with_always_on_top(self):
        pass

    def with_closable(self):
        pass

    def with_content_protection(self):
        pass

    def with_decorations(self):
        pass

    def with_focused(self):
        pass

    def with_fullscreen(self):
        pass

    def with_inner_size(self, width, height):
        pass

    def with_inner_size_constraints(self, min_width, min_height, max_width, max_height):
        pass

    def with_max_inner_size(self, max_width, max_height):
        pass

    def with_maximizable(self):
        pass

    def with_maximized(self):
        pass

    def with_min_inner_size(self, min_width, min_height):
        pass

    def with_minimizable(self):
        pass

    def with_position(self, x, y):
        pass

    def with_resizable(self):
        pass

    def with_theme(self, theme):
        pass

    def with_title(self, title):
        pass

    def with_transparent(self):
        pass

    def with_visible(self):
        pass

    def with_visible_on_all_workspaces(self):
        pass

    def with_window_icon(self, icon_path):
        pass
















class PyFrame:
    def __init__(self):
        pass

    # -------------------------- all Window Methods ----------------------------------
    def available_monitors(self):
        pass

    def current_monitor(self):
        pass

    def cursor_position(self):
        pass

    def fullscreen(self):
        pass

    def id(self):
        pass

    def inner_position(self):
        pass

    def inner_size(self):
        pass

    def is_closable(self):
        pass

    def is_decorated(self):
        pass

    def is_focused(self):
        pass

    def is_maximizable(self):
        pass

    def is_maximized(self):
        pass

    def is_minimizable(self):
        pass

    def is_minimized(self):
        pass

    def is_resizable(self):
        pass

    def is_visible(self):
        pass

    def monitor_from_point(self, x, y):
        pass

    def outer_position(self):
        pass

    def outer_size(self):
        pass

    def primary_monitor(self):
        pass

    def scale_factor(self):
        pass


    def set_always_on_bottom(self, value):
        pass

    def set_always_on_top(self, value):
        pass

    def set_closable(self, value):
        pass

    def set_content_protection(self, value):
        pass

    def set_cursor_grab(self, value):
        pass

    def set_cursor_icon(self, icon):
        pass

    def set_cursor_position(self, x, y):
        pass

    def set_cursor_visible(self, visible):
        pass

    def set_decorations(self, value):
        pass

    def set_focus(self):
        pass

    def set_fullscreen(self, value):
        pass

    def set_ignore_cursor_events(self, ignore):
        pass

    def set_ime_position(self, x, y):
        pass

    def set_inner_size(self, width, height):
        pass

    def set_inner_size_constraints(self, min_width, min_height, max_width, max_height):
        pass

    def set_max_inner_size(self, max_width, max_height):
        pass

    def set_maximizable(self, value):
        pass

    def set_maximized(self, value):
        pass

    def set_min_inner_size(self, min_width, min_height):
        pass

    def set_minimizable(self, value):
        pass

    def set_minimized(self, value):
        pass

    def set_outer_position(self, x, y):
        pass

    def set_progress_bar(self, progress):
        pass

    def set_resizable(self, value):
        pass

    def set_title(self, title):
        pass

    def set_visible(self, visible):
        pass

    def set_visible_on_all_workspaces(self, visible):
        pass

    def set_window_icon(self, icon_path):
        pass

    def drag_resize_window(self, x, y):
        pass

    def drag_window(self, x, y):
        pass

    def request_redraw(self):
        pass

    def request_user_attention(self):
        pass

    def theme(self):
        pass

    def title(self):
        pass




    # -------------------------- all WebView Methods ----------------------------------
    def bounds(self):
        pass

    def clear_all_browsing_data(self):
        pass

    def close_devtools(self):
        pass

    def evaluate_script(self, script):
        pass

    def evaluate_script_with_callback(self, script, callback):
        pass

    def focus(self):
        pass

    def is_devtools_open(self):
        pass

    def load_url(self, url):
        pass

    def load_url_with_headers(self, url, headers):
        pass

    def new(self):
        pass

    def new_as_child(self):
        pass

    def open_devtools(self):
        pass

    def print(self):
        pass

    def set_background_color(self, color):
        pass

    def set_bounds(self, bounds):
        pass

    def set_visible(self, visible):
        pass

    def url(self):
        pass

    def zoom(self, level):
        pass

    # -------------------------- all Eventloop Methods ----------------------------------


    def available_monitors(self):
        pass
    def cursor_position(self):
        pass
    def monitor_from_point(self):
        pass
    def primary_monitor(self):
        pass
    def set_device_event_filter(self):
        pass
    def set_progress_bar(self):
        pass

    def run(self):
        pass





# Python Example


window = PyWindowBuilder()
window.with_always_on_bottom()

webview = PyWebViewBuilder()
webview.with_url()


app = PyFrame(window=window, webview=webview)
app.evaluate_script()





if __name__ == "__main__":
    app.run()


```



# Solution with one thought problem (see build_webview methods )


```rust

use std::{
    borrow::Cow, collections::HashMap, path::PathBuf, rc::Rc, sync::Arc
};
use serde_json::Value;
use tao::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow,EventLoop, EventLoopBuilder, EventLoopProxy},
    window::{Window, WindowAttributes, WindowBuilder, WindowId},
};
use wry::{http::{self, Request, Response}, DragDropEvent, PageLoadEvent, WebView};
mod pyweboptions;
use std::fmt;
use tao::dpi;
use wry::{ProxyConfig, ProxyEndpoint, Rect, WebViewBuilder, RGBA};


pub struct PyWebView {
    pub user_agent: Option<String>,
    pub visible: bool,
    pub transparent: bool,
    pub background_color: Option<RGBA>,
    pub url: Option<String>,
    pub headers: Option<http::HeaderMap>,
    pub zoom_hotkeys_enabled: bool,
    pub html: Option<String>,
    pub initialization_scripts: Vec<String>,
    pub clipboard: bool,
    pub devtools: bool,
    pub accept_first_mouse: bool,
    pub back_forward_navigation_gestures: bool,
    pub incognito: bool,
    pub autoplay: bool,
    pub proxy_config: Option<ProxyConfig>,
    pub focused: bool,
    pub bounds: Option<Rect>,
}


impl fmt::Debug for PyWebView {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("PyWebView")
            .field("user_agent", &self.user_agent)
            .field("visible", &self.visible)
            .field("transparent", &self.transparent)
            .field("background_color", &self.background_color)
            .field("url", &self.url)
            .field("headers", &self.headers)
            .field("zoom_hotkeys_enabled", &self.zoom_hotkeys_enabled)
            .field("html", &self.html)
            .field("initialization_scripts", &self.initialization_scripts)
            .field("clipboard", &self.clipboard)
            .field("devtools", &self.devtools)
            .field("accept_first_mouse", &self.accept_first_mouse)
            .field("back_forward_navigation_gestures", &self.back_forward_navigation_gestures)
            .field("incognito", &self.incognito)
            .field("autoplay", &self.autoplay)
            .field("proxy_config", &self.proxy_config)
            .field("focused", &self.focused)
            .field("bounds", &self.bounds)
            .finish()
    }
}

impl Clone for PyWebView {
    fn clone(&self) -> Self {
        Self {
            zoom_hotkeys_enabled:self.zoom_hotkeys_enabled.clone(),
            user_agent: self.user_agent.clone(),
            visible: self.visible,
            transparent: self.transparent,
            background_color: self.background_color.clone(),
            url: self.url.clone(),
            headers: self.headers.clone(),
            html: self.html.clone(),
            initialization_scripts: self.initialization_scripts.clone(),
            clipboard: self.clipboard,
            devtools: self.devtools,
            accept_first_mouse: self.accept_first_mouse,
            back_forward_navigation_gestures: self.back_forward_navigation_gestures,
            incognito: self.incognito,
            autoplay: self.autoplay,
            proxy_config: self.proxy_config.clone(),
            focused: self.focused,
            bounds: self.bounds.clone(),
        }
    }
}



impl Default for PyWebView {
    fn default() -> Self {
        Self {
            user_agent: None,
            visible: true,
            transparent: false,
            background_color: None,
            url: None,
            headers: None,
            html: None,
            initialization_scripts: vec![],
            clipboard: false,
            #[cfg(debug_assertions)]
            devtools: true,
            #[cfg(not(debug_assertions))]
            devtools: false,
            zoom_hotkeys_enabled: false,
            accept_first_mouse: false,
            back_forward_navigation_gestures: false,
            incognito: false,
            autoplay: true,
            proxy_config: None,
            focused: true,
            bounds: Some(Rect {
                position: dpi::LogicalPosition::new(0, 0).into(),
                size: dpi::LogicalSize::new(200, 200).into(),
            }),
        }
    }
}

impl PyWebView {
    #[allow(dead_code)]
    pub fn new() -> PyWebView {
        PyWebView::default()
    }

    #[allow(dead_code)]
    pub fn with_url(&mut self, url: &str) -> &mut Self {
        self.url = Some(url.into());
        self
    }

    #[allow(dead_code)]
    pub fn with_html(&mut self, html: &str) -> &mut Self {
        self.html = Some(html.into());
        self
    }

    #[allow(dead_code)]
    pub fn with_visible(&mut self, visible: bool) -> &mut Self {
        self.visible = visible;
        self
    }

    #[allow(dead_code)]
    pub fn with_transparent(&mut self, transparent: bool) -> &mut Self {
        self.transparent = transparent;
        self
    }

    #[allow(dead_code)]
    pub fn with_background_color(&mut self, background_color: Option<RGBA>) -> &mut Self {
        self.background_color = background_color;
        self
    }

    #[allow(dead_code)]
    pub fn with_headers(&mut self, headers: Option<http::HeaderMap>) -> &mut Self {
        self.headers = headers;
        self
    }

    #[allow(dead_code)]
    pub fn with_zoom_hotkeys_enabled(&mut self, zoom: bool) -> &mut Self {
        self.zoom_hotkeys_enabled = zoom;
        self
    }

    #[allow(dead_code)]
    pub fn with_initialization_scripts(&mut self, js:&str) -> &mut Self {
        if !js.is_empty() {
            self.initialization_scripts.push(js.to_string());
          }
        self
    }

    #[allow(dead_code)]
    pub fn with_clipboard(&mut self, clipboard: bool) -> &mut Self {
        self.clipboard = clipboard;
        self
    }

    #[allow(dead_code)]
    pub fn with_devtools(&mut self, devtools: bool) -> &mut Self {
        self.devtools = devtools;
        self
    }

    #[allow(dead_code)]
    pub fn with_back_forward_navigation_gestures(&mut self, gestures: bool) -> &mut Self {
        self.back_forward_navigation_gestures = gestures;
        self
    }

    #[allow(dead_code)]
    pub fn with_accept_first_mouse(&mut self, accept_first_mouse: bool) -> &mut Self {
        self.accept_first_mouse = accept_first_mouse;
        self
    }

    #[allow(dead_code)]
    pub fn with_incognito(&mut self, incognito: bool) -> &mut Self {
        self.incognito = incognito;
        self
    }

    #[allow(dead_code)]
    pub fn with_proxy_config(&mut self, proxy_config: Option<ProxyConfig>) -> &mut Self {
        self.proxy_config = proxy_config;
        self
    }


    #[allow(dead_code)]
    pub fn with_autoplay(&mut self, autoplay: bool) -> &mut Self {
        self.autoplay = autoplay;
        self
    }
    #[allow(dead_code)]
    pub fn with_focused(&mut self, focused: bool) -> &mut Self {
        self.focused = focused;
        self
    }

    #[allow(dead_code)]
    pub fn with_bounds(&mut self, bounds: Option<Rect>,) -> &mut Self {
        self.bounds = bounds;
        self
    }


    #[allow(dead_code)]
    pub fn build_webview(
        &self, 
        window: &Window, 
        event_listner: PythonEventlister,
        custom_protocols: impl Fn(Request<Vec<u8>>) -> Response<Cow<'static, [u8]>> + 'static, // Optionally can be None Type
        drag_drop_handler: Option<Box<dyn Fn(DragDropEvent) -> bool>>, // Optionally can be None Type
        navigation_handler: Option<Box<dyn Fn(String) -> bool>>, // Optionally can be None Type
        download_started_handler: Option<Box<dyn FnMut(String, &mut PathBuf) -> bool>>, // Optionally can be None Type
        download_completed_handler: Option<Rc<dyn Fn(String, Option<PathBuf>, bool) + 'static>>, // Optionally can be None Type
        new_window_req_handler: Option<Box<dyn Fn(String) -> bool>>, // Optionally can be None Type
        document_title_changed_handler: Option<Box<dyn Fn(String)>>, // Optionally can be None Type
        on_page_load_handler: Option<Box<dyn Fn(PageLoadEvent, String)>>  // Optionally can be None Type
    ) -> WebView {
            // Erzeuge den WebViewBuilder abhängig vom Betriebssystem
            #[cfg(any(
                target_os = "windows",
                target_os = "macos",
                target_os = "ios",
                target_os = "android"
            ))]
            let mut webview_builder = WebViewBuilder::new(&window);
        
            #[cfg(not(any(
                target_os = "windows",
                target_os = "macos",
                target_os = "ios",
                target_os = "android"
            )))]
            let mut webview_builder = {
                use tao::platform::unix::WindowExtUnix;
                use wry::WebViewBuilderExtUnix;
                let vbox = window.default_vbox().expect("Failed to get default vbox");
                WebViewBuilder::new_gtk(vbox)
            };
        
            // Wende die Konfigurationsoptionen an, sofern sie gesetzt sind
            webview_builder = webview_builder
                .with_ipc_handler(event_listner.handle_receive_message())
                .with_accept_first_mouse(self.accept_first_mouse);
    
            if let Some(user_agent) = &self.user_agent {
                webview_builder = webview_builder.with_user_agent(user_agent);
            }
    
            if let Some(url) = &self.url {
                webview_builder = webview_builder.with_url(url.clone());
            }
    
            if let Some(html) = &self.html {
                webview_builder = webview_builder.with_html(html.clone());
            }
    
            if let Some(background_color) = &self.background_color {
                // Ensure background_color is of type (u8, u8, u8, u8)
                webview_builder = webview_builder.with_background_color(*background_color);
            } else {
                // Use default value as a tuple
                let default_color: RGBA = (0, 0, 0, 0); // Example default value
                webview_builder = webview_builder.with_background_color(default_color);
            }
            if let Some(proxy_config) = &self.proxy_config {
                webview_builder = webview_builder.with_proxy_config(proxy_config.clone());
            } else {
                // Create a default ProxyEndpoint
                let default_endpoint = ProxyEndpoint{
                    host: "127.0.0.1".to_string(),
                    port: "8080".to_string(),
                };
                
                // Create a default ProxyConfig using the default ProxyEndpoint
                let default_proxy = ProxyConfig::Http(default_endpoint);
                
                webview_builder = webview_builder.with_proxy_config(default_proxy);
            }


            // Weitere optionale Konfigurationen
            webview_builder = webview_builder
                .with_clipboard(self.clipboard)
                .with_devtools(self.devtools)
                .with_back_forward_navigation_gestures(self.back_forward_navigation_gestures)
                .with_incognito(self.incognito)
                .with_autoplay(self.autoplay)
                .with_focused(self.focused);
    
            // Custom Protocol Handler
            webview_builder = webview_builder.with_custom_protocol("wry".into(), move |request: Request<Vec<u8>>| {
                custom_protocols(request) // Pass the request to the closure
            });
    
            if let Some(handler) = drag_drop_handler {
                webview_builder = webview_builder.with_drag_drop_handler(move |event| handler(event));
            }

    
            if let Some(handler) = navigation_handler {
                webview_builder = webview_builder.with_navigation_handler(move |url| handler(url));
            }
    
            if let Some(mut handler) = download_started_handler {
                webview_builder = webview_builder.with_download_started_handler(move |url, path| handler(url, path));
            }
    
            if let Some(handler) = download_completed_handler {
                webview_builder = webview_builder.with_download_completed_handler(move |url, path, success| {
                    handler(url, path, success)
                });
            }
            
            if let Some(handler) = new_window_req_handler {
                webview_builder = webview_builder.with_new_window_req_handler(move |url| handler(url));
            }
    
            if let Some(handler) = document_title_changed_handler {
                webview_builder = webview_builder.with_document_title_changed_handler(move |title| handler(title));
            }
    
            if let Some(handler) = on_page_load_handler {
                webview_builder = webview_builder.with_on_page_load_handler(move |event, url| handler(event, url));
            }
    
            // Überprüfe, ob Bounds gesetzt sind, bevor du sie anwendest
            let bounds = self.bounds.clone().expect("Bounds must be set");
            webview_builder = webview_builder.with_bounds(bounds);
    
            // Baue den WebView
            webview_builder.build().expect("Failed to build WebView")
    }
    
    
}






#[allow(dead_code)]
macro_rules! unsafe_impl_sync_send {
    ($type:ty) => {
        unsafe impl Send for $type {}
        unsafe impl Sync for $type{}
    };
}


pub enum PythonEventAPI {
    CloseWindow(WindowId),
    NewTitle(WindowId, String),
    NewWindow,
    PythonEvent(HashMap<String, Value>),
    NewMessageReceived(String),
}









pub struct PythonEventlister {
    window_id: WindowId,
    proxy: EventLoopProxy<PythonEventAPI>,
}

impl PythonEventlister {
    pub fn new(
        window_id: WindowId,
        proxy: EventLoopProxy<PythonEventAPI>,
    ) -> PythonEventlister {
        Self {
            window_id,
            proxy,
        }
    }

    pub fn handle_receive_message(self) -> impl Fn(Request<String>) + 'static {
        move |req: Request<String>| {
            let body = req.body();
            if body == "close" {
                let _ = self.proxy.send_event(PythonEventAPI::CloseWindow(self.window_id));
            }
        }
    }
}








struct PythonEventHandler;

impl PythonEventHandler {
    pub fn new() -> Self {
        PythonEventHandler
    }

    pub fn handle_incoming_event(&self, event: Event<PythonEventAPI>, _proxy:Arc<EventLoopProxy<PythonEventAPI>>) {
        match event {
            Event::UserEvent(PythonEventAPI::CloseWindow(window_id)) => {
                println!("Close window event received for window ID: {:?}", window_id);
                // let event = "";
                // proxy.send_event(PythonEventAPI::PythonEvent(event));
                // Handle window close logic here
            },
            Event::UserEvent(PythonEventAPI::NewTitle(window_id, title)) => {
                println!("New title event received for window ID: {:?}, Title: {}", window_id, title);
                // Handle title change logic here
            },
            Event::UserEvent(PythonEventAPI::NewWindow) => {
                println!("New window event received");
                // Handle new window logic here
            },
            Event::UserEvent(PythonEventAPI::PythonEvent(data)) => {
                println!("Python event received: {:?}", data);
                // Handle Python event logic here
            },
            Event::UserEvent(PythonEventAPI::NewMessageReceived(message)) => {
                println!("New message received: {}", message);
                // Handle new message logic here
            },
            _ => {
                // Handle other events if needed
            }
        }
    }
}




struct PyWindowFrame {
    win_attrs: WindowAttributes,
}

impl PyWindowFrame {
    pub fn new() -> PyWindowFrame {
        Self {
            win_attrs: Default::default(),
        }
    }

    pub fn with_title(&mut self, title: &str) {
        self.win_attrs.title = title.into();
    }
}


pub struct PyFrame {
    event_loop: EventLoop<PythonEventAPI>,
    window: Window,
    webview: WebView,
}

impl PyFrame {
    pub(crate) fn new(
        window: &PyWindowFrame,
        webview: &PyWebView,
        _web_context: Option<bool>,
    ) -> PyFrame {
        let event_loop = EventLoopBuilder::<PythonEventAPI>::with_user_event().build();
        let my_event_proxy = event_loop.create_proxy();
        let mut window_builder = WindowBuilder::new();
        let web = webview.clone();

        window_builder.window = window.win_attrs.clone();
        let main_window = window_builder.build(&event_loop).unwrap();
        let event_listner = PythonEventlister::new(
            main_window.id(),
            my_event_proxy,
        );

                
        // Define the handlers
        let custom_protocols = |_req: Request<Vec<u8>>| -> Response<Cow<'static, [u8]>> {
            Response::new(Cow::Borrowed(b"Custom Protocol Response"))
        };

        let drag_drop_handler: Option<Box<dyn Fn(DragDropEvent) -> bool>> = 
        Some(Box::new(|event: DragDropEvent| -> bool {
            println!("DragDropEvent: {:?}", event);
            true
        }));
    

        let navigation_handler: Option<Box<dyn Fn(String) -> bool>> = 
        Some(Box::new(|url: String| -> bool {
            println!("Navigating to: {}", url);
            true
        }));
    

        let download_started_handler: Option<Box<dyn FnMut(String, &mut PathBuf) -> bool>> = 
        Some(Box::new(|url: String, path: &mut PathBuf| -> bool {
            println!("Download started: {} at {:?}", url, path);
            true
        }));
    
        let download_completed_handler: Option<Rc<dyn Fn(String, Option<PathBuf>, bool)>> = 
        Some(Rc::new(|url: String, path: Option<PathBuf>, success: bool| {
            println!("Download completed: {} at {:?}, success: {}", url, path, success);
        }));
    

        let new_window_req_handler: Option<Box<dyn Fn(String) -> bool>> = 
        Some(Box::new(|url: String| -> bool {
            println!("New window requested for URL: {}", url);
            true
        }));
    
        let document_title_changed_handler: Option<Box<dyn Fn(String)>> = 
        Some(Box::new(|title: String| {
            println!("Document title changed to: {}", title);
        }));
    
        let on_page_load_handler: Option<Box<dyn Fn(PageLoadEvent, String)>> = 
        Some(Box::new(|_event: PageLoadEvent, url: String| {
            println!("Page loaded: {} with event:", url);
        }));
        let main_webview = web.build_webview(
            &main_window,
            event_listner,
            custom_protocols,
            drag_drop_handler,
            navigation_handler,
            download_started_handler,
            download_completed_handler,
            new_window_req_handler,
            document_title_changed_handler,
            on_page_load_handler,
        );
	/* 
	
	let main_webview = web.build_webview(
            &main_window,
            event_listner,
            Some(custom_protocols ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(drag_drop_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(navigation_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all                     
            Some(download_started_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(download_completed_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(new_window_req_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(document_title_changed_handler ),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
            Some(on_page_load_handler),this must also be able to be none, and if it is the case that nothing has been passed, it does not need to be called at all
        );
	
	*/

        Self {
            event_loop,
            window: main_window,
            webview: main_webview,
        }
    }

    pub fn set_title(&self, title: &str) {
        let _ = self.window.set_title(title);
    }

    pub fn evaluate_script(&self, js: &str) {
        let _ = self.webview.evaluate_script(js);
    }

    pub(crate) fn run(self, py:PythonEventHandler) {
        let proxy = Arc::new(self.event_loop.create_proxy());
        self.event_loop.run(move |event, _, control_flow| {
            *control_flow = ControlFlow::Wait;

            if let Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } = event
            {
                *control_flow = ControlFlow::Exit;
            }

            // Clone proxy to pass to the event handler
            py.handle_incoming_event(event, Arc::clone(&proxy));
        });
    }

}


unsafe_impl_sync_send!(PyFrame);
unsafe_impl_sync_send!(PyWebView);

fn main() {

    let backend_api = PythonEventHandler::new();
    let mut window_frame = PyWindowFrame::new();
    window_frame.with_title("Rust Application");



    let mut webview_frame = PyWebView::new();
    //webview_frame.with_url("https://www.example.com");
    let html_code = r#"
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Mein Button</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                text-align: center;
                margin-top: 50px;
            }
            button {
                padding: 10px 20px;
                font-size: 16px;
                cursor: pointer;
            }
        </style>
    </head>
    <body>
        <h1>Willkommen auf meiner Seite</h1>
        <button onclick="close_event()">Klick mich!</button>
        <script>

            function close_event(){
                window.ipc.postMessage('close');
            }
        </script>
    </body>
    </html>
    "#;
    webview_frame.with_html(html_code);




    let py_frame = PyFrame::new(&window_frame, &webview_frame, None);
    py_frame.evaluate_script("alert('Hallo from Rust')");
    py_frame.set_title("Hallo JS from Rust");
    py_frame.run(backend_api);
}

```
