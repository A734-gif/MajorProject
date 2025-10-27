# MajorProject
# ===========================================
# 🎯 BACKEND: FastAPI + SQLAlchemy + JWT Auth
# ===========================================

# --- main.py ---
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from auth import router as auth_router
from ai_engine import chatbot, recommender, doubt_resolver, voice_to_text
from integrations import google_drive, topperrank_lms
from utils.deadline_tracker import check_deadlines

app = FastAPI(title="Topper AI Mentor", version="1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth_router, prefix="/auth", tags=["Authentication"])

@app.get("/")
def home():
    return {"message": "Welcome to Topper AI Mentor!"}

@app.post("/chatbot")
def ask_question(query: str):
    response = chatbot.get_response(query)
    return {"query": query, "response": response}

@app.get("/recommendations/{student_id}")
def get_recommendations(student_id: int):
    recommendations = recommender.suggest_topics(student_id)
    return {"student_id": student_id, "recommendations": recommendations}

@app.post("/resolve_doubt")
def resolve_doubt(query: str):
    return {"query": query, "answer": doubt_resolver.resolve(query)}

@app.get("/deadlines/{student_id}")
def deadlines(student_id: int):
    return check_deadlines(student_id)

@app.post("/voice_to_text")
def voice_to_text_api(audio_file: str):
    text = voice_to_text.convert(audio_file)
    return {"transcript": text}

@app.get("/drive/files")
def list_drive_files():
    return google_drive.list_files()

@app.get("/lms/courses/{student_id}")
def get_courses(student_id: int):
    return topperrank_lms.get_courses(student_id)


# --- database.py ---
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./topper_ai.db"

engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


# --- models.py ---
from sqlalchemy import Column, Integer, String
from database import Base

class Student(Base):
    __tablename__ = "students"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    email = Column(String, unique=True)
    password = Column(String)
    interests = Column(String)


# --- auth.py ---
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from jose import jwt
import bcrypt

router = APIRouter()

SECRET_KEY = "topper_secret"
ALGORITHM = "HS256"

class LoginRequest(BaseModel):
    email: str
    password: str

fake_user_db = {"student@topperrank.com": {"password": bcrypt.hashpw(b"1234", bcrypt.gensalt())}}

@router.post("/login")
def login(request: LoginRequest):
    user = fake_user_db.get(request.email)
    if not user or not bcrypt.checkpw(request.password.encode(), user["password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    token = jwt.encode({"email": request.email}, SECRET_KEY, algorithm=ALGORITHM)
    return {"access_token": token, "token_type": "bearer"}


# --- ai_engine/chatbot.py ---
from transformers import pipeline

qa_pipeline = pipeline("question-answering", model="distilbert-base-cased-distilled-squad")

def get_response(query: str):
    context = """
    TopperRank is an academic platform offering Data Science, Cyber Security, and App Development courses.
    """
    result = qa_pipeline(question=query, context=context)
    return result["answer"]


# --- ai_engine/recommender.py ---
def suggest_topics(student_id: int):
    topics = ["Machine Learning", "React Native", "Cloud Security"]
    return topics


# --- ai_engine/doubt_resolver.py ---
def resolve(query: str):
    return f"The concept behind '{query}' relates to AI/ML fundamentals. Here's a brief explanation..."


# --- ai_engine/voice_to_text.py ---
def convert(audio_file: str):
    return "This is a transcribed example from the audio file."


# --- integrations/google_drive.py ---
def list_files():
    return [{"name": "Project_Report.pdf"}, {"name": "Data_Science_Notes.docx"}]


# --- integrations/topperrank_lms.py ---
def get_courses(student_id: int):
    return [{"course": "AI Fundamentals"}, {"course": "App Development"}]


# --- utils/deadline_tracker.py ---
from datetime import datetime, timedelta

def check_deadlines(student_id: int):
    return [
        {"task": "AI Project Submission", "deadline": str(datetime.now() + timedelta(days=3))},
        {"task": "Cyber Security Assignment", "deadline": str(datetime.now() + timedelta(days=5))}
    ]

# Run backend:
# uvicorn main:app --reload
# Access docs: http://127.0.0.1:8000/docs

# ======================================================
# 📱 FRONTEND: React Native (Expo) – Chat + Login Screen
# ======================================================

# --- package.json ---
"""
{
  "name": "topper_ai_frontend",
  "version": "1.0.0",
  "main": "node_modules/expo/AppEntry.js",
  "scripts": {
    "start": "expo start"
  },
  "dependencies": {
    "axios": "^1.7.2",
    "@react-navigation/native": "^6.1.9",
    "@react-navigation/stack": "^6.3.23",
    "expo": "~52.0.0",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "react-native": "0.76.3",
    "react-native-web": "0.19.12"
  }
}
"""

# --- api/api.js ---
"""
import axios from "axios";

const API_URL = "http://127.0.0.1:8000"; // backend URL

export const api = axios.create({
  baseURL: API_URL,
  headers: { "Content-Type": "application/json" },
});

export const login = async (email, password) => {
  const res = await api.post("/auth/login", { email, password });
  return res.data;
};

export const sendChat = async (query) => {
  const res = await api.post(`/chatbot?query=${encodeURIComponent(query)}`);
  return res.data;
};
"""

# --- context/AuthContext.js ---
"""
import React, { createContext, useState } from "react";

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [token, setToken] = useState(null);
  return (
    <AuthContext.Provider value={{ token, setToken }}>
      {children}
    </AuthContext.Provider>
  );
};
"""

# --- screens/LoginScreen.js ---
"""
import React, { useState, useContext } from "react";
import { View, TextInput, Button, Text, StyleSheet } from "react-native";
import { login } from "../api/api";
import { AuthContext } from "../context/AuthContext";

export default function LoginScreen({ navigation }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const { setToken } = useContext(AuthContext);

  const handleLogin = async () => {
    try {
      const data = await login(email, password);
      setToken(data.access_token);
      navigation.navigate("Chat");
    } catch {
      alert("Invalid credentials");
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Topper AI Mentor</Text>
      <TextInput placeholder="Email" value={email} onChangeText={setEmail} style={styles.input} />
      <TextInput placeholder="Password" secureTextEntry value={password} onChangeText={setPassword} style={styles.input} />
      <Button title="Login" onPress={handleLogin} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: "center", padding: 20 },
  title: { fontSize: 22, textAlign: "center", marginBottom: 20 },
  input: { borderWidth: 1, borderColor: "#ccc", padding: 10, marginBottom: 10, borderRadius: 5 },
});
"""

# --- screens/ChatScreen.js ---
"""
import React, { useState } from "react";
import { View, TextInput, Button, ScrollView, Text, StyleSheet } from "react-native";
import { sendChat } from "../api/api";

export default function ChatScreen() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");

  const handleSend = async () => {
    if (!input.trim()) return;
    const userMsg = { from: "user", text: input };
    setMessages((prev) => [...prev, userMsg]);
    const res = await sendChat(input);
    const botMsg = { from: "bot", text: res.response };
    setMessages((prev) => [...prev, botMsg]);
    setInput("");
  };

  return (
    <View style={styles.container}>
      <ScrollView style={styles.chatArea}>
        {messages.map((msg, i) => (
          <Text key={i} style={msg.from === "user" ? styles.userMsg : styles.botMsg}>
            {msg.text}
          </Text>
        ))}
      </ScrollView>
      <View style={styles.inputArea}>
        <TextInput
          placeholder="Ask me anything..."
          value={input}
          onChangeText={setInput}
          style={styles.input}
        />
        <Button title="Send" onPress={handleSend} />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 10 },
  chatArea: { flex: 1, marginBottom: 10 },
  inputArea: { flexDirection: "row", alignItems: "center" },
  input: { flex: 1, borderWidth: 1, borderColor: "#ccc", padding: 10, marginRight: 5 },
  userMsg: { alignSelf: "flex-end", backgroundColor: "#d1e7ff", padding: 8, margin: 4, borderRadius: 5 },
  botMsg: { alignSelf: "flex-start", backgroundColor: "#f2f2f2", padding: 8, margin: 4, borderRadius: 5 },
});
"""

# --- App.js ---
"""
import React from "react";
import { NavigationContainer } from "@react-navigation/native";
import { createStackNavigator } from "@react-navigation/stack";
import { AuthProvider } from "./context/AuthContext";
import LoginScreen from "./screens/LoginScreen";
import ChatScreen from "./screens/ChatScreen";

const Stack = createStackNavigator();

export default function App() {
  return (
    <AuthProvider>
      <NavigationContainer>
        <Stack.Navigator>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Chat" component={ChatScreen} />
        </Stack.Navigator>
      </NavigationContainer>
    </AuthProvider>
  );
}
"""
