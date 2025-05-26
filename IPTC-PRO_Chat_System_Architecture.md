# IPTC-PRO Chat System Technical Architecture

## Chat Module Overview

The IPTC-PRO chat system provides two main communication channels:

1. **Group Chat**: Course-wide discussions between instructors and all students
2. **Private Chat**: One-on-one conversations between an instructor and a student

## System Architecture

### Overview Diagram

```
+---------------------------+         +---------------------------+
|      Client Side          |         |       Server Side         |
+---------------------------+         +---------------------------+
|                           |         |                           |
| React Components          |         | Laravel Backend           |
| ┌───────────────────────┐ |  HTTP/  | ┌───────────────────────┐ |
| │  CourseChat.jsx       │◄┼─WebSock─┼►│  ChatController.php   │ |
| │  - Group Chat         │ |   API   | │  - API Endpoints      │ |
| └───────────────────────┘ |         | └──────────┬────────────┘ |
|           │               |         |            │              |
| ┌───────────────────────┐ |         | ┌──────────▼────────────┐ |
| │  PrivateChat.jsx      │ |  Real-  | │  Models                │ |
| │  - 1:1 Chat           │◄┼─Time    │ │  - ChatConversation    │ |
| └───────────────────────┘ | Events  | │  - ChatConversationPri.│ |
|                           |         | └──────────┬────────────┘ |
+---------------------------+         |            │              |
                                      | ┌──────────▼────────────┐ |
                                      | │  Events                │ |
                                      | │  - ChatEvent           │ |
                                      | │  - PrivateChatEvent    │ |
                                      | └──────────┬────────────┘ |
                                      |            │              |
                                      | ┌──────────▼────────────┐ |
                                      | │  Notifications         │ |
                                      | │  - ChatMessageNotif.   │ |
                                      | └───────────────────────┘ |
                                      +---------------------------+
```

### Data Flow Diagram

```
┌─────────────────┐  1.Send Message  ┌──────────────────┐
│ React Component │────────────────▶ │ ChatController   │
│ (CourseChat/    │                  │ (sendMessage/    │
│  PrivateChat)   │◀────────────────│ sendPrivateMsg)  │
└─────────────────┘  2.Response      └──────────┬───────┘
        ▲                                       │
        │                                       │ 3.Create
        │                                       ▼
        │                             ┌──────────────────┐
        │                             │ ChatMessage      │
        │                             │ (Database Model) │
        │                             └──────────┬───────┘
        │                                        │
        │                                        │ 4.Trigger
        │                                        ▼
        │                             ┌──────────────────┐
        │                             │ ChatEvent/       │
        │ 6.Real-time                 │ PrivateChatEvent │
        │   WebSocket                 └──────────┬───────┘
        │   Message                              │
        │                                        │ 5.Broadcast
        │                                        ▼
        │                             ┌──────────────────┐
        └─────────────────────────────│ Pusher Service   │
                                      │ (WebSockets)     │
                                      └──────────────────┘
```

## Technology Stack & Build Tools

### Build System
- **Laravel Mix:** Used for asset compilation in the project
- **Vite:** Configured with the `@vitejs/plugin-react` for fast React development
- **Node.js Polyfills:** Configured in Vite for compatibility with browser environment

### Frontend Framework
- **React 18:** Component-based UI library
- **React DOM:** DOM-specific methods for React
- **JSX/TSX:** Extension syntax for embedding XML within JavaScript

### Backend Framework
- **Laravel 10:** PHP MVC framework for web applications
- **PHP 8.1+:** Server-side scripting language

### Real-time Communication
- **Pusher:** WebSocket service for real-time messaging
- **Laravel Echo:** JavaScript library for WebSocket channel subscription

### File Storage & Media
- **Laravel Media Library (Spatie):** For file attachment management
  - Handles uploads, conversions, and thumbnail generation
  - Stores files in collections named 'attachments'

## API Routes and Endpoints

