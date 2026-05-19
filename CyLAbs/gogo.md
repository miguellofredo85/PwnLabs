

Solução do Desafio Gogo (PicoCTF) – Write-up1. Análise Inicial e ReconhecimentoO desafio nos fornece um binário executável chamado enter_password e um endpoint remoto para conexão via nc.Comecei executando o comando file para entender a arquitetura do arquivo:Bashfile enter_password
O retorno indicou que o arquivo é um executável ELF de 32 bits para arquitetura Intel i386, compilado em Go (Golang), estaticamente linkado, mas com dois detalhes cruciais para a engenharia reversa: not stripped (não limpo) e com debug_info.Na sequência, rodei o checksec para mapear as proteções ativas:Bashchecksec --file=enter_password
O binário não possui PIE (Position Independent Executable), o que fixa sua base de memória em 0x8048000, e não possui Stack Canaries, facilitando a leitura de estruturas locais.2. Engenharia Reversa e Primeira FaseComo o binário mantinha os símbolos originais, usei o utilitário nm para listar as funções do pacote principal (main):Bashnm enter_password | grep -i "main\."
Identifiquei as funções chave: main.main, main.checkPassword, main.ambush e main.get_flag.Abri o executável no GDB com a extensão pwndbg e analisei o disassembly da função encarregada de validar a primeira senha:Fragmento de códigopwndbg ./enter_password
break main.checkPassword
run
disass main.checkPassword
Analisando o assembly do Go, encontrei duas coisas fundamentais:Uma verificação inicial que exigia que o input tivesse exatamente 32 bytes de tamanho (cmp ecx, 0x20).Uma string de 32 caracteres hardcoded na pilha em formato Little Endian (convertida para texto como 861836f13e3d627dfa375bdb8389214e).O algoritmo realizava uma operação de XOR caractere por caractere entre o meu input, essa string fixa e um bloco de bytes estático localizado no endereço 0x810fe00 (main.statictmp_4). Recuperei esses bytes com:Fragmento de códigox/32bx 0x810fe00
Utilizando um script rápido em Python para aplicar a propriedade simétrica do XOR ($\text{Input} = \text{String} \oplus \text{Statictmp}$), reverti os bytes e obtive a primeira senha:reverseengineericanbarelyforward3. Segunda Fase: A Emboscada (main.ambush)Ao inserir a primeira senha no servidor remoto, o fluxo foi desviado para a função main.ambush, que exigia uma nova credencial com a pergunta: "What is the unhashed key?".Examinei o disassembly dessa nova função no GDB:Fragmento de códigodisass main.ambush
O mapeamento das instruções revelou a seguinte lógica criptográfica:O programa lia o meu segundo input e gerava o seu hash bruto utilizando a biblioteca nativa do Go crypto/md5.Sum.Na sequência, transformava esse resultado em uma string hexadecimal via encoding/hex.EncodeToString.Por fim, comparava a string resultante do meu hash, caractere por caractere, com a mesma assinatura de 32 bytes da primeira fase: 861836f13e3d627dfa375bdb8389214e.4. Quebra do Hash e Captura da FlagA lógica imposta pelo desafio exigia encontrar o texto limpo (colisão/pre-image) cujo hash MD5 resultasse exatamente na assinatura armazenada no binário.Submeti o hash 861836f13e3d627dfa375bdb8389214e à base de dados do CrackStation, que quebrou instantaneamente a assinatura, revelando a palavra original:goldfishConectei-me ao servidor remoto usando nc, inseri a primeira senha para passar pela validação de XOR e, quando solicitado pela "unhashed key", enviei o segredo obtido.Flag: picoCTF{p1kap1ka_p1c01a475a0d}


