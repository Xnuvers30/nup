import random
import time
import os
import traceback
from datetime import datetime

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select, WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.service import Service

import names

# =========================
# Konfigurasi environment
# =========================
CHROME_DRIVER_PATH = "/usr/bin/chromedriver"
CHROMIUM_PATH = "/usr/bin/chromium"
DEFAULT_WAIT = 25

# =========================
# Util logging real-time
# =========================
def log(msg):
    ts = datetime.now().strftime("%H:%M:%S")
    print(f"[{ts}] {msg}", flush=True)

# =========================
# Data helper
# =========================
bulan_list_id = [
    "Januari","Februari","Maret","April","Mei","Juni",
    "Juli","Agustus","September","Oktober","November","Desember"
]

def generate_random_data():
    first_name = names.get_first_name()
    last_name = names.get_last_name()
    full_name = f"{first_name} {last_name}"
    username = f"{first_name.lower()}{last_name.lower()}{random.randint(100,999)}"
    password = f"{first_name}{random.randint(1000,9999)}!"
    birth_year = random.randint(1985, 2005)
    birth_month = random.choice(bulan_list_id)
    birth_day = random.randint(1, 28)
    return {
        "full_name": full_name,
        "username": username,
        "password": password,
        "birth_year": str(birth_year),
        "birth_month": birth_month,
        "birth_day": str(birth_day)
    }

# =========================
# Selenium helpers
# =========================
def find_any(driver, wait, locators, condition="presence"):
    cond_map = {
        "presence": EC.presence_of_element_located,
        "clickable": EC.element_to_be_clickable,
        "visible": EC.visibility_of_element_located,
    }
    last_err = None
    for how, what in locators:
        try:
            return wait.until(cond_map[condition]((how, what)))
        except Exception as e:
            last_err = e
    if last_err:
        raise last_err

def click_any(driver, wait, locators):
    el = find_any(driver, wait, locators, condition="clickable")
    el.click()
    return el

def first_visible_input(driver, xpaths):
    for xp in xpaths:
        els = driver.find_elements(By.XPATH, xp)
        for el in els:
            try:
                if el.is_displayed() and el.is_enabled():
                    return el
            except Exception:
                pass
    return None

def save_debug_html(driver, filename="debug_page.html"):
    try:
        with open(filename, "w", encoding="utf-8") as f:
            f.write(driver.page_source)
        log(f"HTML halaman disimpan ke {filename}")
    except Exception as e:
        log(f"Gagal simpan debug HTML: {e}")

