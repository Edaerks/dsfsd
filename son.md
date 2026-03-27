# Proje Raporu: Çok Araçlı Dinamik Rotalama için Exact ve Matheuristic Modelleri

Bu doküman, gün ortasında (mid-day) güncellenen araç rotalama ve makine bakım/arıza probleminin çözümü için geliştirilen iki farklı yaklaşımı detaylandırmaktadır:
1. **Exact (Katı) Matematiksel Model:** Problemin global optimumunu arayan, katı kısıtlara sahip model.
2. **Matheuristic (Esnek) Model:** Spatio-Temporal K-Means kümelemesi ve esnek zaman pencereleri (Soft Time Windows) kullanarak büyük ölçekli problemleri saniyeler içinde çözen dayanıklı (robust) model.

---

## 1. Ortak Tanımlar (Setler ve Parametreler)

Her iki modelde de kullanılan temel matematiksel altyapı şöyledir:

### 1.1. Setler ve İndisler
- $K$: Kamyonlar (araçlar) seti, $k \in K = \{0, \dots, n\_trucks-1\}$
- $M$: Hizmet bekleyen makineler seti, $i, j \in M$
- $F$: Arızalı (VIP) makineler seti, $F \subset M$
- $O_k$: $k$ kamyonunun öğlen saatindeki başlangıç konumu (düğüm)
- $D$: Depo (bitiş noktası)
- $V_k$: $k$ kamyonu için olası düğümler seti, $V_k = \{O_k\} \cup M \cup \{D\}$

### 1.2. Parametreler
- $dist_{ij}$: $i$ ve $j$ düğümleri arasındaki seyahat süresi
- $sd_i$: $i$ makinesindeki toplam servis süresi (bakım + arıza onarım)
- $[tw\_s_i, tw\_e_i]$: $i$ makinesinin planlı zaman penceresi
- $\tau_k$: $k$ kamyonunun yola çıkmaya hazır olduğu an ($\ge mid\_day\_time$)
- $\lambda_f$: Arıza bekleme süresi maliyet katsayısı
- $\lambda_d$: Talep bazlı gecikme maliyet katsayısı
- $M_{ij}$: Kısıtları sıkılaştırmak için kullanılan özel "Tight Big-M" katsayısı

---

## 2. SAF MATEMATİKSEL MODEL (EXACT MILP)

Bu model, kurallardan asla taviz vermez. Zaman pencerelerine %100 uyulmasını şart koşar ve alt turları engellemek için klasik MTZ değişkenlerini kullanır.

### 2.1. Karar Değişkenleri
- $x_{kij} \in \{0, 1\}$: $k$ kamyonu $i$'den $j$'ye giderse 1, aksi halde 0.
- $y_k \in \{0, 1\}$: $k$ kamyonu aktif olarak kullanılıyorsa 1, aksi halde 0.
- $T_i \ge 0$: $i$ makinesine varış (servis başlangıç) zamanı.
- $L_i \ge 0$: $i$ arızalı makinesinin tamir edilene kadar geçen süresi ($T_i - failed\_at_i$).
- $u_i \ge 1$: $i$ makinesinin rotadaki ziyaret sırası (MTZ alt tur engelleyici).

### 2.2. Amaç Fonksiyonu
Amaç, arızalı makinelerin bekleme süresini ve normal makinelerin talep oranına bağlı gecikmelerini minimize etmektir.
$$\min Z_{exact} = \lambda_f \cdot \sum_{i \in F} L_i + \lambda_d \cdot \sum_{i \in M} (demand\_rate_i \cdot T_i)$$

### 2.3. Kısıtlar
1. **Akış ve Atama Kısıtları:** Her makine tam bir kez ziyaret edilmelidir ve araca giren akış çıkan akışa eşit olmalıdır.
2. **Katı Zaman Pencereleri (Hard Time Windows):**
   - Normal makineler için: $tw\_s_i \le T_i \le tw\_e_i, \quad \forall i \in M \setminus F$
   - Arızalı (VIP) makineler için: $T_i \ge 0 \quad$ (Üst sınır olmaksızın en kısa sürede gidilir)
