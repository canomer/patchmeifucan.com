# Dijital Arşivlemede Python Dokunuşu: hist2.py ile Akıllı Dosya Yönetimi

Dosya yığınları arasında boğuluyor musunuz? İndirilenler klasörünüz veya proje dizininiz, hangi dosyanın daha önce oluşturulduğunu anlamadığınız bir karmaşaya mı dönüştü? Bugün, dosya sistemindeki zaman damgalarını (timestamps) kullanarak dosyaları otomatik olarak kronolojik sıraya dizen ve bunu yaparken veri güvenliğini ön planda tutan **`hist2.py`** aracını inceliyoruz.

## 1. Temel Mantık: Dosyalar Nasıl Sıralanır?

Bir işletim sistemi, her dosya için üç temel zaman damgası tutar:
*   **mtime (Modification Time):** Dosya içeriğinin en son ne zaman değiştiği.
*   **ctime (Creation/Change Time):** Dosyanın oluşturulduğu veya metadata bilgilerinin değiştiği an.

`hist2.py`, kullanıcıya bu iki kriterden birini seçme şansı vererek dosyaları geçmişten günümüze (veya tam tersi) sıralar.

## 2. Teknik Analiz ve Kod İncelemesi

İşte senin yazdığın ve üzerinde durmamız gereken o fonksiyonel kodun anatomisi:

```python
import os
import re
import argparse
from pathlib import Path
import time

# Dosya başındaki mevcut sayı öneklerini yakalamak için Regex
NUM_PREFIX_RE = re.compile(r'^(\d+)_')

def get_file_list(directory, sort_by='mtime', reverse=False):
    """Dizindeki dosyaları zaman damgasına göre sıralı döndürür."""
    p = Path(directory)
    # Sadece dosyaları filtrele (dizinleri atla)
    files = [f for f in p.iterdir() if f.is_file()]
    
    if sort_by == 'mtime':
        files.sort(key=lambda x: x.stat().st_mtime, reverse=reverse)
    else:
        files.sort(key=lambda x: x.stat().st_ctime, reverse=reverse)
    return files

def safe_rename(file_path, new_name):
    """Çakışmaları önlemek için önce geçici bir isme, sonra hedefe taşır."""
    temp_name = file_path.parent / f".__tmp__{time.time_ns()}_{file_path.name}"
    target_path = file_path.parent / new_name
    
    try:
        file_path.rename(temp_name)
        temp_name.rename(target_path)
        return True
    except Exception as e:
        print(f"[!] Hata: {file_path.name} -> {new_name} değiştirilemedi: {e}")
        return False

def process_directory(directory, sort_by='mtime', dedup=False, dry_run=False):
    files = get_file_list(directory, sort_by)
    seen_names = set()
    
    for idx, file_path in enumerate(files):
        original_name = file_path.name
        # Eğer dosya zaten "0_", "1_" gibi başlıyorsa onu temizle
        clean_name = NUM_PREFIX_RE.sub('', original_name)
        new_name = f"{idx}_{clean_name}"

        # De-duplication (Mükerrer dosya kontrolü)
        if dedup and clean_name in seen_names:
            if not dry_run:
                print(f"[-] Kopya Siliniyor: {original_name}")
                file_path.unlink()
            else:
                print(f"[*] (Dry-Run) Silinecek: {original_name}")
            continue

        seen_names.add(clean_name)

        if original_name != new_name:
            if not dry_run:
                print(f"[+] {original_name} -> {new_name}")
                safe_rename(file_path, new_name)
            else:
                print(f"[*] (Dry-Run) Değişecek: {original_name} -> {new_name}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Dosya Kronoloji Düzenleyici")
    parser.add_argument("dir", help="Hedef dizin")
    parser.add_argument("--sort", choices=['mtime', 'ctime'], default='mtime')
    parser.add_argument("--dedup", action="store_true", help="Aynı isimli eski kopyaları sil")
    parser.add_argument("--dry-run", action="store_true", help="Gerçek işlem yapmadan simüle et")
    
    args = parser.parse_args()
    process_directory(args.dir, args.sort, args.dedup, args.dry_run)
```

## 3. Öne Çıkan Özellikler ve "Best Practices"

### A. Race Condition ve Veri Kaybı Koruması (`safe_rename`)
Kodun en profesyonel kısmı `safe_rename` fonksiyonu. Eğer bir klasörde `1_data.txt` varken başka bir dosyayı `1_data.txt` yapmaya çalışırsanız, üzerine yazma (overwrite) riski doğar. 
*   **Çözüm:** Kod, dosyayı önce `time.time_ns()` (nanosaniye) kullanarak üretilen benzersiz bir geçici isme taşıyor, ardından hedef isme döndürüyor. Bu, "atomik" bir işlem simülasyonu yaratarak veri kaybını önler.

### B. Düzenli İfadeler (Regex) ile Akıllı Temizlik
`NUM_PREFIX_RE` kullanımı, betiğin defalarca çalıştırılmasına olanak tanır. Eğer dosyanızın adı `0_rapor.pdf` ise ve siz betiği tekrar çalıştırırsanız, dosya adını `0_0_rapor.pdf` yapmaz. Önce mevcut sayı önekini temizler (`sub`), sonra yeni indeksi ekler.

### C. De-duplication (Mükerrerlikten Kurtulma)
`--dedup` bayrağı aktifse, kod bir `set` (küme) yapısı kullanarak daha önce aynı isimle (önek hariç) karşılaşılıp karşılaşılmadığını kontrol eder. Eğer dosya daha önce görüldüyse ve kronolojik sırada daha sonraysa (yani daha yeniyse ama ismi aynıysa), onu silerek dizini temiz tutar.

## 4. Kullanım Senaryoları

1.  **Fotoğraf Arşivleme:** Telefonunuzdan farklı zamanlarda yüklediğiniz karmaşık isimli fotoğrafları çekilme sırasına göre `0_img.jpg`, `1_img.jpg` şeklinde dizin.
2.  **Log Analizi:** Farklı sunuculardan gelen log dosyalarını oluşma sırasına göre numaralandırarak analiz sürecini kolaylaştırın.
3.  **Yazılım Versiyonlama:** Manuel tutulan yedeklerin hangisinin en eski olduğunu bir bakışta görün.

## 5. Sonuç

`hist2.py`, Python'un `pathlib` kütüphanesinin gücünü ve modern dosya işleme tekniklerini bir araya getiriyor. Sadece dosya ismini değiştirmekle kalmıyor, aynı zamanda bir dosya sisteminde güvenli bir şekilde nasıl işlem yapılacağına dair harika bir ders niteliği taşıyor.

---
