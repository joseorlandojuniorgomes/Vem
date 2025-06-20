// server/index.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

const PORT = process.env.PORT || 5000;
const JWT_SECRET = process.env.JWT_SECRET || 'secreto';
const MONGO_URI = process.env.MONGO_URI || '';

// --- Models ---

const UserSchema = new mongoose.Schema({
  email: { type: String, unique: true },
  password: String,
  saldo: { type: Number, default: 0 },
  lastSpin: Date,
  referralCode: String,
  referredBy: String,
});

const User = mongoose.model('User', UserSchema);

// --- Helpers ---

const generateToken = (user) => {
  return jwt.sign({ id: user._id, email: user.email }, JWT_SECRET, {
    expiresIn: '7d',
  });
};

// --- Routes ---

// Registro
app.post('/api/register', async (req, res) => {
  const { email, password, referralCode } = req.body;
  try {
    const exists = await User.findOne({ email });
    if (exists) return res.status(400).json({ message: 'Email já cadastrado' });

    // Criar código de indicação próprio
    const newUser = new User({
      email,
      password, // Sem criptografia simples, pra demo. Pode melhorar depois.
      saldo: 0,
      lastSpin: null,
      referralCode: Math.random().toString(36).substring(2, 8).toUpperCase(),
      referredBy: referralCode || null,
    });

    // Se veio código de indicação, dá bônus
    if (referralCode) {
      const refUser = await User.findOne({ referralCode });
      if (refUser) {
        refUser.saldo += 0.5; // bônus por indicação
        await refUser.save();
      }
    }

    await newUser.save();
    const token = generateToken(newUser);
    res.json({ token, email: newUser.email, saldo: newUser.saldo, referralCode: newUser.referralCode });
  } catch (err) {
    res.status(500).json({ message: 'Erro no servidor' });
  }
});

// Login
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });
    if (!user || user.password !== password) {
      return res.status(400).json({ message: 'Email ou senha inválidos' });
    }
    const token = generateToken(user);
    res.json({ token, email: user.email, saldo: user.saldo, referralCode: user.referralCode });
  } catch (err) {
    res.status(500).json({ message: 'Erro no servidor' });
  }
});

// Middleware pra validar token
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ message: 'Token não enviado' });

  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Token inválido' });
  }
};

// Pegar dados do usuário
app.get('/api/me', authMiddleware, async (req, res) => {
  const user = await User.findById(req.user.id);
  if (!user) return res.status(404).json({ message: 'Usuário não encontrado' });

  res.json({ email: user.email, saldo: user.saldo, referralCode: user.referralCode, lastSpin: user.lastSpin });
});

// Girar roleta (1 vez por dia)
app.post('/api/spin', authMiddleware, async (req, res) => {
  const user = await User.findById(req.user.id);
  if (!user) return res.status(404).json({ message: 'Usuário não encontrado' });

  const now = new Date();
  if (user.lastSpin) {
    const lastSpinDate = new Date(user.lastSpin);
    if (
      now.toDateString() === lastSpinDate.toDateString()
    ) {
      return res.status(400).json({ message: 'Você já girou a roleta hoje' });
    }
  }

  // Prêmios possíveis
  const premios = [0.25, 0.5, 1, 5, 10];
  const premio = premios[Math.floor(Math.random() * premios.length)];

  user.saldo += premio;
  user.lastSpin = now;
  await user.save();

  res.json({ premio, saldo: user.saldo });
});

// Rodar servidor
mongoose
  .connect(MONGO_URI)
  .then(() => {
    app.listen(PORT, () => {
      console.log(`Servidor rodando na porta ${PORT}`);
    });
  })
  .catch((err) => {
    console.error('Erro ao conectar no MongoDB:', err);
  });// client/src/App.jsx
import React, { useState, useEffect } from 'react';

const BACKEND_URL = process.env.REACT_APP_BACKEND_URL || 'http://localhost:5000';

function App() {
  const [page, setPage] = useState('login');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [token, setToken] = useState(localStorage.getItem('token') || '');
  const [saldo, setSaldo] = useState(0);
  const [referralCode, setReferralCode] = useState('');
  const [lastSpin, setLastSpin] = useState(null);
  const [message, setMessage] = useState('');
  const [referralInput, setReferralInput] = useState('');

  useEffect(() => {
    if (token) {
      fetch(`${BACKEND_URL}/api/me`, {
        headers: { Authorization: `Bearer ${token}` },
      })
        .then((res) => res.json())
        .then((data) => {
          setSaldo(data.saldo);
          setReferralCode(data.referralCode);
          setLastSpin(data.lastSpin);
          setPage('dashboard');
        })
        .catch(() => {
          setToken('');
          localStorage.removeItem('token');
          setPage('login');
        });
    }
  }, [token]);

  const handleRegister = () => {
    fetch(`${BACKEND_URL}/api/register`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password, referralCode: referralInput }),
    })
      .then((res) => res.json())
      .then((data) => {
        if (data.token) {
          setToken(data.token);
          localStorage.setItem('token', data.token);
          setPage('dashboard');
          setMessage('');
        } else {
          setMessage(data.message || 'Erro no cadastro');
        }
      });
  };

  const handleLogin = () => {
    fetch(`${BACKEND_URL}/api/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })
      .then((res) => res.json())
      .then((data) => {
        if (data.token) {
          setToken(data.token);
          localStorage.setItem('token', data.token);
          setPage('dashboard');
          setMessage('');
        } else {
          setMessage(data.message || 'Erro no login');
        }
      });
  };

  const handleSpin = () => {
    fetch(`${BACKEND_URL}/api/spin`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
    })
      .then((res) => res.json())
      .then((data) => {
        if (data.premio) {
          setSaldo(data.saldo);
          setLastSpin(new Date().toISOString());
          setMessage(`Você ganhou R$${data.premio.toFixed(2)}!`);
        } else {
          setMessage(data.message || 'Erro ao girar');
        }
      });
  };

  const handleLogout = () => {
    setToken('');
    localStorage.removeItem('token');
    setPage('login');
  };

  if (page === 'login')
    return (
      <div className="max-w-md mx-auto p-4">
        <h1 className="text-2xl font-bold mb-4">Login</h1>
        <input
          type="email"
          placeholder="E-mail"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="border p-2 w-full mb-2"
        />
        <input
          type="password"
          placeholder="Senha"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          className="border p-2 w-full mb-2"
        />
        <button onClick={handleLogin} className="bg-blue-500 text-white p-2 w-full mb-2">
          Entrar
        </button>
        <p className="mb-2 text-red-600">{message}</p>
        <p>
          Não tem conta?{' '}
          <button onClick={() => setPage('register')} className="text-blue-600 underline">
            Cadastrar
          </button>
        </p>
      </div>
    );

  if (page === 'register')
    return (
      <div className="max-w-md mx-auto p-4">
        <h1 className="text-2xl font-bold mb-4">Cadastro</h1>
        <input
          type="email"
          placeholder="E-mail"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="border p-2 w-full mb-2"
        />
        <input
          type="password"
          placeholder="Senha"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          className="border p-2 w-full mb-2"
        />
        <input
          type="text"
          placeholder="Código de indicação (opcional)"
          value={referralInput}
          onChange={(
