# MoJi - Project Architecture & Documentation

Tài liệu kiến trúc tổng quan của dự án **MoJi** (chat application với realtime messaging).

---

## 1. Cây thư mục cấp cao

Root project có 2 ứng dụng chính:

- **D:/MoJi/frontend**
- **D:/MoJi/backend**

Cả hai thư mục đều có file `.gitignore`.

### Frontend (`D:/MoJi/frontend`)
- **Core**: `src/main.tsx`, `src/App.tsx`, `src/pages/*`
- **State**: `src/stores/*` (Zustand)
- **API layer**: `src/services/*`, `src/lib/axios.ts`
- **UI**: `src/components/*` (auth/chat/sidebar/profile/ui/skeleton/...)
- **Type contracts**: `src/types/*`
- **Build/config**: `vite.config.ts`, `tsconfig*.json`, `tailwind.config.ts`, `eslint.config.js`, `.env.*`

### Backend (`D:/MoJi/backend`)
- **Entry**: `src/server.js`
- **HTTP layers**: `src/routes/*`, `src/controllers/*`, `src/middlewares/*`
- **Data**: `src/models/*`, `src/libs/db.js`
- **Realtime**: `src/socket/index.js`
- **Helpers/docs**: `src/utils/messageHelper.js`, `src/swagger.json`

---

## 2. Frontend Architecture

### Entry + App Bootstrap
- React mount tại `D:/MoJi/frontend/src/main.tsx`.
- Router + global toaster + socket connect lifecycle tại `D:/MoJi/frontend/src/App.tsx`.

### Routing
- **Public**: `/signin`, `/signup`
- **Private**: `/` qua `ProtectedRoute` (`D:/MoJi/frontend/src/components/auth/ProtectedRoute.tsx`)

### Stores (Zustand)
- `useAuthStore` (`src/stores/useAuthStore.ts`): signin/signup/signout/refresh/fetchMe + persist user
- `useChatStore` (`src/stores/useChatStore.ts`): conversations, messages, markSeen, create/send message
- `useSocketStore` (`src/stores/useSocketStore.ts`): socket lifecycle + event listeners
- `useFriendStore`, `useUserStore`, `useThemeStore`: friend/profile/theme

### Services
- Auth API: `src/services/authService.ts`
- Chat API: `src/services/chatService.ts`
- Friend API: `src/services/friendService.ts`
- User API: `src/services/userService.ts`
- Axios + interceptor refresh token: `src/lib/axios.ts`

### Components quan trọng
- Auth forms: `src/components/auth/signin-form.tsx`, `signup-form.tsx`
- Shell chính: `src/components/sidebar/app-sidebar.tsx`
- Chat layout/body/input: `src/components/chat/ChatWindowLayout.tsx`, `ChatWindowBody.tsx`, `MessageInput.tsx`

---

## 3. Backend Architecture

### Server Entry
- `D:/MoJi/backend/src/server.js`: load env, middleware, routes, swagger, DB connect, start server (qua socket server).

### Routes
- Auth: `src/routes/authRoute.js`
- Users: `src/routes/userRoute.js`
- Friends: `src/routes/friendRoute.js`
- Messages: `src/routes/messageRoute.js`
- Conversations: `src/routes/conversationRoute.js`

### Controllers
- Auth: `src/controllers/authController.js`
- User/Friend/Message/Conversation tương ứng trong `src/controllers/*`

### Middlewares
- JWT route guard: `src/middlewares/authMiddleware.js`
- Socket auth: `src/middlewares/socketMiddleware.js`
- Friendship/group checks: `src/middlewares/friendMiddleware.js`
- Upload: `src/middlewares/uploadMiddleware.js`

### Models
- User, Session, Friend, FriendRequest, Conversation, Message trong `src/models/*`

### Socket Server
- `D:/MoJi/backend/src/socket/index.js`: online users, join rooms (conversation/user), broadcast events.

---

## 4. Luồng Auth End-to-End (signin/signup/signout/refresh/fetch me)

### Signup
- FE gọi `POST /auth/signup` qua `authService.signUp`.
- Store `useAuthStore.signUp` xử lý toast success/error.
- BE validate + hash bcrypt + tạo user.

### Signin
- FE form gọi `useAuthStore.signIn`.
- Store: clear state → `/auth/signin` → set accessToken → `fetchMe()` → preload conversations.
- BE trả:
  - `accessToken` trong JSON body
  - `refreshToken` trong cookie (httpOnly, secure, sameSite=none)

### Fetch Me
- FE: `GET /users/me` với `Authorization: Bearer` (từ axios interceptor).
- BE: lấy `req.user` từ protected middleware.

### Refresh Token
- FE có 2 nhánh:
  1. Chủ động ở `ProtectedRoute` khi chưa có access token.
  2. Tự động qua axios response interceptor (403 → retry tối đa 4 lần).
- BE: `POST /auth/refresh` đọc cookie, check DB Session, trả access token mới.

### Signout
- FE gọi `/auth/signout` + clear local store.
- BE xóa Session theo refresh token + clear cookie.

**Token/Cookie hiện tại**  
- **Access token**: JWT, TTL 30 phút, dùng trong header `Bearer`.  
- **Refresh token**: random string lưu trong DB (Session) + cookie httpOnly.

---

## 5. Luồng Chat/Realtime End-to-End

### Socket Connect
- FE connect khi có accessToken (trong `App.tsx` → `useSocketStore.connectSocket()`).
- BE verify token qua socket middleware, lưu online user map, join rooms.

### Conversations/Messages
- FE fetch qua REST (`chatService`, `useChatStore`).
- Gửi tin:
  - Direct: `POST /messages/direct`
  - Group: `POST /messages/group`
- BE cập nhật conversation + unread + emit `new-message`.

### Seen
- FE: `markAsSeen` khi mở conversation.
- BE: patch seen + emit `read-message`.

### Group Creation
- FE: `createConversation` → emit join room.
- BE: tạo conversation + emit `new-group` tới user rooms.

### Socket Events chính
- `online-users`
- `new-message`
- `read-message`
- `new-group`
- `join-conversation`

---

## 6. Dependencies quan trọng 

### Frontend deps chính (`D:/MoJi/frontend/package.json`)
- React + Vite + TypeScript
- Zustand, Axios, React Router, Socket.io-client
- Sonner (toast), Zod, React Hook Form
- Radix UI + Tailwind stack

### Backend deps chính (`D:/MoJi/backend/package.json`)
- Express, Mongoose, JsonWebToken, Bcrypt
- Cookie-parser, Cors, Socket.io
- Multer, Cloudinary, Dotenv

---

