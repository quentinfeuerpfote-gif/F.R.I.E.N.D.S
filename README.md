# F.R.I.E.N.D.S
mkdir hobby-backend
cd hobby-backend
npm init -y
npm install express mongoose bcrypt jsonwebtoken multer cors socket.io
const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");
const multer = require("multer");
const cors = require("cors");
const http = require("http");
const { Server } = require("socket.io");

const app = express();
app.use(express.json());
app.use(cors());
app.use("/uploads", express.static("uploads"));

const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

mongoose.connect("mongodb://localhost:27017/hobbyApp", { useNewUrlParser:true, useUnifiedTopology:true })
.then(()=>console.log("MongoDB verbunden")).catch(err=>console.log(err));

const userSchema = new mongoose.Schema({username:String,email:String,password:String,profilePic:String,description:String});
const User = mongoose.model("User",userSchema);

const groupSchema = new mongoose.Schema({
  name:String,
  image:String,
  members:[{type:mongoose.Schema.Types.ObjectId,ref:"User"}],
  admins:[{type:mongoose.Schema.Types.ObjectId,ref:"User"}],
  tags:[String]
});
const Group = mongoose.model("Group",groupSchema);

const messageSchema = new mongoose.Schema({
  groupId:{type:mongoose.Schema.Types.ObjectId,ref:"Group"},
  senderId:{type:mongoose.Schema.Types.ObjectId,ref:"User"},
  text:String,
  type:{type:String,default:"text"},
  timestamp:{type:Date,default:Date.now}
});
const Message = mongoose.model("Message",messageSchema);

const chatBgSchema = new mongoose.Schema({userId:{type:mongoose.Schema.Types.ObjectId,ref:"User"},groupId:{type:mongoose.Schema.Types.ObjectId,ref:"Group"},bgImage:String});
const ChatBackground = mongoose.model("ChatBackground",chatBgSchema);

const storage = multer.diskStorage({
  destination:(req,file,cb)=>cb(null,"uploads/"),
  filename:(req,file,cb)=>cb(null,Date.now()+"-"+file.originalname)
});
const upload = multer({ storage });

// Routen: Registrierung, Login, Gruppen, Nachrichten, Profile, Chat-Hintergrund
app.post("/register", async (req,res)=>{
  const hashed = await bcrypt.hash(req.body.password,10);
  const user = new User({...req.body,password:hashed});
  await user.save();
  res.send({message:"User created"});
});

app.post("/login",async(req,res)=>{
  const user = await User.findOne({email:req.body.email});
  if(!user) return res.status(400).send("User not found");
  const valid = await bcrypt.compare(req.body.password,user.password);
  if(!valid) return res.status(400).send("Wrong password");
  const token = jwt.sign({id:user._id},"SECRET_KEY");
  res.send({token,userId:user._id,username:user.username});
});

app.post("/groups", upload.single("image"), async (req,res)=>{
  const { name, userId, tags } = req.body;
  const group = new Group({
    name,
    image: req.file ? req.file.path : "",
    members:[userId],
    admins:[userId],
    tags: tags ? tags.split(",").map(t=>t.trim()) : []
  });
  await group.save();
  res.send(group);
});

app.get("/groups/search", async (req,res)=>{
  const { q } = req.query;
  const groups = await Group.find({name:{$regex:q,$options:"i"}});
  res.send(groups);
});

app.get("/groups/search-tags", async(req,res)=>{
  const { hobby } = req.query;
  const groups = await Group.find({tags:{$regex:hobby,$options:"i"}});
  res.send(groups);
});

app.post("/messages", async(req,res)=>{
  const msg = new Message(req.body);
  await msg.save();
  res.send(msg);
});

app.get("/messages/:groupId", async(req,res)=>{
  const messages = await Message.find({groupId:req.params.groupId}).populate("senderId","username profilePic");
  res.send(messages);
});

app.post("/profile/:userId", upload.single("profilePic"), async(req,res)=>{
  const update = {username:req.body.username,description:req.body.description};
  if(req.file) update.profilePic = req.file.path;
  const user = await User.findByIdAndUpdate(req.params.userId,update,{new:true});
  res.send(user);
});

app.post("/chat-bg", upload.single("bgImage"), async(req,res)=>{
  const { userId, groupId } = req.body;
  let record = await ChatBackground.findOne({userId,groupId});
  if(record){ record.bgImage=req.file.path; await record.save(); } 
  else { record = new ChatBackground({userId,groupId,bgImage:req.file.path}); await record.save(); }
  res.send(record);
});

app.get("/chat-bg/:userId/:groupId", async(req,res)=>{
  const record = await ChatBackground.findOne({userId:req.params.userId,groupId:req.params.groupId});
  res.send(record||{});
});

// Socket.IO Echtzeit-Chat
io.on("connection",(socket)=>{
  console.log("User connected:",socket.id);

  socket.on("joinGroup",(groupId)=>{socket.join(groupId); console.log(socket.id,"joined",groupId);});
  socket.on("sendMessage",async(data)=>{
    const { groupId, senderId, text, type } = data;
    const message = new Message({groupId,senderId,text,type:type||"text"});
    await message.save();
    io.to(groupId).emit("newMessage",await message.populate("senderId","username profilePic"));
  });

  socket.on("disconnect",()=>{console.log("User disconnected:",socket.id);});
});

server.listen(3000,()=>console.log("Server läuft auf Port 3000"));
node server.js
npx expo start

