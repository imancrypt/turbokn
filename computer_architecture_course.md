# Kompüter Arxitekturası: x86_64 və ARM Assembly ilə Dərin Öyrənmə

## Giriş
Bu kurs, kompüter arxitekturasını universitet nəzəriyyəsindən fərqli olaraq, **praktiki Assembly (x86_64 və ARM)** və **C/C++** kodları üzərində qarşılaşdırma yolu ilə öyrədir. Məqsəd, yüksək səviyyəli kodun donanımda necə işlədiyini, registerlərin, yaddaşın və idarəetmə axınının real həyatda necə idarə olunduğunu başa düşməkdir.

---

## 1. Fəlsəfə Fərqi: CISC vs RISC

### x86_64 (CISC - Complex Instruction Set Computer)
- **Məntiq:** "Kompleks əmrlər". Bir əmrlə çoxlu iş görə bilər (məsələn, birbaşa yaddaşdan oxumaq və toplamaq).
- **Xüsusiyyətlər:**
  - Dəyişən uzunluqlu əmrlər (1 baytdan 15 bayta qədər).
  - Az sayda ümumi məqsədli registerlər (RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, R8-R15).
  - Birbaşa yaddaş əməliyyatlarına icazə verir (`add [mem], reg`).
  - Geriyə uyğunluq (backward compatibility) çox güclüdür.

### ARM (RISC - Reduced Instruction Set Computer)
- **Məntiq:** "Sadələşdirilmiş əmrlər". Hər əmr sadədir və adətən bir saat dövrəsində (cycle) icra olunur.
- **Xüsusiyyətlər:**
  - Sabit uzunluqlu əmrlər (ARMv8-də 32 bit, Thumb rejimində 16 bit).
  - Çoxlu registerlər (R0-R30, SP, PC, x0-x30 64-bit rejimdə).
  - **Load/Store Arxitekturası:** Yalnız `LOAD` və `STORE` əmrləri yaddaşa girib-çıxa bilər. Riyazi əməliyyatlar yalnız registerlər arasında aparılır.
  - Enerji səmərəliliyi yüksəkdir (mobil cihazların əsas səbəbi).

---

## 2. Registerlərin Müqayisəsi

| Xüsusiyyət | x86_64 (Intel/AMD) | ARM64 (AArch64) |
| :--- | :--- | :--- |
| **Ümumi Məqsədli** | RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, R8-R15 | X0-X30 (W0-W30 32-bit üçün) |
| **Akumulyator** | RAX (nəticələr burada saxlanılır) | X0 (həm nəticə, həm də 1-ci arqument) |
| **Stack Pointer** | RSP | SP (X31 kimi davranır) |
| **Instruction Ptr** | RIP | PC (Proqram Sayğacı, birbaşa əlçatan deyil) |
| **Bayt Endianlıq** | Little Endian | Little Endian (standart), Big Endian da mümkündür |

---

## 3. Praktiki Nümunə: Sadə Toplama Funksiyası

Gəlin eyni C funksiyasının hər iki memarida necə Assembly-yə çevrildiyinə baxaq.

### C Kodu
```c
long add_numbers(long a, long b) {
    return a + b;
}
```

### x86_64 Assembly (AT&T Sintaksisi - Linux/GCC)
```asm
add_numbers:
    # a -> %rdi, b -> %rsi (System V AMD64 ABI çağırış konvensiyası)
    movq    %rsi, %rax      # b-ni RAX-a köçür (nəticə RAX-da qaytarılır)
    addq    %rdi, %rax      # a-nı RAX-a əlavə et (RAX = RAX + RDI)
    ret                     # Geri qayıt
```
*Qeyd:* x86-da birbaşa `add %rdi, %rsi` edib `mov %rsi, %rax` da edə bilərdik, amma nəticə mütləq `RAX`-da olmalıdır.

