# Proje Raporu: Hibrit Matematiksel Model ve Matheuristic Yaklaşımı

Bu döküman, projenin çözümünde kullanılan **Zaman-Tabanlı Karma Tam Sayılı Programlama (MILP)** modelini ve **Spatio-Temporal K-Means** tabanlı matheuristic yaklaşımını detaylandırmaktadır.

---

## 1. Matematiksel Model (MILP)

### 1.1. Setler ve İndisler

- $K$: Kamyonlar seti, $k \in K = \{0, \dots, n\_trucks-1\}$
- $M$: Hizmet bekleyen makineler seti, $i, j \in M$
- $F$: Arızalı (failed) makineler seti, $F \subset M$
- $O_k$: $k$ kamyonunun öğlen saatindeki başlangıç konumu
- $D$: Depo (bitiş noktası)
- $V_k$: $k$ kamyonu için düğüm seti, $V_k = \{O_k\} \cup M \cup \{D\}$

### 1.2. Parametreler

- $dist_{ij}$: $i$ ve $j$ düğümleri arasındaki seyahat süresi
- $sd_i$: $i$ makinesindeki toplam servis süresi (bakım + arıza onarım)
- $[tw\_s_i, tw\_e_i]$: $i$ makinesinin zaman penceresi (Arızalı makineler için $[0, T_{end}]$)
- $\tau_k$: $k$ kamyonunun öğlen başlangıç zamanı ($\ge mid\_day\_time$)
- $\lambda_f$: Arıza bekleme süresi maliyet katsayısı
- $\lambda_d$: Talep bazlı gecikme maliyet katsayısı
- $P_{late}$: Heuristic modunda uygulanan gecikme cezası (Penalty)
- $M_{ij}$: Sıkıştırılmış (Tightened) Big-M katsayısı

### 1.3. Karar Değişkenleri

- $x_{kij} \in \{0, 1\}$: $k$ kamyonu $i$ düğümünden $j$ düğümüne giderse 1, aksi halde 0.
- $y_k \in \{0, 1\}$: $k$ kamyonu öğleden sonra aktifse 1, değilse 0.
- $T_i \ge 0$: $i$ makinesine varış (servis başlangıç) zamanı.
- $S_i \ge 0$: $i$ makinesine yapılan gecikme (slack) miktarı (Sadece Heuristic modunda).
- $L_i \ge 0$: $i$ arızalı makinesinin tamir edilene kadar geçen süresi ($T_i - failed\_at_i$).
- $u_i \in \mathbb{R}$: MTZ alt tur engelleme değişkeni (Sadece Exact modunda).

### 1.4. Amaç Fonksiyonu

$$\min Z = \lambda_f \cdot \sum_{i \in F} L_i + \lambda_d \cdot \sum_{i \in M} (demand\_rate_i \cdot T_i) + [P_{late} \cdot \sum_{i \in M} S_i]_{heuristic}$$

### 1.5. Kısıtlar

**Akış Korunumu:** Her makine tam bir kez ziyaret edilmelidir.

$$\sum_{k \in K} \sum_{i \in \{O_k\} \cup M, i \neq j} x_{kij} = 1, \quad \forall j \in M$$

**Rota Sürekliliği:**

$$\sum_{j \in M \cup \{D\}} x_{k, O_k, j} = y_k, \quad \forall k \in K$$

$$\sum_{i \in \{O_k\} \cup M} x_{kij} = \sum_{h \in M \cup \{D\}} x_{kjh}, \quad \forall j \in M, \forall k \in K$$

**Zaman İlerlemesi (Time-Based MTZ):**

$$T_j \ge T_i + sd_i + dist_{ij} - M_{ij} \cdot (1 - \sum_{k \in K} x_{kij}), \quad \forall i, j \in M, i \neq j$$

**Zaman Pencereleri (Hybrid):**

- **Exact:** $$tw\_s_i \le T_i \le tw\_e_i$$
- **Heuristic:** $$tw\_s_i \le T_i \le tw\_e_i + S_i$$

**Alt Tur Engelleme (Sıralama - Sadece Exact):**

$$u_j \ge u_i + 1 - |M| \cdot (1 - \sum_{k \in K} x_{kij}), \quad \forall i, j \in M, i \neq j$$

---

## 2. Matheuristic Yaklaşımı: Spatio-Temporal Clustering

Modelin büyük ölçekli problemlerde hızlı çözüm üretmesi için K-Means tabanlı bir kümeleme algoritması kullanılmıştır.

### 2.1. Kümeleme Mantığı

Makineler sadece coğrafi konumlarına ($X, Y$) göre değil, aynı zamanda zaman pencerelerinin orta noktalarına ($T_{mid\_point}$) göre kümelenir. Bu 3 boyutlu veri seti MinMaxScaler ile normalize edilir.

### 2.2. Overlapping Clusters (Kesişen Kümeler)

Sistemin çözümsüz (Infeasible) kalmasını engellemek için her makine en yakın 2 kamyona aday olarak atanır. Bu, Gurobi'ye alternatif rotalar sunarak "load balancing" (yük dengeleme) sağlar.

### 2.3. Heuristic Pseudocode

```
Algorithm: Spatio-Temporal Overlapping Matheuristic

 1: Input: M_set, truck_states, overlap_limit=2
 2: features = Extract(X, Y, TimeWindow_Center) for each m in M_set
 3: features_scaled = MinMaxScaler(features)
 4: clusters = KMeans(n_clusters=n_trucks).fit(features_scaled)
 5: centroids = CalculateSpatialCentroids(clusters)
 6: truck_centroids = AssignCentroidsToNearestTruckOrigin(centroids, truck_states)
 7: For each machine m in M_set:
 8:    dists = CalculateDistancesToAllTruckCentroids(m)
 9:    Assign m to top 'overlap_limit' nearest trucks in truck_clusters
10: End For
11: Solve MILP with restricted x[k,i,j] variables (only if m is in truck_clusters[k])
12: Apply Soft Time Windows and Penalty_Late in Objective Function
```

---

## 3. Varsayımlar

- **Gün Ortası Kilidi (Mid-Day Lock):** Kamyonların öğlen saatindeki (mid_day_time) konumları sabittir ve bu noktadan önceki rotalar değiştirilemez.
- **Hizmet Önceliği (VIP):** Arızalı makineler (F_set), periyodik bakım makinelerinin zaman penceresi kısıtlamalarına tabi değildir; en hızlı şekilde müdahale edilecek şekilde modellenmiştir.
- **Zaman Bazlı MTZ:** Zaman değişkeni ($T_i$) tek yönlü arttığı için alt turların engellenmesi matematiksel olarak garanti altına alınmıştır.
- **Kısıtlı Esneklik:** Heuristic modunda zaman pencerelerinin aşılmasına izin verilir ancak yüksek ceza puanı ile bu durum minimize edilir.
