# Form Bildirimi Kurulumu — Google Sheet + WhatsApp (CallMeBot)

Bu doküman, sitedeki ("Ücretsiz Analiz İçin Tıkla" / "Üye Ol") formunun gönderdiği bilgilerin
**Google E-Tablosu'na satır olarak eklenmesi** ve **Cengiz Hoca'ya WhatsApp bildirimi** gitmesi için
gereken kurulumu anlatır.

**Şu anki durum:** Site kodu (`index.html`) şu an SADECE ön yüz olarak çalışıyor — form doğrulanıyor
(KVKK onayı zorunlu), "Teşekkürler" mesajı gösteriliyor, ama veri hiçbir yere gönderilmiyor (sadece
tarayıcı konsoluna yazılıyor). Bu kurulum tamamlanıp aşağıdaki adım 4'teki URL `index.html` içine
yapıştırılınca gerçek gönderim başlayacak.

**Kim kuracak:** Adım 1 (tabloyu oluşturmak) tamamlandı. Adım 2'den itibaren (Apps Script, Dağıt,
CallMeBot aktivasyonu) Yunus Emre'nin (veya Cengiz Hoca'nın) kendi Google/WhatsApp hesabından yapması
gerekiyor — Claude bunları deploy edemez çünkü Google hesabı erişimi ve WhatsApp numarası onayı gerektirir.

---

## Adım 1 — Google E-Tablosu ✅ oluşturuldu