```

pwndbg ./enter_password                                                                                                                 0|PIPE|0 ✔ 
pwndbg: loaded 212 pwndbg commands. Type pwndbg [filter] for a list.
pwndbg: created 13 GDB functions (can be used with print/break). Type help function to see them.
Reading symbols from ./enter_password...
warning: Missing auto-load script at offset 0 in section .debug_gdb_scripts
of file /home/kali/Work/Gogo/enter_password.
Use `info auto-load python-scripts [REGEXP]' to list them.
------- tip of the day (disable with set show-tips off) -------
Use GDB's pi command to run an interactive Python console where you can use Pwndbg APIs like pwndbg.aglib.memory.read(addr, len), pwndbg.aglib.memory.write(addr, data), pwndbg.aglib.vmmap.get() and so on!                                                                                                                                            
pwndbg> break main.checkPassword
Breakpoint 1 at 0x80d4a80: file /app/enter_password.go, line 30.
pwndbg> run
Starting program: /home/kali/Work/Gogo/enter_password 
[New LWP 1741480]
[New LWP 1741481]
[New LWP 1741483]
[New LWP 1741482]
[New LWP 1741484]
Enter Password: password

Thread 1 "enter_password" hit Breakpoint 1, main.checkPassword (input=..., ~r1=172) at /app/enter_password.go:30
warning: 30     /app/enter_password.go: No such file or directory
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
───────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────────────────────────────────────
 EAX  8
 EBX  0x80
 ECX  0x1848e028 ◂— 'password'
 EDX  0
 EDI  0x1843befc ◂— 1
 ESI  0x1843bf20 ◂— 0
 EBP  0x1843beac —▸ 0x1843bf04 ◂— 0
 ESP  0x1843bf68 —▸ 0x80d48c2 (main.main+194) —▸ 0x2444b60f ◂— 0
 EIP  0x80d4a80 (main[checkPassword]) ◂— mov ecx, dword ptr gs:[0]
─────────────────────────────────────────────────────────────────────[ DISASM / i386 / set emulate on ]─────────────────────────────────────────────────────────────────────
 ► 0x80d4a80 <main[checkPassword]>       mov    ecx, dword ptr gs:[0]        ECX, [runtime.m0+80] => 0x816e8b0 (runtime.m0+80)
   0x80d4a87 <main[checkPassword]+7>     mov    ecx, dword ptr [ecx - 4]
   0x80d4a8d <main[checkPassword]+13>    cmp    esp, dword ptr [ecx + 8]
   0x80d4a90 <main[checkPassword]+16>    jbe    main[checkPassword]+237     <main[checkPassword]+237>
 
   0x80d4a96 <main[checkPassword]+22>    sub    esp, 0x44
   0x80d4a99 <main[checkPassword]+25>    mov    ecx, dword ptr [esp + 0x4c]
   0x80d4a9d <main[checkPassword]+29>    cmp    ecx, 0x20
   0x80d4aa0 <main[checkPassword]+32>    jl     main[checkPassword]+209     <main[checkPassword]+209>
 
   0x80d4aa6 <main[checkPassword]+38>    lea    edi, [esp + 4]
   0x80d4aaa <main[checkPassword]+42>    xor    eax, eax                        EAX => 0
   0x80d4aac <main[checkPassword]+44>    call   runtime[duffzero]+120       <runtime[duffzero]+120>
─────────────────────────────────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────────────────────────────────
00:0000│ esp 0x1843bf68 —▸ 0x80d48c2 (main.main+194) —▸ 0x2444b60f ◂— 0
01:0004│+0c0 0x1843bf6c —▸ 0x1848e028 ◂— 'password'
02:0008│+0c4 0x1843bf70 ◂— 8
03:000c│+0c8 0x1843bf74 —▸ 0x1843bfac —▸ 0x80e1300 (type.*+49920) ◂— 4
04:0010│+0cc 0x1843bf78 ◂— 1
... ↓        2 skipped
07:001c│+0d8 0x1843bf84 ◂— 0
───────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────────────────────────────────
 ► 0 0x80d4a80 main[checkPassword]
   1 0x80d48c2 main.main+194
   2 0x806d846 runtime[main]+518
   3 0x8090a41 runtime[goexit]+1
   4      0x0 None
