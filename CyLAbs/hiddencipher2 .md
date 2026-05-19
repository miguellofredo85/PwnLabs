                                    The flag is right in front of you... kind of. You just need to solve a 
basic math problem to see it. But to get the real flag, you’ll have to 
understand how that math answer is used.
You can download the program files here.
```
 ./hiddencipher2                                                                                                                                       ✔ 
What is 3 * 3? 9
Encoded flag values:
1008, 945, 891, 999, 603, 756, 630, 1107, 918, 873, 963, 909, 855, 918, 972, 873, 927, 1125
                                                                                                                                                                            

      ~/Work  file hiddencipher2                                                                                                                            ✔  3s   
hiddencipher2: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=599eedd164a0821201befcb967a2529efa0cc3ce, for GNU/Linux 3.2.0, not stripped
                                                                                                                                                                            

      ~/Work  strings hiddencipher2                                                                                                                                 ✔ 
/lib64/ld-linux-x86-64.so.2
rewind
puts
perror
__stack_chk_fail
free
fread
__isoc23_scanf
putchar
fflush
fopen
time
stdout
malloc
__libc_start_main
srand
__cxa_finalize
ftell
fclose
printf
fseek
libc.so.6
GLIBC_2.38
GLIBC_2.4
GLIBC_2.34
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
gfffH
VUUUH
Encoded flag values:
Could not open flag file
Memory error
What is %d %c %d? 
Invalid input. Exiting.
Wrong answer! No flag for you.
flag.txt
9*3$"
{"type":"deb","os":"ubuntu","name":"glibc","version":"2.42-0ubuntu3.1","architecture":"amd64"}
GCC: (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Scrt1.o
__abi_tag
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.0
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
hiddencipher2.c
__FRAME_END__
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
free@GLIBC_2.2.5
putchar@GLIBC_2.2.5
__libc_start_main@GLIBC_2.34
_ITM_deregisterTMCloneTable
stdout@GLIBC_2.2.5
puts@GLIBC_2.2.5
fread@GLIBC_2.2.5
_edata
fclose@GLIBC_2.2.5
read_flag_file
_fini
__stack_chk_fail@GLIBC_2.4
printf@GLIBC_2.2.5
rewind@GLIBC_2.2.5
srand@GLIBC_2.2.5
__isoc23_scanf@GLIBC_2.38
__data_start
ftell@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
time@GLIBC_2.2.5
malloc@GLIBC_2.2.5
fflush@GLIBC_2.2.5
_end
fseek@GLIBC_2.2.5
encode_flag
__bss_start
main
fopen@GLIBC_2.2.5
perror@GLIBC_2.2.5
generate_math_question
__TMC_END__
_ITM_registerTMCloneTable
__cxa_finalize@GLIBC_2.2.5
_init
.symtab
.strtab
.shstrtab
.note.gnu.property
.note.gnu.build-id
.interp
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.plt.sec
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.note.ABI-tag
.note.package
.init_array
.fini_array
.dynamic
.data
.bss
.comment

Si asumimos que el formato de este reto empieza por p (de picoCTF, o simplemente una inicial común) o por f (de flag):El primer valor codificado es 1008.Tu respuesta matemática fue 9.Si dividimos el valor codificado por la respuesta del usuario:$$\frac{1008}{9} = 112$$Si buscas en la tabla ASCII, el número decimal 112 corresponde exactamente al carácter: p.¡Bingo! El algoritmo de cifrado de la función encode_flag es una simple multiplicación:$$\text{Valor Codificado} = \text{Carácter ASCII} \times \text{Respuesta Matemática}$$2. Rompiendo el algoritmo (División)Dado que conocemos la clave ($9$), podemos revertir la codificación dividiendo cada uno de los valores del output por $9$ para obtener sus valores ASCII reales:Valor CodificadoDividido por 9Carácter ASCII1008$1008 / 9 = 112$p945$945 / 9 = 105$i891$891 / 9 = 99$c999$999 / 9 = 111$o603$603 / 9 = 67$C756$756 / 9 = 84$T630$630 / 9 = 70$F1107$1107 / 9 = 123${918$918 / 9 = 102$f873$873 / 9 = 97$a963$963 / 9 = 107$k909$909 / 9 = 101$e855$855 / 9 = 95$_918$918 / 9 = 102$f972$972 / 9 = 108$l873$873 / 9 = 97$a927$927 / 9 = 103$g1125$1125 / 9 = 125$}


 nc crystal-peak.picoctf.net 64646                                                                                                                     ✔ 
What is 3 + 6? 9
Encoded flag values:
1008, 945, 891, 999, 603, 756, 630, 1107, 981, 468, 1044, 936, 855, 882, 459, 936, 441, 990, 900, 855, 891, 441, 1008, 936, 459, 1026, 855, 459, 441, 504, 477, 909, 900, 450, 909, 1125

picoCTF{m4th_b3h1nd_c1ph3r_3185ed2e}
```
