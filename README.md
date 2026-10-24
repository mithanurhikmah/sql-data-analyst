# Analisis Penjualan Toko Peralatan Dapur dengan SQL (BigQuery)

Mini project analisis data penjualan toko peralatan dapur menggunakan **SQL di Google BigQuery**. Ada 10 pertanyaan bisnis yang dijawab dengan agregasi, `CASE WHEN`, CTE, subquery, dan window function.

> Dikerjakan dalam bootcamp **Data Analyst with AI** (KarirNex, 1-8 Juli 2026). Dataset yang dipakai adalah **data dummy** dari bootcamp, bukan data perusahaan sungguhan.

## Ringkasan Temuan

| # | Pertanyaan | Hasil |
|---|---|---|
| 1 | Total dan rata-rata ongkos kirim 2025 | Rp570.160.000 total; Rp57.016 per pesanan |
| 2 | Top 5 produk berdasarkan unit vs revenue (completed) | Daftarnya **berbeda** (tidak ada produk yang sama) |
| 3 | Pesanan dan revenue completed Q4 2025 | 2.156 pesanan; Rp1.295.326.400 |
| 4 | Kota dengan rata-rata ongkir termahal vs termurah | Jakarta Barat Rp60.000 vs Bekasi Rp50.000 (selisih Rp10.000) |
| 5 | Nilai pesanan refund terhadap gross sales | Rp279.711.700 = 4,76% dari Rp5.872.699.300 |
| 6 | Top 5 produk dengan rata-rata qty per pesanan tertinggi (min. 50 pesanan) | Wajan Anti Lengket 24 cm (2,13) |
| 7 | Bulan revenue completed tertinggi per kategori | Alat Masak: Agustus; Alat Saji: Juni; Alat Simpan: Juli |
| 8 | Jumlah produk penyumbang 80% revenue | 26 dari 57 produk |
| 9 | Jeda rata-rata antar pesanan pelanggan (>5 pesanan completed) | 40,76 hari; tersingkat Customer_3 (9,93 hari) |
| 10 | Produk dengan refund rate tertinggi dan potensi revenue yang bisa diselamatkan | Panci Stainless 20 cm; potensi Rp3.111.500 |

## Dataset

- Tabel: `toko_peralatan_dapur.orders` (BigQuery)
- Isi: 10.000 pesanan sepanjang 2025 (1 baris = 1 pesanan), 57 produk, 3 kategori (Alat Masak, Alat Simpan, Alat Saji)
- Kolom utama: `sales_date`, `order_id`, `customer_name`, `product_name`, `category`, `price`, `quantity`, `discount`, `total`, `shipping_fee`, `total_sales`, `status`, `shipping_address`
- Kolom hasil pembersihan data di Excel: `city_clean`, `category_clean`, `status_clean` (`complete`, `cancel`, `refund`), `customer_name_clean`
- Data mentah tidak disertakan di repositori ini. Hanya query dan hasilnya.

## Skill SQL yang Dipakai

`SUM` / `AVG` / `COUNT(DISTINCT)` · `GROUP BY` / `HAVING` · `CASE WHEN` · `COUNTIF` · CTE (`WITH`) · subquery · window function (`ROW_NUMBER`, `LAG`, `SUM() OVER`) · fungsi tanggal (`EXTRACT`, `FORMAT_DATE`, `DATE_DIFF`) · `SAFE_DIVIDE` · `GREATEST`

## Query dan Hasil

### 1. Total dan rata-rata ongkos kirim sepanjang 2025

```sql
SELECT
  SUM(shipping_fee) AS total_ongkir,
  AVG(shipping_fee) AS avg_ongkir
FROM `toko_peralatan_dapur.orders`
WHERE EXTRACT(YEAR FROM sales_date) = 2025;
```

| total_ongkir | avg_ongkir |
|---|---|
| 570.160.000 | 57.016 |

### 2. Top 5 produk berdasarkan unit terjual (completed), dibandingkan dengan top 5 revenue

```sql
-- Top 5 berdasarkan unit
SELECT product_name, SUM(quantity) AS total_unit
FROM `toko_peralatan_dapur.orders`
WHERE status_clean = 'complete'
GROUP BY product_name
ORDER BY total_unit DESC
LIMIT 5;

-- Top 5 berdasarkan revenue
SELECT product_name, SUM(total_sales) AS total_revenue
FROM `toko_peralatan_dapur.orders`
WHERE status_clean = 'complete'
GROUP BY product_name
ORDER BY total_revenue DESC
LIMIT 5;
```