───────────────────────────────────────────────────────────────────────────[ THREADS (6 TOTAL) ]────────────────────────────────────────────────────────────────────────────
  ► 1   "enter_password" stopped: 0x80d4a80 <main[checkPassword]> 
    2   "enter_password" stopped: 0x8091504 <runtime[usleep]+68> 
    3   "enter_password" stopped: 0x809183f <runtime[futex]+31> 
    4   "enter_password" stopped: 0x809183f <runtime[futex]+31> 
Not showing 2 thread(s). Use set context-max-threads <number of threads> to change this.
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> disass main.checkPassword
Dump of assembler code for function main.checkPassword:
=> 0x080d4a80 <+0>:     mov    ecx,DWORD PTR gs:0x0
   0x080d4a87 <+7>:     mov    ecx,DWORD PTR [ecx-0x4]
   0x080d4a8d <+13>:    cmp    esp,DWORD PTR [ecx+0x8]
   0x080d4a90 <+16>:    jbe    0x80d4b6d <main.checkPassword+237>
   0x080d4a96 <+22>:    sub    esp,0x44
   0x080d4a99 <+25>:    mov    ecx,DWORD PTR [esp+0x4c]
   0x080d4a9d <+29>:    cmp    ecx,0x20
   0x080d4aa0 <+32>:    jl     0x80d4b51 <main.checkPassword+209>
   0x080d4aa6 <+38>:    lea    edi,[esp+0x4]
   0x080d4aaa <+42>:    xor    eax,eax
   0x080d4aac <+44>:    call   0x8090b18 <runtime.duffzero+120>
   0x080d4ab1 <+49>:    mov    DWORD PTR [esp+0x4],0x38313638
   0x080d4ab9 <+57>:    mov    DWORD PTR [esp+0x8],0x31663633
   0x080d4ac1 <+65>:    mov    DWORD PTR [esp+0xc],0x64336533
   0x080d4ac9 <+73>:    mov    DWORD PTR [esp+0x10],0x64373236
   0x080d4ad1 <+81>:    mov    DWORD PTR [esp+0x14],0x37336166
   0x080d4ad9 <+89>:    mov    DWORD PTR [esp+0x18],0x62646235
   0x080d4ae1 <+97>:    mov    DWORD PTR [esp+0x1c],0x39383338
   0x080d4ae9 <+105>:   mov    DWORD PTR [esp+0x20],0x65343132
   0x080d4af1 <+113>:   lea    edi,[esp+0x24]
   0x080d4af5 <+117>:   lea    esi,ds:0x810fe00
   0x080d4afb <+123>:   call   0x8090fe0 <runtime.duffcopy+1200>
   0x080d4b00 <+128>:   mov    ecx,DWORD PTR [esp+0x48]
   0x080d4b04 <+132>:   mov    edx,DWORD PTR [esp+0x4c]
   0x080d4b08 <+136>:   xor    eax,eax
   0x080d4b0a <+138>:   xor    ebx,ebx
   0x080d4b0c <+140>:   jmp    0x80d4b0f <main.checkPassword+143>
   0x080d4b0e <+142>:   inc    eax
   0x080d4b0f <+143>:   cmp    eax,0x20
   0x080d4b12 <+146>:   jge    0x80d4b3a <main.checkPassword+186>
   0x080d4b14 <+148>:   cmp    eax,edx
   0x080d4b16 <+150>:   jae    0x80d4b66 <main.checkPassword+230>
   0x080d4b18 <+152>:   movzx  ebp,BYTE PTR [ecx+eax*1]
   0x080d4b1c <+156>:   cmp    eax,0x20
   0x080d4b1f <+159>:   jae    0x80d4b66 <main.checkPassword+230>
   0x080d4b21 <+161>:   movzx  esi,BYTE PTR [esp+eax*1+0x4]
   0x080d4b26 <+166>:   xor    ebp,esi
   0x080d4b28 <+168>:   movzx  esi,BYTE PTR [esp+eax*1+0x24]
   0x080d4b2d <+173>:   xchg   ebp,eax
   0x080d4b2e <+174>:   xchg   esi,ebx
   0x080d4b30 <+176>:   cmp    al,bl
   0x080d4b32 <+178>:   xchg   esi,ebx
   0x080d4b34 <+180>:   xchg   ebp,eax
   0x080d4b35 <+181>:   jne    0x80d4b0e <main.checkPassword+142>
   0x080d4b37 <+183>:   inc    ebx
   0x080d4b38 <+184>:   jmp    0x80d4b0e <main.checkPassword+142>
   0x080d4b3a <+186>:   cmp    ebx,0x20
   0x080d4b3d <+189>:   jne    0x80d4b48 <main.checkPassword+200>
   0x080d4b3f <+191>:   mov    BYTE PTR [esp+0x50],0x1
   0x080d4b44 <+196>:   add    esp,0x44
   0x080d4b47 <+199>:   ret
   0x080d4b48 <+200>:   mov    BYTE PTR [esp+0x50],0x0
   0x080d4b4d <+205>:   add    esp,0x44
   0x080d4b50 <+208>:   ret
   0x080d4b51 <+209>:   mov    DWORD PTR [esp],0x0
   0x080d4b58 <+216>:   call   0x80b3ee0 <os.Exit>
   0x080d4b5d <+221>:   mov    ecx,DWORD PTR [esp+0x4c]
   0x080d4b61 <+225>:   jmp    0x80d4aa6 <main.checkPassword+38>
   0x080d4b66 <+230>:   call   0x806aaa0 <runtime.panicindex>
   0x080d4b6b <+235>:   ud2
   0x080d4b6d <+237>:   call   0x808f0f0 <runtime.morestack_noctxt>
   0x080d4b72 <+242>:   jmp    0x80d4a80 <main.checkPassword>