3. **Zaman İlerlemesi:** $$T_j \ge T_i + sd_i + dist_{ij} - M_{ij} \cdot (1 - \sum_{k \in K} x_{kij}), \quad \forall i, j \in M, i \neq j$$
4. **MTZ Sıralama Kısıtı (Alt Tur Engelleme):** $$u_j \ge u_i + 1 - |M| \cdot (1 - \sum_{k \in K} x_{kij}), \quad \forall i, j \in M, i \neq j$$

---

## 3. MATHEURISTIC MODEL (ESNEK YAKLAŞIM)

Bu model, büyük ölçekli problemleri çözebilmek için K-Means kümelemesi ile arama uzayını daraltır. Olası kilitlenmeleri (Infeasibility) engellemek için de kısıtları esnetir.

### 3.1. Spatio-Temporal Overlapping Clustering
1. **Özellik Çıkarımı:** Makineler $X$ koordinatı, $Y$ koordinatı ve Zaman Penceresi Orta Noktası $(tw\_s + tw\_e)/2$ baz alınarak 3 boyutlu uzayda modellenir ve normalize edilir.
2. **K-Means Uygulaması:** Makineler, araç sayısı ($|K|$) kadar kümeye ayrılır.
3. **Esnek Atama (Overlapping):** Her makine sadece tek bir kümeye değil, merkeze olan uzaklığına göre en yakın **2 araca** aday olarak atanır. Bu durum araçlara yük dengeleme esnekliği sağlar.

### 3.2. Ek Karar Değişkenleri
- $S_i \ge 0$: $i$ makinesine yapılan gecikme (slack) süresi (dakika cinsinden). 
- *(Not: Bu modelde hızı artırmak için $u_i$ sıralama değişkenleri kullanılmamış, alt turlar doğrudan $T_i$ değişkeni ile engellenmiştir).*

### 3.3. Ceza Bazlı Amaç Fonksiyonu (Penalty Objective)
Exact modelin amaç fonksiyonuna ek olarak, zaman penceresi aşıldığında sistemin kilitlenmemesi için gecikme cezası ($P_{late}$) eklenmiştir.
$$\min Z_{heuristic} = \lambda_f \cdot \sum_{i \in F} L_i + \lambda_d \cdot \sum_{i \in M} (demand\_rate_i \cdot T_i) + P_{late} \cdot \sum_{i \in M} S_i$$

### 3.4. Esnek Kısıtlar (Soft Constraints)
1. **Arama Uzayı Kısıtı:** Bir araç sadece kendi esnek kümesinde ($C_k$) bulunan makinelere gidebilir.
   $$x_{kij} = 0, \quad \forall j \notin C_k$$
2. **Yumuşak Zaman Pencereleri (Soft Time Windows):** Kamyonların zor durumlarda geç kalmasına izin verilir, ancak $S_i$ değişkeni üzerinden ceza puanı yazar.
   $$tw\_s_i \le T_i \le tw\_e_i + S_i, \quad \forall i \in M \setminus F$$

---

## 4. Temel Varsayımlar ve Operasyonel Kurallar

1. **Gün Ortası Kilidi (Mid-Day Lock):** Araçların öğlen saatinden önceki rotaları sabittir ve değiştirilemez. Planlama sadece $\tau_k$ anından gün sonuna kadar olan kısmı kapsar.
2. **Hizmet Önceliği (Arızalı Makineler):** Arızalı makineler (`F_set`) planlı zaman penceresine tabi değildir. Sistem bu makineleri "en acil iş" olarak görür ve zaman penceresini beklemeden anında müdahale edilmesine olanak tanır.
3. **Best-Bound Hızlandırması:** Exact modelde, ağaç budama (Presolve) performansını artırmak için klasik "Big-M" yerine her düğüm çiftine özel "Tight-M" hesaplaması kullanılmıştır.
