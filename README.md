import os
import time
import random

# Instala webdriver_manager se necessário (gambiarra da mulwxta)
try:
    from webdriver_manager.chrome import ChromeDriverManager
except ModuleNotFoundError:
    os.system("pip install webdriver-manager")
    from webdriver_manager.chrome import ChromeDriverManager

from selenium import webdriver 
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# ===== CONFIG =====
ALVO = "55831234567890"  # número alvo com DDI
LOGS = "logs_whatsapp.txt"

# mensagens que dão calafrios só de ler
mensagens_falsas = [
    f"""[Instagram] Acesso suspeito detectado!
Dispositivo: Galaxy G24
Local: João Pessoa - PB
Hora: {time.strftime('%H:%M:%S')}
IP: 186.215.122.{random.randint(10, 250)}
[1] Bloquear
[2] Permitir
[3] Ver mais""",
    f"""[Instagram] Atividade incomum.
Navegador: Safari
IP: 189.203.8.{random.randint(30, 250)}
Hora: {time.strftime('%H:%M:%S')}
1. Bloquear
2. Confirmar identidade
3. Detalhes"""
]

respostas_automaticas = {
    "1": "🚫 Bloqueio aplicado. Nenhum acesso autorizado.",
    "2": "✅ Identidade confirmada. Sessão ativa.",
    "3": f"""🔎 Logs:
Navegador: Safari iOS
IP: 186.215.122.{random.randint(10, 250)}
Acesso: http://insta-veri-check.gq""",
    "senha": "🔐 Sua senha foi redefinida com sucesso.",
    "confirmar": "✅ Tudo certo. Nenhuma ameaça detectada."
}

# ===== BROWSER =====
options = Options()
options.add_argument("--user-data-dir=selenium_profile")
options.add_argument("--profile-directory=Default")
options.add_argument("--start-maximized")
options.add_argument("--disable-blink-features=AutomationControlled")

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)

# ===== WHATSAPP LOGIN =====
print("[BOT] Aguardando login no WhatsApp Web...")
driver.get("https://web.whatsapp.com")
WebDriverWait(driver, 60).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, "div[data-testid='chat-list']"))
)
print("[BOT] Login bem-sucedido.")

# ===== UTILIDADES =====
def logar(texto):
    with open(LOGS, "a", encoding="utf-8") as arq:
        timestamp = time.strftime('%H:%M:%S')
        arq.write(f"[{timestamp}] {texto}\n")

def fingir_digitando(segundos=2):
    try:
        caixa = WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "div[data-testid='conversation-compose-box-input']"))
        )
        driver.execute_script("arguments[0].focus();", caixa)
        time.sleep(random.uniform(1, segundos))
    except:
        pass

def enviar(numero, texto):
    driver.get(f"https://web.whatsapp.com/send?phone={numero}")
    try:
        campo = WebDriverWait(driver, 30).until(
            EC.element_to_be_clickable((By.CSS_SELECTOR, "div[data-testid='conversation-compose-box-input']"))
        )
        time.sleep(random.uniform(2.5, 4.5))
        campo.send_keys(texto)
        time.sleep(random.uniform(1, 2))
        campo.send_keys(Keys.ENTER)
        print("[BOT] Mensagem enviada com sucesso.")
    except Exception as err:
        print(f"[ERRO] Falha no envio: {err}")

# ===== EXECUÇÃO =====
mensagem = random.choice(mensagens_falsas)
enviar(ALVO, mensagem)

print("[BOT] Iniciando monitoramento do alvo por 10 minutos.")
tempo_final = time.time() + 600
ultima = ""

while time.time() < tempo_final:
    try:
        mensagens = driver.find_elements(By.CSS_SELECTOR, "div.message-in span.selectable-text")
        if not mensagens:
            time.sleep(2)
            continue

        texto = mensagens[-1].text.lower()
        if texto == ultima:
            time.sleep(1.5)
            continue

        ultima = texto
        print(f"[ALVO] {texto}")
        logar(f"[ALVO] {texto}")

        for chave, resposta in respostas_automaticas.items():
            if chave in texto:
                fingir_digitando(random.randint(1, 3))
                campo = driver.find_element(By.CSS_SELECTOR, "div[data-testid='conversation-compose-box-input']")
                campo.send_keys(resposta)
                time.sleep(1)
                campo.send_keys(Keys.ENTER)
                print(f"[BOT] Resposta: {resposta}")
                logar(f"[BOT] {resposta}")
                break

    except Exception as e:
        print(f"[ERRO] Falha durante a escuta: {e}")
        time.sleep(3)

print("[BOT] Encerrando sessão. Até a próxima investida.")
driver.quit()