End of assembler dump.
pwndbg> x/32bx 0x810fe00
0x810fe00 <main.statictmp_4>:   0x4a    0x53    0x47    0x5d    0x41    0x45    0x03    0x54
0x810fe08 <main.statictmp_4+8>: 0x5d    0x02    0x5a    0x0a    0x53    0x57    0x45    0x0d
0x810fe10 <main.statictmp_4+16>:        0x05    0x00    0x5d    0x55    0x54    0x10    0x01    0x0e
0x810fe18 <main.statictmp_4+24>:        0x41    0x55    0x57    0x4b    0x45    0x50    0x46    0x01
pwndbg> run
Starting program: /home/kali/Work/Gogo/enter_password 
[New LWP 1747414]
[New LWP 1747415]
[New LWP 1747416]
[New LWP 1747417]
Enter Password: reverseengineeringcanbarelyforward

Thread 1 "enter_password" hit Breakpoint 1, main.checkPassword (input=..., ~r1=172) at /app/enter_password.go:30
30      in /app/enter_password.go
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
───────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────────────────────────────────────
 EAX  0x22
 EBX  0
 ECX  0x18414120 ◂— 'reverseengineeringcanbarelyforward'
 EDX  0
 EDI  0x1843befc ◂— 1
 ESI  0x1843bf20 ◂— 0
 EBP  0x1843beac —▸ 0x1843bf04 ◂— 0
 ESP  0x1843bf68 —▸ 0x80d48c2 (main.main+194) —▸ 0x2444b60f ◂— 0
 EIP  0x80d4a80 (main[checkPassword]) ◂— mov ecx, dword ptr gs:[0]