# =========================
# Universal Step Detector
# =========================
def detect_and_handle_step(driver, wait, data):
    """
    Scan halaman dan lakukan aksi jika mendeteksi:
    - birthdate selects
    - email OTP input
    - captcha iframe
    - phone number request
    Mengembalikan salah satu string status:
    'birthdate', 'otp_submitted', 'captcha_wait', 'phone', 'unknown', 'done'
    """
    log("🔍 Scan halaman untuk deteksi langkah berikutnya...")
    page_source = (driver.page_source or "").lower()

    # Heuristik: jika sudah sampai feed/home (kadang langsung selesai)
    # Cari indikator UI yang umum (best-effort)
    possible_done_markers = [
        "//a[@href='/explore/']",
        "//a[@aria-label='Beranda' or @aria-label='Home']",
        "//div[contains(text(),'Welcome to Instagram') or contains(text(),'Selamat datang')]",
    ]
    for xp in possible_done_markers:
        if driver.find_elements(By.XPATH, xp):
            log("🔔 Terindikasi sudah masuk beranda / selesai onboarding.")
            return "done"

    # 1) OTP Email: cari input 6-digit atau field khusus
    otp_input = first_visible_input(driver, [
        "//input[@name='confirmationCode']",
        "//input[@name='email_confirmation_code']",
        "//input[@inputmode='numeric']",
        "//input[@type='tel']",
        "//input[@type='text' and (contains(@aria-label,'Code') or contains(@aria-label,'Kode'))]",
        "//input[@autocomplete='one-time-code']",
    ])
    if otp_input:
        log("✅ Form OTP terdeteksi.")
        # Minta input OTP manual di terminal
        otp = input("Masukkan Kode OTP dari email: ").strip()
        try:
            otp_input.clear()
        except Exception:
            pass
        otp_input.send_keys(otp)

        # Coba klik tombol submit/Next
        try:
            click_any(driver, WebDriverWait(driver, 5), [
                (By.XPATH, "//button[contains(text(),'Selanjutnya')]"),
                (By.XPATH, "//button[contains(text(),'Next')]"),
                (By.XPATH, "//button[@type='submit' and not(@disabled)]"),
                (By.XPATH, "//div[@role='button' and (contains(.,'Next') or contains(.,'Selanjutnya'))]"),
            ])
        except Exception:
            log("Tidak menemukan tombol submit OTP yang jelas. Lanjut best-effort.")
        return "otp_submitted"

    # 2) Tanggal lahir: dropdown <select>
    try:
        month = first_visible_input(driver, ["//select[contains(@aria-label,'Bulan') or contains(@aria-label,'Month')]"])
        day   = first_visible_input(driver, ["//select[contains(@aria-label,'Tanggal') or contains(@aria-label,'Day')]"])
        year  = first_visible_input(driver, ["//select[contains(@aria-label,'Tahun') or contains(@aria-label,'Year')]"])
        if month and day and year:
            log("✅ Form tanggal lahir terdeteksi.")
            # Isi bulan (support ID/EN)
            try:
                Select(month).select_by_visible_text(data["birth_month"])
            except Exception:
                map_en = {
                    "Januari":"January","Februari":"February","Maret":"March","April":"April",
                    "Mei":"May","Juni":"June","Juli":"July","Agustus":"August","September":"September",
                    "Oktober":"October","November":"November","Desember":"December"
                }
                Select(month).select_by_visible_text(map_en[data["birth_month"]])
            Select(day).select_by_visible_text(data["birth_day"])
            Select(year).select_by_visible_text(data["birth_year"])

            # Submit
            try:
                click_any(driver, WebDriverWait(driver, 5), [
                    (By.XPATH, "//button[contains(text(),'Selanjutnya')]"),
                    (By.XPATH, "//button[contains(text(),'Next')]"),
                    (By.XPATH, "//button[@type='submit' and not(@disabled)]"),
                ])
            except Exception:
                log("Tombol submit tanggal lahir tidak ditemukan, lanjut best-effort.")
            return "birthdate"
    except Exception:
        pass

    # 3) Captcha (iframe/keyword)
    captcha_iframe = driver.find_elements(
        By.XPATH,
        "//iframe[contains(@src,'captcha') or contains(@title,'captcha') or contains(@src,'arkose') or contains(@src,'hcaptcha')]"
    )
    if captcha_iframe or "captcha" in page_source:
        log("⚠️ CAPTCHA terdeteksi! Silakan selesaikan manual di browser.")
        input("Tekan ENTER setelah captcha selesai...")
        return "captcha_wait"

    # 4) Nomor HP / challenge phone
    phone_markers = [
        "//input[@type='tel']",
        "//label[contains(.,'Phone') or contains(.,'Telepon') or contains(.,'Nomor')]",
        "//input[contains(@aria-label,'Phone') or contains(@aria-label,'Telepon')]",
    ]
    for xp in phone_markers:
        if first_visible_input(driver, [xp]):
            log("⚠️ Halaman meminta nomor HP. Di-skip otomatis (tidak mengisi).")
            return "phone"

    # 5) Tidak ada yang cocok → dump HTML dan lanjut
    log("⚠️ Tidak ada elemen penting terdeteksi. Skip step ini.")
    save_debug_html(driver, "debug_page.html")
    return "unknown"