### ARM64 Assembly
```asm
add_numbers:
    # a -> x0, b -> x1 (AAPCS64 çağırış konvensiyası)
    add     x0, x1, x0      # x0 = x1 + x0 (Birbaşa nəticəni x0-a yazır)
    ret                     # Geri qayıt (x30 registerinə jump edir)
```
*Mühüm Fərq:* ARM-də nəticəni ayrıca bir registerə köçürməyə ehtiyac yoxdur, çünki artıq birinci arqument (`x0`) qaytarış dəyəri üçün istifadə olunur. Əmr daha sadədir: `ADD destination, operand1, operand2`.

---

## 4. Yaddaş Erişimi və Load/Store Modeli

Bu, universitetdə ən çox qarışıq salınan, lakin praktikada ən vacib mövzudur.

### Ssenari: `A = B + C` (Burada A, B, C yaddaş ünvanlarıdır)

#### x86_64 Yanaşması (CISC)
x86 birbaşa yaddaşla riyazi əməliyyat apara bilir.
```asm
mov     rax, [B]      # B-ni yaddaşdan RAX-a yüklə
add     rax, [C]      # C-ni yaddaşdan birbaşa RAX-a əlavə et
mov     [A], rax      # Nəticəni A yaddaşına yaz
```
*Və ya hətta daha qısa (lakin təhlükəli ola bilən):*
```asm
mov     rax, [B]
add     [A], rax      # Birbaşa yaddaş ünvanına [A] əlavə et (C-nin yüklenmesi lazımdır əvvəlcə)
; Doğrusu:
mov     rbx, [C]
add     [A], rbx
```
x86允许 (icazə verir) `add [memory], register`.

#### ARM64 Yanaşması (RISC - Load/Store)
ARM-da riyazi əməliyyat **yalnız** registerlər arasında olur. Yaddaşa birbaşa toxuna bilməzsiniz.
```asm
ldr     x1, [B]       # LOAD: B-ni yaddaşdan x1-ə yüklə
ldr     x2, [C]       # LOAD: C-ni yaddaşdan x2-yə yüklə
add     x3, x1, x2    # ADD: Yalnız registerlərlə topla (x3 = x1 + x2)
str     x3, [A]       # STORE: Nəticəni yaddaşa (A) yaz
```
*Dərs:* ARM kodu daha uzun görünə bilər, lakin borucuq (pipeline) dizaynı üçün daha proqnozlaşdırılrandır.

---

## 5. Şərti Əmrlər və Dallanma (Branching)

### C Kodu
```c
if (a > b) {
    return 1;
} else {
    return 0;
}
```

#### x86_64
x86 adətən `CMP` (müqayisə) əmri ilə bayraqları (flags) qurur, sonra `JMP` (sıçrayış) edir.
```asm
compare_vals:
    cmp     %rsi, %rdi      # a (rdi) - b (rsi) müqayisəsi
    jle     else_block      # Əgər a <= bdirsə, else_block-a get
    mov     $1, %rax        # True halı
    ret
else_block:
    mov     $0, %rax        # False halı
    ret
```

#### ARM64
ARM da `CMP` istifadə edir, lakin şərtli əmrlər daha fleksibeldir. Hətta bəzi hallarda sıçrayışsız (branchless) kod yazmaq asandır.
```asm
compare_vals:
    cmp     x0, x1          # x0 - x1 müqayisəsi
    cset    x0, gt          # Condition Set: Əgər Greater Than (gt) dirsə, x0=1, əks halda 0
    ret                     # Birbaşa qayıt, heç bir jump olmadı!
```
*Qabaqcıl Dərs:* ARM-in `CSET` kimi əmrləri sayəsində "Branch Prediction" (Şərtli Gözləmə) xətasından qaçmaq daha asandır. Bu, performans üçün kritiktir.

---

## 6. Stack və Funksiya Çağırışları

### Calling Conventions (Çağırış Konvensiyaları)
Funksiyalar arqumentləri necə ötürür və kim registerləri qoruyur?

