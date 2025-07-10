# MIMIC-IV VERİ TABANI KULLANILARAK SEPSİSLİ HASTALARDA DELİRYUM VE MORTALİTE RİSKLERİNİN YAPAY ZEKA İLE TAHMİNİ


## 📌 Proje Özeti

Bu projede, **sepsis tanısı almış yoğun bakım hastalarında**, aynı anda hem **14 günlük mortalite** riskini hem de **24 saat içinde gelişebilecek deliryum** durumunu tahmin eden **çok görevli bir yapay zeka modeli** geliştirilmiştir.

Bu çalışma, literatürde genellikle ayrı ayrı ele alınan bu iki klinik sonucu **eşzamanlı olarak öngörerek** klinik karar destek sistemlerine katkı sağlamayı amaçlamaktadır.

---

##  Veri Seti: MIMIC-IV

Projede kullanılan veri seti, [PhysioNet](https://physionet.org) platformundan erişilen **MIMIC-IV v3.1** veri tabanından oluşturulmuştur.

### Erişim Süreci:

- PhysioNet başvurusu
- Etik eğitimlerin tamamlanması (CITI Program)
- Kullanıcı sertifikalarının onaylanması
- Google Cloud BigQuery üzerinden sorgulama
- Google Colab ile BigQuery bağlantısı

### Veri Seçim Kriterleri:

- Sepsis tanısı (ICD-9 kodları: 99591, 99592, 78552)
- SOFA skoru ≥ 2
- ICU kalış süresi ≥ 24 saat
- Yaş ≥ 18
- Tek ICU kaydı
- Deliryum: CAM-ICU kriterlerine göre
- Mortalite: İlk 14 gün içinde ölüm (manuel etiketleme)
- Hariç: Psikiyatrik, nörolojik, travmatik, madde bağımlılığı öyküsü olan hastalar

---

## Değişkenler

- Toplam **585 hasta**, ilk 24 saatlik **klinik ve laboratuvar verileri**
- Kaynak tablolar: `chartevents`, `labevents`, `icustays`
- Örnek değişkenler: SpO2, Glucose, HeartRate, GCS, SAPS-II vb.
- Her değişken için: `min`, `max`, `median`, `avg` özetleri

---

## Veri Ön İşleme

### Eksik Veri Temizleme:

- %30’dan fazla eksik satırlar çıkarıldı → 73 hasta elendi
- Sütun bazında ciddi eksiklik yok
- Kalan: **585 hasta**, **144 değişken**

### Eksik Veri Doldurma:

- %5’ten az eksikler: **MICE + Bayesian Ridge (Iterative Imputer)**
- %5’ten fazla eksikler: **Random Forest Regressor**
- Alternatif yöntem: Medyan (sayısal), Mod (kategorik)

---

## Özellik Seçimi ve EDA

- Sayısal değişkenler: **T-testi / Mann-Whitney U testi**
- Kategorik değişkenler: **Ki-kare testi**

---

## Hedef Değişkenler

- **Deliryum**: İlk 24 saatte CAM-ICU pozitifliği
- **Mortalite**: İlk 14 gün içinde ölüm

---

## Modelleme Yaklaşımı

- Veri dengesizliği: **SMOTE, BorderlineSMOTE**
- Özellik seçimi: **SHAP (SHapley Additive Explanations)**
- Kullanılan algoritmalar:
  - XGBoost
  - CatBoost
  - Random Forest
  - Ensemble: VotingClassifier

---

## Model Aşamaları

Aşağıdaki süreçte 10 farklı model iterasyonu uygulanmıştır. Her adım, bir önceki adımdaki eksikleri tamamlayacak şekilde yapılandırılmıştır:


### Model 1:
İlk aşamada herhangi bir veri ön işleme veya dengeleme yöntemi uygulanmadan, sadece XGBoost algoritması ile mortalite ve deliryum hedef değişkenleri için ayrı ayrı modeller eğitilmiştir.

### Model 2:
Veri kümesi %70 eğitim, %15 doğrulama ve %15 test olacak şekilde ayrılmış; her hedef değişken için bu yapıda XGBoost modelleri yeniden eğitilmiştir.

### Model 3:
Veri oranları yeniden düzenlenerek %60 eğitim, %20 doğrulama ve %20 test oranı ile XGBoost modelleri tekrar eğitilmiştir. Bu adım, farklı veri ayırma stratejilerinin model performansına etkisini test etmeyi amaçlamıştır.

### Model 4:
Veri dengesizliğini gidermek için klasik SMOTE yöntemi uygulanmıştır. Azınlık sınıfa ait sentetik örnekler üretilerek, daha dengeli bir eğitim kümesi elde edilmiş ve XGBoost modelleri bu yeni veriyle eğitilmiştir.

### Model 5:
SMOTE’a alternatif olarak, sınıf sınırlarında örnek üreten BorderlineSMOTE yöntemi uygulanmıştır. Bu yöntem ile daha hassas sentetik veri üretimi sağlanarak XGBoost modelleri eğitilmiştir.

### Model 6:
Model 5’teki BorderlineSMOTE işlemine ek olarak, modelin doğruluğunu artırmak için Stratified K-Fold çapraz doğrulama yöntemi uygulanmıştır. Bu yöntem, her katmanda sınıfların dengeli dağılmasını sağlar ve daha güvenilir sonuçlar elde edilmesini hedefler. Her fold’da ayrı bir XGBoost modeli eğitilmiştir.

### Model 7:
Model 6’nın üzerine, veri ölçekleme işlemi olarak StandardScaler uygulanmıştır. Ardından XGBoost, CatBoost ve Random Forest modelleri birlikte kullanılarak VotingClassifier ensembel modeli oluşturulmuştur. Bu, birden fazla modelin çıktısına dayalı nihai tahmin üretir.

### Model 8:
Model 7’de kullanılan ensembel yapısı korunmuştur. Ancak bu kez istatistiksel olarak anlamlı olmayan değişkenler veri setinden çıkarılmış, böylece modelin karmaşıklığı azaltılarak daha sade bir yapı oluşturulmuştur. Amaç, overfitting’i önlemek ve modeli daha genellenebilir kılmaktır.

### Model 9:
Özellik seçimini daha etkili kılmak için, SHAP (SHapley Additive exPlanations) yöntemiyle değişken önem düzeyleri belirlenmiştir. En anlamlı bulunan değişkenlerle birlikte XGBoost ve CatBoost modelleri kullanılarak yeniden VotingClassifier ensembel modeli kurulmuştur. Bu yapı, en yüksek mortalite tahmin performansını göstermiştir.

### Model 10:
Model 9’un SHAP destekli yapısına ek olarak, istatistiksel testlerle anlamlı bulunmayan değişkenler de veri setinden çıkarılmıştır. Hem SHAP hem de istatistiksel yöntemle seçilen değişkenlerle yeniden eğitim yapılmış ve deliryum tahmin performansında en başarılı model bu aşamada elde edilmiştir.

---

## Model Performans Özeti

### **Mortalite**

| Model | ROC AUC | F1-Score (0) | F1-Score (1) | Accuracy |
|-------|---------|--------------|--------------|----------|
| 9     | 0.84    | 0.83         | 0.65         | 75–78%   |
| 10    | 0.84    | 0.83         | 0.65         | 75–78%   |

🔹 **En iyi model**: Model 9 (SHAP destekli)

### Mortalitenin Model Karşılaştırması

- **Model 1**: Hiçbir işlem yapılmadan XGBoost — temel performans
- **Model 2–3**: Eğitim/test oranları optimize edildi ama sınıf dengesizliği sorun
- **Model 4**: SMOTE uygulandı, sınırlı iyileşme
- **Model 5**: BorderlineSMOTE ile pozitif sınıf başarısı yükseldi
- **Model 6**: Stratified K-Fold ile istikrarlı sonuçlar
- **Model 7–8**: Ensemble model + anlamsız değişken elemesi ile gelişmiş yapı
- **Model 9**: SHAP ile anlamlı değişkenler → **en yüksek başarı**
- **Model 10**: SHAP + istatistiksel değişken seçimi ile optimize model

---

### **Deliryum**

| Model | ROC AUC | F1-Score (0) | F1-Score (1) | Accuracy |
|-------|---------|--------------|--------------|----------|
| 10    | 0.70    | 0.92         | 0.21         | 84–86%   |

🔹 **En iyi model**: Model 10 (istatistiksel + SHAP eleme)

### Deliryumun Model Karşılaştırması 

- **Model 1–3**: XGBoost + oran ayarları, ama pozitif sınıf tahmini başarısız
- **Model 4–5**: SMOTE ve BorderlineSMOTE ile F1-score (1) yükselmeye başladı
- **Model 6–7**: K-Fold + Ensemble ile denge sağlandı
- **Model 8**: Anlamsız değişkenler çıkarıldı → overfitting azaldı
- **Model 9**: SHAP destekli VotingClassifier → büyük sıçrama
- **Model 10**: Tüm metotlar birleştirildi → **en güçlü model**

---

## Tartışma ve Sonuçlar

- **Mortalite tahmininde en iyi performans** Model 9 ile (ROC AUC: 0.84, F1-1: 0.65)
- **Deliryum tahmininde en iyi model** Model 10 (ROC AUC: 0.70, F1-1: 0.21)
- **Veri dengesizliği**, özellikle deliryumda önemli bir zorluk yaratmıştır.
- **SHAP destekli değişken seçimi**, model başarısında belirleyici rol oynamıştır.
- **Ensemble yöntemleri**, genel performansı artırmıştır.


---

## Kaynakça

- Johnson, A. E. W., et al. (2020). MIMIC-IV (version 1.0).  
  [https://physionet.org/content/mimiciv/1.0/](https://physionet.org/content/mimiciv/1.0/)
- CITI Program: [https://about.citiprogram.org/](https://about.citiprogram.org/)

