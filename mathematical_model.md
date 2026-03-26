# Gün Ortası Rota Yeniden Optimizasyonu İçin Karma Tamsayılı Programlama Yaklaşımı

## 1. Giriş

Bu rapor, **Gün Ortası Suffix (Son Ek) Yeniden Rotalama Problemini** çözmek için geliştirilen matematiksel optimizasyon yaklaşımını detaylandırmaktadır. Dinamik operasyonel ortamlarda, öngörülemeyen makine arızaları veya değişen öncelikler nedeniyle araç rotalarının gün ortasında yeniden değerlendirilmesi sıklıkla gereklidir.

Modelimizin amacı, bir servis aracı filosunun kalan rotalarını, belirli bir gün ortası kontrol noktasından (`mid_day_time`) operasyonel günün sonuna kadar yeniden optimize etmektir. Çözüm, standart operasyonlar için tüm zaman kısıtlarına uyulmasını garanti ederken, arıza gecikmelerini ve talep bazlı hizmet önceliklerini temel alan ağırlıklı bir ceza fonksiyonunu minimize eder.

---

## 2. Çözüm Metodolojisi

Problemi, **Zaman Pencereli Çok Araçlı Rotalama Probleminin (VRPTW)** özel bir varyantı olarak ele aldık ve **Karma Tamsayılı Doğrusal Programlama (MILP)** kullanarak çözdük.

Yaklaşımımızın temel mekanizmaları şunları içerir:

**Prefix/Suffix (Ön Ek/Son Ek) Ayrışımı:** Araçların başlangıçta planlanan rotaları `mid_day_time` eşiğinde bölünür. Bu kontrol noktasında aktif olarak yürütülen veya seyahat edilen herhangi bir operasyon, "Prefix" (Ön Ek) rotanın bir parçası olarak kilitlenir. Bu kilitli operasyonun varış düğümü, ilgili aracın "Suffix" (Son Ek) rotalama aşaması için yeni dinamik başlangıç noktası ($o_k$) olur.

**Arızalar İçin VIP Servis Kuralı:** Arıza cezasını ($\lambda_f$) agresif bir şekilde minimize etmek için dinamik bir zaman penceresi esnetmesi (relaxation) getirilmiştir. Arızalı makineler için makineye özel ikmal (replenishment) zaman penceresi kısıtlamaları kaldırılır ve yalnızca operasyonel gün sonu fizibilite sınırları korunur. Bu durum, aracın varış anında anında onarıma başlamasına olanak tanır.

**Boşta Kalan (Idle) Araç Yönetimi:** Eğer bir araç suffix aşamasındaki kalan görevler için matematiksel olarak gerekli (optimal) değilse, model o aracı ek bir karar değişkenine ihtiyaç duymadan dinamik başlangıç noktasından doğrudan depoya yönlendirerek zarif bir şekilde boşta (idle) bırakır.

---

## 3. Matematiksel Formülasyon

### 3.1. Kümeler ve İndisler

| Sembol | Açıklama |
|--------|----------|
| $K$ | Suffix rotalama probleminde dikkate alınan tüm mevcut hizmet araçlarının (kamyonların) kümesi |
| $M$ | `mid_day_time` ayrımından sonra hala hizmet bekleyen makinelerin kümesi |
| $F$ | $M$ kümesi içinde yer alan ve bir arıza yaşamış ($F \subseteq M$), `failed_at` alanı ile tanımlanan makinelerin alt kümesi |
| $d$ | Merkezi depo düğümü (depot node) |
| $o_k$ | $k \in K$ aracının kilitli prefix operasyonunu tamamladıktan sonra ulaştığı dinamik başlangıç düğümü |
| $N_k$ | Suffix aşamasında $k$ aracı için erişilebilir tüm düğümlerin kümesi; $N_k = M \cup \{o_k, d\}$ olarak tanımlanır |

> **Not:** Bu formülasyon anlık görüntü (snapshot) şemasıyla uyumludur; `mid_day_time` itibarıyla tamamlanan veya taahhüt edilen operasyonlar prefix içinde sabit kalır ve yalnızca kalan araç rotaları yeniden optimize edilir.

### 3.2. Parametreler

| Sembol | Açıklama |
|--------|----------|
| $t_{ij}$ | $i$ düğümünden $j$ düğümüne seyahat süresi (dakika cinsinden) |
| $p_i$ | $i$ makinesindeki standart ikmal (replenishment) hizmet süresi |
| $q_i$ | $i$ makinesindeki onarım (repair) süresi (yalnızca $i \in F$ için tanımlıdır) |
| $sd_i$ | $i$ makinesindeki toplam hizmet süresi (aşağıda tanımlı) |
| $[a_i, b_i]$ | $i$ makinesi için kesin zaman penceresi |
| $f_i$ | $i \in F$ makinesi için kesin arıza zamanı (`failed_at`) |
| $r_i$ | $i$ makinesinin talep oranı (öncelik ağırlığı olarak işlev görür) |
| $\tau_k$ | $k$ aracının dinamik başlangıç noktası $o_k$'da suffix rotalaması için hazır hale geldiği en erken zaman |
| $\lambda_f, \lambda_d$ | Arıza gecikmesi ve talep ağırlıklı hizmet başlama zamanları için ceza ağırlık parametreleri |
| $M^{big}$ | Dinamik olarak hesaplanan, yeterince büyük bir sabit (Big-M) |
| $day\_end$ | Operasyonel gün sonu zaman sınırı |

