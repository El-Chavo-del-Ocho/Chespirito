/**
 * +Chespirito - Final Ultra-Real
 * Streaming global realista + Offline + Push + Multi-dispositivo
 * Rodar: node chespirito-final-ultrareal.js
 * Acesse: http://localhost:5000
 */

const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const bodyParser = require("body-parser");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");
const webpush = require("web-push");

const app = express();
const PORT = 5000;
const JWT_SECRET = "+CHESPIRITO_FINAL_SECRET_ULTRA";

// Configuração Web Push (chaves geradas previamente)
const PUBLIC_VAPID_KEY = "BOLANOME_EXEMPLO_PUBLIC";
const PRIVATE_VAPID_KEY = "BOLANOME_EXEMPLO_PRIVATE";
webpush.setVapidDetails("mailto:admin@pluschespirito.com", PUBLIC_VAPID_KEY, PRIVATE_VAPID_KEY);

app.use(cors());
app.use(bodyParser.json());

// ----------------- MongoDB -----------------
mongoose.connect("mongodb://127.0.0.1:27017/plus_chespirito_final", {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(()=>console.log("MongoDB conectado"))
  .catch(err=>console.log(err));

// ----------------- Models -----------------
const UserSchema = new mongoose.Schema({
  username: String,
  password: String,
  role: { type: String, default: "user" },
  favorites: [{ type: mongoose.Schema.Types.ObjectId, ref: "Series" }],
  history: [{ seriesId: mongoose.Schema.Types.ObjectId, episodeTitle: String, watchedAt: Date }],
  pushSubscription: Object // Para push notifications
});
const User = mongoose.model("User", UserSchema);

const SeriesSchema = new mongoose.Schema({
  title: String,
  description: String,
  episodes: [{ 
    title: String,
    videoUrls: { "360p": String, "720p": String, "1080p": String },
    subtitles: { "pt": String, "en": String, "es": String },
    duration: Number
  }],
  image: String,
  genre: [String],
  releaseYear: Number,
  rating: { type: Number, default: 0 },
  languages: [String],
});
const Series = mongoose.model("Series", SeriesSchema);

const ChatSchema = new mongoose.Schema({
  username: String,
  message: String,
  likes: { type: Number, default: 0 },
  timestamp: { type: Date, default: Date.now }
});
const Chat = mongoose.model("Chat", ChatSchema);

// ----------------- Auth -----------------
app.post("/api/register", async (req,res)=>{
  const { username, password } = req.body;
  const hashed = await bcrypt.hash(password,10);
  const user = await User.create({ username, password: hashed });
  const token = jwt.sign({ id: user._id, username, role: user.role }, JWT_SECRET);
  res.json({ token });
});

app.post("/api/login", async (req,res)=>{
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if(!user) return res.status(400).json({ msg:"Usuário não encontrado" });
  const isMatch = await bcrypt.compare(password,user.password);
  if(!isMatch) return res.status(400).json({ msg:"Senha incorreta" });
  const token = jwt.sign({ id: user._id, username, role: user.role }, JWT_SECRET);
  res.json({ token });
});

// ----------------- JWT -----------------
const authMiddleware = (req,res,next)=>{
  const token = req.headers["authorization"];
  if(!token) return res.status(401).json({ msg:"Token faltando" });
  try{
    const decoded = jwt.verify(token,JWT_SECRET);
    req.user = decoded;
    next();
  } catch(e){
    return res.status(401).json({ msg:"Token inválido" });
  }
};

// ----------------- Series -----------------
app.get("/api/series", async (req,res)=>{ const series = await Series.find(); res.json(series); });
app.get("/api/series/:id", async (req,res)=>{ const series = await Series.findById(req.params.id); res.json(series); });
app.post("/api/series", authMiddleware, async (req,res)=>{
  if(req.user.role!=="admin") return res.status(403).json({ msg:"Acesso negado" });
  const series = await Series.create(req.body);
  res.json(series);
});

// ----------------- Recommendations -----------------
app.get("/api/recommendations/:userid", authMiddleware, async (req,res)=>{
  const user = await User.findById(req.params.userid).populate("history.seriesId");
  const watchedGenres = user.history.map(h=>{
    const s = user.history.find(x=>x.seriesId);
    return s ? s.seriesId.genre : [];
  }).flat();
  const recommended = await Series.find({ genre: { $in: watchedGenres } }).limit(5);
  res.json(recommended);
});

// ----------------- Chat -----------------
app.get("/api/chat", async (req,res)=>{ const messages = await Chat.find().sort({timestamp:1}).limit(50); res.json(messages); });
app.post("/api/chat", authMiddleware, async (req,res)=>{
  const { message } = req.body;
  const chatMsg = await Chat.create({ username:req.user.username, message });
  res.json(chatMsg);
});
app.post("/api/chat/like/:id", authMiddleware, async (req,res)=>{
  const chatMsg = await Chat.findById(req.params.id);
  chatMsg.likes += 1;
  await chatMsg.save();
  res.json(chatMsg);
});

// ----------------- Trending -----------------
app.get("/api/trending", async (req,res)=>{
  const trending = await Series.find().sort({rating:-1}).limit(5);
  res.json(trending);
});

// ----------------- Offline / Push -----------------
app.get("/api/offline/:seriesId", authMiddleware, async (req,res)=>{
  const series = await Series.findById(req.params.seriesId);
  res.json({ message:"Offline real simulado. Vídeos e legendas armazenados localmente.", series });
});

app.post("/api/subscribe", authMiddleware, async (req,res)=>{
  const subscription = req.body;
  const user = await User.findById(req.user.id);
  user.pushSubscription = subscription;
  await user.save();
  res.json({ message:"Inscrito para notificações push!" });
});

app.post("/api/push", authMiddleware, async (req,res)=>{
  const { title, message } = req.body;
  const users = await User.find({ pushSubscription: { $ne: null } });
  users.forEach(u=>{
    webpush.sendNotification(u.pushSubscription, JSON.stringify({ title, message })).catch(err=>console.error(err));
  });
  res.json({ message:"Notificações enviadas!" });
});

// ----------------- Frontend React PWA -----------------
app.get("/", (req,res)=>{
res.send(`<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>+Chespirito Ultra-Real</title>
<script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
<script crossorigin src="https://unpkg.com/axios/dist/axios.min.js"></script>
<style>
body{margin:0;padding:0;font-family:sans-serif;background:#f0f0f0;}
header{background:#ff6f61;color:white;padding:1rem;text-align:center;}
.container{max-width:1200px;margin:auto;padding:1rem;}
.series-card{background:white;padding:1rem;margin:1rem 0;border-radius:10px;box-shadow:0 2px 6px rgba(0,0,0,0.1);}
video{width:100%;border-radius:10px;}
button{margin:5px;padding:0.5rem 1rem;border:none;border-radius:5px;background:#ff6f61;color:white;cursor:pointer;}
.chat{background:#fff;padding:1rem;margin:1rem 0;border-radius:10px;height:200px;overflow-y:scroll;}
.chat input{width:70%;padding:0.5rem;margin-right:0.5rem;}
</style>
</head>
<body>
<header><h1>+Chespirito Ultra-Real Global</h1></header>
<div id="root" class="container"></div>
<script>
const e = React.createElement;

function App(){
  const [series,setSeries]=React.useState([]);
  const [token,setToken]=React.useState("");
  const [username,setUsername]=React.useState("");
  const [password,setPassword]=React.useState("");
  const [userId,setUserId]=React.useState("");
  const [chatMsgs,setChatMsgs]=React.useState([]);
  const [newMsg,setNewMsg]=React.useState("");
  const [lang,setLang]=React.useState("pt");

  React.useEffect(()=>{
    if(token) fetchSeries();
    fetchChat();
    const interval = setInterval(fetchChat,5000);
    if('serviceWorker' in navigator){ navigator.serviceWorker.register('/sw.js'); }
    return ()=>clearInterval(interval);
  },[token]);

  function login(){ axios.post('/api/login',{username,password}).then(res=>{
    setToken(res.data.token);
    const payload = JSON.parse(atob(res.data.token.split(".")[1]));
    setUserId(payload.id);
  }).catch(err=>alert(err.response.data.msg)); }

  function fetchSeries(){ axios.get('/api/series').then(res=>setSeries(res.data)); }
  function fetchRecommendations(){ axios.get('/api/recommendations/'+userId,{headers:{authorization:token}}).then(res=>alert('Recomendações: '+res.data.map(s=>s.title).join(", "))); }
  function fetchTrending(){ axios.get('/api/trending').then(res=>alert('Trending: '+res.data.map(s=>s.title).join(", "))); }
  function fetchChat(){ axios.get('/api/chat').then(res=>setChatMsgs(res.data)); }
  function sendChat(){ if(!newMsg) return; axios.post('/api/chat',{message:newMsg},{headers:{authorization:token}}).then(res=>{ setNewMsg("");