─────────────────────────────────────────────────────────────────────[ DISASM / i386 / set emulate on ]─────────────────────────────────────────────────────────────────────
 ► 0x80d4a80 <main[checkPassword]>       mov    ecx, dword ptr gs:[0]        ECX, [runtime.m0+80] => 0x816e8b0 (runtime.m0+80)
   0x80d4a87 <main[checkPassword]+7>     mov    ecx, dword ptr [ecx - 4]
   0x80d4a8d <main[checkPassword]+13>    cmp    esp, dword ptr [ecx + 8]
   0x80d4a90 <main[checkPassword]+16>    jbe    main[checkPassword]+237     <main[checkPassword]+237>
 
   0x80d4a96 <main[checkPassword]+22>    sub    esp, 0x44
   0x80d4a99 <main[checkPassword]+25>    mov    ecx, dword ptr [esp + 0x4c]
   0x80d4a9d <main[checkPassword]+29>    cmp    ecx, 0x20
   0x80d4aa0 <main[checkPassword]+32>    jl     main[checkPassword]+209     <main[checkPassword]+209>
 
   0x80d4aa6 <main[checkPassword]+38>    lea    edi, [esp + 4]
   0x80d4aaa <main[checkPassword]+42>    xor    eax, eax                        EAX => 0
   0x80d4aac <main[checkPassword]+44>    call   runtime[duffzero]+120       <runtime[duffzero]+120>
─────────────────────────────────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────────────────────────────────
00:0000│ esp 0x1843bf68 —▸ 0x80d48c2 (main.main+194) —▸ 0x2444b60f ◂— 0
01:0004│+0c0 0x1843bf6c —▸ 0x18414120 ◂— 'reverseengineeringcanbarelyforward'
02:0008│+0c4 0x1843bf70 ◂— 0x22 /* '"' */
03:000c│+0c8 0x1843bf74 —▸ 0x1843bfac —▸ 0x80e1300 (type.*+49920) ◂— 4
04:0010│+0cc 0x1843bf78 ◂— 1
... ↓        2 skipped
07:001c│+0d8 0x1843bf84 ◂— 0
───────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────────────────────────────────
 ► 0 0x80d4a80 main[checkPassword]
   1 0x80d48c2 main.main+194
   2 0x806d846 runtime[main]+518
   3 0x8090a41 runtime[goexit]+1
   4      0x0 None
───────────────────────────────────────────────────────────────────────────[ THREADS (5 TOTAL) ]────────────────────────────────────────────────────────────────────────────
  ► 1   "enter_password" stopped: 0x80d4a80 <main[checkPassword]> 
    2   "enter_password" stopped: 0x8091504 <runtime[usleep]+68> 
    3   "enter_password" stopped: 0x809183f <runtime[futex]+31> 
    4   "enter_password" stopped: 0x809183f <runtime[futex]+31> 
Not showing 1 thread(s). Use set context-max-threads <number of threads> to change this.
──────────────────────────────────────────────────────────────────────────────────────────────────

python3 -c 'c=[0x38313638,0x31663633,0x64336533,0x64373236,0x37336166,0x62646235,0x39383338,0x65343132]; s=[]; [s.extend([x&0xff,(x>>8)&0xff,(x>>16)&0xff,(x>>24)&0xff]) for x in c]; b=[0x4a,0x53,0x47,0x5d,0x41,0x45,0x03,0x54,0x5d,0x02,0x5a,0x0a,0x53,0x57,0x45,0x0d,0x05,0x00,0x5d,0x55,0x54,0x10,0x01,0x0e,0x41,0x55,0x57,0x4b,0x45,0x50,0x46,0x01]; print("".join(chr(x^y) for x,y in zip(s,b)))'

reverseengineericanbarelyforward
reverseengineeringcanbarelyforward

```

861836f13e3d627dfa375bdb8389214e	md5	goldfish

        nc wily-courier.picoctf.net 65352                                                                                                       ✔  20s   
Enter Password: reverseengineericanbarelyforward
=========================================
This challenge is interrupted by psociety
What is the unhashed key?
goldfish
Flag is:  picoCTF{p1kap1ka_p1c01a475a0d}
