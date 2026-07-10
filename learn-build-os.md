# Sıfırdan İşletim Sistemi İnşa Etmek: Mutlak Başlangıç Rehberi

> **"Recipe for yogurt: Add yogurt to milk."** — Bootstrapping özdeyişi
>
> Bu belge, elinizde sadece bir CPU varken, BIOS'tan kernel'e, derleyiciden
> kullanıcı arayüzüne kadar her şeyi **sıfırdan** nasıl inşa edeceğinizi
> anlatır. Her katman bir öncekine bağımlıdır. Her araç kendinden öncekiyle
> inşa edilir.

---

## İçindekiler

1. [Giriş: Neden Sıfırdan?](#1-giriş-neden-sıfırdan)
2. [Faz 0: Donanım ve Firmware](#2-faz-0-donanım-ve-firmware)
3. [Faz 1: Bootloader (Önyükleyici)](#3-faz-1-bootloader-önyükleyici)
4. [Faz 2: Derleyici Bootstrap Zinciri](#4-faz-2-derleyici-bootstrap-zinciri)
5. [Faz 3: Kernel Çekirdeği](#5-faz-3-kernel-çekirdeği)
6. [Faz 4: Init ve Userspace](#6-faz-4-init-ve-userspace)
7. [Faz 5: Kendi Kendini Derleme (Full Source Bootstrap)](#7-faz-5-kendi-kendini-derleme-full-source-bootstrap)
8. [Faz 6: Gelişmiş İşletim Sistemi](#8-faz-6-gelişmiş-işletim-sistemi)
9. [Referans Projeler ve Kod Kaynakları](#9-referans-projeler-ve-kod-kaynakları)
10. [Öğrenme Yol Haritası](#10-öğrenme-yol-haritası)
11. [Tam Referans Listesi (Linkler)](#11-tam-referans-listesi-linkler)

---

## 1. Giriş: Neden Sıfırdan?

Normal bir işletim sistemi kurarken var olan bir derleyici, bootloader ve
kernel kullanırız. Peki ya hiçbiri yoksa? O zaman **her şeyi en temel
parçalardan başlayarak** inşa etmek zorundayız. Buna **bootstrapping** denir.

### Bootstrapping Problemi

```
GCC derleyicisi → C kaynak kodunu derler
Peki GCC'yi kim derledi? → Daha eski bir GCC
Peki ilk GCC'yi kim derledi? → ...
```

Bu sonsuz döngüyü kırmak için, elle yazılmış **küçük bir ikili dosya**
(binary seed) gerekir. Stage0 projesi bunu **512 byte**'a indirmiştir.

### Ken Thompson'ın Ünlü Saldırısı

1984 Turing Award konuşmasında Ken Thompson, bir derleyiciye **arka kapı**
(backdoor) yerleştirmenin ne kadar kolay olduğunu gösterdi:
- Derleyici, login koduna arka kapı ekleyecek şekilde değiştirildi
- Derleyicinin kendisi de bu değişikliği fark etmeyecek şekilde
  **kendini** değiştirdi
- Kaynak koddan arka kapı silinse bile, binary'de kaldı

**Sonuç:** Binary'ye güvenemezsin. Tek çözüm: **tüm yazılımı kaynaktan,
sıfırdan inşa etmek.**

---

## 2. Faz 0: Donanım ve Firmware

### 2.1 CPU'nun İlk Adımları

Bir x86 CPU açıldığında:

1. **Reset vektörüne gider:** `0xFFFFFFF0` (16-bit mod)
2. **Orada yazılı olan kodu çalıştırır** — bu BIOS/EFI'dir
3. Ama BIOS yoksa, o adreste **sizin kodunuz** olmalı

### 2.2 Minimal Firmware Yazmak

CPU'yu çalışır hale getirmek için gereken en temel adımlar:

```
1. Cache-as-RAM (CAR) kurulumu
   - RAM henüz başlatılmamıştır
   - L2/L3 önbellek geçici RAM olarak kullanılır

2. Memory Controller başlatma
   - SPD (Serial Presence Detect) okuma
   - DIMM timing ayarları
   - Channel training

3. Chipset başlatma
   - PCI config space erişimi
   - Güç yönetimi (ACPI)

4. Interrupt Controller (PIC/APIC)

5. Sistem saati (HPET/PIT)

6. Boot device bulma
   - SATA/NVMe/eMMC controller ilkel sürücüsü
   - Blok cihazdan Stage 2 bootloader okuma
```

> **⚠️ Bu adım tek başına aylar sürebilir.** Her anakarta özel kod gerekir.
> Açık kaynak çözüm: **coreboot** (ancak bunu da bir yerden almanız gerekir)

### Referans Projeler

| Proje | Açıklama |
|-------|----------|
| [coreboot](https://www.coreboot.org) | Açık kaynak firmware. Anakart spesifik |
| [UEFI spec](https://uefi.org/specifications) | Modern firmware standardı |
| [Bare metal programming guides](https://wiki.osdev.org/Bare_Bones) | En basit boot eden kernel |

---

## 3. Faz 1: Bootloader (Önyükleyici)

BIOS/ROM'dan gelen kod diskten bir sonraki aşamayı yükler.

### 3.1 Aşamalar

#### Stage 1 (512 byte — Master Boot Record)

BIOS, boot cihazının ilk 512 byte'ını `0x7C00` adresine yükler ve çalıştırır.

```nasm
; En basit boot sector (NASM)
[org 0x7C00]
bits 16

start:
    mov ah, 0x0E          ; Teletype output
    mov al, 'H'
    int 0x10              ; BIOS video interrupt

    jmp $                 ; Sonsuz döngü

times 510-($-$$) db 0
dw 0xAA55                 ; Boot signature
```

Bu kadar basit başlar. Gerçek bir bootloader çok daha fazlasını yapar:
- **A20 gate** açma (1 MB üstü belleğe erişim)
- **Protected mode** / **Long mode** geçiş
- **Page tables** kurulumu (64-bit)
- **GDT/IDT** yapılandırma
- Kernel imajını diskten okuma (`0x100000` adresine yükleme)
- `jmp` ile kernel'a atlama

#### Stage 2 (Daha büyük, C ile yazılabilir)

```c
// Sözde kod: Kernel yükleyici
void bootloader_main() {
    init_screen();
    enable_a20();
    init_gdt();
    init_idt();
    enter_protected_mode();
    init_page_tables();
    enter_long_mode();
    load_kernel_from_disk();
    jump_to_kernel();
}
```

### Önemli Kavramlar

| Kavram | Açıklama |
|--------|----------|
| **Real Mode** | 16-bit, 1 MB adres alanı, BIOS kesmeleri var |
| **Protected Mode** | 32-bit, 4 GB, segmentation + paging |
| **Long Mode** | 64-bit, sanal adres alanı, paging zorunlu |
| **A20 Gate** | Tarihi uyumluluk kapısı (1 MB wrap) |
| **GDT** | Global Descriptor Table — bellek segmentleri |
| **IDT** | Interrupt Descriptor Table — kesme yönlendirme |
| **Paging** | Sanal → fiziksel adres dönüşümü |

### Referans Projeler

| Proje | Detay |
|-------|-------|
| [cfenollosa/os-tutorial](https://github.com/cfenollosa/os-tutorial) | 30587 ★ — Adım adım boot'tan kernel'a |
| [gmarino2048/64bit-os-tutorial](https://github.com/gmarino2048/64bit-os-tutorial) | 64-bit long mode girişli |
| [JamesM's kernel tutorials](https://web.archive.org/web/20160412174753/http://www.jamesmolloy.co.uk/tutorial_html/index.html) | Klasik, çok kapsamlı |
| [littleosbook](https://littleosbook.github.io) | Küçük ama eksiksiz OS kitabı |

---

## 4. Faz 2: Derleyici Bootstrap Zinciri

Kernel yazmak için C derleyicisi gerekir. Peki C derleyicisini ne derler?

### 4.1 Tarihten: İlk Derleyici Nasıl Yapıldı?

```
1. Elle makine kodu yaz (hex)
2. Bununla basit bir assembler yap (hex → binary)
3. Assembler ile minimal C derleyici yap
4. Minimal C derleyici ile daha iyi C derleyici yap
5. Bu derleyici GCC'yi derleyebilecek seviyeye gel
6. GCC ile her şeyi derle
```

### 4.2 Stage0: Neredeyse Sıfırdan

Jeremiah Orians'ın **stage0** projesi, sadece elle yazılmış **birkaç hex
byte** ile başlayıp tam bir C derleyicisine ulaşan zinciri sağlar.

```
Zincir:
hex0 (elle, ~512 byte)
  → hex1 (assembler benzeri)
  → hex2 (daha yetenekli assembler)
  → M2-Planet (minimal C altkümesi derleyicisi)
  → mescc-tools (linker, loader)
  → GNU Mes (C derleyici + Scheme yorumlayıcı)
  → TinyCC (taşınabilir C derleyicisi)
  → GCC (tam C/C++/Fortran derleyicisi)
  → Linux kernel ve her şey
```

#### hex0 (512 byte)

hex0, bir hex düzenleyicidir. Elle makine kodunda yazılmıştır.
Girdi olarak hex değerleri alır, çıktı olarak ikili dosya üretir.

```hex0
; Örnek: hex0 ile boot sector yazmak
; Her satır: offset: hex_bytes
00000000: B4 0E B0 48 CD 10 EB FE       ; mov ah,0x0E; mov al,'H'; int 0x10; jmp $
00000007: 00 00 ... 00 00                ; padding
000001FE: 55 AA                           ; boot signature
```

### 4.3 M2-Planet

**The PLAtform NEutral Transpiler** — minimal bir C altkümesini derler.
Kendini derleyebilir (self-hosting). Çapraz platform çalışır.

**M2-Planet'in derlediği C altkümesi:**
- `int`, `char`, `pointer` tipleri
- `if/else`, `while`, `for`
- `struct`, `union`
- Fonksiyon çağrıları (kısıtlı)
- **Yok:** `float`, `double`, `long long`, `switch`, `goto`, `inline`

### 4.4 GNU Mes

GNU Mes, **Scheme yorumlayıcı** + **C derleyici** ikilisidir.

- Scheme yorumlayıcı: ~5000 satır C
- C derleyici (MesCC): Scheme ile yazılmış
- **Birbirini derler** (mutual self-hosting)
- MesCC, TinyCC'yi derleyebilir
- TinyCC, GCC'yi derleyebilir

### Temel Kavramlar

| Terim | Açıklama |
|-------|----------|
| **Self-hosting** | Bir programın kendini derleyebilmesi |
| **Cross-compiler** | Farklı bir hedef için derleme yapan derleyici |
| **Tombstone diagram** | Derleyici zincirini gösteren diyagram |
| **Binary seed** | Elle yazılmış, güvenilen ilk ikili dosya |
| **Full Source Bootstrap** | Tüm sistemin sadece kaynak koddan inşa edilmesi |

### Referans Projeler

| Proje | Açıklama |
|-------|----------|
| [stage0](https://github.com/oriansj/stage0) | 1048 ★ — 512 byte ile başlayan bootstrap zinciri |
| [M2-Planet](https://github.com/oriansj/m2-planet) | Minimal C transpiler, kendini derler |
| [GNU Mes](https://gnu.org/software/mes/) | Scheme + C derleyici, full source bootstrap |
| [live-bootstrap](https://github.com/fosslinux/live-bootstrap) | Otomatik, tekrarlanabilir, uçtan uca bootstrap |
| [chibicc](https://github.com/rui314/chibicc) | 11743 ★ — ~5000 satırda C11 derleyicisi |
| [8cc](https://github.com/rui314/8cc) | 6398 ★ — Küçük C derleyici |
| [amacc](https://github.com/jserv/amacc) | 1059 ★ — ARM için küçük C derleyici |
| [bootstrappable](https://bootstrappable.org) | Bootstrappable builds topluluğu |
| [bootstrapping wiki](https://bootstrapping.miraheze.org) | Bootstrap teorisi ve projeleri |

---

## 5. Faz 3: Kernel Çekirdeği

Derleyiciniz çalıştığına göre, artık bir işletim sistemi çekirdeği
yazabilirsiniz.

### 5.1 Minimal Kernel Yapısı

```
kernel/
├── boot/          # Assembly boot kodları
│   ├── boot.asm   # Multiboot header, GDT, long mode giriş
│   └── page.asm   # Page table setup
├── kernel/        # C çekirdek
│   ├── main.c     # Kernel girişi
│   ├── memory.c   # Fiziksel/sanal bellek yöneticisi
│   ├── scheduler.c# Zamanlayıcı
│   ├── interrupt.c# IDT, IRQ, exception handler
│   ├── syscall.c  # Sistem çağrıları
│   ├── timer.c    # APIC/PIT zamanlayıcı
│   └── elf.c      # ELF ikili yükleyici
├── drivers/       # Aygıt sürücüleri
│   ├── keyboard.c
│   ├── screen.c   # VGA text mode / framebuffer
│   ├── serial.c
│   └── disk.c     # ATA/IDE (PIO mod)
├── fs/            # Dosya sistemi
│   ├── vfs.c      # Sanal dosya sistemi soyutlaması
│   ├── initrd.c   # Başlangıç RAM diski
│   └── ext2.c     # ext2 okuma
└── lib/           # Yardımcı fonksiyonlar
    ├── printf.c
    ├── string.c
    └── malloc.c
```

### 5.2 Kernel Bileşenleri (Sırayla)

#### A. Bellek Yöneticisi

```c
// Fiziksel sayfa tahsisi (bitmap ile)
#define PAGE_SIZE 4096
#define TOTAL_PAGES (64 * 1024 * 1024 / PAGE_SIZE)  // 64 MB

uint8_t page_bitmap[TOTAL_PAGES / 8];

void* alloc_page() {
    for (int i = 0; i < TOTAL_PAGES; i++) {
        if (!(page_bitmap[i/8] & (1 << (i%8)))) {
            page_bitmap[i/8] |= (1 << (i%8));
            return (void*)(i * PAGE_SIZE);
        }
    }
    return NULL;  // Bellek yetersiz
}

void free_page(void* addr) {
    int i = (int)addr / PAGE_SIZE;
    page_bitmap[i/8] &= ~(1 << (i%8));
}
```

Ardından sanal bellek yönetimi (paging):

- **x86-64:** 4-level page tables (PML4 → PDP → PD → PT)
- Her sayfa tablosu girişi: 8 byte
- Sayfa: 4 KB (standart) veya 2 MB (huge page)
- `CR3` register'ı PML4 adresini tutar

#### B. Zamanlayıcı (Scheduler)

```c
typedef struct process {
    uint32_t pid;
    uint32_t *stack;       // Kernel stack
    uint32_t *page_dir;    // Sayfa tablosu
    struct context ctx;     // Register durumu
    enum state { READY, RUNNING, BLOCKED, ZOMBIE } state;
    struct process *next;
} process_t;

// Round-robin scheduler
void schedule() {
    current->state = READY;
    current = current->next;
    while (current->state != READY)
        current = current->next;
    current->state = RUNNING;
    switch_context();  // Context switch (assembly)
}
```

#### C. Kesme Yönetimi

- **IDT (Interrupt Descriptor Table):** Her kesme için handler adres
- **IRQ (Interrupt Request):** Donanım kesmeleri (klavye, saat, disk)
- **Exception:** CPU hataları (page fault, divide by zero)
- **System Call (int 0x80 / syscall):** Kullanıcı → kernel geçişi

```c
void interrupt_handler(struct registers *r) {
    switch (r->int_no) {
        case 0x0E:  // Page fault
            handle_page_fault(r);
            break;
        case 0x20:  // PIT timer
            tick++;
            if (tick % SCHEDULER_TICKS == 0)
                schedule();
            break;
        case 0x21:  // Keyboard
            char c = inb(0x60);  // PS/2 data port
            keyboard_buffer[keyboard_head++] = c;
            break;
        case 0x80:  // System call
            handle_syscall(r->eax, r->ebx, r->ecx, r->edx);
            break;
    }
}
```

#### D. Sistem Çağrıları

```c
// Kullanıcı tarafından çağrılabilir en temel fonksiyonlar
void handle_syscall(uint32_t nr, uint32_t arg1, uint32_t arg2, uint32_t arg3) {
    switch (nr) {
        case 0: exit(arg1);
        case 1: write(arg1, (char*)arg2, arg3);   // fd, buf, count
        case 2: read(arg1, (char*)arg2, arg3);
        case 3: open((char*)arg1, arg2);
        case 4: close(arg1);
        case 5: fork();
        case 6: execve((char*)arg1, (char**)arg2, (char**)arg3);
        case 7: waitpid(arg1);
    }
}
```

#### E. ELF Yükleyici

Kernel'in çalıştırabileceği ikili dosyaların formatını anlaması gerekir.

```c
void load_elf(uint8_t *elf_data) {
    Elf64_Ehdr *ehdr = (Elf64_Ehdr*)elf_data;
    // Validate magic: 0x7F 'E' 'L' 'F'
    if (ehdr->e_ident[0] != 0x7F) return;

    // Load each program segment
    Elf64_Phdr *phdr = (Elf64_Phdr*)(elf_data + ehdr->e_phoff);
    for (int i = 0; i < ehdr->e_phnum; i++) {
        if (phdr[i].p_type == PT_LOAD) {
            // Segmenti uygun adrese kopyala
            copy_to_process_space(
                phdr[i].p_vaddr,
                elf_data + phdr[i].p_offset,
                phdir[i].p_filesz
            );
            // BSS (sıfırla doldurulacak alan)
            if (phdr[i].p_memsz > phdr[i].p_filesz)
                memset(phdr[i].p_vaddr + phdr[i].p_filesz, 0,
                       phdr[i].p_memsz - phdr[i].p_filesz);
        }
    }

    // Entry point'e jump
    start_process(ehdr->e_entry);
}
```

### 5.3 Sürücü Geliştirme

| Aygıt | Protokol | Zorluk |
|-------|----------|--------|
| VGA text mode | Port I/O (0x3D4, 0x3D5) | Kolay |
| Serial port | UART 16550, port 0x3F8 | Kolay |
| PS/2 Keyboard | Port 0x60/0x64 | Orta |
| ATA PIO disk | IDE controller, port 0x1F0 | Orta |
| AHCI/SATA | PCI BAR, MMIO | Zor |
| NVMe | PCIe, queue pairs | Zor |
| USB | EHCI/XHCI, çok katmanlı | Çok zor |
| Ethernet | PCI, DMA | Çok zor |

### Referans Projeler

| Proje | Açıklama |
|-------|----------|
| [tuhdo/os01](https://github.com/tuhdo/os01) | 13617 ★ — "OS from 0 to 1" kitabı |
| [phil-opp/blog_os](https://github.com/phil-opp/blog_os) | 17576 ★ — Rust ile OS yazımı |
| [SamyPesse/How-to-Make-a-Computer-Operating-System](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System) | 22416 ★ — C++ ile OS |
| [klange/toaruos](https://github.com/klange/toaruos) | 6790 ★ — Tam bağımsız OS, desktop ile |
| [SerenityOS](https://github.com/SerenityOS/serenity) | Grafikal, 90'lar UI, x86/ARM/RISC-V |
| [xv6-riscv](https://github.com/mit-pdos/xv6-riscv) | MIT dersi için Unix v6 benzeri |
| [nuta/kerla](https://github.com/nuta/kerla) | 3467 ★ — Rust, Linux binary uyumlu |
| [vvaltchev/tilck](https://github.com/vvaltchev/tilck) | 3106 ★ — Linux uyumlu minimal kernel |
| [EtchedPixels/FUZIX](https://github.com/EtchedPixels/FUZIX) | 2459 ★ — Küçük sistemler için Unix |
| [OSDev Wiki](https://wiki.osdev.org) | **En kapsamlı kaynak** — tüm başlıklar |
| [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) | Intel Manual — 5000 sayfa, her şey |
| [AMD Programmer's Manual](https://www.amd.com/en/support/tech-docs) | AMD işlemci dokümantasyonu |
| [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/) | Ücretsiz OS ders kitabı |
| [0xAX/linux-insides](https://github.com/0xAX/linux-insides) | Linux kernel iç yapısı |
| [remzi-arpacidusseau/ostep-projects](https://github.com/remzi-arpacidusseau/ostep-projects) | 5888 ★ — OS proje ödevleri |

---

## 6. Faz 4: Init ve Userspace

Kernel çalıştıktan sonra ilk kullanıcı sürecini (PID 1) başlatır.

### 6.1 Minimal Init

```c
// /sbin/init
int main() {
    // /dev ve /proc gibi sanal dosya sistemlerini bağla
    mkdir("/dev", 0755);
    mkdir("/proc", 0755);
    mkdir("/sys", 0755);
    mknod("/dev/null", 0666, makedev(1, 3));
    mknod("/dev/console", 0666, makedev(5, 1));
    mknod("/dev/tty0", 0666, makedev(4, 0));

    // Shell başlat
    if (fork() == 0) {
        execve("/bin/sh", NULL, NULL);
    }

    // Zombie process'leri bekle
    while (1) {
        waitpid(-1, NULL, 0);
    }
}
```

### 6.2 Minimal Shell

Bir shell temelde:
1. Komut satırını oku
2. Parse et (argümanlara ayır)
3. `fork()` + `execve()` ile çalıştır
4. `waitpid()` ile bekle
5. Başa dön

```c
void shell() {
    char line[256];
    while (1) {
        printf("$ ");
        read_line(line, sizeof(line));

        char *args[16];
        int argc = parse_args(line, args);

        int pid = fork();
        if (pid == 0) {
            execve(find_path(args[0]), args, NULL);
            printf("Command not found\n");
            exit(1);
        } else {
            waitpid(pid, NULL, 0);
        }
    }
}
```

### 6.3 Temel Kullanıcı Komutları

| Komut | Görevi |
|-------|--------|
| `ls` | Dizin listele |
| `cat` | Dosya oku |
| `echo` | Yazı yazdır |
| `sh` | Shell (kendisi) |
| `mkdir` | Dizin oluştur |
| `rm` | Dosya sil |
| `ps` | Process listele |
| `kill` | Process sonlandır |
| `init` | PID 1 |

---

## 7. Faz 5: Kendi Kendini Derleme (Full Source Bootstrap)

Bu noktada tüm araç zinciri ve OS çalışıyordur. Sıradaki adım, bu sistem
**içinde** kendini yeniden inşa etmektir.

### 7.1 Live-Bootstrap Projesi

[**live-bootstrap**](https://github.com/fosslinux/live-bootstrap),
tamamen otomatik, tekrarlanabilir bir bootstrap zinciri sunar:

```
hex0 (elle yazılmış, ~512 byte)
  → hex1, hex2
  → M2-Planet
  → GNU Mes (Scheme interpreter + MesCC)
  → TinyCC
  → GCC eski sürüm (2.95.3)
  → binutils, glibc eski sürüm
  → GCC modern sürüm
  → Linux kernel
  → Tam GNU/Linux sistemi
```

Bu zincir artık **her şeyi kaynaktan derler** — tek bir binary seed olmadan
(güvenilen ilk ikili dosya dışında).

### 7.2 Full Source Bootstrap'ın Anlamı

1. Tüm kaynak kodu inceleyebilirsin
2. Hiçbir binary'ye güvenmek zorunda değilsin
3. Her şey tekrarlanabilir (deterministic builds)
4. Ken Thompson backdoor saldırısına karşı korumalı

### 7.3 Mevcut Durum

- **GNU Guix 1.0+:** Full source bootstrap başarıldı (2023)
- **live-bootstrap:** x86_64 üzerinde çalışıyor
- **Bootstrappable distros:** Guix, Gentoo, NetBSD, FreeBSD tam bootstrap yapabiliyor

---

## 8. Faz 6: Gelişmiş İşletim Sistemi

Temel sistem çalıştıktan sonra eklenebilecek özellikler:

### 8.1 Çekirdek Özellikleri

| Özellik | Açıklama | Zorluk |
|---------|----------|--------|
| **SMP** | Çoklu CPU desteği | Zor |
| **Preemptive multitasking** | Zaman paylaşımlı çalışma | Orta |
| **Virtual memory** | Sayfalama, swap | Orta |
| **User mode** | Kullanıcı/çekirdek ayrımı | Orta |
| **Shared libraries** | Dinamik bağlama (.so) | Zor |
| **Copy-on-write** | Fork optimizasyonu | Orta |
| **Signals** | POSIX sinyalleri | Orta |
| **Pipes** | Process arası iletişim | Kolay |
| **Sockets** | Ağ iletişimi | Zor |
| **IPC** | Shared memory, semaphores | Orta |
| **Modular drivers** | Çekirdek modülleri | Zor |

### 8.2 Dosya Sistemi

| FS | Özellikler |
|----|------------|
| **initramfs** | Basit, RAM'de, boot için |
| **ext2** | Basit, anlaşılır, iyi başlangıç |
| **FAT32** | Basit, evrensel uyumlu |
| **sfs/tarfs** | Salt-okunur, kolay implementasyon |
| **tmpfs** | RAM tabanlı, geçici dosyalar |

### 8.3 Ağ Yığını

```
Ethernet driver (NIC)
  → ARP (Address Resolution Protocol)
  → IP (Internet Protocol)
  → ICMP (ping)
  → UDP (User Datagram Protocol)
  → TCP (Transmission Control Protocol) — **en zor**
  → Socket API (Berkeley sockets)
```

### 8.4 GUI

| Katman | Açıklama |
|--------|----------|
| **Framebuffer** | Doğrudan piksel yazma |
| **VESA/VBE** | BIOS mode set, lineer framebuffer |
| **Simple compositor** | Pencere yöneticisi |
| **Font rendering** | Bitmap veya TrueType |
| **Widget toolkit** | Buton, menü, input |

### Tam Bağımsız OS Projeleri (İlham için)

| Proje | Özellikler |
|-------|------------|
| [ToaruOS](https://github.com/klange/toaruos) | 6790 ★ — Kernel, libc, desktop, hepsi kendi |
| [SerenityOS](https://github.com/SerenityOS/serenity) | 34k+ ★ — 90'lar estetiği, browser var |
| [TempleOS](https://templeos.org) | Terry Davis — Tek adam, sıfırdan, 64-bit |
| [Collapse OS](https://github.com/hsoft/collapseos) | Medeniyet çökerse diye, Z80 temelli |
| [Redox OS](https://www.redox-os.org) | Rust ile yazılmış mikro-kernel |
| [Haiku](https://www.haiku-os.org) | BeOS klonu, C++ tabanlı |

---

## 9. Referans Projeler ve Kod Kaynakları

### En Önemli 5 Kaynak (Önce Bunlar)

| # | Kaynak | Açıklama |
|---|--------|----------|
| 1 | [OSDev Wiki](https://wiki.osdev.org) | **İncil.** Tüm konular burada |
| 2 | [tuhdo/os01](https://github.com/tuhdo/os01) | En iyi başlangıç kitabı |
| 3 | [cfenollosa/os-tutorial](https://github.com/cfenollosa/os-tutorial) | En pratik adım adım tutorial |
| 4 | [phil-opp/blog_os](https://github.com/phil-opp/blog_os) | Rust ile en modern yaklaşım |
| 5 | [Intel Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) | Gerçek referans. 5000 sayfa |

### Bootstrapping (Sıfırdan Derleyici)

| Kaynak | Açıklama |
|--------|----------|
| [stage0](https://github.com/oriansj/stage0) | 1048 ★ — 512 byte ile başlar |
| [M2-Planet](https://github.com/oriansj/m2-planet) | Minimal C transpiler |
| [GNU Mes](https://gnu.org/software/mes/) | Scheme interpreter + C derleyici |
| [live-bootstrap](https://github.com/fosslinux/live-bootstrap) | Full source bootstrap |
| [Rui Ueyama chibicc](https://github.com/rui314/chibicc) | 11743 ★ — C11 derleyici, öğrenmek için |
| [bootstrappable.org](https://bootstrappable.org) | Topluluk merkezi |
| [bootstrapping wiki](https://bootstrapping.miraheze.org) | Bootstrap teorisi |
| [Reflections on Trusting Trust](https://www.cs.cmu.edu/~rdriley/487/papers/Thompson_1984_ReflectionsonTrustingTrust.pdf) | Ken Thompson'ın ünlü makalesi |
| [AIM-039](ftp://publications.ai.mit.edu/ai-publications/pdf/AIM-039.pdf) | İlk self-hosting Lisp |
| [bootstrappable distros](https://codeberg.org/vasi/bootstrappable-distros) | Bootstrap yapılabilir dağıtım listesi |

### OS Geliştirme (İşletim Sistemi)

| Kaynak | Açıklama |
|--------|----------|
| [OSDev Wiki Bare Bones](https://wiki.osdev.org/Bare_Bones) | En basit kernel |
| [OSDev Required Knowledge](https://wiki.osdev.org/Required_Knowledge) | Neyi bilmen gerektiği |
| [JamesM's tutorials](https://web.archive.org/web/20160412174753/http://www.jamesmolloy.co.uk/tutorial_html/index.html) | Klasik 10 bölümlük tutorial |
| [littleosbook](https://littleosbook.github.io) | Küçük OS kitabı |
| [64-bit OS tutorial](https://github.com/gmarino2048/64bit-os-tutorial) | Long mode odaklı |
| [xv6-riscv](https://github.com/mit-pdos/xv6-riscv) | MIT dersi, RISC-V |
| [Tilck](https://github.com/vvaltchev/tilck) | Linux uyumlu, okunabilir kernel |
| [ToaruOS](https://github.com/klange/toaruos) | Tam bağımsız, desktop'lu |

### Teori ve Kitaplar

| Kaynak | Açıklama |
|--------|----------|
| [OSTEP (Free Book)](https://pages.cs.wisc.edu/~remzi/OSTEP/) | En iyi ücretsiz OS kitabı |
| [OSTEP Projects](https://github.com/remzi-arpacidusseau/ostep-projects) | 5888 ★ — OS projeleri |
| [Linux Insides](https://github.com/0xAX/linux-insides) | Linux kernel iç dünyası |
| [Write a C Interpreter](https://github.com/lotabout/write-a-C-interpreter) | C yorumlayıcı yazmak |
| [mal - Make a Lisp](https://github.com/kanaka/mal) | 10691 ★ — Lisp yorumlayıcı, her dilde |

---

## 10. Öğrenme Yol Haritası

### Aşama 1: Temeller (1-3 ay)

```
1. x86_64 assembly öğren
   - Kaynak: "Programming from the Ground Up" (Jonathan Bartlett)
   - veya: Intel Manual Volume 2

2. C programlama (pointer, memory model)
   - Kaynak: K&R C, "Modern C" (Gustedt)

3. Dijital mantık / CPU mimarisi
   - Kaynak: "But How Do It Know?" (J. Clark Scott)
   - veya: nand2tetris.org

4. Bilgisayarın boot süreci
   - Kaynak: OSDev Wiki "Boot Sequence"
```

### Aşama 2: Boot ve Assembly (1-2 ay)

```
1. Boş bir boot sector yaz
   - Kaynak: os-tutorial (lesson 1-5)

2. Protected mode geçiş
3. Long mode (64-bit) geçiş
4. GDT, IDT kurulumu
5. Assembly'den C'ye geçiş
```

### Aşama 3: Minimal Kernel (3-6 ay)

```
1. VGA text mode driver
2. GDT/IDT (C'den)
3. IRQ handler (PIT timer, keyboard)
4. Fiziksel bellek yöneticisi
5. Sanal bellek (paging)
6. Heap allocator (kmalloc)
7. Scheduler (round-robin)
8. System calls (int 0x80)
9. ELF loader
10. User mode geçiş
```

### Aşama 4: Userspace (2-4 ay)

```
1. init process (PID 1)
2. minimal libc (printf, malloc, string)
3. Shell
4. Temel komutlar (ls, cat, echo)
5. Dosya sistemi (initrd veya ext2)
```

### Aşama 5: Bootstrapping (6-12 ay)

```
1. stage0 hex0 ile başla (elle)
2. hex1 → hex2 zinciri
3. M2-Planet ile minimal C derle
4. GNU Mes ile Scheme + C derle
5. TinyCC'yi derle
6. GCC'yi derle
7. Kendi derleyicini derle (self-host)
```

### Aşama 6: Gelişmiş OS (6-12 ay)

```
1. Ağ yığını (TCP/IP)
2. Çoklu işlemci (SMP)
3. Grafik arayüz (framebuffer)
4. Sanal dosya sistemi (VFS)
5. Dinamik kütüphaneler (.so)
6. Paket yöneticisi
7. POSIX uyumluluğu
```

### Toplam Süre Tahmini

| Senaryo | Süre |
|---------|------|
| Sadece kernel + boot (tutorial ile) | 3-6 ay |
| Tam OS (kernel + userspace + shell) | 6-12 ay |
| + Full source bootstrap | 12-24 ay |
| + Gelişmiş özellikler (GUI, network, SMP) | 2-4 yıl |
| **Sıfırdan her şey (tek başına)** | **~3-5 yıl** |

---

## 11. Tam Referans Listesi (Linkler)

### Bootstrapping

| Link | Açıklama |
|------|----------|
| https://github.com/oriansj/stage0 | 512 byte ile bootstrap |
| https://github.com/oriansj/m2-planet | Minimal C transpiler |
| https://gnu.org/software/mes | GNU Mes |
| https://github.com/fosslinux/live-bootstrap | Full source bootstrap |
| https://bootstrappable.org | Topluluk |
| https://bootstrapping.miraheze.org | Bootstrap wiki |
| https://github.com/rui314/chibicc | Küçük C11 derleyici |
| https://github.com/rui314/8cc | Küçük C derleyici |
| https://github.com/jserv/amacc | ARM C derleyici |
| https://codeberg.org/vasi/bootstrappable-distros | Bootstrap distro listesi |

### OS Geliştirme

| Link | Açıklama |
|------|----------|
| https://wiki.osdev.org | OSDev Wiki |
| https://wiki.osdev.org/Bare_Bones | Minimal kernel |
| https://wiki.osdev.org/Required_Knowledge | Gereken bilgi |
| https://github.com/cfenollosa/os-tutorial | Adım adım boot'tan kernel'a |
| https://github.com/tuhdo/os01 | "OS from 0 to 1" kitabı |
| https://github.com/phil-opp/blog_os | Rust ile OS |
| https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System | C++ ile OS |
| https://github.com/gmarino2048/64bit-os-tutorial | 64-bit long mode |
| https://github.com/mit-pdos/xv6-riscv | MIT xv6, RISC-V |
| https://github.com/klange/toaruos | Tam bağımsız OS |
| https://github.com/SerenityOS/serenity | SerenityOS |
| https://github.com/nuta/kerla | Linux uyumlu Rust kernel |
| https://github.com/vvaltchev/tilck | Linux uyumlu minimal kernel |
| https://github.com/EtchedPixels/FUZIX | Small Unix |
| https://littleosbook.github.io | Little OS Book |
| https://github.com/hsoft/collapseos | Collapse OS |
| https://templeos.org | TempleOS |

### Donanım Referansları

| Link | Açıklama |
|------|----------|
| https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html | Intel Manuals (SDM) |
| https://www.amd.com/en/support/tech-docs | AMD Programmer's Manuals |
| https://www.coreboot.org | Açık kaynak firmware |
| https://uefi.org/specifications | UEFI spec |
| https://pci-specifications.com | PCI/PCIe spec |

### Teori ve Kitaplar

| Link | Açıklama |
|------|----------|
| https://pages.cs.wisc.edu/~remzi/OSTEP/ | Ücretsiz OS kitabı (OSTEP) |
| https://github.com/remzi-arpacidusseau/ostep-projects | OS projeleri |
| https://github.com/0xAX/linux-insides | Linux kernel iç yapısı |
| https://github.com/lotabout/write-a-C-interpreter | C yorumlayıcı |
| https://github.com/kanaka/mal | Lisp her dilde |
| https://www.cs.cmu.edu/~rdriley/487/papers/Thompson_1984_ReflectionsonTrustingTrust.pdf | Reflections on Trusting Trust |
| ftp://publications.ai.mit.edu/ai-publications/pdf/AIM-039.pdf | İlk self-hosting Lisp |
| http://www.allaboutcircuits.com/textbook/ | Temel elektronik |
| https://www.nand2tetris.org | Nand'den Tetris'e (tam bilgisayar inşası) |

### Okuma Sırası (Önerilen)

```
1. OSTEP (teori) → os-tutorial (pratik) → OSDev Wiki (detay)
2. stage0 README + bootstrapping wiki (bootstrap teorisi)
3. Intel Manual Volume 3 (x86 sistem programlama)
4. chibicc kaynak kodu (derleyici nasıl yazılır)
```

---

## Ek: Zaman Çizelgesi Görseli

```
Zaman (yıl)  0        1        2        3        4
            |--------|--------|--------|--------|
Firmware    [=======]
Bootloader           [====]
Assembler/CC          [===========]
Kernel                         [=============]
libc + init                         [=====]
Userspace                              [====]
Bootstrap                                [=====]
Ağ/GUI/SMP                                     [======]

Toplam: ~3-5 yıl (tek kişi)
         ~1-2 yıl (ekip)
```

---

> **Son Söz:** Bu yolculuk, insanın bilgisayarı anlama sınırlarını zorlar.
> Her katman bir öncekine güvenir. En altta sadece transistörler ve
> elektrik vardır. Gerisi sizin yarattığınız soyutlamalardır.
>
> *"Her şeyi sıfırdan yapmak zorunda değilsiniz. Ama nasıl yapıldığını
> bilmek sizi özgürleştirir."*
