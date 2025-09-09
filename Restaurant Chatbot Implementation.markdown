# Restaurant Chatbot Implementation

This implementation includes a Django backend with Channels for WebSocket communication and a Vite/React frontend, using Groq's Llama 3.1 model for the chatbot.

## Backend Setup (Django with Channels)

### 1. Project Structure
```
restaurant_chatbot/
├── backend/
│   ├── config/
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   └── urls.py
│   ├── chat/
│   │   ├── consumers.py
│   │   ├── routing.py
│   │   ├── __init__.py
│   │   └── apps.py
│   ├── manage.py
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── Chat.jsx
│   │   └── main.jsx
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
```

### 2. Backend Code

#### requirements.txt
```
django==4.2.16
channels==4.1.0
daphne==4.1.2
groq==0.11.0
python-dotenv==1.0.1
```

#### config/settings.py
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'daphne',
    'chat',
]

ASGI_APPLICATION = 'config.asgi.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
    },
}

# Add at the end of settings.py
import os
from dotenv import load_dotenv
load_dotenv()
GROQ_API_KEY = os.getenv('GROQ_API_KEY')
```

#### config/asgi.py
```python
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from chat.routing import websocket_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(
            websocket_urlpatterns
        )
    ),
})
```

#### chat/routing.py
```python
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/$', consumers.ChatConsumer.as_asgi()),
]
```

#### chat/consumers.py
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from groq import Groq
from django.conf import settings

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.group_name = 'restaurant_chat'
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        
        # Initialize Groq client
        client = Groq(api_key=settings.GROQ_API_KEY)
        
        # Define system prompt for restaurant context
        system_prompt = """
        You are a helpful restaurant chatbot for Sunny's Bistro. Provide accurate information about:
        - Menu: Italian cuisine, including pasta, pizza, salads, and desserts
        - Hours: Mon-Fri 11AM-10PM, Sat-Sun 10AM-11PM
        - Reservations: Available via phone or website
        - Location: 123 Main St, Foodville
        Answer concisely and professionally. If unsure, suggest contacting staff.
        """
        
        try:
            # Get response from Groq
            chat_completion = await client.chat.completions.create(
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": message}
                ],
                model="llama3-8b-8192",
                temperature=0.5,
                max_tokens=200
            )
            
            response = chat_completion.choices[0].message.content
            
            # Send response back to client
            await self.send(text_data=json.dumps({
                'message': response,
                'sender': 'bot'
            }))
            
        except Exception as e:
            await self.send(text_data=json.dumps({
                'message': f"Sorry, I encountered an error: {str(e)}",
                'sender': 'bot'
            }))
```

### 3. Frontend Setup (Vite/React)

#### package.json
```json
{
  "name": "restaurant-chatbot-frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "marked": "^14.1.2"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.8"
  }
}
```

#### index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Restaurant Chatbot</title>
</head>
<body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
</body>
</html>
```

#### src/main.jsx
```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App.jsx';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

#### src/App.jsx
```jsx
import React from 'react';
import Chat from './Chat.jsx';
import './App.css';

function App() {
  return (
    <div className="min-h-screen bg-gray-100 flex items-center justify-center">
      <div className="w-full max-w-md bg-white rounded-lg shadow-lg p-4">
        <h1 className="text-2xl font-bold text-center mb-4">Sunny's Bistro Chatbot</h1>
        <Chat />
      </div>
    </div>
  );
}

export default App;
```

#### src/Chat.jsx
```jsx
import React, { useState, useEffect, useRef } from 'react';
import { marked } from 'marked';

function Chat() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [ws, setWs] = useState(null);
  const messagesEndRef = useRef(null);

  useEffect(() => {
    const websocket = new WebSocket('ws://localhost:8000/ws/chat/');
    
    websocket.onopen = () => console.log('WebSocket connected');
    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setMessages((prev) => [...prev, { text: data.message, sender: data.sender }]);
    };
    websocket.onclose = () => console.log('WebSocket disconnected');

    setWs(websocket);

    return () => websocket.close();
  }, []);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const sendMessage = () => {
    if (input.trim() && ws) {
      ws.send(JSON.stringify({ message: input }));
      setMessages((prev) => [...prev, { text: input, sender: 'user' }]);
      setInput('');
    }
  };

  const handleKeyDown = (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <div className="flex flex-col h-96">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg, index) => (
          <div
            key={index}
            className={`p-3 rounded-lg max-w-[80%] ${
              msg.sender === 'user'
                ? 'bg-blue-500 text-white ml-auto'
                : 'bg-gray-200 text-black'
            }`}
          >
            <div dangerouslySetInnerHTML={{ __html: marked(msg.text) }} />
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>
      <div className="flex gap-2 p-4">
        <textarea
          className="flex-1 p-2 border rounded-lg resize-none"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Ask about our menu, hours, or reservations..."
        />
        <button
          className="bg-blue-500 text-white px-4 py-2 rounded-lg"
          onClick={sendMessage}
        >
          Send
        </button>
      </div>
    </div>
  );
}

export default Chat;
```

#### src/App.css
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 4. Setup Instructions

1. **Backend Setup**:
   - Create a virtual environment: `python -m venv venv`
   - Activate it: `source venv/bin/activate` (Linux/Mac) or `venv\Scripts\activate` (Windows)
   - Install dependencies: `pip install -r requirements.txt`
   - Create a `.env` file in the backend directory with your Groq API key:
     ```
     GROQ_API_KEY=your_groq_api_key_here
     ```
   - Run migrations: `python manage.py migrate`
   - Start the Django server: `python manage.py runserver`

2. **Frontend Setup**:
   - Navigate to the frontend directory: `cd frontend`
   - Install dependencies: `npm install`
   - Start the Vite development server: `npm run dev`

3. **Get Groq API Key**:
   - Sign up at [console.groq.com](https://console.groq.com) and generate a free API key.

### 5. How to Use
- Open the frontend at `http://localhost:5173`
- Type questions like "What's on the menu?" or "Can I make a reservation?"
- The chatbot responds in real-time using WebSocket communication, powered by Groq's Llama 3.1 model.

### 6. Notes
- The backend uses an in-memory channel layer for simplicity. For production, use Redis.
- The frontend includes basic styling with Tailwind CSS (add via CDN or npm if needed).
- The Groq API key is stored securely in a `.env` file.
- The chatbot maintains context via the system prompt, optimized for restaurant queries.