### Group Chat Endpoints
- **GET** `/web-chat-api/history/{courseId}/{groupId}` - Get chat history
- **GET** `/web-chat-api/participants/{courseId}/{groupId}` - Get participants
- **POST** `/web-chat-api/send` - Send a message to group chat

### Private Chat Endpoints
- **POST** `/web-chat-api/private/history` - Get private chat history
- **POST** `/web-chat-api/private/send` - Send a private message
- **POST** `/web-chat-api/private/participants` - Get private chat participants
- **GET** `/web-chat-api/private/unread-counts` - Count of unread messages
- **POST** `/web-chat-api/private/mark-read` - Mark notifications as read
- **POST** `/web-chat-api/private/notify-unread` - Send unread notification

## WebSocket Channels

### Channel Structure
- **Group Chat:** `course-chat.{courseId}.{groupId}`
- **Private Chat:** `private-chat.{courseId}.{groupId}.{studentId}.{instructorId}`
- **User Notifications:** `private-user.{userId}`
- **Unread Notifications:** `unread-notification.{courseId}.{groupId}`

### Events
- `message.sent` - New message event
- `private-message` - Private notification event
- `unread-message` - Unread counter update event

## Database Schema

```
┌─────────────────────┐      ┌───────────────────────┐
│ ChatConversation    │      │ ChatMessage           │
├─────────────────────┤      ├───────────────────────┤
│ id                  │      │ id                    │
│ conversation_type   │◀─────│ conversation_id       │
│ course_id           │      │ message               │
│ group_id            │      │ sender_id, sender_type│
└─────────────────────┘      │ receiver_id, type     │
         │                   │ attachment            │
         │                   └───────────────────────┘
         │
         ▼
┌─────────────────────┐
│ ChatConversationPri.│
├─────────────────────┤
│ id                  │
│ conversation_id     │
│ student_id          │
│ instructor_id       │
└─────────────────────┘
```

## Project File Structure

```
IPTC-PRO/
├── app/
│   ├── Events/
│   │   ├── ChatEvent.php               # Group chat broadcast event
│   │   └── PrivateChatEvent.php        # Private chat broadcast event
│   ├── Http/
│   │   └── Controllers/
│   │       └── Api/
│   │           └── ChatController.php  # Handles all chat API endpoints
│   ├── Models/
│   │   ├── ChatConversation.php        # Main conversation model
│   │   ├── ChatConversationPrivate.php # Private conversation details 
│   │   └── ChatMessage.php             # Message model with polymorphic relations
│   └── Notifications/
│       └── ChatMessageNotification.php # Notification handler for messages
├── database/
│   └── migrations/
│       ├── xxxx_create_chat_conversations_table.php
│       ├── xxxx_create_chat_messages_table.php
│       └── xxxx_create_chat_conversation_private_table.php
├── config/
│   ├── broadcasting.php              # WebSocket configuration
│   └── filesystems.php               # Media storage configuration
├── resources/
│   ├── js/
│   │   └── react/
│   │       ├── components/
│   │       │   └── course/
│   │       │       ├── CourseChat.jsx      # Group chat component
│   │       │       ├── PrivateChat.jsx     # Private chat component
│   │       │       ├── chatUtils.js        # Shared utilities
│   │       │       ├── course-chat.css     # Chat styling
│   │       │       └── private-chat.css    # Private chat styling
│   │       └── index.jsx                # React entry point
│   └── views/
│       └── Web/
│           └── course/
│               └── course-details.blade.php # Chat tab container
├── routes/
│   └── web-chat-api.php                # Chat API routes
├── public/
│   └── storage/                        # Public storage for message attachments
├── package.json                        # NPM dependencies
├── vite.config.js                      # Vite configuration for React
└── composer.json                      # PHP dependencies
```

## Key Technical Features

1. **Polymorphic Relationships**: For flexible sender/receiver models
2. **Real-time WebSockets**: Using Pusher for instant messaging
3. **Media Library Integration**: For file attachments
4. **Optimistic UI Updates**: Immediate message display before server confirmation
5. **Unread Message Tracking**: Notification counters for new messages
6. **Message Pagination**: Loading history in chunks for performance