Tablo hazır, başlık satırı yazılı: **[Özlem Fitness — Form Başvuruları](https://docs.google.com/spreadsheets/d/1-uoC7MkGRYOOvMLcqaPyJiXREalBm0MtKDA2QKdvfv0/edit)**
(yeaydin.digital@gmail.com hesabında).

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| Tarih-Saat | Ad | Telefon | Doğum Günü | E-posta | Hangi Butondan Geldi | Aydınlatma Okundu | Ticari İleti İzni |

Sekme adı şu an "Untitled" duruyor — dilersen açıp alt sekmeye çift tıklayıp "Başvurular" gibi bir isim verebilirsin, zorunlu değil.

## Adım 2 — Apps Script kodunu ekle

1. E-tabloda üst menüden **Uzantılar → Apps Script** aç.
2. Açılan boş dosyadaki örnek kodu SİL, aşağıdaki kodu **birebir kopyala-yapıştır**:

```javascript
/**
 * Özlem Fitness — form bildirimi: Google Sheet'e satır ekler + Cengiz Hoca'ya
 * WhatsApp bildirimi gönderir (CallMeBot üzerinden, ücretsiz).
 *
 * KURULUM:
 * 1) Aşağıdaki CALLMEBOT_PHONE ve CALLMEBOT_APIKEY değerlerini doldur (Adım 5'e bak).
 * 2) Üstte Dağıt (Deploy) → Yeni dağıtım → Tür: Web uygulaması.
 *    - Yürütülecek kişi: Ben (kendi hesabın)
 *    - Erişebilenler: Herkes
 * 3) Verilen Web App URL'sini index.html'deki FORM_ENDPOINT_URL'ye yapıştır.
 */

var CALLMEBOT_PHONE = "90XXXXXXXXXX";   // Cengiz Hoca'nın WhatsApp numarası, ülke kodu ile, + işareti YOK
var CALLMEBOT_APIKEY = "XXXXXXX";        // CallMeBot'un sana verdiği apikey (Adım 5)

function doPost(e) {
  try {
    var data = JSON.parse(e.postData.contents);
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();

    var simdi = new Date();
    var tarihSaat = Utilities.formatDate(simdi, "Europe/Istanbul", "dd.MM.yyyy HH:mm");
    var dogumGunu = "";
    if (data.dogum_gun && data.dogum_ay) {
      dogumGunu = data.dogum_gun + "/" + data.dogum_ay;
    }

    sheet.appendRow([
      tarihSaat,
      data.ad || "",
      data.telefon || "",
      dogumGunu,
      data.eposta || "",
      data.kaynak || "",
      data.aydinlatma_okundu || "Hayır",
      data.ticari_ileti_izni || "Hayır"
    ]);

    // WhatsApp bildirimi (CallMeBot) — kurulmadıysa (apikey boşsa) bu adım sessizce atlanır
    if (CALLMEBOT_APIKEY && CALLMEBOT_APIKEY !== "XXXXXXX") {
      var mesaj = "Yeni form başvurusu!\n" +
        "Ad: " + (data.ad || "-") + "\n" +
        "Telefon: " + (data.telefon || "-") + "\n" +
        "Nereden: " + (data.kaynak || "-") + "\n" +
        "Tarih: " + tarihSaat;
      var url = "https://api.callmebot.com/whatsapp.php" +
        "?phone=" + CALLMEBOT_PHONE +
        "&text=" + encodeURIComponent(mesaj) +
        "&apikey=" + CALLMEBOT_APIKEY;
      try { UrlFetchApp.fetch(url); } catch (waErr) { /* WhatsApp gitmese de Sheet kaydı zaten yapıldı */ }
    }

    return ContentService.createTextOutput(JSON.stringify({ ok: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ ok: false, error: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// Tarayıcıdan linke tıklayarak test etmek için (form POST değil, GET deneme):
function doGet(e) {
  return ContentService.createTextOutput("Çalışıyor. Bu adres sadece POST (form) isteği kabul eder.");
}
```

3. Üstte **Kaydet** (disket ikonu), dosyaya bir isim ver (örn. "FormBildirimi").

## Adım 3 — Web App olarak yayınla (Deploy)

1. Sağ üstte **Dağıt (Deploy) → Yeni dağıtım (New deployment)**.
2. Tür seç: **Web uygulaması (Web app)**.
3. Ayarlar:
   - **Yürütülecek kişi (Execute as):** Ben (kendi hesabın)
   - **Erişebilenler (Who has access):** Herkes (Anyone) — form herkesten gelecek, bu normal.
4. **Dağıt**'a bas. Google ilk seferde "doğrulanmamış uygulama" uyarısı gösterebilir:
   "Gelişmiş (Advanced)" → "[Proje adı]'na git (devam et)" → izin ver.
5. Sana bir **Web App URL** verecek, şuna benzer:
   `https://script.google.com/macros/s/AKfycb.../exec`
   Bu URL'yi kopyala.

## Adım 4 — URL'yi siteye ekle

`web sitesi/index.html` içinde şu satırı bul (form submit script'inin başında, "FORM" yorumunun altında):

```javascript
var FORM_ENDPOINT_URL = ""; // YOUR_APPS_SCRIPT_WEB_APP_URL buraya gelecek
```

Tırnak içine Adım 3'te aldığın URL'yi yapıştır:

```javascript
var FORM_ENDPOINT_URL = "https://script.google.com/macros/s/AKfycb.../exec";
```

Kaydet, siteyi yeniden yayınla (bu adım ayrı bir onay/deploy süreci — danışman KVKK/Çerez metinlerini
onaylamadan site zaten yayına girmeyecek, o yüzden bu değişikliği de aynı yayın turunda yapabilirsin).

## Adım 5 — CallMeBot ile WhatsApp bildirimi (ücretsiz, tek seferlik kurulum)

CallMeBot, kendi WhatsApp numarandan tek bir onay mesajı göndererek seni "bot"un gönderebileceği
kişiler listesine ekleyen ücretsiz bir servistir. Cengiz Hoca'nın kendi telefonuyla yapması gerekir:

1. Cengiz Hoca'nın WhatsApp'ından şu numarayı rehbere ekle: **+34 644 59 71 20**
2. Bu numaraya WhatsApp'tan şu mesajı gönder (birebir):
   `I allow callmebot to send me messages`
3. Birkaç dakika içinde CallMeBot'tan bir **apikey** (sayılardan oluşan bir kod) içeren cevap gelecek.
4. Bu apikey'i Adım 2'deki kod içinde `CALLMEBOT_APIKEY = "XXXXXXX"` satırına yapıştır, ve
   `CALLMEBOT_PHONE = "90XXXXXXXXXX"` satırına Cengiz Hoca'nın numarasını (ülke kodu 90 ile, boşluksuz,
   başında + olmadan) yaz.
5. Apps Script'te kodu güncelleyip tekrar **Dağıt → Dağıtımları Yönet → Düzenle (kalem ikonu) → Yeni
   sürüm → Dağıt** yaparak güncellemeyi yayına al (URL değişmez, sadece kod güncellenir).

**Not:** CallMeBot ücretsizdir ama resmî bir WhatsApp Business API değildir; gönderim hacmi çok
artarsa (günde çok sayıda mesaj) CallMeBot geçici olarak mesajları geciktirebilir veya reddedebilir.
Düşük hacimli bir salon formu için yeterlidir. Google Sheet'e satır eklenmesi CallMeBot'tan bağımsız
çalışır — WhatsApp bildirimi gitmese de başvuru kaydı güvenceye alınmış olur.

## Adım 6 — Test et

1. Siteyi aç, formu bir kez gerçek bilgilerinle (veya test bilgisiyle) doldurup gönder.
2. Google Sheet'e dön, yeni bir satır eklendiğini kontrol et.
3. Cengiz Hoca'nın WhatsApp'ına bildirim geldi mi bak.
4. Tarayıcı konsolunda (F12 → Console) kırmızı hata olup olmadığını kontrol et.

## Google E-Tablosu sütunları (referans)

| Sütun | Form alanı | Örnek |
|---|---|---|
| Tarih-Saat | otomatik, sunucu saati | 08.10.2026 14:32 |
| Ad | Adın | Ahmet Yılmaz |
| Telefon | Telefon numaran | 0532 123 45 67 |
| Doğum Günü | Doğum günün (gün/ay, isteğe bağlı) | 14/3 |
| E-posta | E-posta adresin (isteğe bağlı) | ahmet@mail.com |
| Hangi Butondan Geldi | gizli "kaynak" alanı | Hero / Salonumuz / Üye Ol / Üst Menü / Alt Köşe Butonu vb. |
| Aydınlatma Okundu | zorunlu KVKK onay kutusu | Evet |
| Ticari İleti İzni | isteğe bağlı kampanya onay kutusu | Evet / Hayır |

**KVKK notu:** "Ticari İleti İzni" sütunu "Hayır" olan hiçbir kayda pazarlama/kampanya/doğum günü
mesajı gönderilmemeli — bu onay olmadan gönderilen ticari e-posta/WhatsApp mesajı ceza riski taşır
(bkz. `~/.claude/notes/n8n-otomasyon-dersleri.md` ve KVKK genel kuralları). Takip/hatırlatma
otomasyonu kurulacaksa önce İYS kaydı netleşmeli.
