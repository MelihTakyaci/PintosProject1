                 P I N T O S  P R O J E C T  1   –  T E S L İ M  F O R M U
                 ================================================

---- G R U P ----------------------------------------------------

2023280154  Melih Takyacı     melih.takyaci@ogr.deu.edu.tr
2023280140  Emre Özdemir      emre.ozdemir23@ogr.deu.edu.tr
2022280128  Eren Önder        eren.onder@ogr.deu.edu.tr
2022280050  Emir Mutlu        emir.mutlu@ogr.deu.edu.tr


---- Ö N  H A Z I R L I K L A R --------------------------------

>> Teslimatınıza ilişkin ek yorum  
–  Kod; Ubuntu 20.04 üzerinde, orijinal Pintos v1.0 starter‑kit üzerinden inşa edildi.  
–  Tüm testler `make check` ile **temiz geçiyor** (alarm*, priority*, mlfqs*).

>> Başvurulan ek kaynaklar  
* Stanford CS140 Pintos sitesi (resmî yönergeler)  
* *Operating‑System Concepts (9 ed.)* Silberschatz d‑u  
* https://www.cnblogs.com/laiy/p/pintos_project1_thread.html  
* Zhihu & CSDN’deki detaylı proje blogları (README’de tam liste)  

=================================================================
                         A L A R M  S A A T İ
=================================================================

---- V E R İ  Y A P I L A R I ----------------------------------

A1 | Eklenen / değişen tanımlar
```c
/* thread.h */
struct thread {
    …
    struct list_elem sleepelem;     /* Uykuda olan thread için liste düğümü */
    int64_t          wake_tick;     /* Uyanacağı global tik               */
};
/* timer.c */
static struct list sleep_list;      /* wake_tick’e göre sıralı uyku listesi */
static int64_t     next_tick;       /* En yakın uyandırma zamanı            */
```

---- A L G O R İ T M A L A R ------------------------------

A2 | `timer_sleep(n)` akışı  
1.  `intr_disable()` → kritik bölge  
2.  `cur->wake_tick = ticks + n`; `list_insert_ordered(&sleep_list, …)`  
3.  `next_tick = MIN(next_tick, cur->wake_tick)`  
4.  `thread_block()`; eski kesme seviyesi geri yüklenir.  

`timer_interrupt()` ISR’de  
* `ticks++`  
* `while (!list_empty(&sleep_list) &&
          list_entry(list_front(&sleep_list), struct thread, sleepelem)->wake_tick <= ticks)`  
    `thread_unblock()`  
* Uyandırılacak kalmadıysa `next_tick = INT64_MAX`.

A3 | ISR süresini kısaltma  
* Uyku listesi **sıralı** → sadece listenin başı kontrol edilir.  
* `next_tick` önbelleği, gereksiz liste erişimini azaltır.  
* Ağır işler (`thread_unblock` sonrası yeniden sıralama) kesme dışı.

---- S E N K R O N İ Z A S Y O N ------------------------

A4 | Birden çok thread `timer_sleep()` çağırırsa  
Tüm liste güncellemesi kesme kapalıyken yapılır → yarış yok.

A5 | `timer_sleep()` sırasında kesme gelirse  
ISR kritik bölgede çalışamaz; kesme kapalıdır, bitince açılır.

---- G E R E K Ç E ---------------------------------------

A6 | Tasarım gerekçesi  
* Sıralı liste + `next_tick` → O(1) beklenen uyandırma maliyeti.  
* Heap tasarımı ek karmaşıklık getirdi, fayda sağlamadı.  
* `wake_tick` alanı `struct thread`’de doğal, başka modülleri bozmaz.

=================================================================
                      Ö N C E L İ K L E  P L A N L A M A
=================================================================

---- V E R İ  Y A P I L A R I ----------------------------------

B1 | Eklenen / değişen tanımlar
```c
/* thread.h */
struct thread {
    …
    int               real_priority; /* Asıl (bağışsız) öncelik      */
    struct list       locks_held;    /* Sahip olunan kilitler        */
    struct lock      *waiting_lock;  /* Beklediği kilit              */
};
/* synch.h */
struct lock {
    …
    struct list_elem  elem;          /* locks_held link’i             */
    int               max_priority;  /* Waiters listesindeki maks.    */
};
struct semaphore_elem {
    …
    int               priority;      /* Bekleyen grubun maks. öncelik */
};
```

B2 | Bağış izleme diyagramı  
```
  T_high (33) ─wait─▶ lock B ◀─held─ T_med (32)
      │                           │
      └──────donate───────────────┘
                     holds lock A
                     ↑
                 T_low (31→33)
```
`waiting_lock` geriye zincir kurar, `locks_held` sayesinde kilit
bırakılırken bağış geri alınır.

---- A L G O R İ T M A L A R ------------------------------

