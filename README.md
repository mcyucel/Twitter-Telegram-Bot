# Twitter-Telegram-Bot
Twitter'dan otomatik tweet çekip çeviri yapıp Telegram'a gönderen Python botu

```
import os
import time
import logging
from playwright.sync_api import sync_playwright
from deep_translator import GoogleTranslator
import requests
import random

# === AYARLAR ===
bot_token = "<bot token>"
chat_id = "<chat id>"
kullanici_adi = "<veri çekeceğin twitter kullanıcısı başında @ olmadan>"
dosya_adi = "onceki_tweet.txt"
USER_DATA_DIR = "./twitter_session"
CHECK_INTERVAL = 60  # Kontrol aralığı (saniye)

# Log ayarları
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("twitter_bot.log", encoding='utf-8')
    ]
)

# === TWEET ÇEKME FONKSİYONU ===
def son_tweeti_al(kullanici_adi):
    with sync_playwright() as p:
        browser = None
        try:
            # Rastgele user agent oluştur
            user_agents = [
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36",
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15",
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:126.0) Gecko/20100101 Firefox/126.0",
                "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
            ]
            
            # Kalıcı oturum için context oluştur
            browser = p.chromium.launch_persistent_context(
                user_data_dir=USER_DATA_DIR,
                headless=False,
                viewport={"width": 1280, "height": 720},
                args=[
                    "--disable-blink-features=AutomationControlled",
                    "--no-sandbox",
                    "--disable-dev-shm-usage",
                    "--disable-extensions",
                    "--disable-gpu",
                    "--disable-web-security",
                    "--disable-features=IsolateOrigins,site-per-process",
                    "--disable-site-isolation-trials",
                    "--dns-server=8.8.8.8"  # Google DNS
                ],
                locale="en-US",
                user_agent=random.choice(user_agents),
                timeout=180000  # 3 dakika
            )
            
            page = browser.new_page()
            page.set_default_timeout(180000)  # 3 dakika
            url = f"https://twitter.com/{kullanici_adi}"
            
            # JavaScript engellemeyi atlat
            page.add_init_script("""
                delete navigator.__proto__.webdriver;
                Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
                Object.defineProperty(navigator, 'plugins', { get: () => [1, 2, 3, 4, 5] });
                Object.defineProperty(navigator, 'languages', { get: () => ['en-US', 'en'] });
                window.localStorage.setItem('debug', '*');
            """)
            
            # Sayfayı yükleme stratejisi
            max_retries = 3
            for attempt in range(max_retries):
                try:
                    # Sayfayı yükle (sadece DOM yüklensin)
                    page.goto(url, wait_until="domcontentloaded", timeout=120000)
                    
                    # Çerez onayını kontrol et ve kabul et
                    try:
                        accept_button = page.get_by_role("button", name="Accept all cookies", timeout=5000)
                        if accept_button:
                            accept_button.click()
                            logging.info("Çerezler kabul edildi")
                            time.sleep(1)
                    except:
                        pass
                    
                    # Giriş kontrolü
                    login_required = False
                    try:
                        if (page.is_visible('text=Log in', timeout=10000) or 
                            page.is_visible('text=Sign in', timeout=10000) or 
                            page.is_visible('text="Log in"', timeout=10000) or
                            page.is_visible('input[name="username"]', timeout=5000)):
                            login_required = True
                    except:
                        pass
                        
                    if login_required:
                        logging.warning("Giriş gerekli! Lütfen tarayıcı penceresinde giriş yapın ve 'Beni hatırla' seçin")
                        input("Giriş yaptıktan sonra bu pencereye dönüp ENTER'a basın...")
                        page.goto(url, wait_until="domcontentloaded", timeout=120000)
                    
                    # Tweet'leri bekle
                    try:
                        page.wait_for_selector('article[data-testid="tweet"]', timeout=90000)
                    except:
                        try:
                            page.wait_for_selector('div[data-testid="tweetText"]', timeout=90000)
                        except:
                            page.wait_for_selector('article', timeout=90000)
                    
                    # Ek bekleme süresi
                    time.sleep(3)
                    
                    # Tüm tweet'leri al ve sabit tweet'leri atla
                    tweet_articles = page.query_selector_all('article')
                    if tweet_articles:
                        # Sabit tweet'leri atla ve ilk normal tweet'i bul
                        for article in tweet_articles:
                            try:
                                # Yeni sabit tweet kontrolü
                                pinned_element = article.query_selector('div[data-testid="socialContext"]')
                                if pinned_element and "Sabitlendi" in pinned_element.inner_text():
                                    logging.info("Sabit tweet atlandı")
                                    continue

                                # Tweet metnini bulmaya çalış
                                tweet_text = article.query_selector('div[data-testid="tweetText"]')
                                if tweet_text:
                                    return tweet_text.inner_text().strip()
                                
                                # Alternatif metin bulma
                                tweet_text = article.query_selector('div[lang]')
                                if tweet_text:
                                    return tweet_text.inner_text().strip()
                                
                                # Daha genel yaklaşım
                                tweet_text = article.query_selector('div[data-testid="tweet"] div[lang]')
                                if tweet_text:
                                    return tweet_text.inner_text().strip()
                            except Exception as e:
                                logging.error(f"Tweet işleme hatası: {str(e)}")
                                continue
                    
                    return "Tweet bulunamadı."
                    
                except Exception as e:
                    if attempt < max_retries - 1:
                        wait_time = (attempt + 1) * 10
                        logging.warning(f"Deneme {attempt+1}/{max_retries} başarısız. {wait_time} saniye sonra yeniden denenecek... Hata: {str(e)}")
                        time.sleep(wait_time)
                        page.reload(wait_until="domcontentloaded", timeout=90000)
                    else:
                        logging.error(f"Tüm denemeler başarısız oldu: {str(e)}")
                        return f"Hata: {str(e)}"
            
            return "Tweet bulunamadı."
            
        except Exception as e:
            logging.error(f"Hata oluştu: {str(e)}")
            return f"Hata: {str(e)}"
            
        finally:
            if browser:
                try:
                    browser.close()
                except:
                    pass

# === ÇEVİRİ FONKSİYONU ===
def cevir_turkceye(metin):
    try:
        # Uzun metinleri parçalara ayır
        if len(metin) > 5000:
            parts = [metin[i:i+4500] for i in range(0, len(metin), 4500)]
            return "".join(GoogleTranslator(source='auto', target='tr').translate(part) for part in parts)
        return GoogleTranslator(source='auto', target='tr').translate(metin)
    except Exception as e:
        logging.error(f"Çeviri hatası: {str(e)}")
        return "Çeviri yapılamadı"

# === TELEGRAM GÖNDERME FONKSİYONU ===
def telegrama_gonder(metin):
    max_length = 3900
    if len(metin) <= max_length:
        parts = [metin]
    else:
        parts = [metin[i:i+max_length] for i in range(0, len(metin), max_length)]
    
    for i, part in enumerate(parts):
        mesaj = f"{part}"
        if len(parts) > 1:
            mesaj += f"\n\n📌 Bölüm {i+1}/{len(parts)}"
        
        url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
        payload = {
            "chat_id": chat_id,
            "text": mesaj,
            "parse_mode": "Markdown",
            "disable_web_page_preview": True
        }
        
        try:
            response = requests.post(url, json=payload, timeout=20)
            if response.status_code == 200:
                logging.info(f"Telegram'a gönderildi (Bölüm {i+1}/{len(parts)})")
            else:
                logging.error(f"Telegram hatası ({response.status_code}): {response.text[:100]}")
        except Exception as e:
            logging.error(f"Telegram bağlantı hatası: {str(e)}")
        
        time.sleep(1)

# === ANA İŞLEM ===
def main():
    logging.info(f"{kullanici_adi} için tweet kontrolü başlatılıyor...")
    
    # Tweet'i al
    tweet = son_tweeti_al(kullanici_adi)
    
    # Kısaltılmış tweet logu
    if tweet and len(tweet) > 70:
        log_tweet = tweet[:70] + "..."
    else:
        log_tweet = tweet or "Boş tweet"
    logging.info(f"Alınan tweet: {log_tweet}")

    # Hata kontrolü
    if tweet is None or tweet.startswith("Hata:") or tweet == "Tweet bulunamadı.":
        logging.warning("Tweet alınamadı, işlem iptal edildi")
        return

    # Dosya kontrolü
    if not os.path.exists(dosya_adi):
        with open(dosya_adi, "w", encoding="utf-8") as f:
            f.write("")

    # Önceki tweet'i oku
    try:
        with open(dosya_adi, "r", encoding="utf-8") as f:
            onceki_tweet = f.read().strip()
    except:
        onceki_tweet = ""

    # Yeni tweet kontrolü
    if tweet != onceki_tweet:
        logging.info("Yeni tweet tespit edildi!")
        
        # Çevir ve gönder
        try:
            ceviri = cevir_turkceye(tweet)
            telegrama_gonder(ceviri)
        except Exception as e:
            logging.error(f"Gönderim hatası: {str(e)}")
        
        # Dosyayı güncelle
        with open(dosya_adi, "w", encoding="utf-8") as f:
            f.write(tweet)
        logging.info("Sistem güncellendi")
    else:
        logging.info("Yeni tweet yok")

# === SÜREKLİ ÇALIŞAN BOT ===
def run_bot():
    logging.info("Twitter Bot başlatıldı")
    logging.info(f"Her {CHECK_INTERVAL} saniyede bir kontrol edecek")
    
    # Oturum dizini yoksa oluştur
    if not os.path.exists(USER_DATA_DIR):
        os.makedirs(USER_DATA_DIR)
    
    while True:
        try:
            main()
        except Exception as e:
            logging.error(f"Ana işlem hatası: {str(e)}")
```
        
        logging.info(f"{CHECK_INTERVAL} saniye sonra tekrar kontrol edilecek...")
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    run_bot()
