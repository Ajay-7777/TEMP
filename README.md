const express = require('express');
const app = express();
const http = require('http').Server(app);
const io = require('socket.io')(http);
const path = require('path'); 
const Watcher = require('./watcher');

//app.use(express.static(path.join(__dirname, '/index.html')))


 
let watcher = new Watcher("newt.txt");

watcher.start();


app.get('/log', (req, res) => {
    console.log("request received");
    var options = {
        root: path.join(__dirname)
    };
     
    var fileName = 'index.html';
    res.sendFile(fileName, options, function (err) {
        if (err) {
            next(err);
        } else {
            console.log('Sent:', fileName);
        }
    });
})

io.on('connection', function(socket){
   // console.log(socket);
    console.log("new connection established:"+socket.id);

      watcher.on("process", function process(data) {
        socket.emit("update-log",data);
      });
      let data = watcher.getLogs();
      socket.emit("init",data);
   });

http.listen(3000, function(){
    console.log('listening on localhost:3000');
});



WATCHER.JS


const events = require("events");
const fs = require("fs");
//  const watchFile = "test.log";
const bf = require('buffer');
const TRAILING_LINES = 10;
const buffer = new Buffer.alloc(bf.constants.MAX_STRING_LENGTH);
  
  
class Watcher extends events.EventEmitter {
  constructor(watchFile) {
    super();
    this.watchFile = watchFile;
    this.store = [];
  }
  getLogs()
  {
      return this.store;
  }

  watch(curr,prev) {
    const watcher = this;
    fs.open(this.watchFile,(err,fd) => {
        if(err) throw err;
        let data = '';
        let logs = [];
        fs.read(fd,buffer,0,buffer.length,prev.size,(err,bytesRead) => {
            if(err) throw err;
            if(bytesRead > 0)
            {
                data = buffer.slice(0,bytesRead).toString();
                logs = data.split("\n").slice(1);
                console.log("logs read:"+logs);
                if(logs.length >= TRAILING_LINES)
                {
                    logs.slice(-10).forEach((elem) => this.store.push(elem));
                }
                else{
                    logs.forEach((elem) => {
                        if(this.store.length == TRAILING_LINES)
                        {
                            console.log("queue is full");
                            this.store.shift();
                        }
                        this.store.push(elem);
                    });
                }
                watcher.emit("process",logs);
            }
        });
    });
   
    }


  start() {
    var watcher = this;
    fs.open(this.watchFile,(err,fd) => {
        if(err) throw err;
        let data = '';
        let logs = [];
        fs.read(fd,buffer,0,buffer.length,0,(err,bytesRead) => {
            if(err) throw err;
            if(bytesRead > 0)
            {   
                data = buffer.slice(0,bytesRead).toString();
                logs = data.split("\n");
                this.store = [];
                logs.slice(-10).forEach((elem) => this.store.push(elem));
            }
            fs.close(fd);
            });
    fs.watchFile(this.watchFile,{"interval":1000}, function(curr,prev) {
        watcher.watch(curr,prev);
    });
  });
}
}
  
module.exports = Watcher;







<!DOCTYPE html>
<html>
   <head><title>app</title></head>
   <script src="/socket.io/socket.io.js"></script>
   <script>
      var socket = io("ws://localhost:3000");
      socket.on('update-log', function(data){
          console.log(data,"by");
          for(elem of data) document.getElementById('message-container').innerHTML +='<p>' + elem + '</br><p>';
      });
      socket.on('init',function(data){
        console.log(data,"hi");
        document.getElementById('message-container').innerHTML ='';
        for(elem of data) document.getElementById('message-container').innerHTML +='<p>' + elem + '</br><p>';      })
   </script>
   <body>
       <h1>Log monitoring app</h1>
      <div id="message-container"></div>
      </body>
   </html>