Toplam hizmet süresi şu şekilde tanımlanır:

$$sd_i = \begin{cases} p_i + q_i, & i \in F \\ p_i, & i \notin F \end{cases}$$

### 3.3. Karar Değişkenleri

| Değişken | Tür | Açıklama |
|----------|-----|----------|
| $x_{kij} \in \{0,1\}$ | İkili (binary) | Eğer $k$ aracı doğrudan $i$ düğümünden $j$ düğümüne giderse 1, aksi takdirde 0 |
| $S_i \ge 0$ | Sürekli (continuous) | $i$ makinesindeki fiili hizmet başlama zamanı |
| $W_i \ge 0$ | Sürekli | Arızalı $i \in F$ makinesi için gecikme (arıza sonrası bekleme süresi) |
| $u_i \ge 1$ | Sürekli yardımcı | MTZ alt tur (subtour) elemesi için kullanılır |

### 3.4. Amaç Fonksiyonu

Amaç fonksiyonu, acil onarımlar ile yüksek talepli ikmaller arasında denge kurarak toplam sistem cezasını minimize eder:

$$\min Z = \lambda_f \sum_{i \in F} W_i + \lambda_d \sum_{i \in M} r_i \cdot S_i$$

### 3.5. Kısıtlar

**1. Tekil Ziyaret Ataması:**
Kalan her makine tam olarak bir kez ziyaret edilmelidir:

$$\sum_{k \in K} \sum_{i \in N_k \setminus \{j\}} x_{kij} = 1 \quad \forall\, j \in M$$

**2. Dinamik Başlangıçtan Çıkış:**
Her araç kendi özel dinamik başlangıcından ayrılmalıdır (kullanılmıyorsa doğrudan depoya veya bir makineye):

$$\sum_{j \in M \cup \{d\}} x_{k,\, o_k,\, j} = 1 \quad \forall\, k \in K$$

**3. Depoya Dönüş:**
Her araç suffix aşamasını depoda sonlandırmalıdır:

$$\sum_{i \in M \cup \{o_k\}} x_{k,\, i,\, d} = 1 \quad \forall\, k \in K$$

**4. Akış Korunumu:**
Her makine ve araç için gelen ve giden akışlar eşleşmelidir:

$$\sum_{\substack{i \in M \cup \{o_k\} \\ i \neq j}} x_{kij} - \sum_{\substack{h \in M \cup \{d\} \\ h \neq j}} x_{kjh} = 0 \quad \forall\, j \in M,\ \forall\, k \in K$$

**5. Dinamik Başlangıçtan İtibaren Zaman Tutarlılığı:**
Eğer $k$ aracı başlangıç noktasından $j$ makinesine seyahat ederse, hizmet, aracın hazır olma zamanı artı seyahat süresinden önce başlayamaz:

$$S_j \ge \tau_k + t_{o_k,\, j} - M^{big}(1 - x_{k,\, o_k,\, j}) \quad \forall\, j \in M,\ \forall\, k \in K$$

**6. Ardışık Makineler Arası Zaman İlerlemesi:**
Hizmet süreleri ve seyahat sürelerini içeren kronolojik zaman ilerlemesi:

$$S_j \ge S_i + sd_i + t_{ij} - M^{big}\left(1 - \sum_{k \in K} x_{kij}\right) \quad \forall\, i, j \in M,\ i \neq j$$

**7. Zaman Penceresi Mantığı:**
Standart operasyonlar, makineye özel sınırlara kesinlikle uyar. Arızalı makineler için makineye özel sınırlar kaldırılarak yalnızca operasyonel gün fizibilite sınırı uygulanır:

$$a_i \le S_i \le b_i \quad \forall\, i \in M \setminus F$$

$$0 \le S_i \le day\_end \quad \forall\, i \in F$$

**8. Arıza Gecikmesi:**
Gecikme, arıza anından fiili hizmet başlangıcına kadar olan süre olarak ölçülür:

$$W_i \ge S_i - f_i \quad \forall\, i \in F$$

**9. Alt Tur Eleme (MTZ):**
Miller-Tucker-Zemlin kısıtları birbiriyle bağlantısı olmayan kapalı döngüleri (alt turları) engeller:

$$u_i - u_j + |M| \cdot x_{kij} \le |M| - 1 \quad \forall\, i, j \in M,\ i \neq j,\ \forall\, k \in K$$

---

## 4. Kod Düzeyindeki Yorumlamalara Dair Notlar

Uygulanan MILP modeli, araca özel dinamik başlangıçlar, makine atama kısıtları, akış korunumu, zamansal tutarlılık kısıtları, arıza gecikme cezaları ve MTZ alt tur eleme yöntemlerini kullanarak her bir araç rotasının yalnızca `mid_day_time` sonrasındaki suffix kısmını yeniden optimize eder. Arızalı makineler için kod, algoritmik fizibiliteyi sağlamak adına makineye özel ikmal zaman penceresi sınırlarını fiziksel olarak kaldırır ve yalnızca operasyonel gün sınırlarını uygular.

Buna ek olarak, implementasyon aşamasında, orijinal JSON çıktı şeması gereksinimlerini birebir karşılamak adına arızalı makineler için `["Repair", "Replenishment"]` sırasının ardışık olarak eklendiği dizisel bir yürütme mantığı varsayılmış ve örnek (instance) dosyalarına özgü `n_trucks` parametresi dinamik olarak kullanılmıştır.
