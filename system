import sqlite3
import json
import random
import string
import threading
import uvicorn
from datetime import datetime
import os
import uuid

from fastapi import FastAPI, Request, Form, HTTPException
from fastapi.responses import HTMLResponse, RedirectResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from openai import OpenAI
from elevenlabs.client import ElevenLabs

# ==============================================================================
# CONFIGURAÇÃO
# ==============================================================================
app = FastAPI(title="Oráculo Mental IA - Sistema Terapêutico")
DB_FILE = "dr_aion_pro.db"
AUDIO_DIR = "static_audio"
os.makedirs(AUDIO_DIR, exist_ok=True)
app.mount("/static_audio", StaticFiles(directory=AUDIO_DIR), name="static_audio")

# ==============================================================================
# BANCO DE DADOS
# ==============================================================================
def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    # Tabela de Usuários
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        role TEXT NOT NULL,
        cpf TEXT UNIQUE,
        doctor_id INTEGER DEFAULT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )''')
    
    # Tabela de Logs Clínicos
    c.execute('''CREATE TABLE IF NOT EXISTS logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        patient_id INTEGER NOT NULL,
        mode TEXT NOT NULL,
        timestamp TEXT NOT NULL,
        msg TEXT,
        reply TEXT,
        alienation TEXT,
        symptoms TEXT,
        risk TEXT,
        audio_path TEXT,
        FOREIGN KEY(patient_id) REFERENCES users(id)
    )''')
    
    # Tabela para armazenar chaves API
    c.execute('''CREATE TABLE IF NOT EXISTS api_keys (
        service TEXT PRIMARY KEY,
        key_value TEXT NOT NULL,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )''')
    
    # Verificar e inserir usuário admin padrão se não existir
    admin = c.execute("SELECT * FROM users WHERE username='admin'").fetchone()
    if not admin:
        c.execute("INSERT INTO users (username, password, role) VALUES (?, ?, ?)", 
                 ("admin", "admin123", "admin"))
        print("✅ Usuário admin criado: admin / admin123 (role: admin)")
    else:
        print("✅ Usuário admin já existe")
    
    conn.commit()
    conn.close()

# Funções para gerenciar chaves API
def get_api_key(service_name):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    key = c.execute("SELECT key_value FROM api_keys WHERE service=?", (service_name,)).fetchone()
    conn.close()
    return key[0] if key else None

def update_api_key(service_name, key_value):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute('''INSERT OR REPLACE INTO api_keys (service, key_value) 
                 VALUES (?, ?)''', (service_name, key_value))
    conn.commit()
    conn.close()

# Funções Auxiliares
def generate_fake_cpf():
    part1 = ''.join(random.choices(string.digits, k=3))
    part2 = ''.join(random.choices(string.digits, k=3))
    part3 = ''.join(random.choices(string.digits, k=3))
    part4 = ''.join(random.choices(string.digits, k=2))
    return f"{part1}.{part2}.{part3}-{part4}"

def get_system_prompt(mode):
    prompts = {
        "acolhimento": "Você é o Dr. AION em modo de Acolhimento. Seja extremamente empático, validador e caloroso. Foque em ouvir sem julgar.",
        "cbt": "Você é o Dr. AION em modo TCC (Terapia Cognitivo-Comportamental). Identifique pensamentos distorcidos e proponha desafios cognitivos.",
        "psicanalise": "Você é o Dr. AION em modo Psicanalítico. Busque o inconsciente, sonhos, padrões da infância e significados ocultos.",
        "crise": "Você é o Dr. AION em modo de Gestão de Crise. Seja direto, calmo e utilize técnicas de grounding (enraizamento)."
    }
    base = prompts.get(mode, prompts["acolhimento"])
    return f"{base}\n\nINSTRUÇÃO DE DADOS: Ao final, adicione JSON oculto:\n<<<DATA>>>\n{{\"alienacao\":\"Baixo|Medio|Alto\",\"sintomas\":[],\"risco\":\"Estavel|Atencao|Emergencia\"}}\n<<<DATA>>>"

# ==============================================================================
# LÓGICA DE BACK-END
# ==============================================================================
def generate_audio(text):
    try:
        eleven_key = get_api_key('elevenlabs')
        if not eleven_key:
            return None
            
        client_voice = ElevenLabs(api_key=eleven_key)
        fname = f"{uuid.uuid4()}.mp3"
        fpath = os.path.join(AUDIO_DIR, fname)
        stream = client_voice.text_to_speech.convert(
            text=text, voice_id="21m00Tcm4TlvDq8ikWAM", model_id="eleven_multilingual_v2"
        )
        with open(fpath, 'wb') as f: 
            for chunk in stream: f.write(chunk)
        return f"/static_audio/{fname}"
    except Exception as e:
        print(f"Erro ao gerar áudio: {e}")
        return None

def process_ai(message, mode):
    try:
        openrouter_key = get_api_key('openrouter')
        if not openrouter_key:
            return "Chave da API de IA não configurada. Contate o administrador.", {}, None
            
        client_ai = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=openrouter_key)
        sys_prompt = get_system_prompt(mode)
        res = client_ai.chat.completions.create(
            model="deepseek/deepseek-chat",
            messages=[
                {"role":"system","content":sys_prompt}, 
                {"role":"user","content":message}
            ]
        )
        txt = res.choices[0].message.content
        reply = txt
        data = {"alienacao":"Indef", "sintomas":[], "risco":"Desconhecido"}
        
        if "<<<DATA>>>" in txt:
            p = txt.split("<<<DATA>>>")
            reply = p[0].strip()
            try: 
                data = json.loads(p[1].strip())
            except: pass
            
        audio = generate_audio(reply)
        return reply, data, audio
    except Exception as e:
        print(f"Erro na API de IA: {e}")
        return f"Erro de conexão com a IA. Verifique as configurações do administrador.", {}, None

# ==============================================================================
# HTML TEMPLATES
# ==============================================================================

LOGIN_HTML = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Oráculo Mental IA - Login</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css">
    <style>
        body { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); }
        .glass { background: rgba(255, 255, 255, 0.1); backdrop-filter: blur(10px); border: 1px solid rgba(255, 255, 255, 0.2); }
    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">
    <div class="glass rounded-2xl p-8 w-full max-w-md">
        <div class="text-center mb-8">
            <div class="w-20 h-20 bg-gradient-to-r from-indigo-500 to-purple-600 rounded-full flex items-center justify-center mx-auto mb-4">
                <i class="fas fa-brain text-3xl text-white"></i>
            </div>
            <h1 class="text-3xl font-bold text-white">Oráculo Mental IA</h1>
            <p class="text-white/80 mt-2">Sistema Terapêutico Inteligente</p>
        </div>
        
        <div id="error-msg" class="hidden bg-red-500/20 border border-red-500 text-red-100 px-4 py-2 rounded-lg mb-4"></div>
        <div id="success-msg" class="hidden bg-green-500/20 border border-green-500 text-green-100 px-4 py-2 rounded-lg mb-4"></div>
        
        <!-- Login Form -->
        <form id="loginForm" class="space-y-4">
            <div>
                <label class="block text-white/80 text-sm mb-2">Usuário</label>
                <input type="text" name="username" required
                    class="w-full px-4 py-3 bg-white/10 border border-white/20 rounded-lg text-white placeholder-white/50 focus:outline-none focus:border-white/40"
                    placeholder="Digite seu usuário">
            </div>
            <div>
                <label class="block text-white/80 text-sm mb-2">Senha</label>
                <input type="password" name="password" required
                    class="w-full px-4 py-3 bg-white/10 border border-white/20 rounded-lg text-white placeholder-white/50 focus:outline-none focus:border-white/40"
                    placeholder="Digite sua senha">
            </div>
            <button type="submit" 
                class="w-full bg-gradient-to-r from-indigo-500 to-purple-600 text-white py-3 rounded-lg font-semibold hover:opacity-90 transition">
                Entrar
            </button>
            <p class="text-center text-white/80">
                Não tem conta? 
                <button type="button" onclick="showRegister()" class="text-indigo-300 hover:text-indigo-200 underline">
                    Criar conta
                </button>
            </p>
        </form>

        <!-- Register Form -->
        <form id="registerForm" class="hidden space-y-4">
            <div>
                <label class="block text-white/80 text-sm mb-2">Novo Usuário</label>
                <input type="text" name="username" required
                    class="w-full px-4 py-3 bg-white/10 border border-white/20 rounded-lg text-white placeholder-white/50 focus:outline-none focus:border-white/40"
                    placeholder="Escolha um nome de usuário">
            </div>
            <div>
                <label class="block text-white/80 text-sm mb-2">Senha</label>
                <input type="password" name="password" required
                    class="w-full px-4 py-3 bg-white/10 border border-white/20 rounded-lg text-white placeholder-white/50 focus:outline-none focus:border-white/40"
                    placeholder="Crie uma senha">
            </div>
            <div>
                <label class="block text-white/80 text-sm mb-2">Tipo de Conta</label>
                <!-- CORREÇÃO VISUAL DO SELECT -->
                <select name="role" required
                    class="w-full px-4 py-3 bg-gray-800 border border-gray-600 rounded-lg text-white focus:outline-none focus:border-white/40 appearance-none">
                    <option value="patient">Paciente</option>
                    <option value="doctor">Médico/Profissional</option>
                </select>
            </div>
            <button type="submit" 
                class="w-full bg-gradient-to-r from-green-500 to-teal-600 text-white py-3 rounded-lg font-semibold hover:opacity-90 transition">
                Criar Conta
            </button>
            <p class="text-center text-white/80">
                Já tem conta? 
                <button type="button" onclick="showLogin()" class="text-indigo-300 hover:text-indigo-200 underline">
                    Fazer login
                </button>
            </p>
        </form>
    </div>

    <script>
        function showRegister() {
            document.getElementById('loginForm').classList.add('hidden');
            document.getElementById('registerForm').classList.remove('hidden');
            hideMessages();
        }
        
        function showLogin() {
            document.getElementById('registerForm').classList.add('hidden');
            document.getElementById('loginForm').classList.remove('hidden');
            hideMessages();
        }
        
        function hideMessages() {
            document.getElementById('error-msg').classList.add('hidden');
            document.getElementById('success-msg').classList.add('hidden');
        }
        
        function showError(message) {
            const errorDiv = document.getElementById('error-msg');
            errorDiv.innerText = message;
            errorDiv.classList.remove('hidden');
            document.getElementById('success-msg').classList.add('hidden');
        }
        
        function showSuccess(message) {
            const successDiv = document.getElementById('success-msg');
            successDiv.innerText = message;
            successDiv.classList.remove('hidden');
            document.getElementById('error-msg').classList.add('hidden');
        }
        
        document.querySelectorAll('form').forEach(f => {
            f.onsubmit = async (e) => {
                e.preventDefault();
                hideMessages();
                
                const fd = new FormData(f);
                const action = f.id === 'loginForm' ? 'login' : 'register';
                fd.append('action', action);
                
                try {
                    const res = await fetch('/auth', { 
                        method: 'POST', 
                        body: fd 
                    });
                    
                    const data = await res.json();
                    
                    if (!res.ok) {
                        showError(data.message || 'Erro ao processar solicitação.');
                    } else {
                        showSuccess(data.message || 'Operação realizada com sucesso!');
                        if (data.redirect) {
                            setTimeout(() => {
                                window.location.href = data.redirect;
                            }, 800);
                        }
                    }
                } catch (error) {
                    console.error('Error:', error);
                    showError('Erro de conexão. Tente novamente.');
                }
            };
        });
    </script>
</body>
</html>
"""

# NOVO: HTML DO PAINEL DO ADMINISTRADOR
ADMIN_HTML = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Oráculo Mental IA - Admin</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css">
</head>
<body class="bg-gradient-to-br from-[#0b1020] to-[#070b16] text-white h-screen overflow-hidden">
    <div class="flex h-full">
        <!-- Sidebar -->
        <div class="w-80 bg-[#0d1224] border-r border-white/5 flex flex-col">
            <div class="p-6 border-b border-white/5">
                <h2 class="text-xl font-bold text-red-400">Painel do Administrador</h2>
                <p class="text-white/60 text-sm">{{USER}}</p>
            </div>
            
            <nav class="flex-1 p-4 space-y-2">
                <button onclick="showSection('keys')" class="nav-btn w-full text-left py-3 px-4 rounded-lg bg-white/10 flex items-center gap-3 hover:bg-white/20 transition">
                    <i class="fas fa-key"></i> Chaves API
                </button>
                <button onclick="showSection('users')" class="nav-btn w-full text-left py-3 px-4 rounded-lg hover:bg-white/5 flex items-center gap-3 transition">
                    <i class="fas fa-users"></i> Gerenciar Usuários
                </button>
                <button onclick="showSection('system')" class="nav-btn w-full text-left py-3 px-4 rounded-lg hover:bg-white/5 flex items-center gap-3 transition">
                    <i class="fas fa-cogs"></i> Status do Sistema
                </button>
            </nav>
            
            <div class="p-4 border-t border-white/5">
                <a href="/" class="flex items-center gap-2 text-white/60 hover:text-white transition">
                    <i class="fas fa-sign-out-alt"></i> Sair
                </a>
            </div>
        </div>

        <!-- Main Content -->
        <div class="flex-1 flex flex-col">
            <header class="p-6 border-b border-white/5">
                <h1 class="text-3xl font-bold">Painel Administrativo</h1>
                <p class="text-white/60">Gerenciamento do Sistema Oráculo Mental IA</p>
            </header>
            
            <div id="main-content" class="flex-1 p-6 overflow-y-auto">
                <!-- API Keys Section -->
                <div id="keys-section" class="content-section">
                    <div class="bg-yellow-500/10 border border-yellow-500/30 rounded-xl p-6">
                        <h2 class="text-2xl font-bold mb-4 flex items-center gap-2">
                            <i class="fas fa-key text-yellow-400"></i>
                            Configurações de API
                        </h2>
                        <p class="text-white/60 mb-6">Configure as chaves de API para habilitar a IA e geração de áudio.</p>
                        
                        <div class="grid md:grid-cols-2 gap-6">
                            <div>
                                <label class="block text-sm font-medium mb-2">OpenRouter API Key</label>
                                <input type="password" id="openrouter-key" 
                                    class="w-full bg-gray-800 border border-gray-600 rounded-lg px-4 py-3 text-white placeholder-white/50"
                                    placeholder="sk-or-v1-...">
                            </div>
                            <div>
                                <label class="block text-sm font-medium mb-2">ElevenLabs API Key</label>
                                <input type="password" id="elevenlabs-key" 
                                    class="w-full bg-gray-800 border border-gray-600 rounded-lg px-4 py-3 text-white placeholder-white/50"
                                    placeholder="sk_...">
                            </div>
                        </div>
                        <button onclick="saveApiKeys()" 
                            class="mt-4 bg-yellow-500 hover:bg-yellow-600 text-black font-semibold px-6 py-3 rounded-lg transition">
                            <i class="fas fa-save mr-2"></i>Salvar Chaves
                        </button>
                        <div id="api-status" class="mt-4 text-sm"></div>
                    </div>
                </div>

                <!-- Users Section -->
                <div id="users-section" class="content-section hidden">
                    <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-6">
                        <h2 class="text-2xl font-bold mb-4">Gerenciar Usuários</h2>
                        <div class="overflow-x-auto">
                            <table class="w-full text-left">
                                <thead>
                                    <tr class="border-b border-white/10">
                                        <th class="pb-3">Usuário</th>
                                        <th class="pb-3">Tipo</th>
                                        <th class="pb-3">CPF</th>
                                        <th class="pb-3">Criado em</th>
                                    </tr>
                                </thead>
                                <tbody id="users-list">
                                    <tr>
                                        <td colspan="4" class="text-center py-4 text-white/60">Carregando...</td>
                                    </tr>
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>

                <!-- System Status Section -->
                <div id="system-section" class="content-section hidden">
                    <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-6">
                            <h3 class="text-lg font-semibold mb-2">Total de Usuários</h3>
                            <p class="text-3xl font-bold" id="total-users">-</p>
                        </div>
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-6">
                            <h3 class="text-lg font-semibold mb-2">Sessões Totais</h3>
                            <p class="text-3xl font-bold" id="total-sessions">-</p>
                        </div>
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-6">
                            <h3 class="text-lg font-semibold mb-2">Status da API</h3>
                            <p class="text-3xl font-bold" id="api-status-badge">-</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        const currentUser = "{{USER}}";

        // Section Management
        function showSection(sectionName) {
            // Hide all sections
            document.querySelectorAll('.content-section').forEach(section => {
                section.classList.add('hidden');
            });
            
            // Remove active class from all nav buttons
            document.querySelectorAll('.nav-btn').forEach(btn => {
                btn.classList.remove('bg-white/10');
                btn.classList.add('hover:bg-white/5');
            });
            
            // Show selected section
            document.getElementById(sectionName + '-section').classList.remove('hidden');
            
            // Add active class to clicked button
            event.target.classList.add('bg-white/10');
            event.target.classList.remove('hover:bg-white/5');
            
            // Load data for the section
            if (sectionName === 'keys') {
                loadApiKeys();
            } else if (sectionName === 'users') {
                loadUsers();
            } else if (sectionName === 'system') {
                loadSystemStatus();
            }
        }

        // API Key Management
        async function loadApiKeys() {
            try {
                const response = await fetch(`/api/admin/keys?user=${currentUser}`);
                const data = await response.json();
                
                if (data.openrouter_key) {
                    document.getElementById('openrouter-key').value = data.openrouter_key;
                }
                if (data.elevenlabs_key) {
                    document.getElementById('elevenlabs-key').value = data.elevenlabs_key;
                }
            } catch (error) {
                console.error('Error loading API keys:', error);
            }
        }

        async function saveApiKeys() {
            const openrouterKey = document.getElementById('openrouter-key').value;
            const elevenlabsKey = document.getElementById('elevenlabs-key').value;
            const statusDiv = document.getElementById('api-status');
            
            try {
                const response = await fetch('/api/admin/keys', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        user: currentUser,
                        openrouter_key: openrouterKey,
                        elevenlabs_key: elevenlabsKey
                    })
                });
                
                if (response.ok) {
                    statusDiv.innerHTML = '<span class="text-green-400"><i class="fas fa-check-circle mr-1"></i>Chaves salvas com sucesso!</span>';
                    setTimeout(() => { statusDiv.innerHTML = ''; }, 3000);
                } else {
                    throw new Error('Failed to save keys');
                }
            } catch (error) {
                statusDiv.innerHTML = '<span class="text-red-400"><i class="fas fa-exclamation-circle mr-1"></i>Erro ao salvar chaves.</span>';
                console.error('Error saving API keys:', error);
            }
        }

        // Users Management
        async function loadUsers() {
            try {
                const response = await fetch('/api/admin/users');
                const users = await response.json();
                
                const list = document.getElementById('users-list');
                list.innerHTML = '';
                
                users.forEach(user => {
                    const row = document.createElement('tr');
                    row.className = 'border-b border-white/5';
                    row.innerHTML = `
                        <td class="py-3">${user.username}</td>
                        <td class="py-3">
                            <span class="px-2 py-1 rounded text-xs ${
                                user.role === 'admin' ? 'bg-red-500/20 text-red-400' :
                                user.role === 'doctor' ? 'bg-blue-500/20 text-blue-400' :
                                'bg-green-500/20 text-green-400'
                            }">${user.role}</span>
                        </td>
                        <td class="py-3">${user.cpf || '-'}</td>
                        <td class="py-3 text-white/60">${new Date(user.created_at).toLocaleDateString('pt-BR')}</td>
                    `;
                    list.appendChild(row);
                });
            } catch (error) {
                console.error('Error loading users:', error);
                document.getElementById('users-list').innerHTML = '<tr><td colspan="4" class="text-center py-4 text-red-400">Erro ao carregar usuários</td></tr>';
            }
        }

        // System Status
        async function loadSystemStatus() {
            try {
                const response = await fetch('/api/admin/status');
                const status = await response.json();
                
                document.getElementById('total-users').textContent = status.total_users || '0';
                document.getElementById('total-sessions').textContent = status.total_sessions || '0';
                
                const apiBadge = document.getElementById('api-status-badge');
                if (status.api_configured) {
                    apiBadge.innerHTML = '<span class="text-green-400"><i class="fas fa-check-circle"></i> Online</span>';
                } else {
                    apiBadge.innerHTML = '<span class="text-yellow-400"><i class="fas fa-exclamation-triangle"></i> Configurar</span>';
                }
            } catch (error) {
                console.error('Error loading system status:', error);
            }
        }

        // Initialize
        document.addEventListener('DOMContentLoaded', () => {
            showSection('keys');
        });
    </script>
</body>
</html>
"""

PATIENT_HTML = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Oráculo Mental IA - Paciente</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css">
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    /* Correção de layout para evitar sobreposição */
    .input-wrapper {
      display: flex;
      flex-direction: column;
      gap: 1rem;
    }
    
    .chip {
      padding: 0.5rem 1rem;
      border-radius: 9999px;
      background: rgba(255,255,255,0.05);
      border: 1px solid rgba(255,255,255,0.1);
      font-size: 0.875rem;
      transition: all 0.2s;
      cursor: pointer;
    }
    .chip:hover {
      background: rgba(99,102,241,0.2);
      transform: translateY(-2px);
    }
    
    /* Estilos para as Tags (Modos) que ficam fixas */
    .tag {
      padding: 0.25rem 0.75rem;
      border-radius: 9999px;
      background: rgba(255,255,255,0.08);
      font-size: 0.75rem;
      cursor: pointer;
      transition: all 0.2s;
      border: 1px solid transparent;
    }
    .tag:hover {
      background: rgba(255,255,255,0.15);
      border-color: rgba(255,255,255,0.2);
    }
    .tag.active {
      background: rgba(99, 102, 241, 0.3);
      border-color: rgba(99, 102, 241, 0.5);
      color: #a5b4fc;
    }
  </style>
</head>

<body class="bg-gradient-to-br from-[#0b1020] to-[#070b16] text-white h-screen overflow-hidden">

<div class="flex h-full">

  <!-- SIDEBAR -->
  <aside class="w-64 bg-[#0d1224] border-r border-white/5 flex flex-col p-4">
    <div class="text-center mb-6">
      <div class="w-16 h-16 rounded-full bg-gradient-to-r from-indigo-500 to-purple-600 flex items-center justify-center mx-auto mb-3">
        <i class="fas fa-brain text-2xl"></i>
      </div>
      <h3 class="text-lg font-semibold">{{USER}}</h3>
      <p class="text-white/60 text-sm">Paciente</p>
    </div>

    <nav class="space-y-2">
      <button id="chat-tab" class="w-full text-left py-2 px-4 rounded-lg bg-white/10 flex items-center gap-3">
        <i class="fas fa-comments"></i> Sessão Atual
      </button>
      <button id="history-tab" class="w-full text-left py-2 px-4 rounded-lg hover:bg-white/5 flex items-center gap-3 transition">
        <i class="fas fa-history"></i> Histórico
      </button>
      <button id="profile-tab" class="w-full text-left py-2 px-4 rounded-lg hover:bg-white/5 flex items-center gap-3 transition">
        <i class="fas fa-user"></i> Meus Dados
      </button>
    </nav>

    <div class="mt-auto space-y-3 text-sm text-white/70">
      <button class="flex items-center gap-2 hover:text-white transition w-full text-left">
        <i class="fas fa-cog"></i> Configurações
      </button>
      <button id="clear-chat-btn" class="flex items-center gap-2 hover:text-white transition w-full text-left">
        <i class="fas fa-trash"></i> Limpar Chat
      </button>
      <a href="/" class="flex items-center gap-2 hover:text-white transition">
        <i class="fas fa-sign-out-alt"></i> Sair
      </a>
    </div>
  </aside>

  <!-- MAIN -->
  <main class="flex-1 flex flex-col">

    <!-- HEADER -->
    <header class="text-center py-6 px-6 border-b border-white/5">
      <h1 class="text-4xl font-bold bg-gradient-to-r from-blue-400 to-emerald-400 bg-clip-text text-transparent">
        ORÁCULO MENTAL IA 🧠
      </h1>
      <p class="text-white/60 mt-2">Seu Assistente Terapêutico Inteligente</p>
    </header>

    <!-- CHAT CONTAINER -->
    <div id="chat-container" class="flex-1 overflow-y-auto p-6 space-y-4 scroll-smooth">
      <!-- Welcome Message -->
      <div class="flex items-start gap-3">
        <div class="w-8 h-8 rounded-full bg-gradient-to-r from-indigo-500 to-purple-600 flex items-center justify-center flex-shrink-0">
          <i class="fas fa-brain text-sm"></i>
        </div>
        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-4 max-w-lg">
          <p class="text-white/90">Olá! Eu sou o Oráculo Mental. Estou aqui para ouvir você sem julgamentos. Por onde você gostaria de começar hoje?</p>
        </div>
      </div>
    </div>

    <!-- INPUT AREA -->
    <div class="w-full p-6">
      <!-- Container com flex-col para garantir ordem e evitar sobreposição -->
      <div class="max-w-3xl mx-auto bg-white/5 border border-white/10 rounded-2xl p-4 input-wrapper">
        
        <!-- CATEGORIES / TAGS (Ficam fixas, não somem) -->
        <div class="flex gap-2 flex-wrap justify-center">
          <span class="tag active" onclick="setMode('acolhimento', this)">🧠 Geral</span>
          <span class="tag" onclick="setMode('cbt', this)">😰 Ansiedade</span>
          <span class="tag" onclick="setMode('psicanalise', this)">😔 Depressão</span>
          <span class="tag" onclick="setMode('crise', this)">😤 Estresse</span>
          <span class="tag" onclick="setMode('psicanalise', this)">❤️ Relacionamentos</span>
        </div>

        <!-- QUICK ACTIONS / SUGGESTIONS (Sugestões) -->
        <div id="suggestions-container" class="flex flex-wrap gap-3 justify-center">
          <button class="quick-action chip">Como lidar com ansiedade?</button>
          <button class="quick-action chip">Estou me sentindo estressado</button>
          <button class="quick-action chip">Preciso de apoio emocional</button>
          <button class="quick-action chip">Como melhorar meu bem-estar?</button>
          <button class="quick-action chip">Técnicas de relaxamento</button>
          <button class="quick-action chip">Autoestima e autoconfiança</button>
        </div>

        <!-- MODE SELECTOR (Oculto visualmente mas funcional, estilo corrigido) -->
        <div>
          <select id="mode" class="bg-gray-800 border border-gray-600 rounded-lg px-3 py-2 text-white w-full appearance-none">
            <option value="acolhimento">Acolhimento</option>
            <option value="cbt">TCC (Cognitiva)</option>
            <option value="psicanalise">Psicanálise</option>
            <option value="crise">Gestão de Crise</option>
          </select>
        </div>

        <!-- TEXT INPUT -->
        <div class="flex gap-3 items-center">
          <input
            id="user-input"
            class="flex-1 bg-transparent outline-none text-white placeholder-white/40"
            placeholder="Compartilhe seus pensamentos e sentimentos..."
          >
          <button id="send-button" class="bg-indigo-600 hover:bg-indigo-700 transition p-3 rounded-xl disabled:opacity-50">
            <i class="fas fa-paper-plane"></i>
          </button>
        </div>
        <div id="thinking-indicator" class="hidden text-sm text-white/50 text-center">
            Oráculo está pensando<span class="animate-pulse">...</span>
        </div>

      </div>
    </div>

  </main>
</div>

<!-- HISTORY MODAL -->
<div id="history-modal" class="hidden fixed inset-0 bg-black/80 z-50 flex items-center justify-center p-4">
  <div class="bg-[#0d1224] border border-white/10 rounded-2xl p-6 max-w-4xl w-full max-h-[80vh] overflow-y-auto">
    <div class="flex justify-between items-center mb-4">
      <h2 class="text-2xl font-bold">Histórico de Sessões</h2>
      <button onclick="closeHistory()" class="text-white/60 hover:text-white">
        <i class="fas fa-times text-xl"></i>
      </button>
    </div>
    <div id="history-list" class="space-y-4">
      <p class="text-white/60 text-center py-4">Carregando histórico...</p>
    </div>
  </div>
</div>

<!-- PROFILE MODAL -->
<div id="profile-modal" class="hidden fixed inset-0 bg-black/80 z-50 flex items-center justify-center p-4">
  <div class="bg-[#0d1224] border border-white/10 rounded-2xl p-6 max-w-4xl w-full max-h-[80vh] overflow-y-auto">
    <div class="flex justify-between items-center mb-4">
      <h2 class="text-2xl font-bold">Meus Dados</h2>
      <button onclick="closeProfile()" class="text-white/60 hover:text-white">
        <i class="fas fa-times text-xl"></i>
      </button>
    </div>
    
    <div class="grid md:grid-cols-2 gap-6">
      <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-4">
        <h3 class="text-lg font-semibold mb-3">Informações Pessoais</h3>
        <div class="space-y-2">
          <div class="flex justify-between">
            <span class="text-white/60">Nome:</span>
            <span>{{USER}}</span>
          </div>
          <div class="flex justify-between">
            <span class="text-white/60">CPF:</span>
            <span id="patient-cpf">Carregando...</span>
          </div>
          <div class="flex justify-between">
            <span class="text-white/60">Total de Sessões:</span>
            <span id="total-sessions">Carregando...</span>
          </div>
        </div>
      </div>
      
      <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-4">
        <h3 class="text-lg font-semibold mb-3">Progresso Recente</h3>
        <canvas id="progressChart" width="400" height="200"></canvas>
      </div>
    </div>
  </div>
</div>

<!-- JAVASCRIPT -->
<script>
  const chatContainer = document.getElementById('chat-container');
  const userInput = document.getElementById('user-input');
  const sendButton = document.getElementById('send-button');
  const thinkingIndicator = document.getElementById('thinking-indicator');
  const clearChatBtn = document.getElementById('clear-chat-btn');
  const quickActions = document.querySelectorAll('.quick-action');
  const modeSelect = document.getElementById('mode');
  const suggestionsContainer = document.getElementById('suggestions-container');
  
  // Tab elements
  const chatTab = document.getElementById('chat-tab');
  const historyTab = document.getElementById('history-tab');
  const profileTab = document.getElementById('profile-tab');
  const historyModal = document.getElementById('history-modal');
  const profileModal = document.getElementById('profile-modal');
  
  const currentUser = "{{USER}}";

  // Initialize
  document.addEventListener('DOMContentLoaded', () => {
    loadPatientData();
  });

  // Tab navigation
  historyTab.addEventListener('click', () => {
    historyModal.classList.remove('hidden');
    loadHistory();
  });

  profileTab.addEventListener('click', () => {
    profileModal.classList.remove('hidden');
    loadProfile();
  });

  function closeHistory() {
    historyModal.classList.add('hidden');
  }

  function closeProfile() {
    profileModal.classList.add('hidden');
  }
  
  // Set Mode Function (for tags)
  function setMode(modeValue, element) {
    modeSelect.value = modeValue;
    // Update visual active state
    document.querySelectorAll('.tag').forEach(t => t.classList.remove('active'));
    element.classList.add('active');
  }

  // Chat functions
  function addMessage(sender, text, audioUrl = null) {
    const messageDiv = document.createElement('div');
    messageDiv.classList.add('flex', 'items-start', 'gap-3', 'animate-fade-in');

    let avatar = '';
    let messageContentClass = 'bg-white/5';

    if (sender === 'user') {
      messageDiv.classList.add('flex-row-reverse');
      avatar = '<div class="w-8 h-8 rounded-full bg-gray-600 flex items-center justify-center flex-shrink-0"><i class="fas fa-user text-sm"></i></div>';
      messageContentClass = 'bg-indigo-600/20';
    } else {
      avatar = '<div class="w-8 h-8 rounded-full bg-gradient-to-r from-indigo-500 to-purple-600 flex items-center justify-center flex-shrink-0"><i class="fas fa-brain text-sm"></i></div>';
    }

    let audioElement = '';
    if (audioUrl) {
      audioElement = `<audio controls class="mt-2 w-full"><source src="${audioUrl}" type="audio/mpeg">Seu navegador não suporta o elemento de áudio.</audio>`;
    }

    messageDiv.innerHTML = `
      ${avatar}
      <div class="${messageContentClass} backdrop-blur-xl border border-white/10 rounded-2xl p-4 max-w-lg">
        <p class="text-white/90 whitespace-pre-wrap">${text}</p>
        ${audioElement}
      </div>
    `;

    chatContainer.appendChild(messageDiv);
    scrollToBottom();
  }

  function scrollToBottom() {
    chatContainer.scrollTop = chatContainer.scrollHeight;
  }

  async function sendMessage() {
    const text = userInput.value.trim();
    if (text === '') return;

    addMessage('user', text);
    userInput.value = '';
    
    // Esconder sugestões ao enviar mensagem manualmente
    if (suggestionsContainer) {
      suggestionsContainer.style.display = 'none';
    }
    
    thinkingIndicator.classList.remove('hidden');
    sendButton.disabled = true;
    
    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          username: currentUser,
          message: text,
          mode: modeSelect.value
        })
      });
      
      const data = await response.json();
      
      thinkingIndicator.classList.add('hidden');
      sendButton.disabled = false;
      
      if (data.reply) {
        addMessage('ai', data.reply, data.audio_url);
      } else {
        addMessage('ai', 'Desculpe, ocorreu um erro ao processar sua mensagem. Tente novamente.');
      }
    } catch (error) {
      console.error('Error sending message:', error);
      thinkingIndicator.classList.add('hidden');
      sendButton.disabled = false;
      addMessage('ai', 'Desculpe, ocorreu um erro de conexão. Tente novamente.');
    }
    
    userInput.focus();
  }

  // Event Listeners
  sendButton.addEventListener('click', sendMessage);
  userInput.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  });

  quickActions.forEach(button => {
    button.addEventListener('click', () => {
      userInput.value = button.textContent;
      // CORREÇÃO: Remove apenas o botão clicado, não todos
      button.remove();
      sendMessage();
    });
  });

  clearChatBtn.addEventListener('click', () => {
    if (confirm('Tem certeza de que deseja limpar o chat atual?')) {
      chatContainer.innerHTML = `
        <div class="flex items-start gap-3">
          <div class="w-8 h-8 rounded-full bg-gradient-to-r from-indigo-500 to-purple-600 flex items-center justify-center flex-shrink-0">
            <i class="fas fa-brain text-sm"></i>
          </div>
          <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-4 max-w-lg">
            <p class="text-white/90">Chat limpo. Um novo começo. Por onde você gostaria de recomeçar?</p>
          </div>
        </div>`;
      // Restaurar sugestões ao limpar chat
      if (suggestionsContainer) {
        suggestionsContainer.style.display = 'flex';
      }
    }
  });

  // Load patient data
  async function loadPatientData() {
    try {
      const response = await fetch(`/api/patient/data?user=${currentUser}`);
      const data = await response.json();
      
      if (data.cpf) {
        document.getElementById('patient-cpf').textContent = data.cpf;
      }
      
      if (data.dates && data.scores) {
        document.getElementById('total-sessions').textContent = data.scores.length;
      }
    } catch (error) {
      console.error('Error loading patient data:', error);
    }
  }

  // Load history
  async function loadHistory() {
    try {
      const response = await fetch(`/api/patient/history?user=${currentUser}`);
      const history = await response.json();
      
      const historyList = document.getElementById('history-list');
      historyList.innerHTML = '';
      
      if (history.length === 0) {
        historyList.innerHTML = '<p class="text-white/60 text-center py-4">Nenhum histórico encontrado.</p>';
        return;
      }
      
      history.forEach(item => {
        const date = new Date(item.date);
        const formattedDate = date.toLocaleDateString('pt-BR') + ' ' + date.toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' });
        
        const historyItem = document.createElement('div');
        historyItem.className = 'bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-4';
        historyItem.innerHTML = `
          <div class="flex justify-between items-start mb-2">
            <span class="text-white/60 text-sm">${formattedDate}</span>
            <span class="tag bg-indigo-500/30">${item.mode}</span>
          </div>
          <p class="text-white/90">${item.msg.substring(0, 200)}${item.msg.length > 200 ? '...' : ''}</p>
        `;
        
        historyList.appendChild(historyItem);
      });
    } catch (error) {
      console.error('Error loading history:', error);
      document.getElementById('history-list').innerHTML = '<p class="text-white/60 text-center py-4">Erro ao carregar histórico.</p>';
    }
  }

  // Load profile
  async function loadProfile() {
    try {
      const response = await fetch(`/api/patient/data?user=${currentUser}`);
      const data = await response.json();
      
      if (data.cpf) {
        document.getElementById('patient-cpf').textContent = data.cpf;
      }
      
      if (data.dates && data.scores) {
        document.getElementById('total-sessions').textContent = data.scores.length;
        
        // Create progress chart
        const ctx = document.getElementById('progressChart').getContext('2d');
        new Chart(ctx, {
          type: 'line',
          data: {
            labels: data.dates,
            datasets: [{
              label: 'Nível de Risco',
              data: data.scores,
              borderColor: '#10b981',
              backgroundColor: 'rgba(16, 185, 129, 0.1)',
              tension: 0.4,
              fill: true
            }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
              legend: {
                display: false
              }
            },
            scales: {
              y: {
                beginAtZero: true,
                max: 4,
                ticks: {
                  color: 'rgba(255, 255, 255, 0.7)'
                },
                grid: {
                  color: 'rgba(255, 255, 255, 0.1)'
                }
              },
              x: {
                ticks: {
                  color: 'rgba(255, 255, 255, 0.7)'
                },
                grid: {
                  color: 'rgba(255, 255, 255, 0.1)'
                }
              }
            }
          }
        });
      }
    } catch (error) {
      console.error('Error loading profile:', error);
    }
  }
</script>

</body>
</html>
"""

# DOCTOR_HTML SEM O PAINEL DE CHAVES
DOCTOR_HTML = """
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Oráculo Mental IA - Médico</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-gradient-to-br from-[#0b1020] to-[#070b16] text-white h-screen overflow-hidden">
    <div class="flex h-full">
        <!-- Sidebar -->
        <div class="w-80 bg-[#0d1224] border-r border-white/5 flex flex-col">
            <div class="p-6 border-b border-white/5">
                <h2 class="text-xl font-bold text-indigo-400">Painel do Médico</h2>
                <p class="text-white/60 text-sm">Dr. {{USER}}</p>
            </div>
            
            <div class="flex-1 overflow-y-auto p-4">
                <h3 class="text-sm font-semibold text-white/60 mb-3">PACIENTES</h3>
                <div id="patient-list" class="space-y-2">
                    <!-- Patients will be loaded here -->
                </div>
            </div>
            
            <div class="p-4 border-t border-white/5">
                <div class="flex gap-2">
                    <input type="text" id="cpf-input" placeholder="CPF do paciente" 
                        class="flex-1 bg-white/10 border border-white/20 rounded-lg px-3 py-2 text-white placeholder-white/50 text-sm">
                    <button onclick="addPatient()" 
                        class="bg-indigo-600 hover:bg-indigo-700 px-4 py-2 rounded-lg transition">
                        <i class="fas fa-plus"></i>
                    </button>
                </div>
            </div>
            
            <div class="p-4 border-t border-white/5">
                <a href="/" class="flex items-center gap-2 text-white/60 hover:text-white transition">
                    <i class="fas fa-sign-out-alt"></i> Sair
                </a>
            </div>
        </div>

        <!-- Main Content -->
        <div class="flex-1 flex flex-col">
            <header class="p-6 border-b border-white/5">
                <h1 class="text-3xl font-bold">Prontuário dos Pacientes</h1>
                <p class="text-white/60">Visualize e acompanhe o progresso terapêutico</p>
            </header>
            
            <div id="main-content" class="flex-1 p-6 overflow-y-auto">
                <div class="text-center py-20 text-white/40">
                    <i class="fas fa-users text-6xl mb-4"></i>
                    <p class="text-xl">Selecione um paciente para visualizar o prontuário</p>
                </div>
            </div>
        </div>
    </div>

    <script>
        const currentUser = "{{USER}}";
        let currentPatient = null;

        async function loadPatients() {
            try {
                const response = await fetch(`/api/doctor/patients?doc=${currentUser}`);
                const patients = await response.json();
                
                const list = document.getElementById('patient-list');
                list.innerHTML = '';
                
                if (patients.length === 0) {
                    list.innerHTML = '<p class="text-white/40 text-sm">Nenhum paciente vinculado</p>';
                    return;
                }
                
                patients.forEach(patient => {
                    const item = document.createElement('div');
                    item.className = 'bg-white/5 hover:bg-white/10 rounded-lg p-3 cursor-pointer transition';
                    item.innerHTML = `
                        <div class="flex items-center justify-between">
                            <div>
                                <p class="font-semibold">${patient.username}</p>
                                <p class="text-xs text-white/60">CPF: ${patient.cpf}</p>
                            </div>
                            <i class="fas fa-chevron-right text-white/40"></i>
                        </div>
                    `;
                    item.onclick = () => selectPatient(patient.username, patient.cpf);
                    list.appendChild(item);
                });
            } catch (error) {
                console.error('Error loading patients:', error);
            }
        }

        async function addPatient() {
            const cpf = document.getElementById('cpf-input').value.trim();
            if (!cpf) return;
            
            try {
                const response = await fetch('/api/doctor/add', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ doctor: currentUser, cpf: cpf })
                });
                
                if (response.ok) {
                    document.getElementById('cpf-input').value = '';
                    loadPatients();
                } else {
                    alert('Paciente não encontrado ou já vinculado.');
                }
            } catch (error) {
                console.error('Error adding patient:', error);
            }
        }

        async function selectPatient(username, cpf) {
            currentPatient = username;
            
            try {
                const response = await fetch(`/api/doctor/data?pat=${username}`);
                const data = await response.json();
                
                const mainContent = document.getElementById('main-content');
                
                if (!data.dates) {
                    mainContent.innerHTML = `
                        <div class="text-center py-20 text-white/40">
                            <i class="fas fa-clipboard text-6xl mb-4"></i>
                            <p class="text-xl">Nenhuma sessão registrada para este paciente</p>
                        </div>
                    `;
                    return;
                }
                
                const riskClass = data.last_risk === 'Emergencia' ? 'text-red-400' : 
                                 data.last_risk === 'Atencao' ? 'text-yellow-400' : 'text-green-400';
                
                mainContent.innerHTML = `
                    <div class="mb-6">
                        <h2 class="text-2xl font-bold mb-2">${username}</h2>
                        <p class="text-white/60">CPF: ${cpf}</p>
                    </div>
                    
                    <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                            <p class="text-white/60 text-sm mb-1">Risco Atual</p>
                            <p class="text-2xl font-bold ${riskClass}">${data.last_risk || 'N/A'}</p>
                        </div>
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                            <p class="text-white/60 text-sm mb-1">Alienação</p>
                            <p class="text-2xl font-bold">${data.last_alienation || 'N/A'}</p>
                        </div>
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                            <p class="text-white/60 text-sm mb-1">Total Sessões</p>
                            <p class="text-2xl font-bold">${data.total_sessions || 0}</p>
                        </div>
                    </div>
                    
                    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                            <h3 class="text-lg font-semibold mb-4">Evolução da Alienação</h3>
                            <canvas id="evolution-chart"></canvas>
                        </div>
                        <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                            <h3 class="text-lg font-semibold mb-4">Distribuição de Modos</h3>
                            <canvas id="modes-chart"></canvas>
                        </div>
                    </div>
                    
                    <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4">
                        <h3 class="text-lg font-semibold mb-4">Últimas Sessões</h3>
                        <div class="space-y-3">
                            ${(data.logs || []).map(log => `
                                <div class="border-b border-white/10 pb-3">
                                    <div class="flex justify-between items-start mb-2">
                                        <span class="text-white/60 text-sm">${log.date}</span>
                                        <span class="text-xs bg-indigo-500/30 px-2 py-1 rounded">${log.mode}</span>
                                    </div>
                                    <p class="text-white/90">${log.msg}</p>
                                </div>
                            `).join('')}
                        </div>
                    </div>
                `;
                
                // Render charts
                setTimeout(() => {
                    renderCharts(data);
                }, 100);
                
            } catch (error) {
                console.error('Error loading patient data:', error);
            }
        }

        function renderCharts(data) {
            // Evolution Chart
            const evolutionCtx = document.getElementById('evolution-chart');
            if (evolutionCtx) {
                new Chart(evolutionCtx, {
                    type: 'line',
                    data: {
                        labels: data.dates || [],
                        datasets: [{
                            label: 'Nível de Alienação',
                            data: data.alienation_scores || [],
                            borderColor: '#818cf8',
                            backgroundColor: 'rgba(129, 140, 248, 0.1)',
                            tension: 0.4,
                            fill: true
                        }]
                    },
                    options: {
                        responsive: true,
                        plugins: {
                            legend: { display: false }
                        },
                        scales: {
                            y: {
                                beginAtZero: true,
                                max: 4,
                                ticks: { color: 'rgba(255, 255, 255, 0.7)' },
                                grid: { color: 'rgba(255, 255, 255, 0.1)' }
                            },
                            x: {
                                ticks: { color: 'rgba(255, 255, 255, 0.7)' },
                                grid: { color: 'rgba(255, 255, 255, 0.1)' }
                            }
                        }
                    }
                });
            }
            
            // Modes Chart
            const modesCtx = document.getElementById('modes-chart');
            if (modesCtx && data.modes) {
                new Chart(modesCtx, {
                    type: 'doughnut',
                    data: {
                        labels: Object.keys(data.modes),
                        datasets: [{
                            data: Object.values(data.modes),
                            backgroundColor: ['#818cf8', '#34d399', '#fbbf24', '#f87171']
                        }]
                    },
                    options: {
                        responsive: true,
                        plugins: {
                            legend: {
                                position: 'bottom',
                                labels: { color: 'rgba(255, 255, 255, 0.7)' }
                            }
                        }
                    }
                });
            }
        }

        // Initialize
        loadPatients();
    </script>
</body>
</html>
"""

# ==============================================================================
# ROTAS DA APLICAÇÃO
# ==============================================================================

@app.get("/", response_class=HTMLResponse)
async def root():
    return HTMLResponse(content=LOGIN_HTML)

@app.post("/auth")
async def auth(req: Request, username: str=Form(...), password: str=Form(...), action: str=Form(...), role: str=Form("patient")):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    try:
        if action == "register":
            existing = c.execute("SELECT username FROM users WHERE username=?", (username,)).fetchone()
            if existing:
                return JSONResponse(content={"message": "Nome de usuário já existe. Escolha outro."}, status_code=400)
            
            cpf = None
            if role == "patient":
                cpf = generate_fake_cpf()
            
            c.execute("INSERT INTO users (username, password, role, cpf) VALUES (?,?,?,?)", 
                      (username, password, role, cpf))
            conn.commit()
            
            print(f"✅ Novo usuário criado: {username} ({role})")
            # Retorna JSON com a URL para o frontend redirecionar
            return JSONResponse(content={"redirect": f"/dashboard/{role}?user={username}", "message": "Conta criada com sucesso!"})
            
        elif action == "login":
            user = c.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password)).fetchone()
            if user:
                print(f"✅ Login bem-sucedido: {username} ({user[3]})")
                return JSONResponse(content={"redirect": f"/dashboard/{user[3]}?user={username}", "message": "Login realizado com sucesso!"})
            else:
                return JSONResponse(content={"message": "Usuário ou senha incorretos."}, status_code=401)
                
    except sqlite3.Error as e:
        print(f"❌ Erro no banco de dados: {e}")
        return JSONResponse(content={"message": "Erro ao processar sua solicitação. Tente novamente."}, status_code=500)
    finally:
        conn.close()

# NOVA: Rota para o dashboard do admin
@app.get("/dashboard/admin", response_class=HTMLResponse)
async def admin_dash(req: Request, user: str):
    html = ADMIN_HTML.replace("{{USER}}", user)
    return HTMLResponse(content=html)

@app.get("/dashboard/patient", response_class=HTMLResponse)
async def patient_dash(req: Request, user: str):
    html = PATIENT_HTML.replace("{{USER}}", user)
    return HTMLResponse(content=html)

@app.get("/dashboard/doctor", response_class=HTMLResponse)
async def doctor_dash(req: Request, user: str):
    html = DOCTOR_HTML.replace("{{USER}}", user)
    return HTMLResponse(content=html)

@app.get("/api/patient/data")
async def patient_data(user: str):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    user_row = c.execute("SELECT cpf FROM users WHERE username=?", (user,)).fetchone()
    logs = c.execute("SELECT timestamp, risk FROM logs WHERE patient_id=(SELECT id FROM users WHERE username=?) ORDER BY timestamp ASC LIMIT 10", (user,)).fetchall()
    conn.close()
    
    scores_map = {"Estavel": 1, "Atencao": 2, "Emergencia": 3}
    return {
        "cpf": user_row[0] if user_row else None,
        "dates": [l[0].split(' ')[0] for l in logs],
        "scores": [scores_map.get(l[1], 1) for l in logs]
    }

@app.get("/api/patient/history")
async def patient_history(user: str):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    logs = c.execute("SELECT timestamp, mode, msg FROM logs WHERE patient_id=(SELECT id FROM users WHERE username=?) ORDER BY timestamp DESC", (user,)).fetchall()
    conn.close()
    return [{"date": l[0], "mode": l[1], "msg": l[2]} for l in logs]

@app.post("/api/chat")
async def api_chat(req: Request):
    d = await req.json()
    u, m, mode = d.get("username"), d.get("message"), d.get("mode", "acolhimento")
    
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    pat = c.execute("SELECT id FROM users WHERE username=?", (u,)).fetchone()
    if not pat: 
        conn.close()
        return {"error": "User not found"}
    pat_id = pat[0]
    
    reply, data, audio = process_ai(m, mode)
    
    c.execute("INSERT INTO logs (patient_id, mode, timestamp, msg, reply, alienation, symptoms, risk, audio_path) VALUES (?,?,?,?,?,?,?,?,?)",
              (pat_id, mode, str(datetime.now()), m, reply, data.get("alienacao"), json.dumps(data.get("sintomas")), data.get("risco"), audio))
    conn.commit()
    conn.close()
    
    return {"reply": reply, "audio_url": audio}

@app.get("/api/doctor/patients")
async def get_doctor_patients(doc: str):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    rows = c.execute("SELECT username, cpf FROM users WHERE role='patient' AND doctor_id=(SELECT id FROM users WHERE username=?)", (doc,)).fetchall()
    conn.close()
    return [{"username": r[0], "cpf": r[1]} for r in rows]

@app.post("/api/doctor/add")
async def add_patient_to_doctor(req: Request):
    d = await req.json()
    doc_name, cpf = d.get("doctor"), d.get("cpf")
    
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    doc = c.execute("SELECT id FROM users WHERE username=? AND role='doctor'", (doc_name,)).fetchone()
    pat = c.execute("SELECT id, doctor_id FROM users WHERE cpf=? AND role='patient'", (cpf,)).fetchone()
    
    if not doc or not pat:
        conn.close()
        raise HTTPException(400, "Dados inválidos")
    
    c.execute("UPDATE users SET doctor_id=? WHERE id=?", (doc[0], pat[0]))
    conn.commit()
    conn.close()
    return {"success": True}

@app.get("/api/doctor/data")
async def get_patient_analysis(pat: str):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    logs = c.execute("SELECT * FROM logs WHERE patient_id=(SELECT id FROM users WHERE username=?) ORDER BY timestamp ASC", (pat,)).fetchall()
    conn.close()
    
    if not logs: 
        return {}
    
    alien_map = {"Baixo":1, "Medio":2, "Alto":3}
    dates = [l[3].split(' ')[0] for l in logs]
    alien_scores = [alien_map.get(l[6], 0) for l in logs]
    
    modes = {}
    for l in logs:
        m = l[2]
        modes[m] = modes.get(m, 0) + 1
        
    return {
        "last_risk": logs[-1][8],
        "last_alienation": logs[-1][6],
        "total_sessions": len(logs),
        "last_audio": logs[-1][9],
        "dates": dates,
        "alienation_scores": alien_scores,
        "modes": modes,
        "logs": [{"date": l[3], "mode": l[2], "msg": l[5]} for l in logs[-5:]]
    }

# NOVAS: Rotas para o Admin
@app.get("/api/admin/keys")
async def get_admin_keys(user: str):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    user_check = c.execute("SELECT role FROM users WHERE username=?", (user,)).fetchone()
    if not user_check or user_check[0] != 'admin':
        conn.close()
        raise HTTPException(403, "Acesso negado")
    
    openrouter_key = get_api_key('openrouter')
    elevenlabs_key = get_api_key('elevenlabs')
    
    conn.close()
    return {
        "openrouter_key": openrouter_key or "",
        "elevenlabs_key": elevenlabs_key or ""
    }

@app.post("/api/admin/keys")
async def save_admin_keys(req: Request):
    data = await req.json()
    user = data.get("user")
    openrouter_key = data.get("openrouter_key")
    elevenlabs_key = data.get("elevenlabs_key")
    
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    user_check = c.execute("SELECT role FROM users WHERE username=?", (user,)).fetchone()
    if not user_check or user_check[0] != 'admin':
        conn.close()
        raise HTTPException(403, "Acesso negado")
    
    if openrouter_key:
        update_api_key('openrouter', openrouter_key)
    if elevenlabs_key:
        update_api_key('elevenlabs', elevenlabs_key)
    
    conn.close()
    return {"status": "success", "message": "Chaves API atualizadas com sucesso"}

@app.get("/api/admin/users")
async def get_all_users():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    users = c.execute("SELECT username, role, cpf, created_at FROM users ORDER BY created_at DESC").fetchall()
    conn.close()
    return [{"username": u[0], "role": u[1], "cpf": u[2], "created_at": u[3]} for u in users]

@app.get("/api/admin/status")
async def get_system_status():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    
    total_users = c.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    total_sessions = c.execute("SELECT COUNT(*) FROM logs").fetchone()[0]
    
    openrouter_key = get_api_key('openrouter')
    elevenlabs_key = get_api_key('elevenlabs')
    api_configured = bool(openrouter_key and elevenlabs_key)
    
    conn.close()
    return {
        "total_users": total_users,
        "total_sessions": total_sessions,
        "api_configured": api_configured
    }

# ==============================================================================
# PONTO DE ENTRADA
# ==============================================================================
if __name__ == "__main__":
    print("\n" + "="*60)
    print("🧠 ORÁCULO MENTAL IA - SISTEMA TERAPÊUTICO")
    print("="*60)
    print("\n📋 Informações:")
    print("   • Banco de dados: dr_aion_pro.db")
    print("   • Áudios salvos em: static_audio/")
    print("   • Usuário admin padrão: admin / admin123")
    print("\n⚠️  IMPORTANTE:")
    print("   1. Faça login como 'admin' para acessar o painel administrativo")
    print("   2. Configure as chaves API na seção 'Chaves API'")
    print("   3. Médicos não têm mais acesso às configurações de API")
    print("\n🚀 Iniciando servidor...")
    print("   • Acesse: http://localhost:8000")
    print("   • Pressione Ctrl+C para parar\n")
    print("="*60 + "\n")
    
    init_db()
    
    uvicorn.run(app, host="0.0.0.0", port=8000)
