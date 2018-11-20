# Sử dụng Pusher trong Laravel để tạo ứng dụng chat

Chức năng live chat gần như là một chức năng cần thiết và không thể thiếu trong mỗi ứng dụng web. Để xây dựng chức năng chat cho ứng dụng web thì không hề khó. Ngày nay, cũng có vô vàn thư viện hỗ trợ những lập trình viên thực hiện điều này. Trong bài viết này, tôi xin giới thiệu tới bạn đọc cách xây dựng một ứng dụng real chat bằng **Pusher** được tích hợp sẵn trong framework Laravel.

**Tạo project mới**

Trước tiên, bạn hãy tạo một project laravel mới. Sau khi tạo được project mới, việc tiếp theo cần làm trước đó là register thư viện _App\Providers\BroadcastServiceProvider_. Để làm được điều này, hãy  mở file _config/app.php_ và bỏ comment dòng

```
// App\Providers\BroadcastServiceProvider
```
trong array **providers**

Cần khai báo trong file _.env_ để cho Laravel hiểu được chúng ta sẽ gọi **Pusher**

```
// .env

BROADCAST_DRIVER=pusher
```

**Setting up Pusher**

Trước hết, hãy tạo một Pusher account trên trang [https://pusher.com/signup](https://pusher.com/signup) nếu bạn chưa có tài khoản của Pusher. Việc đăng ký này là hoàn toàn miễn phí, vì vậy bạn không cần phải e ngại bất cứ điều gì nhé.

Thay đổi đoạn config sau trong file 'config/broadcasting.php':

```
/ Don't add your credentials here!
// config/broadcasting.php

'pusher' => [
  'driver' => 'pusher',
  'key' => env('PUSHER_APP_KEY'),
  'secret' => env('PUSHER_APP_SECRET'),
  'app_id' => env('PUSHER_APP_ID'),
  'options' => [],
],
```
bằng đoạn code sau:

```
'pusher' => [
      'driver' => 'pusher',
      'key' => env('PUSHER_APP_KEY'),
      'secret' => env('PUSHER_APP_SECRET'),
      'app_id' => env('PUSHER_APP_ID'),
      'options' => [
          'cluster' => env('PUSHER_CLUSTER'),
          'encrypted' => true,
      ],
  ],
```

Các bạn cũng có thể thấy là trong file config này chúng ta đang đọc thông tin từ file .env vì vậy hãy thêm đoạn config này vào file .env như sau:

```
// .env

PUSHER_APP_ID=xxxxxx
PUSHER_APP_KEY=xxxxxxxxxxxxxxxxxxxx
PUSHER_APP_SECRET=xxxxxxxxxxxxxxxxxxxx
PUSHER_CLUSTER=xx
```

Chú ý rằng nhưng thông tin này bạn có thể lấy được ở tabs **Overview** khi bạn login vào account của mình.

Cài đặt các package cần thiết qua lệnh npm

```
npm install

npm install --save laravel-echo pusher-js
```

Mở file _resources/assets/js/bootstrap.js_ thêm vào cuối file đoạn code sau:

```
// resources/assets/js/bootstrap.js

import Echo from "laravel-echo"

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'xxxxxxxxxxxxxxxxxxxx',
    cluster: 'eu',
    encrypted: true
});
```

Chú ý đổi giá trị "xxxxxxxxxxxxxxxxxxxx" thành giá trị pusher app key mà bạn đã đăng ký ở bước bên trên nhé.

**AUTHENTICATING USERS**

Việc authenticate user trong laravel thì vô cùng đơn giản, bạn chỉ cần chạy lệnh

```
php artisan make:auth
```

Config trong file .env để project của bạn có thể kết nối với cơ sở dữ liệu mysql

```
// .env

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel-chat
DB_USERNAME=root
DB_PASSWORD=root
```

Hãy điền thông tin của cơ sở dữ liệu tương ứng với cái bạn đang cài trên máy.

Chạy migrate mới nhất:

```
php artisan migrate
```

**MESSAGE MODEL AND MIGRATION**

Tạo **Message** modal với file migration của nó bằng lệnh

```
php artisan make:model Message -m
```

Mở *Message* modal và thêm đoạn code sau:

```
// app/Message.php

/**
 * Fields that are mass assignable
 *
 * @var array
 */
protected $fillable = ['message'];
```

Mở file migration của bảng messages và thêm đoạn code;

```
Schema::create('messages', function (Blueprint $table) {
  $table->increments('id');
  $table->integer('user_id')->unsigned();
  $table->text('message');
  $table->timestamps();
});
```

Cập nhật bảng messages vào trong cơ sở dữ liệu bằng lệnh:

```
php artisan migrate
```

**USER TO MESSAGE RELATIONSHIP**

Thêm những đoạn code sau vào các file tương ứng:

```
// app/User.php

/**
 * A user can have many messages
 *
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
public function messages()
{
  return $this->hasMany(Message::class);
}

```

Trong migration

```
// app/Message.php

/**
 * A message belong to a user
 *
 * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
 */
public function user()
{
  return $this->belongsTo(User::class);
}
```
Định nghĩa route cho ứng dụng của chúng ta

```
// routes/web.php

Auth::routes();

Route::get('/', 'ChatsController@index');
Route::get('messages', 'ChatsController@fetchMessages');
Route::post('messages', 'ChatsController@sendMessage');
```

**Controller**

Tạo mới ChatsController với câu lệnh

```
php artisan make:controller ChatsController
```

Mở file vừa được tạo và thêm những dòng code sau:

```
// app/Http/Controllers/ChatsController.php

use App\Message;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

public function __construct()
{
  $this->middleware('auth');
}

/**
 * Show chats
 *
 * @return \Illuminate\Http\Response
 */
public function index()
{
  return view('chat');
}

/**
 * Fetch all messages
 *
 * @return Message
 */
public function fetchMessages()
{
  return Message::with('user')->get();
}

/**
 * Persist message to database
 *
 * @param  Request $request
 * @return Response
 */
public function sendMessage(Request $request)
{
  $user = Auth::user();

  $message = $user->messages()->create([
    'message' => $request->input('message')
  ]);

  return ['status' => 'Message Sent!'];
}
```

**Views**

Tạo view cho ứng dụng trong file _resources/views/chat.blade.php_

```
<!-- resources/views/chat.blade.php -->

@extends('layouts.app')

@section('content')

<div class="container">
    <div class="row">
        <div class="col-md-8 col-md-offset-2">
            <div class="panel panel-default">
                <div class="panel-heading">Chats</div>

                <div class="panel-body">
                    <chat-messages :messages="messages"></chat-messages>
                </div>
                <div class="panel-footer">
                    <chat-form
                        v-on:messagesent="addMessage"
                        :user="{{ Auth::user() }}"
                    ></chat-form>
                </div>
            </div>
        </div>
    </div>
</div>
@endsection
```
Ở đây chúng ta extends từ layouts app.blade.php. Vì vậy thêm đoạn code sau vào app.blade.php

```
<!-- resources/views/layouts/app.blade.php -->

<style>
  .chat {
    list-style: none;
    margin: 0;
    padding: 0;
  }

  .chat li {
    margin-bottom: 10px;
    padding-bottom: 5px;
    border-bottom: 1px dotted #B3A9A9;
  }

  .chat li .chat-body p {
    margin: 0;
    color: #777777;
  }

  .panel-body {
    overflow-y: scroll;
    height: 350px;
  }

  ::-webkit-scrollbar-track {
    -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,0.3);
    background-color: #F5F5F5;
  }

  ::-webkit-scrollbar {
    width: 12px;
    background-color: #F5F5F5;
  }

  ::-webkit-scrollbar-thumb {
    -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,.3);
    background-color: #555;
  }
</style>
```

Ngoài ra, hãy để ý rằng chúng ta còn gọi tới một component ChatMessages.vue. Hãy register component đó như sau:

```
// resources/assets/js/components/ChatMessages.vue

<template>
    <ul class="chat">
        <li class="left clearfix" v-for="message in messages">
            <div class="chat-body clearfix">
                <div class="header">
                    <strong class="primary-font">
                        {{ message.user.name }}
                    </strong>
                </div>
                <p>
                    {{ message.message }}
                </p>
            </div>
        </li>
    </ul>
</template>

<script>
  export default {
    props: ['messages']
  };
</script>
```

Thêm component ChatForm

```
// resources/assets/js/components/ChatForm.vue

<template>
    <div class="input-group">
        <input id="btn-input" type="text" name="message" class="form-control input-sm" placeholder="Type your message here..." v-model="newMessage" @keyup.enter="sendMessage">

        <span class="input-group-btn">
            <button class="btn btn-primary btn-sm" id="btn-chat" @click="sendMessage">
                Send
            </button>
        </span>
    </div>
</template>

<script>
    export default {
        props: ['user'],

        data() {
            return {
                newMessage: ''
            }
        },

        methods: {
            sendMessage() {
                this.$emit('messagesent', {
                    user: this.user,
                    message: this.newMessage
                });

                this.newMessage = ''
            }
        }    
    }
</script>
```

Cuối cùng là update lại trong app.js

```
// resources/assets/js/app.js

require('./bootstrap');

Vue.component('chat-messages', require('./components/ChatMessages.vue'));
Vue.component('chat-form', require('./components/ChatForm.vue'));

const app = new Vue({
    el: '#app',

    data: {
        messages: []
    },

    created() {
        this.fetchMessages();
    },

    methods: {
        fetchMessages() {
            axios.get('/messages').then(response => {
                this.messages = response.data;
            });
        },

        addMessage(message) {
            this.messages.push(message);

            axios.post('/messages', message).then(response => {
              console.log(response.data);
            });
        }
    }
});
```

**MESSAGE SENT EVENT**

Tạo một envent mới với tên MessageSent bằng lệnh

```
php artisan make:event MessageSent
```

Mở file vừa được tạo và thêm dòng code

```
// app/Events/MessageSent.php

use App\User;
use App\Message;
use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * User that sent the message
     *
     * @var User
     */
    public $user;

    /**
     * Message details
     *
     * @var Message
     */
    public $message;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(User $user, Message $message)
    {
        $this->user = $user;
        $this->message = $message;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('chat');
    }
}
```

Trong ChatController update một số dòng code như sau:

```
// app/Http/Controllers/ChatsController.php

//remember to use
use App\Events\MessageSent;

/**
 * Persist message to database
 *
 * @param  Request $request
 * @return Response
 */
public function sendMessage(Request $request)
{
  $user = Auth::user();

  $message = $user->messages()->create([
    'message' => $request->input('message')
  ]);

  broadcast(new MessageSent($user, $message))->toOthers();

  return ['status' => 'Message Sent!'];
}
```

Trong routes thì khai báo thêm route cho Chanel như sau

```
Broadcast::channel('chat', function ($user) {
  return Auth::check();
});
```
Lắng nghe sự kiện từ Event bằng đoạn code

```
// resources/assets/js/app.js

Echo.private('chat')
  .listen('MessageSent', (e) => {
    this.messages.push({
      message: e.message.message,
      user: e.user
    });
  });
```

Chạy 

```
npm run dev

php artisan serve

```

Rồi th