# =========================
# Main
# =========================
def main():
    EMAIL = input("Masukkan email: ").strip()
    log("Membuat data acak (nama, username, password)...")
    data = generate_random_data()
    log(f"Nama: {data['full_name']}")
    log(f"Username: {data['username']}")
    log(f"Password: {data['password']}")
    log(f"Tgl lahir: {data['birth_day']} {data['birth_month']} {data['birth_year']}")

    # Chrome options
    chrome_options = webdriver.ChromeOptions()
    chrome_options.binary_location = CHROMIUM_PATH
    chrome_options.add_argument("--disable-notifications")
    chrome_options.add_argument("--disable-infobars")
    chrome_options.add_argument("--disable-extensions")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--incognito")
    chrome_options.add_argument("--guest")
    chrome_options.add_argument("--headless=new")  # matikan kalau mau lihat GUI
    # (Opsional) spoof UA biar lebih stabil:
    chrome_options.add_argument("--user-agent=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
                                "(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36")

    log(f"Menggunakan Chromium: {CHROMIUM_PATH}")
    service = Service(executable_path=CHROME_DRIVER_PATH)

    try:
        driver = webdriver.Chrome(service=service, options=chrome_options)
        driver.set_window_size(1280, 900)
        wait = WebDriverWait(driver, DEFAULT_WAIT)
        log("Browser berhasil dijalankan.")

        # 1) Buka Instagram home → klik "Buat akun"
        driver.get("https://www.instagram.com/")
        log("Membuka instagram.com ...")

        click_any(
            driver, wait,
            [
                (By.XPATH, "//a[contains(text(),'Buat akun')]"),
                (By.XPATH, "//a[contains(text(),'Sign up')]"),
                (By.XPATH, "//a[@href and contains(@href, 'accounts/emailsignup')]"),
            ],
        )
        log("Masuk ke halaman pendaftaran.")

        # 2) Isi form pendaftaran
        log("Mengisi form pendaftaran (email/nama/username/password)...")
        find_any(driver, wait, [(By.NAME, "emailOrPhone")], "visible").send_keys(EMAIL)
        find_any(driver, wait, [(By.NAME, "fullName")], "visible").send_keys(data['full_name'])
        find_any(driver, wait, [(By.NAME, "username")], "visible").send_keys(data['username'])
        find_any(driver, wait, [(By.NAME, "password")], "visible").send_keys(data['password'])

        log("Mengirim form pendaftaran...")
        click_any(
            driver, wait,
            [
                (By.XPATH, "//button[@type='submit' and not(@disabled)]"),
                (By.XPATH, "//button[contains(text(),'Daftar')]"),
                (By.XPATH, "//button[contains(text(),'Sign up')]"),
            ],
        )

        # 3) Loop deteksi langkah berikutnya agar tidak nunggu error
        max_cycles = 8
        for i in range(max_cycles):
            step = detect_and_handle_step(driver, wait, data)
            log(f"➡️ Step terdeteksi: {step}")
            # Jika sudah OTP dikirim atau selesai, boleh break
            if step in ("done", "otp_submitted"):
                break
            # Beri jeda kecil biar halaman update
            time.sleep(2)

        # 4) Verifikasi sukses (best effort)
        try:
            wait.until(EC.url_contains("instagram.com"))
            log("✅ Pendaftaran tampaknya berhasil/lanjut. Periksa akunmu.")
        except Exception:
            log("ℹ️ Tidak ada redirect yang jelas. Mungkin ada langkah tambahan.")

        # Simpan data
        with open("akun_instagram.txt", "a") as f:
            f.write(f"Email: {EMAIL}\n")
            f.write(f"Username: {data['username']}\n")
            f.write(f"Password: {data['password']}\n")
            f.write("="*30 + "\n")
        log("Data akun disimpan di 'akun_instagram.txt'.")

    except Exception as e:
        print("\n=== ERROR DETAIL ===")
        print(f"Jenis Error : {type(e).__name__}")
        print(f"Pesan Error : {e}")
        print("Traceback   :")
        traceback.print_exc()
        print("====================")
    finally:
        try:
            driver.quit()
            log("Browser ditutup.")
        except Exception:
            log("Browser sudah ditutup.")

if __name__ == "__main__":
    main()
