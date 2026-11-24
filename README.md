Solaris STM32 ADC Monitor & KiCad SchematicDokuz Eylül Üniversitesi Solaris Güneş Enerjili Araç Takımı — Elektronik Alt Grubu kapsamında geliştirilen bu proje; STM32 Nucleo-F103RB mikrodenetleyicisi üzerinden analog veri okuma (ADC), gerçek zamanlı akım/direnç hesaplaması, eşik tabanlı LED kontrolü ve devrenin KiCad şematik tasarımını içermektedir.1. Projenin AmacıGüneş enerjili araç alt sistemlerinde (gaz pedalı konum sensörü, batarya gerilim izleme ve analog telemetri girişleri) kullanılan Analog-Dijital Dönüşüm (ADC) mimarisini uygulamak, mikrodenetleyici üzerinden dolaylı akım ve eşdeğer direnç hesaplaması gerçekleştirmek ve belirlenen kritik eşik gerilimi (2.64 V) aşıldığında donanımsal uyarı üretmektir.2. Donanım ve Pin Bağlantı TablosuMikrodenetleyici giriş pinleri yüksek empedanslı olduğu için doğrudan ampermetre gibi seri bağlanarak akım ölçemez. Bu nedenle devreden geçen ortak akımı fiziksel olarak tespit edebilmek amacıyla reosta (değişken direnç) + sabit yük direnci seri topolojisi kurulmuştur:Bileşen / PinDeğer / EtiketDevre Bağlantısı ve GöreviMikrodenetleyiciSTM32F103RBT6Nucleo-F103RB geliştirme kartı (12-bit ADC, 3.3V lojik)Potansiyometre (RV1)10 kΩPin 1: +3.3V, Pin 2: Ölçüm düğümü, Pin 3: Boşta (No-Connect)Sabit Direnç (R1)20 kΩPotansiyometre orta ucu (Pin 2) ile GND arasında seri yükAnalog Giriş (PA0 / A0)ADC1_IN0RV1 (Pin 2) ve R1 birleşim düğümünden gerilim ölçümüDurum LED'i (PA5 / LD2)GPIO_OutputDüğüm gerilimi 2.64 V eşiğini aştığında aktif olan uyarı LED'i3. Çalışma Aralıkları ve Hesaplama FormülleriDevredeki elektriksel büyüklükler 12-bit ADC çözünürlüğü (0 – 4095) ve Ohm Yasası kullanılarak gerçek zamanlı hesaplanır:20 kΩ Direnç Gerilimi: V_20k = (ADC_raw / 4095.0) * 3.3 VSeri Devre Akımı: I = V_20k / 20000 ΩToplam ve Potansiyometre Direnci: R_toplam = 3.3 V / I ve R_pot = R_toplam - 20000 ΩPotansiyometre KonumuR_pot DeğeriDüğüm Gerilimi (PA0)Devre Akımı (I)LED Durumu (LD2)En Üst Konum~ 0 Ω3.30 V0.165 mAYANAR (> 2.64 V)Orta Eşik Noktası~ 5 kΩ2.64 V0.132 mAEŞİK SINIRIEn Alt Konum~ 10 kΩ2.20 V0.110 mASÖNÜK (< 2.64 V)4. Depo Klasör YapısıSolaris-STM32-ADC-Monitor/
├── Firmware/                  # STM32CubeIDE proje dosyaları (Core, Drivers, .ioc)
├── Hardware/                  # KiCad proje dosyaları (.kicad_pro, .kicad_sch)
└── README.md                  # Proje teknik dokümantasyonu
5. Uygulanan Gömülü C Kodu (Firmware/Core/Src/main.c)/* USER CODE BEGIN PV */
uint16_t adc_ham_deger = 0;          // 0 - 4095 ham ADC verisi
float direnc_voltaji_V = 0.0f;       // 20k direnc uzerindeki gerilim (V)
float devre_akimi_mA = 0.0f;         // Seri devreden gecen ortak akim (mA)
float toplam_direnc_Ohm = 0.0f;      // R_pot + R_20k toplam direnc (Ohm)
float sadece_pot_direnci_Ohm = 0.0f; // Potansiyometrenin anlik direnci (Ohm)

const float SABIT_DIRENC_OHM = 20000.0f; // 20k Ohm sabit yuk direnci
const float BESLEME_VOLTAJI = 3.3f;      // 3.3V sistem besleme gerilimi
const float ESIK_VOLTAJ = 2.64f;         // Orta nokta esik gerilimi (V)
/* USER CODE END PV */

/* USER CODE BEGIN 3 */
HAL_ADC_Start(&hadc1);

if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK)
{
    adc_ham_deger = HAL_ADC_GetValue(&hadc1);

    // 1. 20k direnc uzerindeki gerilim farki (PA0 - GND)
    direnc_voltaji_V = ((float)adc_ham_deger / 4095.0f) * BESLEME_VOLTAJI;

    // 2. Seri koldan gecen ortak akim (Amper ve mA)
    float akim_Amper = direnc_voltaji_V / SABIT_DIRENC_OHM;
    devre_akimi_mA = akim_Amper * 1000.0f;

    // 3. Toplam direnc ve potansiyometre direnci hesabi (sifira bolme korumali)
    if (akim_Amper > 0.000001f)
    {
        toplam_direnc_Ohm = BESLEME_VOLTAJI / akim_Amper;
        sadece_pot_direnci_Ohm = toplam_direnc_Ohm - SABIT_DIRENC_OHM;
    }
    else
    {
        toplam_direnc_Ohm = 999999.0f;
        sadece_pot_direnci_Ohm = 999999.0f;
    }

    // 4. Threshold (Esik) gerilimi kontrolu
    if (direnc_voltaji_V > ESIK_VOLTAJ)
    {
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
    }
    else
    {
        HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    }
}

HAL_ADC_Stop(&hadc1);
HAL_Delay(50);
/* USER CODE END 3 */
6. Derleme ve ÇalıştırmaFirmware/ klasörü içindeki .ioc veya .project dosyasını STM32CubeIDE ile içe aktarın (File > Open Projects from File System).Debug modunda projeyi derleyip kart üzerine yükleyin.Live Expressions sekmesine direnc_voltaji_V, devre_akimi_mA ve sadece_pot_direnci_Ohm değişkenlerini ekleyerek potansiyometre hareketini canlı izleyin.
