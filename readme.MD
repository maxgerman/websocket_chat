
# Websockets chat documentation

     

## General flow
Here is the desciption of the general flow, each of the steps is described in more detail after in the subtopics that follow. Lastly, there is working example page (html + js) useful for testing and reference.

Steps:
- open websocket by url containing *other_member_id* - the one you want to open the chat with
- authorize by sending the access_token in the first message
- if authorization is successful you will get the response with the chat history -- as the list of messages (json) in predefined format
- if auth is unsuccessful - websocket is closed
- history is sent back to you
- send/receive messages in real time to/from the open socket; if the other member is offline - messages are saved in the history and will be delivered as soon as the other member logins
- if any erros occur after authorization (incorrect msg format, file not found, etc.) - they will be sent to websocket as json with error description, web socket is NOT closed in this case
- closed tab/browser closes the socket

## url 
Use the url like this to open the websocket:
>	ws://localhost:8000/members/chat/?other_member_id={other_member_id}

## authorizaiton
Send the first message to WS as:
>Bearer *access_token*

Use the same access_token as with other parts of the site -- received after POST to login endpoint.

## chat history
Sent to websocket once you authorize - in the form of the list of the message objects. Each of the messages have the field 'type' (see message types). 

## message types
There are three types of messages: **text**, **smile**, **file**.
Which when received look like these:

-   {"id": 87, "member_id": 1, "chat_id": 2, "date": "2022-09-02T15:27:34.857355", "type": "text", "text": "Hello, world!"}
-   {"id": 88, "member_id": 1, "chat_id": 2, "date": "2022-09-02T15:27:49.783958", "type": "smile", "smile_id": 2, "smile_file_uuid": "b1d24593-9166-408a-bf68-a8f7be00e181"}
-   {"id": 89, "member_id": 1, "chat_id": 2, "date": "2022-09-02T15:29:23.598871", "type": "file", "file_uuid": "84d8a07a-67ca-4d92-8d1e-f5e646b72fb7", "text": "Here's the file!"}


## message sending
Each type has its format and has some requirements.
- ### text
Just send to the open websocket the message of explicit type or plain text which will be considered text format as well:
>{"type": "text", "text": "Hello, world!"}

Or
>"hey"

&nbsp;

- ### smile
Send to the open websocket:
> -   {"type": "smile", "smile_id": 2}

Smile must exist. To create a smile, upload any image and get its id:
: POST file as a superuser to */storage/smiles* (as 'form-data' with field name = 'file')
&nbsp;

-  ### file
Sending of files is a bit more complex as we cannot send them in json, nor do the websockets have forms like http.
In essence, frontend codes the needed file info + message (if any) and combines it with the file data in order to send the resulting byte stream to the socket in one go. The example JS code is included below. Full example is at the end.
Can be with or without text attached to them.
Size limitation checked at backed (default = 5 000 000 bytes)

    function sendFile() {
                var file = document.getElementById('filename').files[0];
                var reader = new FileReader();
                var rawData = new ArrayBuffer();            
                
                reader.onload = function(e) {
                    rawData = e.target.result;
                    //getting the file info JSON + add text message
                    var fileInfo = JSON.stringify({
                        name: file.name,
                        size: file.size,
                        type: file.type,
                        text: document.getElementById("messageText").value
                }); 
            
            	const enc = new TextEncoder();
                const buf1 = enc.encode('!');
                const buf2 = enc.encode(fileInfo);
                const buf3 = enc.encode('#');
                const buf4 = rawData; 
                
               let sendData = new Uint8Array(buf1.byteLength + buf2.byteLength + buf3.byteLength + buf4.byteLength);
               sendData.set(new Uint8Array(buf1), 0);
               sendData.set(new Uint8Array(buf2), buf1.byteLength);
               sendData.set(new Uint8Array(buf3), buf1.byteLength + buf2.byteLength);
               sendData.set(new Uint8Array(buf4), buf1.byteLength + buf2.byteLength + buf3.byteLength); 
               
               ws.binaryType = "arraybuffer";
               window.ws.send(sendData);
	           ws.binaryType = "blob"; 
                
               alert("the File has been transferred.")
            }
            reader.readAsArrayBuffer(file);
        }