| Xüsusiyyət | x86_64 (System V - Linux) | ARM64 (Linux/Android) |
| :--- | :--- | :--- |
| **Arqumentlər** | RDI, RSI, RDX, RCX, R8, R9 (ilk 6) | X0-X7 (ilk 8) |
| **Qaytarış** | RAX | X0 |
| **Caller-Saved** | RAX, RCX, RDX, RSI, RDI, R8-R11 (Çağıran qorumalıdır) | X0-X8, X16-X18 |
| **Callee-Saved** | RBX, RBP, R12-R15 (Çağırılan qorumalıdır) | X19-X29 |
| **Stack Frame** | RBP adətən frame pointer kimi işlədilir | FP (X29) və Link Register (LR/X30) |

### Stack Çərçivəsi (Frame) Qurulması
**x86_64:**
```asm
push    %rbp            # Köhnə frame pointer-i yaddaşa at
mov     %rsp, %rbp      # Yeni frame pointer-i təyin et
sub     $32, %rsp       # Yerli dəyişənlər üçün yer ayır
...
leave                   # mov %rbp, %rsp; pop %rbp (Əks əməliyyat)
ret
```

**ARM64:**
```asm
stp     x29, x30, [sp, #-16]!   # Frame Pointer (x29) və Link Register (x30)-i yaddaşa yaz, SP-ni azalt
mov     x29, sp                 # Yeni Frame Pointer-i təyin et
sub     sp, sp, #32             # Yerli dəyişənlər üçün əlavə yer
...
ldp     x29, x30, [sp], #16     # Dəyərləri geri yüklə və SP-ni artır
ret                             # x30 (LR)-ə qayıt
```
*Qeyd:* ARM-də `Link Register (LR/x30)` funksiyaya çağırış ünvanını saxlayır. `BL` (Branch with Link) əmri çağırış zamanı avtomatik olaraq növbəti əmrin ünvanını X30-a yazır. x86-da isə bu ünvan `CALL` əmri tərəfindən birbaşa Stack-ə atılır (`RET` stack-dən oxuyur).

---

## 7. Niyə Bu Yanaşma Universitetdən Daha Yaxşıdır?

1.  **Optimizasiyanı Başa Düşmək:** Derleyicinin (`gcc -O2`, `-O3`) niyə bəzi dövrələri (loops) açdığını (unroll) və ya dəyişənləri registerlərdə saxladığını assembly səviyyəsində görürsünüz.
2.  **Debugging Gücü:** `gdb` və ya `lldb` istifadə edərkən, proqramın niyə çökdüyünü (segfault) yalnız C koduna baxaraq yox, registerlərin vəziyyətinə baxaraq dəqiq tapa bilirsiniz.
3.  **Platform Bağımsızlıq:** Mobil (ARM) və Server (x86) dünyası arasındakı fərqləri bildiyiniz üçün, cross-platform proqramlarda performans dar boğazlarını (bottlenecks) daha yaxşı analiz edirsiniz.
4.  **Güvenlik:** Buffer overflow, ROP (Return Oriented Programming) və Stack smashing hücumlarının necə işlədiyini yalnız bu səviyyədə tam dərk etmək olar.

---

## Tapşırıq: Özünüz Yoxlayın

Aşağıdakı addımları yerinə yetirin:
1.  Yuxarıdakı `add_numbers` funksiyasını C dilində yazın.
2.  **x86_64-də:** `gcc -S -O2 add.c` əmrini işlədin və `.s` faylında assembly kodunu incələyin.
3.  **ARM-də:** Əgər ARM cihazınız (Raspberry Pi, M1/M2 Mac, və ya Android telefon) varsa, orada eyni əmri işlədin. Yoxdursa, `qemu-user-static` və ya onlayn Compiler Explorer (godbolt.org) istifadə edərək ARM64 çıxışına baxın.
4.  Fərqləri qeyd edin: Neçə əmr var? Hangi registerlər istifadə olunub?

