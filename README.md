import requests
import hashlib
import json
import os
import io
from datetime import datetime

# Tenta importar PyPDF2 para extração de texto
try:
    import PyPDF2
except ImportError:
    os.system('pip install PyPDF2')
    import PyPDF2

# --- CONFIGURAÇÃO DE FONTES (URLs das Sociedades Médicas) ---
SOURCES = [
    {"nome": "Diretriz Cardio SBC", "url": "https://aberto.cardiol.br/diretriz/exemplo.pdf"},
    {"nome": "Manual de Perícia Federal", "url": "https://www.gov.br/servidor/exemplo.pdf"}
]

def get_hash(data):
    return hashlib.sha256(data).hexdigest()

def extract_text_from_pdf(pdf_content):
    try:
        pdf_file = io.BytesIO(pdf_content)
        reader = PyPDF2.PdfReader(pdf_file)
        text = ""
        for page in reader.pages:
            content = page.extract_text()
            if content:
                text += content + " "
        return text.strip()
    except Exception as e:
        return f"Erro na extração: {str(e)}"

def run_helios_sync():
    print(f"🚀 Iniciando HELIOS DNA Sync - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    
    extracted_sinapses = []

    # 1. Processamento das Diretrizes (Scraping)
    for source in SOURCES:
        try:
            print(f"🔍 Varrendo: {source['nome']}...")
            response = requests.get(source['url'], timeout=30, headers={'User-Agent': 'Mozilla/5.0'})
            
            if response.status_code == 200:
                content = response.content
                doc_hash = get_hash(content)
                raw_text = extract_text_from_pdf(content)
                
                # Estrutura compatível com o Dexie (IA-PEX v1000)
                extracted_sinapses.append({
                    "id": f"synapse-{doc_hash[:8]}",
                    "nome": source['nome'],
                    "hash": doc_hash,
                    "materia": "Atualização Automática",
                    "texto": raw_text,
                    "source": source['url'],
                    "provenance": f"Crawler Helios: {datetime.now().isoformat()}",
                    "data": int(datetime.now().timestamp() * 1000)
                })
                print(f"✅ {source['nome']} processado com sucesso.")
            else:
                print(f"❌ Erro {response.status_code} em {source['nome']}")
        except Exception as e:
            print(f"⚠️ Falha crítica em {source['nome']}: {e}")

    # 2. Integração com o seu DNA_DATA (Configurações de Nuvem)
    full_payload = {
        "version": "1.0.0",
        "last_sync": datetime.now().strftime("%Y-%m-%d"),
        "project_name": "Helios Nuvem",
        "system_config": {
            "auto_sync": True,
            "cloud_provider": "Vercel/GitHub",
            "encryption": "SHA-256 Enabled"
        },
        # Aqui injetamos os dados capturados pelo crawler
        "dna_sequences": extracted_sinapses 
    }

    # 3. Salvamento do Arquivo Final
    with open('dna_evolution.json', 'w', encoding='utf-8') as f:
        json.dump(full_payload, f, ensure_ascii=False, indent=4)
    
    print(f"🧬 DNA EVOLUTION concluído. {len(extracted_sinapses)} sinapses integradas.")

if __name__ == "__main__":
    run_helios_sync()
