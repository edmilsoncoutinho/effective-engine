Python
# OmniTech AI - Plataforma Inteligente Total Avançada com SINE Bahia
# Proprietário: Edmilson Gomes - Código Blindado

import json, os, time, requests, re
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI(title="OmniTech AI", version="2.1.0")
DB = "omnitech_ai_db.json"

# ---------------------------
# Modelos
# ---------------------------
class User(BaseModel):
    name: str
    email: str
    skills: List[str]
    resume: str
    calendar: List[str] = []

class Application(BaseModel):
    company: str
    position: str
    status: str = "pending"

# ---------------------------
# DB Helpers
# ---------------------------
def load_db():
    if not os.path.exists(DB):
        json.dump({"users":[], "applications":[]}, open(DB,"w"))
    return json.load(open(DB))

def save_db(db):
    json.dump(db, open(DB,"w"), indent=2)

# ---------------------------
# Cadastro de usuário
# ---------------------------
@app.post("/register")
def register(user: User):
    db = load_db()
    if any(u["email"]==user.email for u in db["users"]):
        raise HTTPException(400, "Usuário já registrado")
    db["users"].append(user.dict())
    save_db(db)
    return {"msg": f"{user.name} registrado com sucesso."}

# ---------------------------
# Busca de vagas padrão
# ---------------------------
def fetch_standard_jobs(skills):
    jobs = [
        {"company":"TechCorp","position":"Dev Python","skills":["Python"]},
        {"company":"InovaSoft","position":"Eng Dados","skills":["Python","SQL"]},
        {"company":"AI Labs","position":"Especialista IA","skills":["Python","ML"]}
    ]
    return [j for j in jobs if any(s in skills for s in j["skills"])]

# ---------------------------
# Busca de vagas SINE Bahia (simulação scraping)
# ---------------------------
def fetch_sine_bahia_jobs():
    """
    Simula busca de vagas do SINE Bahia usando scraping.
    Em produção, pode ser substituído por integração real com Emprega Brasil/SINE Fácil.
    """
    try:
        url = "https://servicos.mte.gov.br/spme-v2/#/login"
        res = requests.get(url, timeout=10)
        html = res.text

        # Simples extração de texto de vagas a partir de palavras-chaves (exemplo)
        found = re.findall(r"[A-Za-zÀ-ÿ ]{5,30}", html)
        jobs = [{"company": "SINE Bahia", "position": f"{txt.strip()}"} for txt in found[:10]]
        return jobs
    except Exception as e:
        return []

# ---------------------------
# Aplicação automática de vagas
# ---------------------------
@app.post("/apply/{email}")
def apply_jobs(email: str):
    db = load_db()
    user = next((u for u in db["users"] if u["email"] == email), None)
    if not user: raise HTTPException(404, "Usuário não encontrado")

    # Busca clássica + SINE Bahia
    std_jobs = fetch_standard_jobs(user["skills"])
    sine_jobs = fetch_sine_bahia_jobs()
    all_jobs = std_jobs + sine_jobs

    sent = []
    for job in all_jobs:
        app_data = Application(company=job["company"], position=job["position"]).dict()
        db["applications"].append(app_data)
        sent.append(app_data)
        time.sleep(1)
    save_db(db)
    return {"msg": f"{len(sent)} aplicações enviadas (inclui SINE Bahia).", "applications": sent}

# ---------------------------
# Listar aplicações
# ---------------------------
@app.get("/applications/{email}")
def list_applications(email: str):
    db = load_db()
    return {"applications": [a for a in db["applications"] if any(u["email"]==email for u in db["users"])]}

# ---------------------------
# Assistente IA avançado
# ---------------------------
@app.get("/ai/{email}")
def ai_assistant(email: str, query: str):
    db = load_db()
    user = next((u for u in db["users"] if u["email"]==email), None)
    if not user: raise HTTPException(404, "Usuário não encontrado")

    q = query.lower()
    if "financeiro" in q:
        return {"response": f"{user['name']}, sugiro revisar seus gastos e priorizar investimentos seguros."}
    if "emprego" in q:
        jobs = fetch_standard_jobs(user["skills"]) + fetch_sine_bahia_jobs()
        return {"response": f"{user['name']}, encontrei {len(jobs)} vagas compatíveis (incluindo SINE Bahia)."}
    if "agenda" in q:
        return {"response": f"{user['name']}, sua agenda: {user.get('calendar', [])}"}
    return {"response": f"{user['name']}, analisando sua solicitação: {query}"}

# ---------------------------
# Agenda / Calendário
# ---------------------------
@app.post("/calendar/{email}")
def add_calendar(email: str, event: str):
    db = load_db()
    user = next((u for u in db["users"] if u["email"]==email), None)
    if not user: raise HTTPException(404, "Usuário não encontrado")
    user.setdefault("calendar", []).append(event)
    save_db(db)
    return {"msg": f"Evento adicionado à agenda de {user['name']}.", "calendar": user["calendar"]}