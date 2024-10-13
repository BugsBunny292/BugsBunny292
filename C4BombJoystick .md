#include <LiquidCrystal.h>

LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

const int buzzerPin = 8;
const int joystickX = A0;  // X ekseni
const int joystickY = A1;  // Y ekseni
const int joystickButton = 9;

const int passwordLength = 8; // Şifre uzunluğu
const char password[passwordLength] = "7355608"; // Doğru şifre
char enteredPassword[passwordLength]; // Girilen şifre
int passwordIndex = 0; // Girilen şifre indeksi
bool isBombActivated = false;
bool isArmed = false;
long startTime = 0;
long currentTime = 0;
long lastBeepTime = 0;
long buttonPressStart = 0;
int countdown = 40;  // Bombanın patlama süresi
const int initialCountdown = 40;  // Geri sayım için başlangıç değeri
const int defuseTime = 10;  // Bombayı imha etmek için süre (10 saniye)

void setup() {
  lcd.begin(16, 2);
  pinMode(buzzerPin, OUTPUT);
  pinMode(joystickButton, INPUT_PULLUP);
  showPasswordPrompt();  // Şifreyi başlangıçta göster
}

void loop() {
  int joystickXValue = analogRead(joystickX);  // Joystick X eksenini oku
  int joystickYValue = analogRead(joystickY);  // Joystick Y eksenini oku
  bool buttonPressed = digitalRead(joystickButton) == LOW;

  // Patlama süresini ayarlamak için joystick yan hareketi
  if (joystickXValue < 400 && countdown < 60) {  // Maksimum 60 saniye
    countdown++;  // Patlama süresini artır
    showCountdown();  // Yeni sayıyı ekranda göster
    delay(300);  // Değişiklik için kısa bekleme süresi
  } else if (joystickXValue > 600 && countdown > 1) {
    countdown--;  // Patlama süresini azalt
    showCountdown();  // Yeni sayıyı ekranda göster
    delay(300);  // Değişiklik için kısa bekleme süresi
  }

  // Joystick yukarı hareket ettiğinde şifre girme
  if (joystickYValue < 400 && passwordIndex < passwordLength - 1) {
    enteredPassword[passwordIndex] = password[passwordIndex]; // Şifreyi ekle
    passwordIndex++;
    showPassword();  // Şifreyi ekranda göster
    delay(300);  // Değişiklik için kısa bekleme süresi
  }

  // Joystick butonuna basıldığında şifreyi kontrol et
  if (buttonPressed && !isBombActivated && passwordIndex == passwordLength - 1) {
    enteredPassword[passwordIndex] = '\0';  // Girilen şifreyi sonlandır
    if (strcmp(enteredPassword, password) == 0) {  // Doğru şifre kontrolü
      lcd.clear();
      lcd.print("Bomb Planted!!!");
      isBombActivated = true;  // Bomba kuruldu
      isArmed = true;          // Bomba aktif
      startTime = millis();    // Zamanlayıcıyı başlat
    } else {  // Yanlış şifre seçilmişse
      lcd.clear();
      lcd.print("WORNG!");
      delay(2000);  // "YANLIS" mesajını 2 saniye göster
      showPasswordPrompt();  // Şifreyi tekrar göster
      passwordIndex = 0; // Girilen şifreyi sıfırla
      memset(enteredPassword, 0, passwordLength); // Girilen şifreyi temizle
    }
  }

  // Bomba aktif olduğunda geri sayımı başlat
  if (isArmed && countdown > 0) {
    currentTime = millis();

    // Her saniyede bir buzzer sesi çıkar ve geri sayımı güncelle
    if (currentTime - lastBeepTime >= 1000) {
      lastBeepTime = currentTime;
      countdown--;  // Geri sayımı azalt
      lcd.setCursor(0, 1);
      lcd.print("              ");  // Önceki sayıyı temizle
      lcd.setCursor(0, 1);
      lcd.print(countdown);  // Yeni geri sayımı yazdır

      tone(buzzerPin, 1000);  // Buzzer'dan bip sesi her saniye çıkar
      delay(500);             // 0.5 saniye buzzer açık
      noTone(buzzerPin);      // Buzzer'ı kapat
    }

    // Joystick'e 10 saniye basılı tutulduğunda bombayı etkisiz hale getir
    if (buttonPressed) {
      if (buttonPressStart == 0) {
        buttonPressStart = millis();  // Basma süresini başlat
      }
      
      if (millis() - buttonPressStart >= defuseTime * 1000) {  // 10 saniye kontrolü
        lcd.clear();
        lcd.print("Defused!");
        noTone(buzzerPin);  // Buzzer'ı kapat
        isArmed = false;    // Bombayı etkisiz hale getir
        countdown = initialCountdown;  // Geri sayımı sıfırla
        delay(2000);
        lcd.clear();
        showPasswordPrompt();  // Şifreyi tekrar göster
        buttonPressStart = 0;  // Sıfırla
      }
    } else {
      buttonPressStart = 0;  // Eğer buton bırakılırsa zamanlayıcı sıfırlanır
    }

    // Geri sayım sona erdiyse patlat
    if (countdown == 0) {
      lcd.clear();
      lcd.print("BOOM!");
      boomSound(); // Patlama sesi
      while (true) {
        // Buzzer sürekli açık kalacak ve LCD'de "BOOM!" yazacak
      }
    }
  }
}

// Patlama sesi oluşturma fonksiyonu
void boomSound() {
  for (int i = 0; i < 3; i++) { // Üç kez ses çıkar
    tone(buzzerPin, 2000); // 2 kHz ton (yüksek frekans)
    delay(500);            // 0.5 saniye bekle
    noTone(buzzerPin);    // Tonu kapat
    delay(500);            // 0.5 saniye bekle
  }
}

// Şifreyi gösterme fonksiyonu
void showPasswordPrompt() {
  lcd.clear();
  lcd.print("Password:");
  lcd.setCursor(0, 1);
  lcd.print("Enter: _ _ _ _ _ _ _"); // Girilecek şifre için boşluklar
}

// Girilen şifreyi ekranda gösterme fonksiyonu
void showPassword() {
  lcd.setCursor(0, 1);
  lcd.print("Enter: ");
  lcd.print(enteredPassword); // Girilen şifreyi yazdır
  for (int i = passwordIndex; i < passwordLength; i++) {
    lcd.print("_"); // Girilmeyen karakterler için alt çizgi göster
  }
}

// Geri sayımı ekranda gösterme fonksiyonu
void showCountdown() {
  lcd.setCursor(0, 1);
  lcd.print("Countdown: ");
  lcd.print(countdown); // Yeni geri sayımı yazdır
}
