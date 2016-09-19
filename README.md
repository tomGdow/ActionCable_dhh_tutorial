### Action Cable Demo (David Heinemeier Hansson tutorial)

The tutorial (*Rails 5: Action Cable demo* by *David Heinemeier Hansson*) is available at https://www.youtube.com/watch?v=n0WUjGkDFS0  

### Method

Developed locally from the above [tutorial](https://www.youtube.com/watch?v=n0WUjGkDFS0) on a vagrant virtual machine with Ubuntu/trusty64,
Rails 5.0.0.1 and  Ruby 2.3.1, as follows: 

    rails new campfire
    cd campfire
    bundle install
    rails g controller rooms show

> /config/routes.rb
    
    root to: 'rooms#show'

&hellip;   
 
    rails s -b 0.0.0.0
    localhost:3000 # browser

&hellip;
    
    rails g model message content:text
    rails db:migrate

> /app/controllers/rooms_controller.rb 

    def show
      @messages = Message.all
     end

create partial  

    cd app/views/
    mkdir messages && cd $_
    touch _message.html.erb

add  

    <div>
       <p><%= message.content %></p>
    </div>

> app/views/rooms/show.html.erb  

Delete all and add  

    <h1> Chat Room</h1>
    <div id="messages">
      <%= render @messages %>
    </div>

&hellip; 
    
    rails c
    Message.create! content: "hello world!"
    exit

&hellip; 
 
    rails s -b 0.0.0.0  
    localhost:3000 # browser

Generate channel  

    rails g channel room speak  

> /config/routes.rb

    mount ActionCable.server => '/cable'

> app/assets/javascripts/cable.js  

Check that the following is uncommented 

    (function() {
       this.App || (this.App = {});    
       App.cable = ActionCable.createConsumer();
    }).call(this);

> app/assets/javascripts/channels/room.coffee

    speak: (message) -> 
      @perform 'speak', message: message

> app/channels/room_channel.rb

    def subscribed
      stream_from "room_channel"
    end

    def speak(data)
      ActionCable.server.broadcast 'room_channel', message: data['message']
    end

> app/assets/javascripts/channels/room.coffee 

    received: (data) -> 
      alert data['message']

!! restart server  
Open app in two separate browsers  
In (Chrome) browser console: 

    App.room.speak('Hello World')

Both browsers get message instantaneously

&hellip;  

> app/views/rooms/show.html.erb  

add the following: 

    <form>
      <label>Say something </label> <br>
      <input type ="text" data-behavior="room_speaker">
    </form>

> app/assets/javascripts/channels/room.coffee  

add

    $(document).on 'keypress', '[data-behavior~=room_speaker]', (event) ->
      if event.keyCode is 13 # return = send
        App.room.speak event.target.value
        event.target.value = ''
        event.preventDefault()

&hellip;    

    rails s -b 0.0.0.0
    localhost:3000

Should get alert box as before, but now from input box

Now want to 'talk' to database

> app/channels/room_channel.rb  

Change 'def/speak' to the following: 
 
    def speak(data)
      Message.create! content: data['message']
    end

> /app/models/message.rb

    after_create_commit { MessageBroadcastJob.perform_later self }

&hellip; 

    rails g job MessageBroadcast

> app/jobs/message_broadcast_job.rb

    queue_as :default

    def perform(message)
      ActionCable.server.broadcast 'room_channel', message: render_message(message)
    end

    private
      def render_message(message)
        ApplicationController.renderer.render(partial: 'messages/message', locals: {message: message})
      end

> app/assets/javascripts/channels/room.coffee

    received: (data) -> 
      $('#messages').append data['message']

### Deployment

[Deployed](http://dhhaction.tomgdow.com) on [Digital Ocean](https://www.digitalocean.com/) with Puma as stand-alone server behind an Apache reverse proxy (see [here](https://www.phusionpassenger.com/library/deploy/standalone/reverse_proxy.html))

http://dhhaction.tomgdow.com

### Notes

**To use port 4000 instead of port 3000**
> config/environments/development.rb  

add the following line:

    config.action_cable.allowed_request_origins = ['http://localhost:4000'] 
  
The following will now work:  

    rails s -p 4000 -b 0.0.0.0

**To include caching**
> app/views/messages/_message.html.erb  

change to the following:

    <% cache message do %>
    <div class="message">
       <p><%= message.content %></p>
    </div>
    <% end %>

**Comment**  

The following piece of code

    <%= action_cable_meta_tag %>

is not present in:

> app/views/layouts/application.html.erb

Perhaps it should be included?