| Top 5 unit | Unit | Top 5 revenue | Revenue (Rp) |
|---|---|---|---|
| Tempat Tisu Meja | 370 | Mesin Kopi Espresso Rumahan | 640.217.900 |
| Cobek Granit | 365 | Food Processor | 447.392.900 |
| Keranjang Buah Besi | 361 | Oven Listrik 20 L | 296.670.000 |
| Mangkuk Keramik Set 6 pcs | 360 | Air Fryer 4 L | 246.153.600 |
| Rice Cooker 1.8 L | 355 | Kompor Induksi | 237.592.800 |

**Insight:** daftar kedua ranking sama sekali berbeda. Produk terlaris berdasarkan unit tidak otomatis menjadi penyumbang revenue terbesar, karena revenue didominasi produk elektronik dapur berharga tinggi.

### 3. Jumlah pesanan dan revenue completed pada Q4 2025

```sql
SELECT
  COUNT(DISTINCT order_id) AS jumlah_pesanan,
  SUM(total_sales) AS total_revenue
FROM `toko_peralatan_dapur.orders`
WHERE status_clean = 'complete'
  AND sales_date BETWEEN '2025-10-01' AND '2025-12-31';
```

| jumlah_pesanan | total_revenue |
|---|---|
| 2.156 | 1.295.326.400 |

### 4. Kota dengan rata-rata ongkos kirim termahal dan selisihnya dengan termurah

```sql
WITH per_kota AS (
  SELECT city_clean, AVG(shipping_fee) AS avg_ongkir
  FROM `toko_peralatan_dapur.orders`
  GROUP BY city_clean
)
SELECT
  (SELECT city_clean FROM per_kota ORDER BY avg_ongkir DESC LIMIT 1) AS kota_termahal,
  (SELECT MAX(avg_ongkir) FROM per_kota) AS ongkir_termahal,
  (SELECT city_clean FROM per_kota ORDER BY avg_ongkir ASC LIMIT 1) AS kota_termurah,
  (SELECT MIN(avg_ongkir) FROM per_kota) AS ongkir_termurah,
  (SELECT MAX(avg_ongkir) - MIN(avg_ongkir) FROM per_kota) AS selisih;
```

| kota_termahal | ongkir_termahal | kota_termurah | ongkir_termurah | selisih |
|---|---|---|---|---|
| Jakarta Barat | 60.000 | Bekasi | 50.000 | 10.000 |

### 5. Nilai pesanan refund dan persentasenya terhadap gross sales

```sql
SELECT
  SUM(CASE WHEN status_clean = 'refund' THEN total_sales ELSE 0 END) AS total_refund,
  SUM(total_sales) AS gross_sales,
  ROUND(
    SUM(CASE WHEN status_clean = 'refund' THEN total_sales ELSE 0 END)
    / SUM(total_sales) * 100, 2
  ) AS persen_refund
FROM `toko_peralatan_dapur.orders`;
```

| total_refund | gross_sales | persen_refund |
|---|---|---|
| 279.711.700 | 5.872.699.300 | 4,76 |

### 6. Top 5 produk dengan rata-rata quantity per pesanan tertinggi (minimal 50 pesanan completed)

```sql
SELECT
  product_name,
  COUNT(DISTINCT order_id) AS jumlah_pesanan,
  AVG(quantity) AS avg_qty_per_pesanan
FROM `toko_peralatan_dapur.orders`
WHERE status_clean = 'complete'
GROUP BY product_name
HAVING COUNT(DISTINCT order_id) >= 50
ORDER BY avg_qty_per_pesanan DESC
LIMIT 5;
```

| product_name | jumlah_pesanan | avg_qty_per_pesanan |
|---|---|---|
| Wajan Anti Lengket 24 cm | 161 | 2,13 |
| Penggiling Bumbu Manual | 133 | 2,12 |
| Kompor Induksi | 155 | 2,12 |
| Saringan Teh Stainless | 161 | 2,09 |
| Panci Set 3 pcs | 151 | 2,08 |

### 7. Bulan dengan revenue completed tertinggi untuk setiap kategori

```sql
WITH per_bulan AS (
  SELECT
    category_clean,
    FORMAT_DATE('%Y-%m', sales_date) AS bulan,
    SUM(total_sales) AS revenue
  FROM `toko_peralatan_dapur.orders`
  WHERE status_clean = 'complete'
  GROUP BY category_clean, bulan
),
ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY category_clean ORDER BY revenue DESC) AS rn
  FROM per_bulan
)
SELECT category_clean, bulan, revenue
FROM ranked
WHERE rn = 1;
```

| category_clean | bulan | revenue |
|---|---|---|
| Alat Masak | 2025-08 | 384.961.500 |
| Alat Saji | 2025-06 | 52.461.100 |
| Alat Simpan | 2025-07 | 51.585.700 |

### 8. Jumlah produk yang menyumbang 80% dari total revenue completed (analisis Pareto)