## message pricing 
Ensure the  users have some balance for chat testing OR they're superusers/employees/premium users.
For ordinary users smiles have custom price for each, other message types have the  default price of 5 coins.


## Full example

    <!doctype html>
    <html>
        <head>
            <title>chat</title>
        </head>
        <body>
            <h1>websocket chat</h1>
            <form action="" onsubmit="openWs(event)">
                <label for="other_member_id">other_member_id: </label><br>  
                <input type="text" id="other_member_id" />
                <button>open websocket</button>
            </form>
            
           <form action="" onsubmit="sendFirstMessage(event)">
                <label for="messageText">First message (Bearer): </label><br>
                <input type="text" id="bearerText" autocomplete="off"/>
                <button>send</button>
            </form> 
            
           <input type="file" id="filename" /> 
           <input type="button" value="Get Info" onclick="getInfo()" />
           <input type="button" value="Upload" onclick="sendFile()" />
            
            <form action="" onsubmit="sendMessage(event)">
                <label for="messageText">Message: </label><br>
                <textarea id="messageText" rows="15" cols="50"></textarea>
                
                <br> 
                <button style="margin-bottom: 10px">send</button>

            </form>
            
            <form action="" onsubmit="clearMessages(event)">
                <button>Clear</button>
            </form>
            
            <ul id='messages'>
            </ul>
            
            
            <script>
                function sendMessage(event) {
                    var input = document.getElementById("messageText")
                    window.ws.send(input.value)
                    input.value = ''
                    event.preventDefault()
                }
                
               function sendFirstMessage(event) {
                    var input = document.getElementById("bearerText")
                    window.ws.send(input.value)
                    event.preventDefault()
                } 
                
               function clearMessages(event) {
                    var messages = document.getElementById('messages')
                    messages.innerHTML = ''
                    event.preventDefault()
                } 
                
                function openWs(event) {
                    
                    var other_member_id = document.querySelector("#other_member_id").value;
                    window.ws = new WebSocket(`ws://localhost:8000/members/chat/?other_member_id=${other_member_id}`);
                    
                   ws.onmessage = function(event) {
                    var messages = document.getElementById('messages')
                    var message = document.createElement('li')
                    var content = document.createTextNode(event.data)
                    message.appendChild(content)
                    messages.appendChild(message)
                    
                    ws.onclose = function() {
                    alert("Connection is closed...");
                };

                ws.onerror = function(e) {
                    alert(e.msg);
                } 
                    
                }; 
                    
                    
                    console.log(window.ws)
                    event.preventDefault()
                }
                
               function sendFile() {
                    var file = document.getElementById('filename').files[0];
                    var reader = new FileReader();
                    var rawData = new ArrayBuffer();            
                    
                    reader.onload = function(e) {
                        rawData = e.target.result;
                        //getting the file info JSON + add text message
                        var fileInfo = JSON.stringify({
                            name: file.name,
                            size: file.size,
                            type: file.type,
                            text: document.getElementById("messageText").value
                    }); 
                
                    const enc = new TextEncoder();
                    const buf1 = enc.encode('!');
                    const buf2 = enc.encode(fileInfo);
                    const buf3 = enc.encode('#');
                    const buf4 = rawData; 
                    
                   let sendData = new Uint8Array(buf1.byteLength + buf2.byteLength + buf3.byteLength + buf4.byteLength);
                   sendData.set(new Uint8Array(buf1), 0);
                   sendData.set(new Uint8Array(buf2), buf1.byteLength);
                   sendData.set(new Uint8Array(buf3), buf1.byteLength + buf2.byteLength);
                   sendData.set(new Uint8Array(buf4), buf1.byteLength + buf2.byteLength + buf3.byteLength); 
                   
                   ws.binaryType = "arraybuffer";
                   window.ws.send(sendData);
                   ws.binaryType = "blob"; 
                    
                   alert("the File has been transferred.")
                }
                reader.readAsArrayBuffer(file);
            } 
            
           function getInfo() { 
               var inputElement = document.getElementById('filename').files[0];

                var fileInfo = JSON.stringify({
                    name: inputElement.name,
                    size: inputElement.size,
                    type: inputElement.type,
                    text: document.getElementById("messageText").value
                });
                console.log(fileInfo);
                return fileInfo;
           } 
            </script>
        </body>
    </html>

     
## Questions
t.me/Max_G