B3 | Yüksek öncelikli bekleyen ilk uyanır  
Semafor/kilit/koşul waiters listeleri `list_insert_ordered()` ile
önceliğe göre tutulur; `sema_up()` en öndekini uyandırır.

B4 | `lock_acquire()` bağış akışı  
1.  Kilit doluysa `cur->waiting_lock = lock`.  
2.  `donate_priority(holder)` (özyinelemeli).  
3.  Holder kilit bırakana dek bağış sürer.

B5 | `lock_release()` akışı  
1.  Kilit `locks_held` listesinden çıkar.  
2.  Holder’ın önceliği, artık tuttuğu kilitlerin `max_priority`si ile
   `real_priority`nin maksimumuna düşürülür.  
3.  `sema_up()` ile en yüksek bekleyen uyanır.

---- S E N K R O N İ Z A S Y O N ------------------------

B6 | `thread_set_priority()` yarış riski  
Öncelik düşürme ile eşzamanlı bağış çakışabilir; çözüm: tüm işlem
`intr_disable()` – `intr_set_level()` bloğu içinde atomik. Kilit
kullanmak gerekmez, ters sıralı kilitlenme riskini önler.

---- G E R E K Ç E ---------------------------------------

B7 | Tasarım gerekçesi  
* Klasik donation zinciri (basit, test‑dostu).  
* `max_priority` alanı O(1) hesap; global tarama yok.  
* Merkezi “donation tablosu” denendi, karmaşık & yavaş çıktı.

=================================================================
                       G E L İ Ş M İ Ş  P L A N L A M A
=================================================================

---- V E R İ  Y A P I L A R I ----------------------------------

C1 | Eklenen / değişen tanımlar
```c
/* thread.h */
struct thread {
    …
    int nice;              /* Kullanıcı “nezaket” (‑20…20)           */
    int recent_cpu;        /* Sabit‑nokta, yakın CPU zamanı          */
};
/* threads.c */
static int load_avg;       /* Sabit‑nokta, sistem yük ortalaması     */
```
`fixed.h` makroları sabit‑nokta işlemleri kapsüller.

---- A L G O R İ T M A L A R ------------------------------

C2 | A, B, C nice = 0,1,2 — 36 tiklik örnek  
| tick | rcpu A | rcpu B | rcpu C | pri A | pri B | pri C | Run |
|----:|:------:|:------:|:------:|:-----:|:-----:|:-----:|:---:|
|  0  |   0    |   0    |   0    | 63    | 61    | 59    |  A  |
|  4  |   4    |   0    |   0    | 62    | 61    | 59    |  A  |
|  8  |   8    |   0    |   0    | 61    | 61    | 59    |  A  |
| 12  |  12    |   0    |   0    | 60    | **61**| 59    | **B**|
| 16  |  12    |   4    |   0    | 60    | 60    | 59    |  B  |
| 20  |  12    |   8    |   0    | 60    | 59    | 59    |  A  |
| 24  |  16    |   8    |   0    | 59    | 59    | 59    |  A  |
| 28  |  20    |   8    |   0    | **58**| 59    | **59**| **C**|
| 32  |  20    |   8    |   4    | 58    | **59**| 58    | **B**|
| 36  |  20    |  12    |   4    | 58    | 58    | 58    |  B  |

C3 | Belirsizlik  
Spesifikasyon eşit öncelik pre‑emption davranışını söylemiyor.
Kuralımız: **daha yüksek >**, eşitse mevcut devam eder. Kod & tablo aynı.

C4 | Kesme vs kullanıcı hesap maliyeti  
`recent_cpu++` ISR’de tek adım; pahalı `recalc_all()` saniyede 1× ve
kesme dışı → düşük gecikme, iyi öncelik doğruluğu.

---- G E R E K Ç E ---------------------------------------

C5 | Öz‑eleştiri & geliştirme fikirleri  
+ Sabit‑nokta yerine 64‑bit float donanımı kodu basitleştirirdi.  
+ Ready‑queue’yu 4 sabit “bin” (pri 0‑15,16‑31…) ile O(1) seçmek
  bellek & hız kazandırırdı.  
+ `nice` değeri dinamik API’si zaman darlığından eklenemedi.

C6 | Sabit‑nokta soyutlama seçimi  
`fixed.h` makroları (`INT`, `FRACT`, `ADD/MUL/DIV`) kullanılınca
MLFQS formülleri doğrudan kodlanabildi; taşma tek yerde kontrol edildi.
Alternatif (satır içi işlemler) okunabilirliği düşürüyordu.

```c
/* fixed.h kısaltılmış */
#define F (1<<14)
#define INT(n)     ((n)*F)
#define TO_INT(x)  ((x)/F)
#define ADD(x,y)   ((x)+(y))
#define SUB(x,y)   ((x)-(y))
#define MUL(x,y)   ((int64_t)(x)*(y)/F)
#define DIV(x,y)   ((int64_t)(x)*F/(y))
```

(Form, README’de açıklanan tasarımla **tam uyumludur**.)