```sql
WITH revenue_per_produk AS (
  SELECT product_name, SUM(total_sales) AS revenue
  FROM `toko_peralatan_dapur.orders`
  WHERE status_clean = 'complete'
  GROUP BY product_name
),
cum AS (
  SELECT
    product_name,
    revenue,
    SUM(revenue) OVER (ORDER BY revenue DESC) AS cum_revenue,
    SUM(revenue) OVER () AS total_revenue,
    ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rank_produk
  FROM revenue_per_produk
)
SELECT MIN(rank_produk) AS jumlah_produk_80_persen
FROM cum
WHERE cum_revenue >= 0.8 * total_revenue;
```

| jumlah_produk_80_persen |
|---|
| 26 |

**Insight:** 26 dari 57 produk (sekitar 46%) menyumbang 80% revenue completed.

### 9. Jeda rata-rata antar pesanan untuk pelanggan dengan lebih dari 5 pesanan completed

```sql
WITH pelanggan_valid AS (
  SELECT customer_name_clean
  FROM `toko_peralatan_dapur.orders`
  WHERE status_clean = 'complete'
  GROUP BY customer_name_clean
  HAVING COUNT(DISTINCT order_id) > 5
),
tanggal_urut AS (
  SELECT
    t.customer_name_clean,
    t.sales_date,
    LAG(t.sales_date) OVER (
      PARTITION BY t.customer_name_clean ORDER BY t.sales_date
    ) AS tanggal_sebelumnya
  FROM `toko_peralatan_dapur.orders` t
  JOIN pelanggan_valid p ON t.customer_name_clean = p.customer_name_clean
  WHERE t.status_clean = 'complete'
),
jeda AS (
  SELECT
    customer_name_clean,
    AVG(DATE_DIFF(sales_date, tanggal_sebelumnya, DAY)) AS avg_jeda_hari
  FROM tanggal_urut
  WHERE tanggal_sebelumnya IS NOT NULL
  GROUP BY customer_name_clean
)
SELECT
  (SELECT AVG(avg_jeda_hari) FROM jeda) AS rata_rata_jeda_keseluruhan,
  (SELECT customer_name_clean FROM jeda ORDER BY avg_jeda_hari ASC LIMIT 1) AS jeda_pelanggan_tersingkat,
  (SELECT MIN(avg_jeda_hari) FROM jeda) AS jeda_tersingkat_hari;
```

| rata_rata_jeda_keseluruhan | jeda_pelanggan_tersingkat | jeda_tersingkat_hari |
|---|---|---|
| 40,76 | Customer_3 | 9,93 |

### 10. Produk dengan refund rate tertinggi dan potensi revenue yang bisa diselamatkan

Potensi revenue dihitung dari pesanan refund produk tersebut jika refund rate turun ke rata-rata toko (sekitar 5%).

```sql
WITH per_produk AS (
  SELECT
    product_name,
    COUNTIF(status_clean = 'refund') AS jumlah_refund,
    COUNT(*) AS total_pesanan,
    SUM(CASE WHEN status_clean = 'refund' THEN total_sales ELSE 0 END) AS revenue_refund,
    SAFE_DIVIDE(COUNTIF(status_clean = 'refund'), COUNT(*)) AS refund_rate
  FROM `toko_peralatan_dapur.orders`
  GROUP BY product_name
)
SELECT
  product_name,
  refund_rate,
  revenue_refund,
  revenue_refund * GREATEST(refund_rate - 0.05, 0) / refund_rate AS potensi_revenue_terselamatkan
FROM per_produk
ORDER BY refund_rate DESC
LIMIT 1;
```

| product_name | refund_rate | revenue_refund | potensi_revenue_terselamatkan |
|---|---|---|---|
| Panci Stainless 20 cm | 8,29% | 7.840.000 | 3.111.500 |

## Catatan Asumsi

- `total_sales` pada data contoh sama dengan `total` (setelah diskon) ditambah `shipping_fee`, sehingga revenue di project ini **sudah termasuk ongkos kirim**.
- `gross_sales` (soal 5) adalah jumlah `total_sales` dari semua status, termasuk pesanan cancel dan refund.
- Refund rate (soal 10) dihitung dari jumlah pesanan refund dibagi seluruh pesanan produk tersebut (semua status).
- Soal 4: jika ada lebih dari satu kota dengan rata-rata ongkir yang sama, `LIMIT 1` hanya menampilkan salah satunya.
- Soal 9: rata-rata keseluruhan adalah rata-rata dari rata-rata jeda per pelanggan.

## Proyek Terkait

- Dashboard Looker Studio (Kitchen Equipment Sales Performance): [tambahkan link]
- Pembersihan data dengan Excel dan analisis dengan Python (Google Colab): [tambahkan link repositori]

## Tools

Google BigQuery (SQL) · Microsoft Excel (pembersihan data)
